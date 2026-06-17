# -*- coding: utf-8 -*-
"""
robot.py — Boucle principale du Momentum Scanner.

Architecture (slot unique, LONG only) :
  - Scan D1 a 23h05 Paris : update cache H1 + calcul scores -> watchlist.json
  - Chaque minute : si en trade -> check exit + trailing SL
                    si libre    -> check signal H1 sur watchlist (1 check / barre H1)
  - Vendredi 20h30 UTC : fermeture position si ouverte, pas de nouveau trade

Parametres dans config.py. Kill switch : ENABLE_TRADING = False.
"""

import time
from datetime import date, timedelta

import pandas as pd

import config
import risk_manager as rm
import scanner
import signal_h1 as sig
import trade_executor as te
from logger import log, telegram

LOOP_SLEEP = 60  # secondes entre chaque iteration de la boucle principale


def _load_symbols() -> list:
    """Charge la liste des symboles faisables depuis symbol_specs.csv."""
    try:
        specs = pd.read_csv(config.SPECS_FILE)
        syms = specs.loc[specs["faisable"] == True, "symbole"].tolist()
        log.info(f"Univers : {len(syms)} symboles faisables")
        return syms
    except Exception as exc:
        log.error(f"Impossible de charger {config.SPECS_FILE}: {exc}")
        return []


def _run_daily_scan(symbols: list) -> None:
    """Met a jour le cache H1 et calcule les scores D1 -> watchlist.json."""
    log.info("=== Scan D1 quotidien ===")
    telegram("Scan D1 en cours...")
    scanner.update_h1_caches(symbols)
    watchlist = scanner.run_scan(symbols)
    if watchlist:
        names = ", ".join(f"{w['symbole']}({w['score_global']:.3f})" for w in watchlist)
        log.info(f"Watchlist : {names}")
        telegram(f"Watchlist : {names}")
    else:
        log.info("Watchlist vide — aucun signal fort LONG aujourd'hui")
        telegram("Watchlist vide")


def _manage_open_position(pos) -> bool:
    """Gere une position ouverte : trailing SL + check exit.
    Retourne True si la position a ete fermee par signal (pas par SL broker)."""
    sym = pos.symbol
    bars = te.fetch_h1_bars(sym)
    if bars is None:
        return False

    # Trailing SL
    new_sl = sig.compute_trailing_sl(bars, pos.sl)
    if new_sl > pos.sl + 1e-8:
        te.update_sl(pos.ticket, new_sl)

    # Exit : close < EMA50
    if sig.check_exit(bars):
        log.info(f"EXIT signal {sym} | close < EMA50")
        if te.close_position(pos.ticket):
            return True
    return False


def _check_entries(watchlist: list, last_h1_seen: dict) -> bool:
    """
    Parcourt la watchlist et entre sur le premier signal valide.
    last_h1_seen : dict sym -> dernier timestamp H1 traite (evite les doublons).
    Retourne True si un trade a ete ouvert.
    """
    for asset in watchlist:
        sym = asset["symbole"]
        bars = te.fetch_h1_bars(sym)
        if bars is None:
            continue

        # Ne verifier l'entree que si une nouvelle barre H1 est disponible
        last_ts = int(bars[-1]["time"])
        if last_h1_seen.get(sym) == last_ts:
            continue
        last_h1_seen[sym] = last_ts

        triggered, entry, sl, atr = sig.check_entry(bars)
        if not triggered:
            continue

        if not sig.spread_ok(sym, atr):
            log.info(f"ENTRY skip {sym}: spread trop large")
            continue

        lot = te.compute_lot(sym, entry, sl)
        if lot is None:
            log.warning(f"ENTRY skip {sym}: lot non calculable")
            continue

        ticket = te.open_long(sym, lot, sl)
        if ticket:
            return True  # slot unique : une seule entree par iteration

    return False


def main() -> None:
    log.info("=== Momentum Scanner demarrage ===")
    telegram("Momentum Scanner demarre")

    if not te.connect():
        log.error("Connexion MT5 impossible — arret")
        return

    symbols = _load_symbols()
    if not symbols:
        log.error("Liste de symboles vide — arret")
        te.shutdown()
        return

    last_scan_date: date | None = None
    last_h1_seen: dict = {}   # sym -> last H1 bar timestamp vu en entree

    # Circuit-breaker A2+20j
    asset_consec_sl: dict = {}        # sym -> nb SL consecutifs
    asset_blacklist_until: dict = {}  # sym -> datetime fin blacklist
    prev_ticket: int | None = None    # ticket suivi la derniere iteration
    prev_sym: str | None = None       # symbole de la derniere position

    log.info("Boucle principale demarree (Ctrl+C pour arreter)")

    try:
        while True:
            # --- Connexion ---
            if not te.ensure_connected():
                log.error("Reconnexion echouee, nouvelle tentative dans 30s")
                time.sleep(30)
                continue

            # --- Fermeture vendredi ---
            if not rm.is_trading_hours():
                positions = te.get_positions()
                if positions:
                    log.info("Hors heures de trading — fermeture position")
                    for pos in positions:
                        te.close_position(pos.ticket)
                    telegram("Hors heures — position fermee")
                time.sleep(LOOP_SLEEP)
                continue

            # --- Scan D1 (une fois par jour a l'heure de scan) ---
            today = rm.paris_now().date()
            if rm.is_scan_time() and last_scan_date != today:
                _run_daily_scan(symbols)
                last_scan_date = today
                last_h1_seen.clear()  # reset : nouvelle watchlist, nouveaux signaux

            # --- Positions ouvertes (slot unique) ---
            positions = te.get_positions()

            if positions:
                pos = positions[0]
                closed_by_signal = _manage_open_position(pos)
                if closed_by_signal:
                    # Sortie propre (EMA50) : reinitialise le compteur SL de cet actif
                    asset_consec_sl[pos.symbol] = 0
                    prev_ticket = None
                    prev_sym    = None
                else:
                    prev_ticket = pos.ticket
                    prev_sym    = pos.symbol
            else:
                if prev_ticket is not None:
                    # La position a disparu sans fermeture volontaire -> SL broker
                    sym   = prev_sym
                    count = asset_consec_sl.get(sym, 0) + 1
                    asset_consec_sl[sym] = count
                    log.info(f"SL broker detecte {sym} | SL consecutifs: {count}")
                    if count >= 2:
                        until = rm.paris_now() + timedelta(days=20)
                        asset_blacklist_until[sym] = until
                        asset_consec_sl[sym] = 0
                        log.info(f"CB A2+20j: {sym} blackliste jusqu'au {until.date()}")
                        telegram(f"CB: {sym} blackliste 20j (2 SL consecutifs)")
                prev_ticket = None
                prev_sym    = None

                # Recherche d'entree sur la watchlist (filtree par le CB)
                watchlist = scanner.load_watchlist()
                if watchlist:
                    now_dt = rm.paris_now()
                    wl_ok = [
                        w for w in watchlist
                        if w["symbole"] not in asset_blacklist_until
                        or now_dt >= asset_blacklist_until[w["symbole"]]
                    ]
                    if wl_ok:
                        _check_entries(wl_ok, last_h1_seen)
                elif last_scan_date is None:
                    log.info("Aucun scan effectue aujourd'hui — utilisation de la watchlist existante")

            time.sleep(LOOP_SLEEP)

    except KeyboardInterrupt:
        log.info("Arret demande (Ctrl+C)")
        telegram("Momentum Scanner arrete (Ctrl+C)")
    except Exception as exc:
        log.exception(f"Erreur inattendue : {exc}")
        telegram(f"ERREUR ROBOT : {exc}")
    finally:
        te.shutdown()
        log.info("MT5 deconnecte — au revoir")


if __name__ == "__main__":
    main()
