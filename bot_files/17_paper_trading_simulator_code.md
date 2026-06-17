# -*- coding: utf-8 -*-
"""
17_paper_trading_simulator.py
==============================
Paper trading fictif continu (24/7 boucle horaire).

Simule le comportement du robot en temps réel prospectif:
- Met à jour les caches H1 depuis MT5 chaque heure
- À 23h05 Paris: scan D1, génère watchlist
- Chaque heure: gère les positions ouvertes (exit, trailing SL)
- Chaque heure: cherche entrée sur watchlist disponible
- Journalise COMPLÈTEMENT dans journal_paper_trading.csv
- CB A2+20j intégré
- Reconnect auto si MT5 crash

Usage:
  python 17_paper_trading_simulator.py
  (tourne en boucle infinie, relance chaque heure)

Output:
  journal_paper_trading.csv : tous les H1 bars, décisions, trades, états CB
  paper_trading_simulator.log : logs des erreurs et événements
"""

import time
import logging
import json
from datetime import datetime, timedelta
from pathlib import Path
import csv
import traceback

import numpy as np
import pandas as pd
import MetaTrader5 as mt5

BASE_DIR = Path(__file__).resolve().parent
H1_DIR = BASE_DIR / "h1_cache"
SPECS_FILE = BASE_DIR / "symbol_specs.csv"
JOURNAL_FILE = BASE_DIR / "journal_paper_trading.csv"
LOG_FILE = BASE_DIR / "paper_trading_simulator.log"

# --- MT5 ---
MT5_PATH = r"C:\Program Files\Raise Global MT5 Terminal\terminal64.exe"
LOGIN = 5007258
PASSWORD = "zL3!gvG8Ol"
SERVER = "RaiseGlobal-Live"

# --- Simulation ---
RISK_EUR = 250.0
CAPITAL = 50_000.0
SL_MULT = 2.5
ATR_PERIOD = 14
ADX_PERIOD = 14
SEUIL_FORT = 0.65
MAX_BARS_TRADE = 240
ENTRY_HOUR = 23
ENTRY_MIN_PARIS = 5

# --- CB ---
CB_A_N = 2
CB_A_DAYS = 20


# ================================================================
# Setup logging
# ================================================================

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    handlers=[
        logging.FileHandler(LOG_FILE, encoding="utf-8"),
        logging.StreamHandler(),
    ],
)
log = logging.getLogger(__name__)


# ================================================================
# MT5 connexion avec reconnect auto
# ================================================================

def connect_mt5(max_retries=3):
    """Connexion MT5 avec retry."""
    for attempt in range(1, max_retries + 1):
        try:
            if mt5.initialize(path=MT5_PATH, login=LOGIN, password=PASSWORD, server=SERVER):
                info = mt5.account_info()
                if info:
                    log.info(f"MT5 connecté | Balance: {info.balance:.2f} {info.currency}")
                    return True
        except Exception as e:
            log.warning(f"Tentative {attempt}/{max_retries} échouée: {e}")
        time.sleep(5)
    return False


def ensure_connected():
    """Vérifie la connexion, reconnecte si nécessaire."""
    try:
        if mt5.terminal_info() is None:
            log.warning("MT5 déconnecté, reconnexion...")
            return connect_mt5()
        return True
    except Exception as e:
        log.error(f"Erreur check connexion: {e}")
        return connect_mt5()


# ================================================================
# Update caches H1 depuis MT5
# ================================================================

def update_h1_cache(sym: str, max_bars: int = 500) -> bool:
    """
    Récupère les barres H1 les plus récentes depuis MT5 et met à jour le cache.
    Retourne True si succès.
    """
    cache_path = H1_DIR / f"{sym}_H1.csv"

    try:
        mt5.symbol_select(sym, True)
        # Récupère les max_bars barres H1 les plus récentes
        rates = mt5.copy_rates_from_pos(sym, mt5.TIMEFRAME_H1, 0, max_bars)

        if rates is None or len(rates) == 0:
            log.debug(f"Pas de nouvelles données H1 pour {sym}")
            return False

        # Charge le cache existant s'il existe
        existing = None
        if cache_path.exists():
            existing = pd.read_csv(cache_path)

        # Convertit les nouvelles barres en DataFrame
        df_new = pd.DataFrame(rates)
        df_new["datetime_utc"] = df_new["time"].apply(
            lambda ts: datetime.utcfromtimestamp(int(ts)).strftime("%Y-%m-%d %H:%M:%S")
        )
        df_new = df_new[["time", "datetime_utc", "open", "high", "low", "close",
                         "tick_volume", "spread", "real_volume"]]

        # Fusionne avec existant (évite les doublons)
        if existing is not None:
            df_combined = pd.concat([df_new, existing], ignore_index=True)
            df_combined = df_combined.drop_duplicates(subset="time", keep="first")
            df_combined = df_combined.sort_values("time").reset_index(drop=True)
        else:
            df_combined = df_new.sort_values("time").reset_index(drop=True)

        df_combined.to_csv(cache_path, index=False)
        return True

    except Exception as e:
        log.error(f"Erreur update H1 {sym}: {e}")
        return False


# ================================================================
# Indicateurs H1
# ================================================================

def _ema(s: pd.Series, span: int) -> pd.Series:
    return s.ewm(span=span, adjust=False, min_periods=span).mean()


def _atr14(df: pd.DataFrame) -> pd.Series:
    h, l, c = df["high"], df["low"], df["close"]
    p = c.shift(1)
    tr = pd.concat([h - l, (h - p).abs(), (l - p).abs()], axis=1).max(axis=1)
    return tr.ewm(alpha=1/ATR_PERIOD, adjust=False, min_periods=ATR_PERIOD).mean()


def _adx14(df: pd.DataFrame) -> np.ndarray:
    h, l, c = df["high"], df["low"], df["close"]
    p = c.shift(1)
    up = h.diff(); down = -l.diff()
    plus_dm = np.where((up > down) & (up > 0), up, 0.0)
    minus_dm = np.where((down > up) & (down > 0), down, 0.0)
    tr = pd.concat([h - l, (h - p).abs(), (l - p).abs()], axis=1).max(axis=1)
    atr_w = pd.Series(tr).ewm(alpha=1/ADX_PERIOD, adjust=False, min_periods=ADX_PERIOD).mean()
    plus_di = 100 * pd.Series(plus_dm).ewm(alpha=1/ADX_PERIOD, adjust=False, min_periods=ADX_PERIOD).mean() / atr_w
    minus_di = 100 * pd.Series(minus_dm).ewm(alpha=1/ADX_PERIOD, adjust=False, min_periods=ADX_PERIOD).mean() / atr_w
    denom = (plus_di + minus_di).replace(0, np.nan)
    dx = 100 * (plus_di - minus_di).abs() / denom
    return dx.ewm(alpha=1/ADX_PERIOD, adjust=False, min_periods=ADX_PERIOD).mean().values


def enrich_h1(h1_df: pd.DataFrame) -> pd.DataFrame:
    """Ajoute ema50, atr14, adx14 au DataFrame H1."""
    df = h1_df.copy()
    if not pd.api.types.is_datetime64_any_dtype(df["datetime_utc"]):
        df["datetime_utc"] = pd.to_datetime(df["datetime_utc"])
    c = df["close"]
    df["ema50"] = _ema(c, 50)
    df["atr14"] = _atr14(df)
    df["adx14"] = _adx14(df)
    return df


# ================================================================
# Check entry (EMA50 cross + ADX > 20)
# ================================================================

def check_entry(h1_enriched: pd.DataFrame) -> dict | None:
    """
    Cherche un front montant EMA50 sur les 2 dernières barres H1.
    Retourne {entry_price, sl_price, atr} ou None.
    """
    if len(h1_enriched) < 2:
        return None

    # Deux dernières barres
    bar_prev = h1_enriched.iloc[-2]
    bar_curr = h1_enriched.iloc[-1]

    # Condition: close monte au-dessus de EMA50, ADX > 20
    prev_close = bar_prev["close"]
    prev_ema = bar_prev["ema50"]
    curr_close = bar_curr["close"]
    curr_ema = bar_curr["ema50"]
    adx = bar_curr["adx14"]
    atr = bar_curr["atr14"]

    if (pd.isna(prev_ema) or pd.isna(curr_ema) or pd.isna(adx) or pd.isna(atr)):
        return None

    if prev_close < prev_ema and curr_close >= curr_ema and adx > 20:
        sl = curr_close - SL_MULT * atr
        return {"entry_price": curr_close, "sl_price": sl, "atr": atr}

    return None


# ================================================================
# Check exit (close < EMA50)
# ================================================================

def check_exit(h1_enriched: pd.DataFrame) -> bool:
    """Vérifie si on doit sortir (close < EMA50 sur la dernière barre)."""
    if len(h1_enriched) == 0:
        return False
    bar = h1_enriched.iloc[-1]
    ema = bar.get("ema50")
    if pd.isna(ema):
        return False
    return bar["close"] < ema


# ================================================================
# Compute trailing SL
# ================================================================

def compute_trailing_sl(h1_enriched: pd.DataFrame, current_sl: float) -> float:
    """Trailing SL: max(current_SL, EMA50 - 2.5*ATR)."""
    if len(h1_enriched) == 0:
        return current_sl
    bar = h1_enriched.iloc[-1]
    ema = bar.get("ema50")
    atr = bar.get("atr14")
    if pd.isna(ema) or pd.isna(atr):
        return current_sl
    new_sl = ema - SL_MULT * atr
    return max(current_sl, new_sl)


# ================================================================
# Journalisation
# ================================================================

def init_journal():
    """Crée le header du CSV s'il n'existe pas."""
    if not JOURNAL_FILE.exists():
        with open(JOURNAL_FILE, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "timestamp",
                "event_type",  # "h1_bar", "scan", "entry", "exit", "trailing_sl", "cb_state"
                "symbole",
                "close", "ema50", "atr14", "adx14",
                "decision",  # "seeking_entry", "in_trade", "exit_signal", "sl_hit", "check_entry_failed"
                "entry_price", "sl_price", "exit_price",
                "pnl_r", "pnl_eur",
                "raison_exit",  # "signal_ema50", "sl_hit", "max_bars"
                "slot_libre",
                "cb_a_consec_sl",  # comma-separated dict
                "cb_a_blacklist",  # comma-separated dict
                "position_id",  # pour tracker une position de son ouverture à sa fermeture
            ])
            writer.writeheader()


def log_event(event_type: str, **kwargs):
    """Enregistre un événement dans le journal."""
    row = {
        "timestamp": datetime.utcnow().isoformat(),
        "event_type": event_type,
    }
    row.update(kwargs)

    with open(JOURNAL_FILE, "a", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=[
            "timestamp", "event_type", "symbole", "close", "ema50", "atr14", "adx14",
            "decision", "entry_price", "sl_price", "exit_price", "pnl_r", "pnl_eur",
            "raison_exit", "slot_libre", "cb_a_consec_sl", "cb_a_blacklist", "position_id",
        ])
        writer.writerow(row)


# ================================================================
# Paris time utils
# ================================================================

def paris_now() -> datetime:
    """Heure actuelle Paris (UTC+1 ou UTC+2 selon DST)."""
    import pytz
    utc_now = datetime.now(pytz.UTC)
    paris_tz = pytz.timezone("Europe/Paris")
    return utc_now.astimezone(paris_tz)


def is_scan_time() -> bool:
    """True si on est entre 23h05 et 23h15 Paris."""
    now = paris_now()
    return now.hour == ENTRY_HOUR and ENTRY_MIN_PARIS <= now.minute < ENTRY_MIN_PARIS + 10


# ================================================================
# Main boucle
# ================================================================

def main():
    log.info("=" * 70)
    log.info("Paper Trading Simulator - démarrage")
    log.info(f"Compte: {LOGIN} | Server: {SERVER}")
    log.info("=" * 70)

    # Init
    H1_DIR.mkdir(exist_ok=True)
    init_journal()

    # Connexion MT5
    if not connect_mt5():
        log.error("Impossible de connecter MT5. Arrêt.")
        return

    # Charge la liste des symboles
    specs = pd.read_csv(SPECS_FILE)
    all_syms = specs.loc[specs["faisable"] == True, "symbole"].tolist()
    log.info(f"Univers: {len(all_syms)} symboles chargés")

    # État du robot
    position_open = None  # {sym, entry_price, sl, entry_dt, entry_id}
    asset_consec_sl = {}  # sym -> nb SL consecutifs
    asset_blacklist_until = {}  # sym -> datetime
    watchlist = []  # signaux fort LONG du dernier scan
    last_scan_date = None
    position_id_counter = 0

    # Boucle horaire
    log.info("Entrée boucle horaire infinie. Ctrl+C pour arrêter.")

    try:
        while True:
            try:
                now = paris_now()
                log.info(f"\n[{now.strftime('%Y-%m-%d %H:%M:%S Paris')}] --- Itération horaire ---")

                # Reconnect MT5 si nécessaire
                if not ensure_connected():
                    log.error("Reconnexion MT5 échouée. Attente 60s...")
                    time.sleep(60)
                    continue

                # Update des H1 caches depuis MT5
                log.info(f"Mise à jour des {len(all_syms)} caches H1...")
                for sym in all_syms:
                    update_h1_cache(sym)

                # --- Check scan D1 (23h05 Paris) ---
                # Note: le scan est fait par robot.py, on lit juste la watchlist.json
                watchlist_file = JOURNAL_FILE.parent / "watchlist.json"
                if watchlist_file.exists():
                    try:
                        with open(watchlist_file, "r", encoding="utf-8") as f:
                            watchlist_data = json.load(f)
                            watchlist = [w["symbole"] for w in watchlist_data]
                            if watchlist:
                                log.info(f"[WATCHLIST] {len(watchlist)} signaux fort LONG lus depuis watchlist.json")
                                log_event("scan", decision=f"watchlist={len(watchlist)}")
                    except Exception as e:
                        log.warning(f"Erreur lecture watchlist.json: {e}")
                        watchlist = []

                # --- Gestion position ouverte ---
                if position_open:
                    sym = position_open["sym"]
                    cache_path = H1_DIR / f"{sym}_H1.csv"

                    if not cache_path.exists():
                        log.warning(f"Cache {sym} introuvable")
                    else:
                        h1 = pd.read_csv(cache_path, parse_dates=["datetime_utc"])
                        h1_enr = enrich_h1(h1)

                        # Log la barre actuelle
                        bar = h1_enr.iloc[-1]
                        log_event(
                            "h1_bar",
                            symbole=sym,
                            close=round(bar["close"], 5),
                            ema50=round(bar["ema50"], 5) if not pd.isna(bar["ema50"]) else None,
                            atr14=round(bar["atr14"], 5) if not pd.isna(bar["atr14"]) else None,
                            adx14=round(bar["adx14"], 2) if not pd.isna(bar["adx14"]) else None,
                            decision="in_trade",
                            position_id=position_open["id"],
                        )

                        # Trailing SL
                        old_sl = position_open["sl"]
                        new_sl = compute_trailing_sl(h1_enr, old_sl)
                        if new_sl > old_sl + 1e-5:
                            position_open["sl"] = new_sl
                            log.info(f"{sym} trailing SL: {old_sl:.5f} -> {new_sl:.5f}")
                            log_event(
                                "trailing_sl",
                                symbole=sym,
                                sl_price=round(new_sl, 5),
                                position_id=position_open["id"],
                            )

                        # Check exit (EMA50)
                        if check_exit(h1_enr):
                            exit_price = h1_enr.iloc[-1]["close"]
                            pnl_r = (exit_price - position_open["entry_price"]) / (position_open["entry_price"] - position_open["sl"])
                            pnl_eur = pnl_r * RISK_EUR

                            log.info(f"{sym} EXIT signal (close < EMA50) | PnL: {pnl_r:.3f}R ({pnl_eur:.0f}EUR)")
                            log_event(
                                "exit",
                                symbole=sym,
                                exit_price=round(exit_price, 5),
                                pnl_r=round(pnl_r, 3),
                                pnl_eur=round(pnl_eur, 0),
                                raison_exit="signal_ema50",
                                position_id=position_open["id"],
                            )

                            # Reset SL counter (sortie propre)
                            asset_consec_sl[sym] = 0
                            position_open = None

                        # Check SL hit
                        elif bar["low"] <= position_open["sl"]:
                            pnl_r = -SL_MULT
                            pnl_eur = pnl_r * RISK_EUR

                            log.info(f"{sym} SL HIT | PnL: {pnl_r:.3f}R ({pnl_eur:.0f}EUR)")
                            log_event(
                                "exit",
                                symbole=sym,
                                exit_price=round(position_open["sl"], 5),
                                pnl_r=round(pnl_r, 3),
                                pnl_eur=round(pnl_eur, 0),
                                raison_exit="sl_hit",
                                position_id=position_open["id"],
                            )

                            # Increment SL counter
                            asset_consec_sl[sym] = asset_consec_sl.get(sym, 0) + 1
                            log.info(f"CB: {sym} SL consecutifs = {asset_consec_sl[sym]}")

                            if asset_consec_sl[sym] >= CB_A_N:
                                until = now + timedelta(days=CB_A_DAYS)
                                asset_blacklist_until[sym] = until
                                asset_consec_sl[sym] = 0
                                log.info(f"CB A2+20j: {sym} blackliste jusqu'au {until.date()}")

                            position_open = None

                # --- Recherche entrée (slot libre) ---
                if not position_open and watchlist:
                    # Filtre watchlist par CB blacklist
                    wl_ok = [
                        w for w in watchlist
                        if w not in asset_blacklist_until or now >= asset_blacklist_until[w]
                    ]

                    for sym in wl_ok:
                        cache_path = H1_DIR / f"{sym}_H1.csv"
                        if not cache_path.exists():
                            log.debug(f"Cache {sym} introuvable pour entry check")
                            continue

                        h1 = pd.read_csv(cache_path, parse_dates=["datetime_utc"])
                        h1_enr = enrich_h1(h1)

                        entry = check_entry(h1_enr)
                        if entry:
                            position_id_counter += 1
                            position_open = {
                                "sym": sym,
                                "entry_price": entry["entry_price"],
                                "sl": entry["sl_price"],
                                "entry_dt": now,
                                "id": position_id_counter,
                            }
                            log.info(f"{sym} ENTRY | price={entry['entry_price']:.5f} | SL={entry['sl_price']:.5f}")
                            log_event(
                                "entry",
                                symbole=sym,
                                entry_price=round(entry["entry_price"], 5),
                                sl_price=round(entry["sl_price"], 5),
                                position_id=position_id_counter,
                            )
                            break  # Slot unique

                # Attendre 60 min
                log.info("Sleep 60 min...")
                time.sleep(3600)

            except Exception as e:
                log.error(f"Erreur dans boucle: {e}")
                log.error(traceback.format_exc())
                log.info("Attente 60s avant retry...")
                time.sleep(60)

    except KeyboardInterrupt:
        log.info("\nArrêt demandé (Ctrl+C)")

    finally:
        mt5.shutdown()
        log.info("MT5 déconnecté. Fin.")


if __name__ == "__main__":
    main()

