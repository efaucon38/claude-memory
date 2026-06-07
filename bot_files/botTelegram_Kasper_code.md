
# -*- coding: utf-8 -*-
"""
FORTRESS v3.0 — Robot de trading Telegram → MT5
================================================
Historique des versions
-----------------------
v1   : version initiale (signal follower basique)
v2   : parser renforcé, SL ATR dynamique, BE/trailing ajustés, bug CSV corrigé
v2.1 : ajout Silver, EURJPY, NZDCAD ; filtre provider par symbole
v2.2 : corrections post-revue de code (8 points)
v2.3 : calibration spread_max sur données M1 réelles
v2.4 : lots fixes calibrés à 1% de risque ; bug log démarrage corrigé
v2.5 : refonte stratégie de sortie — 3 tranches par signal
       Phase 1 (signal rapide) : 3 trades simultanés avec SL/TP automatiques
         T1 → SLauto / TP1auto = 1.0 × SLauto
         T2 → SLauto / TP2auto = 2.0 × SLauto
         T3 → SLauto / TP3auto = 4.0 × SLauto
       Phase 2 (signal formel ~20-120s après) : mise à jour avec paramètres Kasper
       BE après TP1 : SL T2+T3 → prix_entrée ± spread
       Cooldown adaptatif, identification tranches par comment MT5
v2.6 : lots dynamiques — risque constant quelle que soit la volatilité ou la balance
         lot/tranche = (balance × RISK_PCT / 3) / (sl_pts × tick_value_eur)
         tick_value_eur encodée dans SYMBOL_CONFIG (spécifications broker réelles)
         → S'ajuste automatiquement si balance passe de 1000€ à 10 000€
         → Risque réel loggé en EUR à chaque ouverture
       sl_atr_mult : 1.5 → 2.0 (meilleure survie avant signal formel)
       FORMAL_SIGNAL_WINDOW : 120s → 180s (couvre le cas +126s observé en prod)
       Parser formel enrichi : gère les signaux sans SL explicite (TP seulement)
       sl_min Silver : 30 → 100 pts (évite fermeture en <10s avant signal formel)
v3.1 : corrections post-déploiement (mai 2026)
       Bug 1 — Boucle infinie dans manage_groups : les tickets clôturés étaient
         retraités à chaque itération de 2s car logged_positions n'était mis à
         jour que dans update_performance_log. Ajout d'un set qualifié_exits
         au niveau module pour éviter le retraitement.
       Bug 2 — exit_type toujours UNKNOWN : history_deals_get(now-30min, now)
         manquait les deals plus anciens. La fenêtre utilise maintenant
         l'heure d'ouverture du groupe (group["opened_at"]) comme borne basse,
         avec un fallback sur -24h.
       Bug 3 — Doublons CSV (tranche=NaN) : update_performance_log loggait
         certaines positions avant que manage_groups ne les retire d'active_groups,
         générant une 2ème ligne sans tranche. Les positions encore dans un groupe
         actif sont désormais exclues de update_performance_log.
       Bug 4 — Silver surdimensionné : tick_value_eur Silver corrigé de 4.608
         à 0.921 (recalibration : 5000oz × 0.001USD/pt vs spéc broker réelle).
         L'impact par trade Silver est désormais comparable à Gold.
v3.0 : remplacement du BE fixe sur T2 et T3 par des trailing stops dynamiques
       Analyse de performance (mai 2026, 9 signaux Gold) a montré que le BE
       transformait systématiquement les trades avec bon MFE (500-1300pts) en
       sorties quasi-nulles, alors que le marché continuait. Résultats simulés :
         - Trailing 1×ATR sur T2+T3 : +89$ supplémentaires sur la période
         - Trailing 0.5×ATR sur T2+T3 : +144$ supplémentaires (trop serré en prod)
       Paramètres trailing (facilement ajustables via constantes dédiées) :
         TRAILING_MULT_T2 = 1.0  → recul max toléré depuis MFE = 1 × ATR
         TRAILING_MULT_T3 = 1.0  → recul max toléré depuis MFE = 1 × ATR
       Comportement :
         - Activé seulement après TP1 atteint (T1 fermé), comme le BE avant
         - Le trailing ne peut jamais reculer sous le BE (protection plancher)
         - Si le TP2/TP3 de Kasper est atteint, le trade clôture normalement
         - Sans signal formel (informel seul) : trailing sur SL ATR d'origine
       Journal de performance enrichi pour comparatif v2.6 vs v3.0 :
         - exit_type   : "SL" / "TP_KASPER" / "TRAILING" / "BE" / "UNKNOWN"
         - trailing_active : True/False (trailing était-il actif à la clôture ?)
         - mfe_at_close    : MFE en points au moment de la clôture
         - tp_kasper_price : prix TP Kasper assigné (0 si non reçu)
         - be_price        : prix BE calculé (0 si non déclenché)
         - bot_version     : "v2.6" ou "v3.0" pour filtrer dans les analyses
"""

import time
import MetaTrader5 as mt5
import asyncio
import hashlib
import os
import csv
import re
from datetime import datetime, date, timedelta
from telethon import TelegramClient, events

# ==========================================================
# CONFIGURATION GENERALE
# ==========================================================
BOT_NAME     = "FORTRESS_v3.1"
BOT_VERSION  = "v3.1"           # Tracé dans le CSV pour comparatif avec v2.6 / v3.0
MAGIC_NUMBER = 20260416

# ==========================================================
# TRAILING STOP — PARAMÈTRES AJUSTABLES
# ==========================================================
# Multiplié par l'ATR à l'entrée pour définir le recul toléré depuis le MFE.
# Exemples :
#   1.0 → le trailing se déclenche si le prix recule de 1×ATR depuis le MFE
#   0.5 → trailing plus serré (plus de profits capturés, mais sorties prématurées possibles)
#   2.0 → trailing plus large (moins de bruit, mais capture moins)
#
# Analyse sur 9 signaux Gold (mai 2026) :
#   0.5×ATR → +144$ vs système BE fixe  (trop serré pour la prod)
#   1.0×ATR → +89$  vs système BE fixe  ← recommandé pour démarrage
#   2.0×ATR → +21$  vs système BE fixe  (trop large)
#
# Le trailing ne peut jamais reculer sous le BE (plancher de protection).
# Il n'est activé qu'après que T1 a atteint son TP (comme le BE en v2.6).
TRAILING_MULT_T2 = 1.0   # multiple ATR pour le trailing de T2
TRAILING_MULT_T3 = 1.0   # multiple ATR pour le trailing de T3

MT5_PATH = r"C:\Program Files\Raise Global MT5 Terminal Demo01\terminal64.exe"
LOGIN    = 5004591
PASSWORD = "nB4@wqv#ox"
SERVER   = "RaiseGlobal-Live"

DEVIATION              = 50
MAX_DAILY_LOSS_PERCENT = 20
RISK_PCT               = 0.01

# ==========================================================
# FIREWALL CONFIG
# ==========================================================
LOCK_TIMEOUT         = 5
COOLDOWN_MIN         = 120    # secondes minimum absolu
COOLDOWN_ATR_FACTOR  = 0.5   # secondes par point d'ATR
MAX_SIGNAL_LATENCY   = 20
FORMAL_SIGNAL_WINDOW = 180   # délai max pour signal formel (126s observé en prod → marge 180s)

trade_locks     = {}
symbol_cooldown = {}

# ==========================================================
# TELEGRAM
# ==========================================================
api_id   = 31164648
api_hash = "0ffe1b119aea530cbe70d946d39241d6"

PROVIDERS = [
    {"name": "Kasper_TA",    "channel_id": -1001770543299},
    {"name": "Kasper_Forex", "channel_id": -1002364564741},
]

# ==========================================================
# LOGGING
# ==========================================================
LOG_FILE = "botTelegramMT5Kasper_v2_log.txt"

def log(message: str, level: str = "INFO"):
    now  = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    line = f"[{now}] [{level}] [{BOT_NAME}] [MAGIC:{MAGIC_NUMBER}] {message}"
    print(line)
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(line + "\n")

# ==========================================================
# CONFIGURATION PAR PAIRE
# ==========================================================
# sl_atr_mult      : multiplicateur ATR M1 → 2.0 (v2.6, vs 1.5 en v2.5)
#                    Meilleure survie avant réception du signal formel
# tick_value_eur   : EUR par point par lot — spécifications Raise Global
#                    Gold   : contrat 100oz, 1pt=0.01USD → 100×0.01/EURUSD = 0.922
#                    Bitcoin: contrat 1BTC, 1pt=0.01USD → 1×0.01/EURUSD  = 0.009
#                    Silver : contrat 5000oz, 1pt=0.001USD → 5000×0.001/EURUSD = 4.608
#                    EURJPY : contrat 100K EUR, 1pt=0.001JPY → 100K×0.001/USDJPY/EURUSD
#                    NZDCAD : contrat 100K NZD, 1pt=0.00001CAD → 100K×0.00001/USDCAD/EURUSD
# sl_min Silver    : 30 → 100 pts (évite fermeture en <10s avant signal formel)
# lot              : conservé à titre indicatif — N'EST PLUS UTILISÉ en v2.6
#                    Le lot est calculé dynamiquement à chaque ouverture

SYMBOL_CONFIG = {

    "Gold": {
        "aliases"           : ["GOLD", "XAUUSD"],
        "broker_symbol"     : "Gold",
        "sl_atr_mult"       : 2.0,    # v2.5: 1.5 → v2.6: 2.0
        "sl_min"            : 200,
        "sl_max"            : 1200,
        "spread_max"        : 60,
        "tick_value_eur"    : 0.92166, # 100oz × 0.01USD/pt / 1.085 EURUSD
        "vol_min"           : 0.01,
        "vol_step"          : 0.01,
        "atr_bars"          : 14,
        "providers_allowed" : None,
    },

    "BTCUSD": {
        "aliases"           : ["BTCUSD", "BTC", "BITCOIN"],
        "broker_symbol"     : "Bitcoin",
        "sl_atr_mult"       : 2.0,
        "sl_min"            : 1000,
        "sl_max"            : 5000,
        "spread_max"        : 1500,
        "tick_value_eur"    : 0.00922, # 1BTC × 0.01USD/pt / 1.085 EURUSD
        "vol_min"           : 0.01,
        "vol_step"          : 0.01,
        "atr_bars"          : 14,
        "providers_allowed" : ["Kasper_TA"],
    },

    "XAGUSD": {
        "aliases"           : ["XAGUSD", "SILVER", "ARGENT", "XAG"],
        "broker_symbol"     : "Silver",
        "sl_atr_mult"       : 2.0,
        "sl_min"            : 100,    # v2.5: 30 → v2.6: 100 pts
        "sl_max"            : 400,
        "spread_max"        : 90,
        "tick_value_eur"    : 0.92166, # recalibré v3.1 : spéc broker réelle
                                       # (était 4.60829 — erreur de contrat)
        "vol_min"           : 0.01,
        "vol_step"          : 0.01,
        "atr_bars"          : 14,
        "providers_allowed" : ["Kasper_Forex"],
    },

    "EURJPY": {
        "aliases"           : ["EURJPY", "EUR/JPY", "EUR JPY"],
        "broker_symbol"     : "EURJPY",
        "sl_atr_mult"       : 2.0,
        "sl_min"            : 80,
        "sl_max"            : 400,
        "spread_max"        : 15,
        "tick_value_eur"    : 0.54311, # 100K × 0.001JPY/pt / 169.7 USDJPY / 1.085 EURUSD
        "vol_min"           : 0.01,
        "vol_step"          : 0.01,
        "atr_bars"          : 14,
        "providers_allowed" : ["Kasper_Forex"],
    },

    "NZDCAD": {
        "aliases"           : ["NZDCAD", "NZD/CAD", "NZD CAD"],
        "broker_symbol"     : "NZDCAD",
        "sl_atr_mult"       : 2.0,
        "sl_min"            : 50,
        "sl_max"            : 400,
        "spread_max"        : 15,
        "tick_value_eur"    : 0.65833, # 100K × 0.00001CAD/pt / 1.40 USDCAD / 1.085 EURUSD
        "vol_min"           : 0.01,
        "vol_step"          : 0.01,
        "atr_bars"          : 14,
        "providers_allowed" : ["Kasper_Forex"],
    },
}

# ==========================================================
# PARSER — SIGNAL RAPIDE
# ==========================================================

BUY_KEYWORDS = [
    "BUY", "ACHAT", "J'ACHÈTE", "J ACHÈTE", "ACHÈTE", "ACHETE",
    "J'ACHETE", "J ACHETE", "LONG",
]
SELL_KEYWORDS = [
    "SELL", "VENTE", "JE VENDS", "J VENDS", "VENDS", "SHORT",
    "JE VEND", "J VEND",
]
BLACKLIST_CONTEXT = [
    "POSSIBLE", "PEUT-ÊTRE", "PEUT ETRE", "PEUT ÊTRE",
    "BILAN", "RÉSULTATS", "RESULTATS", "SEMAINE", "DERNIERS TRADES",
    "TP1 HIT", "TP2 HIT", "TP3 HIT", "TP HIT",
    "SL HIT", "STOPPÉ", "STOPPE",
    "BE", "BREAK EVEN", "BREAK-EVEN",
    "JE QUITTE", "JE DOIS QUITTER", "J'AI PLUS", "PLUS AUCUNE",
    "LIVE ", "COACHING", "WEBINAR", "ZOOM", "HTTP",
    "FÉLICITATIONS", "FELICITATIONS", "BRAVO",
    "ANALYSE", "ZONE", "OB ", "ORDER BLOCK",
    "RISQUE PLEIN",
    "SI VOUS ETES", "SI VOUS ÊTES",
    "TIKTOK", "ABONNÉS", "ABONNES",
]
ACTION_WORDS = ["NOW", "MAINTENANT", "ICI", "📈", "📉", "✅"]


def parse_signal(message: str, provider_name: str = None):
    """Retourne (direction, symbol) ou (None, None)."""
    text = message.upper()
    for word in BLACKLIST_CONTEXT:
        if word in text:
            return None, None
    direction = None
    if any(kw in text for kw in BUY_KEYWORDS):
        direction = "BUY"
    elif any(kw in text for kw in SELL_KEYWORDS):
        direction = "SELL"
    else:
        return None, None
    symbol = None
    for sym, cfg in SYMBOL_CONFIG.items():
        for alias in cfg["aliases"]:
            if alias.upper() in text:
                symbol = sym
                break
        if symbol:
            break
    if not symbol:
        return None, None
    allowed = SYMBOL_CONFIG[symbol].get("providers_allowed")
    if allowed is not None and provider_name not in allowed:
        log(f"Symbole {symbol} non autorisé pour {provider_name} — ignoré")
        return None, None
    has_price = bool(re.search(r'\d{2,6}[,\.]\d{1,3}', message))
    if not any(aw in text for aw in ACTION_WORDS) and not has_price:
        return None, None
    return direction, symbol


# ==========================================================
# PARSER — SIGNAL FORMEL (extraction SL/TP)
# ==========================================================

def parse_formal_signal(message: str):
    """
    Tente d'extraire SL, TP1, TP2, TP3 depuis un signal Kasper formel.
    Retourne un dict ou None si extraction impossible.
    Les valeurs retournées sont des PRIX (float).
    """
    text = message.upper()

    # Direction
    direction = None
    if any(kw in text for kw in BUY_KEYWORDS):
        direction = "BUY"
    elif any(kw in text for kw in SELL_KEYWORDS):
        direction = "SELL"
    else:
        return None

    # Symbole
    symbol = None
    for sym, cfg in SYMBOL_CONFIG.items():
        for alias in cfg["aliases"]:
            if alias.upper() in text:
                symbol = sym
                break
        if symbol:
            break
    if not symbol:
        return None

    def extract_price(pattern):
        """
        Extrait un prix depuis le message brut en gérant tous les formats :
          '4793.20'  → point décimal (anglais)
          '4793,20'  → virgule décimale (français)
          '4,793.20' → milliers virgule + décimal point
          '4.793,20' → milliers point + décimal virgule (européen)
          '4 793,20' → milliers espace (français typique) + décimal virgule
          '4 793.20' → milliers espace + décimal point
          '4793'     → entier sans décimale
        """
        m = re.search(pattern, message, re.IGNORECASE)
        if not m:
            return None
        raw = m.group(1).strip()

        # Supprimer les espaces internes (séparateur milliers espace)
        # Double passe pour couvrir les grands nombres (ex: '10 000 000')
        raw = re.sub(r'(\d)\s(\d)', r'\1\2', raw)
        raw = re.sub(r'(\d)\s(\d)', r'\1\2', raw)

        # Supprimer les séparateurs de milliers (virgule ou point suivi exactement de 3 chiffres)
        raw = re.sub(r'[,\.](?=\d{3}(?:[,\.\s]|$))', '', raw)

        # Normaliser le séparateur décimal restant (virgule → point)
        raw = raw.replace(',', '.')

        try:
            return float(raw)
        except ValueError:
            return None

    sl  = extract_price(r'SL\s*[:\-]?\s*([\d][\d\s,\.]*)')
    tp1 = extract_price(r'TP\s*1\s*[:\-]?\s*([\d][\d\s,\.]*)')
    tp2 = extract_price(r'TP\s*2\s*[:\-]?\s*([\d][\d\s,\.]*)')
    tp3 = extract_price(r'TP\s*3\s*[:\-]?\s*([\d][\d\s,\.]*)')
    if tp3 is None:
        tp3 = extract_price(r'TP\s*(?:FINAL|FIN)\s*[:\-]?\s*([\d][\d\s,\.]*)')

    # Cas signal formel sans SL explicite (ex: Bitcoin avec TP seulement)
    # On retourne quand même si au moins TP1 est présent —
    # apply_formal_signal utilisera le SL automatique du groupe existant
    if tp1 is None:
        return None

    return {
        "direction": direction,
        "symbol"   : symbol,
        "sl"       : sl,    # peut être None — géré dans apply_formal_signal
        "tp1"      : tp1,
        "tp2"      : tp2,
        "tp3"      : tp3,
    }


# ==========================================================
# ETAT DES GROUPES ACTIFS
# ==========================================================
active_groups = {}
# symbol → {
#   "direction"     : "BUY"/"SELL",
#   "entry_price"   : float,
#   "spread"        : float (en points),
#   "opened_at"     : timestamp,
#   "formal_applied": bool,
#   "be_applied"    : bool,
#   "tickets"       : {"T1": int, "T2": int, "T3": int},
#   "sl_pts"        : float,
#   "atr_val"       : float,
# }

# ==========================================================
# MODULE PERFORMANCE / MFE
# ==========================================================
PERF_FILE = "botTelegramMT5Kasper_v3_performance.csv"

mfe_points       = {}
risk_points      = {}
position_meta    = {}
logged_positions = set()
# v3.1 — Bug 1 : évite de requalifier le même ticket clôturé à chaque itération
# de manage_groups (boucle infinie précédente)
qualified_exits  = set()


def init_performance_file():
    if not os.path.exists(PERF_FILE):
        with open(PERF_FILE, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f, delimiter=";")
            writer.writerow([
                # ── Identification ──────────────────────────────────────────
                "time_close", "symbol", "direction", "position_ticket",
                "tranche", "volume", "entry_price", "exit_price",
                # ── P&L ─────────────────────────────────────────────────────
                "profit_money", "profit_points",
                "risk_points", "R_multiple", "MFE_points",
                # ── Paramètres à l'entrée ────────────────────────────────────
                "sl_dynamic_pts", "atr_at_entry", "spread_at_entry",
                # ── Flags signal ─────────────────────────────────────────────
                "formal_applied",
                # ── Colonnes v3.0 : comparatif trailing vs BE ────────────────
                # exit_type     : comment la position s'est fermée
                #                 "SL"        → stop-loss touché
                #                 "TP_KASPER" → TP Kasper atteint normalement
                #                 "TRAILING"  → trailing stop déclenché
                #                 "BE"        → break-even atteint (plancher trailing)
                #                 "UNKNOWN"   → clôture externe / non identifiée
                "exit_type",
                # trailing_active : True si le trailing était activé sur cette
                #                   tranche au moment de la clôture
                "trailing_active",
                # mfe_at_close  : MFE (pts) au moment exact de la clôture
                #                 Permet de recalculer a posteriori ce qu'un
                #                 autre multiple ATR aurait donné
                "mfe_at_close",
                # tp_kasper_price : prix TP Kasper assigné à cette tranche
                #                   (0.0 si signal formel non reçu)
                #                   Permet de simuler l'exit TP fixe vs trailing
                "tp_kasper_price",
                # be_price       : prix BE calculé pour cette tranche
                #                   (0.0 si BE non déclenché = T1 jamais atteint)
                #                   Plancher du trailing en v3
                "be_price",
                # trailing_sl_at_close : valeur du trailing SL au moment de
                #                        la clôture (prix)
                #                        Permet de recalculer l'exit trailing
                #                        avec un autre multiple
                "trailing_sl_at_close",
                # bot_version    : "v2.6" ou "v3.0" — filtre pour comparatif
                "bot_version",
            ])


def update_performance_log():
    start_dt = datetime.combine(date.today(), datetime.min.time()) - timedelta(days=1)
    deals    = mt5.history_deals_get(start_dt, datetime.now())
    if deals is None:
        return

    positions_deals = {}
    for d in deals:
        if d.magic != MAGIC_NUMBER or d.position_id == 0:
            continue
        positions_deals.setdefault(d.position_id, []).append(d)

    if not positions_deals:
        return

    # v3.1 — Bug 3 : exclure les tickets encore dans un groupe actif pour éviter
    # les doublons CSV (tranche=NaN) causés par un logging prématuré
    tickets_in_active_groups = set()
    for grp in active_groups.values():
        for t in grp["tickets"].values():
            if t:
                tickets_in_active_groups.add(t)

    with open(PERF_FILE, "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f, delimiter=";")
        for pos_id, dlist in positions_deals.items():
            if pos_id in logged_positions:
                continue
            # v3.1 — Bug 3 : ne pas logger tant que la position est encore
            # dans un groupe actif (évite les doublons tranche=NaN)
            if pos_id in tickets_in_active_groups:
                continue
            entry_deal = exit_deal = None
            total_profit = volume = 0.0
            for d in dlist:
                total_profit += d.profit
                volume = max(volume, abs(d.volume))
                if d.entry == mt5.DEAL_ENTRY_IN:
                    entry_deal = d
                elif d.entry == mt5.DEAL_ENTRY_OUT:
                    exit_deal = d
            if entry_deal is None or exit_deal is None:
                continue
            symbol   = exit_deal.symbol
            sym_info = mt5.symbol_info(symbol)
            if sym_info is None:
                continue
            point       = sym_info.point
            entry_price = entry_deal.price
            exit_price  = exit_deal.price
            time_close  = datetime.fromtimestamp(exit_deal.time)
            if entry_deal.type == mt5.DEAL_TYPE_BUY:
                direction     = "BUY"
                profit_points = (exit_price - entry_price) / point
            elif entry_deal.type == mt5.DEAL_TYPE_SELL:
                direction     = "SELL"
                profit_points = (entry_price - exit_price) / point
            else:
                direction, profit_points = "UNKNOWN", 0.0

            meta              = position_meta.get(pos_id, {})
            rp                = risk_points.get(pos_id, 0.0)
            mfe               = mfe_points.get(pos_id, 0.0)
            r_mult            = (profit_points / rp) if rp > 0 else 0.0

            # ── Colonnes v3.0 ─────────────────────────────────────────────
            exit_type          = meta.get("exit_type",           "UNKNOWN")
            trailing_active    = meta.get("trailing_active",     False)
            mfe_at_close       = mfe  # MFE courant = mfe_points[pos_id] à l'instant du log
            tp_kasper_price    = meta.get("tp_kasper_price",     0.0)
            be_price_log       = meta.get("be_price",            0.0)
            trailing_sl_close  = meta.get("trailing_sl_at_close",0.0)

            writer.writerow([
                time_close.strftime("%Y-%m-%d %H:%M:%S"),
                symbol, direction, pos_id,
                meta.get("tranche", ""),
                volume,
                f"{entry_price:.5f}", f"{exit_price:.5f}",
                f"{total_profit:.2f}", f"{profit_points:.1f}",
                f"{rp:.1f}", f"{r_mult:.2f}", f"{mfe:.1f}",
                f"{meta.get('sl_dynamic_pts',  0.0):.1f}",
                f"{meta.get('atr_at_entry',    0.0):.1f}",
                f"{meta.get('spread_at_entry', 0.0):.1f}",
                meta.get("formal_applied", False),
                # v3.0
                exit_type,
                trailing_active,
                f"{mfe_at_close:.1f}",
                f"{tp_kasper_price:.5f}",
                f"{be_price_log:.5f}",
                f"{trailing_sl_close:.5f}",
                BOT_VERSION,
            ])
            logged_positions.add(pos_id)
            for store in (mfe_points, risk_points, position_meta):
                store.pop(pos_id, None)


# ==========================================================
# ATR DYNAMIQUE
# ==========================================================
def get_atr_points(broker_symbol: str, n_bars: int) -> float:
    rates = mt5.copy_rates_from_pos(broker_symbol, mt5.TIMEFRAME_M1, 0, n_bars + 1)
    if rates is None or len(rates) < 2:
        return 0.0
    info = mt5.symbol_info(broker_symbol)
    if info is None:
        return 0.0
    point  = info.point
    ranges = [(r["high"] - r["low"]) / point for r in rates]
    return sum(ranges) / len(ranges)


def compute_sl_points(symbol: str) -> tuple:
    """Retourne TOUJOURS (sl_pts: float, atr_val: float)."""
    cfg           = SYMBOL_CONFIG[symbol]
    broker_symbol = cfg["broker_symbol"]
    atr           = get_atr_points(broker_symbol, cfg["atr_bars"])
    if atr == 0.0:
        log(f"[{symbol}] ATR indisponible → fallback SL_MIN={cfg['sl_min']}pts", "WARNING")
        return float(cfg["sl_min"]), 0.0
    sl_raw     = atr * cfg["sl_atr_mult"]
    sl_clamped = max(cfg["sl_min"], min(cfg["sl_max"], sl_raw))
    log(f"[{symbol}] ATR={atr:.0f}pts | SL brut={sl_raw:.0f}pts | SL final={sl_clamped:.0f}pts")
    return sl_clamped, atr


def compute_cooldown(atr_val: float) -> float:
    return max(COOLDOWN_MIN, atr_val * COOLDOWN_ATR_FACTOR)


def compute_lot_per_tranche(symbol: str, sl_pts: float) -> tuple:
    """
    Calcule le lot par tranche dynamiquement pour maintenir un risque constant.
    lot = (balance × RISK_PCT / 3) / (sl_pts × tick_value_eur)
    Retourne (lot: float, risque_eur: float)
    """
    cfg           = SYMBOL_CONFIG[symbol]
    step          = cfg["vol_step"]
    vol_min       = cfg["vol_min"]
    tick_value    = cfg["tick_value_eur"]

    account = mt5.account_info()
    balance = account.balance if account else 1000.0

    risk_per_tranche = (balance * RISK_PCT) / 3.0

    if sl_pts > 0 and tick_value > 0:
        lot_brut = risk_per_tranche / (sl_pts * tick_value)
    else:
        lot_brut = vol_min

    lot = round(round(lot_brut / step) * step, 3)
    lot = max(lot, vol_min)

    risque_reel = lot * sl_pts * tick_value

    if lot_brut < vol_min:
        log(f"[{symbol}] Lot calculé {lot_brut:.4f} < vol_min {vol_min} "
            f"→ forcé à {vol_min} | risque réel={risque_reel:.2f}€", "WARNING")

    return lot, risque_reel


# ==========================================================
# WATCHDOG MT5
# ==========================================================
def connect_mt5() -> bool:
    if not mt5.initialize(path=MT5_PATH, login=LOGIN, password=PASSWORD, server=SERVER):
        log(f"Echec connexion MT5 : {mt5.last_error()}", "ERROR")
        return False
    account = mt5.account_info()
    log(f"Connecté compte {account.login} | Balance : {account.balance:.2f}")
    return True


def ensure_connection() -> bool:
    try:
        if mt5.terminal_info() is None:
            raise Exception("terminal_info() None")
    except Exception:
        log("MT5 freeze détecté — reconnexion...", "WARNING")
        mt5.shutdown()
        time.sleep(2)
        return connect_mt5()
    return True


# ==========================================================
# FIREWALL
# ==========================================================
def firewall_check(symbol: str) -> bool:
    now = time.time()
    if symbol in trade_locks:
        if now - trade_locks[symbol] < LOCK_TIMEOUT:
            log(f"Signal bloqué (lock actif) {symbol}")
            return False
        del trade_locks[symbol]
    if symbol in symbol_cooldown:
        entry    = symbol_cooldown[symbol]
        duration = entry["duration"]
        if now - entry["ts"] < duration:
            remaining = duration - (now - entry["ts"])
            log(f"Signal bloqué (cooldown {duration:.0f}s, reste {remaining:.0f}s) {symbol}")
            return False
    trade_locks[symbol] = now
    return True


# ==========================================================
# CONTROLE PERTE JOURNALIERE
# ==========================================================
def daily_loss_exceeded() -> bool:
    start_dt = datetime.combine(date.today(), datetime.min.time())
    history  = mt5.history_deals_get(start_dt, datetime.now())
    if history is None:
        return False
    today_profit = sum(d.profit for d in history if d.magic == MAGIC_NUMBER)
    account      = mt5.account_info()
    if account is None:
        return False
    loss_pct = (-today_profit / account.balance * 100) if today_profit < 0 else 0
    if loss_pct >= MAX_DAILY_LOSS_PERCENT:
        log(f"Perte journalière {loss_pct:.2f}% ≥ limite {MAX_DAILY_LOSS_PERCENT}%.", "ERROR")
        return True
    return False


# ==========================================================
# ENVOI D'ORDRE AVEC FALLBACK IOC → FOK
# ==========================================================
def send_order(request: dict):
    result = mt5.order_send(request)
    if result is None:
        return None
    if result.retcode == 10030:
        log("IOC refusé → tentative FOK", "WARNING")
        request["type_filling"] = mt5.ORDER_FILLING_FOK
        result = mt5.order_send(request)
    return result


# ==========================================================
# MODIFICATION POSITION AVEC VERIFICATION
# ==========================================================
def modify_position(ticket: int, broker_symbol: str,
                    sl: float, tp: float,
                    label: str, retry: int = 2) -> bool:
    """Modifie SL/TP d'une position et vérifie la bonne application."""
    for attempt in range(retry):
        request = {
            "action"  : mt5.TRADE_ACTION_SLTP,
            "symbol"  : broker_symbol,
            "sl"      : sl,
            "tp"      : tp,
            "position": ticket,
            "magic"   : MAGIC_NUMBER,
            "comment" : BOT_NAME,
        }
        result = mt5.order_send(request)
        if result is None or result.retcode != mt5.TRADE_RETCODE_DONE:
            log(f"Echec modify {label} ticket={ticket} : "
                f"retcode={result.retcode if result else 'None'} "
                f"(tentative {attempt+1}/{retry})", "WARNING")
            time.sleep(0.5)
            continue
        # Vérification post-modify
        time.sleep(0.2)
        positions = mt5.positions_get(ticket=ticket)
        if positions:
            pos    = positions[0]
            sl_ok  = abs(pos.sl - sl) < 0.011
            tp_ok  = abs(pos.tp - tp) < 0.011
            if sl_ok and tp_ok:
                log(f"{label} ticket={ticket} ✓ SL={sl:.5f} TP={tp:.5f}")
                return True
            log(f"{label} ticket={ticket} discordance "
                f"SL attendu={sl:.5f} obtenu={pos.sl:.5f} | "
                f"TP attendu={tp:.5f} obtenu={pos.tp:.5f}", "WARNING")
        time.sleep(0.5)
    log(f"Echec définitif modify {label} ticket={ticket}", "ERROR")
    return False


# ==========================================================
# OUVERTURE GROUPE DE 3 TRADES
# ==========================================================
def open_trade_group(direction: str, symbol: str, signal_type: str = "INFORMEL"):
    if daily_loss_exceeded():
        return
    if symbol in active_groups:
        log(f"[{symbol}] Groupe déjà actif — trade ignoré.")
        return
    if not ensure_connection():
        log("MT5 indisponible — trade annulé.", "ERROR")
        return

    cfg           = SYMBOL_CONFIG[symbol]
    broker_symbol = cfg["broker_symbol"]
    info          = mt5.symbol_info(broker_symbol)
    tick          = mt5.symbol_info_tick(broker_symbol)
    if not info or not tick:
        log(f"[{symbol}] Données symbole indisponibles.", "ERROR")
        return

    point  = info.point
    spread = abs(tick.ask - tick.bid) / point

    if spread > cfg["spread_max"] * 2:
        log(f"[{symbol}] Spread extrême ({spread:.0f}pts). Groupe bloqué.")
        return
    if spread > cfg["spread_max"]:
        log(f"[{symbol}] Spread trop élevé ({spread:.0f}pts). Groupe bloqué.")
        return

    sl_pts, atr_val = compute_sl_points(symbol)
    lot, risque_eur = compute_lot_per_tranche(symbol, sl_pts)

    log(f"[{symbol}] Lot dynamique={lot} | SL={sl_pts:.0f}pts | "
        f"risque/tranche={risque_eur:.2f}€ | risque total={risque_eur*3:.2f}€")

    if direction == "BUY":
        price      = tick.ask
        sl_price   = price - sl_pts * point
        order_type = mt5.ORDER_TYPE_BUY
        tp1_price  = price + sl_pts * 1.0 * point
        tp2_price  = price + sl_pts * 2.0 * point
        tp3_price  = price + sl_pts * 4.0 * point
    else:
        price      = tick.bid
        sl_price   = price + sl_pts * point
        order_type = mt5.ORDER_TYPE_SELL
        tp1_price  = price - sl_pts * 1.0 * point
        tp2_price  = price - sl_pts * 2.0 * point
        tp3_price  = price - sl_pts * 4.0 * point

    tranches_config = [
        ("T1", tp1_price),
        ("T2", tp2_price),
        ("T3", tp3_price),
    ]

    tickets = {}

    for tranche, tp_price in tranches_config:
        request = {
            "action"      : mt5.TRADE_ACTION_DEAL,
            "symbol"      : broker_symbol,
            "volume"      : lot,
            "type"        : order_type,
            "price"       : price,
            "sl"          : sl_price,
            "tp"          : tp_price,
            "deviation"   : DEVIATION,
            "magic"       : MAGIC_NUMBER,
            "comment"     : f"FORTRESS_{tranche}",
            "type_time"   : mt5.ORDER_TIME_GTC,
            "type_filling": mt5.ORDER_FILLING_IOC,
        }
        result = send_order(request)

        if result is None or result.retcode != mt5.TRADE_RETCODE_DONE:
            retcode = result.retcode if result else "None"
            log(f"[{symbol}] Echec ouverture {tranche} : retcode={retcode}", "ERROR")
            continue

        # Récupérer le ticket de position réel
        time.sleep(0.1)
        positions  = mt5.positions_get(symbol=broker_symbol)
        pos_ticket = None
        if positions:
            for pos in positions:
                if (pos.magic == MAGIC_NUMBER
                        and pos.comment == f"FORTRESS_{tranche}"
                        and pos.ticket not in tickets.values()):
                    pos_ticket = pos.ticket
                    break
        if pos_ticket is None:
            log(f"[{symbol}] {tranche} ouvert mais ticket non trouvé "
                f"(fallback ordre={result.order})", "WARNING")
            pos_ticket = result.order

        tickets[tranche] = pos_ticket
        position_meta[pos_ticket] = {
            "sl_dynamic_pts"   : sl_pts,
            "atr_at_entry"     : atr_val,
            "spread_at_entry"  : spread,
            "tranche"          : tranche,
            "formal_applied"   : False,
            # v3.0 — initialisés à l'ouverture, mis à jour au fil du trade
            "exit_type"        : "UNKNOWN",
            "trailing_active"  : False,
            "tp_kasper_price"  : 0.0,   # renseigné par apply_formal_signal
            "be_price"         : 0.0,   # renseigné par manage_groups au moment du BE
            "trailing_sl_at_close": 0.0,
        }
        risk_points[pos_ticket] = sl_pts

        log(f"[{symbol}] {tranche} ouvert | ticket={pos_ticket} | lot={lot} | "
            f"prix={price:.5f} | SL={sl_price:.5f} ({sl_pts:.0f}pts) | "
            f"TP={tp_price:.5f} | ATR={atr_val:.0f}pts | signal={signal_type}")

    if len(tickets) > 0:
        active_groups[symbol] = {
            "direction"     : direction,
            "entry_price"   : price,
            "sl_price"      : sl_price,   # prix SL auto — fallback si formel sans SL
            "spread"        : spread,
            "opened_at"     : time.time(),
            "formal_applied": False,
            "be_applied"    : False,
            "tickets"       : tickets,
            "sl_pts"        : sl_pts,
            "atr_val"       : atr_val,
            # v3.0 — trailing
            "trailing_active": False,   # True après que T1 est clôturé
            "be_price"       : 0.0,     # prix BE calculé (plancher trailing)
            "trailing_sl_t2" : 0.0,     # valeur courante du trailing SL de T2
            "trailing_sl_t3" : 0.0,     # valeur courante du trailing SL de T3
        }
        cooldown_val = compute_cooldown(atr_val)
        trade_locks.pop(symbol, None)
        symbol_cooldown[symbol] = {"ts": time.time(), "duration": cooldown_val}
        log(f"[{symbol}] Groupe {list(tickets.keys())} actif | cooldown={cooldown_val:.0f}s")
    else:
        trade_locks.pop(symbol, None)


# ==========================================================
# APPLICATION DU SIGNAL FORMEL
# ==========================================================
def apply_formal_signal(symbol: str, parsed: dict):
    if symbol not in active_groups:
        log(f"[{symbol}] Signal formel sans groupe actif — ignoré.", "WARNING")
        return

    group = active_groups[symbol]
    if group["formal_applied"]:
        log(f"[{symbol}] Signal formel déjà appliqué — ignoré.")
        return

    elapsed = time.time() - group["opened_at"]
    if elapsed > FORMAL_SIGNAL_WINDOW:
        log(f"[{symbol}] Signal formel trop tardif ({elapsed:.0f}s > {FORMAL_SIGNAL_WINDOW}s).")
        return

    cfg           = SYMBOL_CONFIG[symbol]
    broker_symbol = cfg["broker_symbol"]
    info          = mt5.symbol_info(broker_symbol)
    if not info:
        return
    point = info.point

    direction = group["direction"]
    entry     = group["entry_price"]
    sl_k      = parsed["sl"]
    tp1_k     = parsed["tp1"]
    tp2_k     = parsed["tp2"]
    tp3_k     = parsed["tp3"]

    # Si SL absent du signal formel → conserver le SL automatique du groupe
    if sl_k is None:
        sl_k = group["sl_price"]  # prix du SL auto calculé à l'ouverture
        log(f"[{symbol}] SL absent du signal formel → conservé SL auto ({sl_k:.5f})")

    # Recalcul TP2/TP3 depuis TP1Kasper si manquants
    if tp1_k is not None:
        tp1_dist = abs(tp1_k - entry)
        if tp2_k is None:
            tp2_k = entry + tp1_dist * 2.0 if direction == "BUY" else entry - tp1_dist * 2.0
            log(f"[{symbol}] TP2 manquant → recalculé à {tp2_k:.5f}")
        if tp3_k is None:
            tp3_k = entry + tp1_dist * 4.0 if direction == "BUY" else entry - tp1_dist * 4.0
            log(f"[{symbol}] TP3 manquant → recalculé à {tp3_k:.5f}")

    mapping = {
        "T1": (sl_k, tp1_k),
        "T2": (sl_k, tp2_k),
        "T3": (sl_k, tp3_k),
    }

    all_ok = True
    for tranche, (sl, tp) in mapping.items():
        ticket = group["tickets"].get(tranche)
        if ticket is None:
            log(f"[{symbol}] Ticket {tranche} absent du groupe", "WARNING")
            all_ok = False
            continue
        if sl is None or tp is None:
            log(f"[{symbol}] {tranche} SL ou TP None — modification ignorée", "WARNING")
            all_ok = False
            continue
        ok = modify_position(ticket, broker_symbol, sl, tp, f"[{symbol}] {tranche}")
        if ok and ticket in position_meta:
            sl_pts_k = abs(entry - sl) / point
            position_meta[ticket]["sl_dynamic_pts"]  = sl_pts_k
            position_meta[ticket]["formal_applied"]  = True
            position_meta[ticket]["tp_kasper_price"] = tp   # v3.0 — pour le CSV
            risk_points[ticket] = sl_pts_k
        elif not ok:
            all_ok = False

    group["formal_applied"] = True
    status = "✓ complet" if all_ok else "⚠️ partiel"
    log(f"[{symbol}] Signal formel appliqué ({status}) | "
        f"SLK={sl_k} TP1K={tp1_k} TP2K={tp2_k:.5f} TP3K={tp3_k:.5f}")


# ==========================================================
# GESTION DES GROUPES — MFE + TRAILING STOP après TP1
# ==========================================================
# v3.0 : remplacement du BE fixe (v2.6) par des trailing stops dynamiques
# sur T2 et T3. Le trailing est activé dès que T1 est fermé (TP1 atteint).
#
# Logique trailing :
#   trail_dist_t2 = atr_val × TRAILING_MULT_T2  (en prix)
#   trail_dist_t3 = atr_val × TRAILING_MULT_T3  (en prix)
#
#   BUY  → trailing_sl = max(mfe_price - trail_dist, be_price)
#   SELL → trailing_sl = min(mfe_price + trail_dist, be_price)
#
# Le trailing SL est mis à jour à chaque itération (toutes les 2s via main).
# Si le prix courant touche le trailing SL → la position se ferme via le SL
# positionné dans MT5. Le log enregistre exit_type="TRAILING".
#
# Si TP2/TP3 de Kasper est atteint avant le trailing → exit_type="TP_KASPER".
# Si le prix touche le SL initial (avant T1 clôturé) → exit_type="SL".
# ==========================================================
def manage_groups():
    if not active_groups:
        return

    for symbol in list(active_groups.keys()):
        group         = active_groups[symbol]
        cfg           = SYMBOL_CONFIG[symbol]
        broker_symbol = cfg["broker_symbol"]

        info = mt5.symbol_info(broker_symbol)
        tick = mt5.symbol_info_tick(broker_symbol)
        if not info or not tick:
            continue

        point     = info.point
        direction = group["direction"]
        entry     = group["entry_price"]
        spread    = group["spread"]
        atr_val   = group["atr_val"]

        price_current = tick.ask if direction == "SELL" else tick.bid
        profit_distance = (
            (price_current - entry) / point if direction == "BUY"
            else (entry - price_current) / point
        )

        # ── MFE global du groupe (partagé entre T1/T2/T3 car même entrée) ──
        for ticket in group["tickets"].values():
            if ticket and profit_distance > 0:
                mfe_points[ticket] = max(mfe_points.get(ticket, 0.0), profit_distance)

        # ── Positions encore ouvertes dans ce groupe ─────────────────────
        open_tickets = set()
        positions = mt5.positions_get(symbol=broker_symbol)
        if positions:
            for pos in positions:
                if pos.magic == MAGIC_NUMBER and pos.ticket in group["tickets"].values():
                    open_tickets.add(pos.ticket)

        t1_ticket = group["tickets"].get("T1")
        t2_ticket = group["tickets"].get("T2")
        t3_ticket = group["tickets"].get("T3")

        t1_closed = t1_ticket is not None and t1_ticket not in open_tickets
        t2_open   = t2_ticket in open_tickets if t2_ticket else False
        t3_open   = t3_ticket in open_tickets if t3_ticket else False

        # ── Activation du trailing : TP1 fermé ET trailing pas encore actif ─
        if t1_closed and (t2_open or t3_open) and not group["trailing_active"]:

            # Calcul du BE (plancher de protection du trailing)
            if direction == "BUY":
                be_price = entry + spread * point
            else:
                be_price = entry - spread * point

            group["be_price"]        = be_price
            group["trailing_active"] = True

            # Stocker be_price dans position_meta pour le CSV
            for ticket in [t2_ticket, t3_ticket]:
                if ticket and ticket in position_meta:
                    position_meta[ticket]["be_price"] = be_price

            log(f"[{symbol}] TP1 atteint → TRAILING activé sur T2/T3 "
                f"| BE plancher={be_price:.5f} "
                f"| dist T2={atr_val * TRAILING_MULT_T2:.0f}pts "
                f"| dist T3={atr_val * TRAILING_MULT_T3:.0f}pts")

        # ── Mise à jour du trailing SL à chaque itération ────────────────
        if group["trailing_active"] and (t2_open or t3_open):

            be_price  = group["be_price"]
            mfe_price = entry + mfe_points.get(t1_ticket or t2_ticket, 0.0) * point \
                        if direction == "BUY" \
                        else entry - mfe_points.get(t1_ticket or t2_ticket, 0.0) * point

            for tranche, ticket, mult in [
                ("T2", t2_ticket, TRAILING_MULT_T2),
                ("T3", t3_ticket, TRAILING_MULT_T3),
            ]:
                if ticket not in open_tickets:
                    continue

                trail_dist = atr_val * mult * point

                if direction == "BUY":
                    new_trailing_sl = max(mfe_price - trail_dist, be_price)
                else:
                    new_trailing_sl = min(mfe_price + trail_dist, be_price)

                # Récupérer le SL et TP actuels de la position
                pos_list = mt5.positions_get(ticket=ticket)
                if not pos_list:
                    continue
                pos        = pos_list[0]
                current_sl = pos.sl
                current_tp = pos.tp

                # On ne déplace le SL que si le nouveau trailing est plus
                # favorable (on ne recule jamais le SL dans le mauvais sens)
                sl_improved = (
                    (direction == "BUY"  and new_trailing_sl > current_sl + point) or
                    (direction == "SELL" and new_trailing_sl < current_sl - point)
                )

                if sl_improved:
                    ok = modify_position(
                        ticket, broker_symbol,
                        new_trailing_sl, current_tp,
                        f"[{symbol}] {tranche} trailing"
                    )
                    if ok:
                        group[f"trailing_sl_{tranche.lower()}"] = new_trailing_sl
                        if ticket in position_meta:
                            position_meta[ticket]["trailing_active"]     = True
                            position_meta[ticket]["trailing_sl_at_close"] = new_trailing_sl
                        log(f"[{symbol}] {tranche} trailing SL={new_trailing_sl:.5f} "
                            f"(MFE={profit_distance:.0f}pts, "
                            f"recul={atr_val * mult:.0f}pts, "
                            f"plancher BE={be_price:.5f})")

        # ── Détection des positions fermées pour qualifier exit_type ─────
        # v3.1 — Bug 1 : on utilise qualified_exits (set module) pour ne
        # traiter chaque ticket qu'une seule fois, quelle que soit l'itération.
        # v3.1 — Bug 2 : la fenêtre history_deals_get part de l'heure
        # d'ouverture du groupe (pas now-30min) pour couvrir tous les deals.
        group_open_dt = datetime.fromtimestamp(group["opened_at"]) - timedelta(seconds=10)

        for tranche, ticket in group["tickets"].items():
            if ticket is None:
                continue
            if ticket in open_tickets:
                continue
            if ticket in qualified_exits:
                continue   # v3.1 Bug 1 — déjà traité, ne pas reboucler

            meta = position_meta.get(ticket, {})

            # Récupérer le deal de clôture — fenêtre depuis l'ouverture du groupe
            deals_out = mt5.history_deals_get(group_open_dt, datetime.now())
            exit_type = "UNKNOWN"
            if deals_out:
                for d in deals_out:
                    if d.position_id == ticket and d.entry == mt5.DEAL_ENTRY_OUT:
                        exit_px  = d.price
                        tp_k     = meta.get("tp_kasper_price",     0.0)
                        be_p     = meta.get("be_price",            0.0)
                        trail_sl = meta.get("trailing_sl_at_close",0.0)
                        tol      = point * 5

                        if group["trailing_active"] and trail_sl > 0:
                            if direction == "BUY":
                                if be_p > 0 and abs(exit_px - be_p) <= tol:
                                    exit_type = "BE"
                                elif tp_k > 0 and abs(exit_px - tp_k) <= tol:
                                    exit_type = "TP_KASPER"
                                elif abs(exit_px - trail_sl) <= tol:
                                    exit_type = "TRAILING"
                                else:
                                    exit_type = "TRAILING"
                            else:
                                if be_p > 0 and abs(exit_px - be_p) <= tol:
                                    exit_type = "BE"
                                elif tp_k > 0 and abs(exit_px - tp_k) <= tol:
                                    exit_type = "TP_KASPER"
                                elif abs(exit_px - trail_sl) <= tol:
                                    exit_type = "TRAILING"
                                else:
                                    exit_type = "TRAILING"
                        else:
                            tp_k = meta.get("tp_kasper_price", 0.0)
                            if tp_k > 0 and abs(exit_px - tp_k) <= tol:
                                exit_type = "TP_KASPER"
                            else:
                                exit_type = "SL"
                        break

            if ticket in position_meta:
                position_meta[ticket]["exit_type"] = exit_type
            qualified_exits.add(ticket)   # v3.1 Bug 1 — marquer comme traité
            log(f"[{symbol}] {tranche} clôturé → exit_type={exit_type}")

        # ── Nettoyage groupe entièrement clôturé ─────────────────────────
        all_closed = all(
            t not in open_tickets
            for t in group["tickets"].values()
            if t is not None
        )
        if all_closed:
            log(f"[{symbol}] Groupe entièrement clôturé — nettoyage.")
            del active_groups[symbol]


# ==========================================================
# TELEGRAM
# ==========================================================
telegram_client   = TelegramClient("session", api_id, api_hash)
last_message_hash = {p["name"]: None for p in PROVIDERS}


@telegram_client.on(events.NewMessage())
async def handler(event):

    for provider in PROVIDERS:
        if event.chat_id != provider["channel_id"]:
            continue

        message = event.message.text
        if not message:
            return

        if time.time() - event.message.date.timestamp() > MAX_SIGNAL_LATENCY:
            log(f"[{provider['name']}] Signal ignoré (trop ancien)")
            return

        msg_hash = hashlib.md5(message.encode()).hexdigest()
        if msg_hash == last_message_hash[provider["name"]]:
            return
        last_message_hash[provider["name"]] = msg_hash

        log(f"Signal reçu de {provider['name']} : {message[:120].replace(chr(10), ' ')}")

        # Tenter d'abord le parsing formel
        formal = parse_formal_signal(message)

        # Si signal formel et groupe actif sur ce symbole → mise à jour
        if formal and formal["symbol"] in active_groups:
            grp = active_groups[formal["symbol"]]
            if not grp["formal_applied"]:
                log(f"Signal formel détecté → mise à jour groupe [{formal['symbol']}]")
                apply_formal_signal(formal["symbol"], formal)
                return

        # Sinon : signal d'entrée (rapide ou formel sans groupe existant)
        direction, symbol = parse_signal(message, provider_name=provider["name"])

        if direction and symbol:
            text        = message.upper()
            is_formal   = any(x in text for x in ["📈", "📉", "TP 1", "TP1", "J'ACHÈTE", "JE VENDS"])
            signal_type = "FORMEL" if is_formal else "INFORMEL"

            if firewall_check(symbol):
                open_trade_group(direction, symbol, signal_type)

                # Si le signal est déjà formel avec SL/TP → appliquer immédiatement
                if formal and formal["symbol"] == symbol and formal["sl"] is not None:
                    time.sleep(0.5)
                    apply_formal_signal(symbol, formal)
        else:
            log(f"[{provider['name']}] Message ignoré (parser)")

        return


# ==========================================================
# MAIN LOOP
# ==========================================================
async def main():
    log("=" * 62)
    log(f"BOT {BOT_NAME} DÉMARRAGE EN COURS | MAGIC={MAGIC_NUMBER}")
    log(f"  Stratégie : 3 tranches (T1×1R / T2×2R / T3×4R)")
    log(f"  Lots : DYNAMIQUES — (balance × {RISK_PCT*100:.0f}% / 3) / (SL × TV_EUR)")
    log(f"  Cooldown adaptatif : max({COOLDOWN_MIN}s, ATR×{COOLDOWN_ATR_FACTOR}s/pt)")
    log(f"  Fenêtre signal formel : {FORMAL_SIGNAL_WINDOW}s")
    log(f"  Sortie T2 : TRAILING STOP {TRAILING_MULT_T2}×ATR (plancher = BE)")
    log(f"  Sortie T3 : TRAILING STOP {TRAILING_MULT_T3}×ATR (plancher = BE)")
    log(f"  (v2.6 utilisait un BE fixe sur T2 et T3)")
    log(f"  Corrections v3.1 : boucle infinie, exit_type, doublons CSV, Silver TV")
    log("  Symboles :")
    for sym, cfg in SYMBOL_CONFIG.items():
        prov = cfg.get("providers_allowed") or "tous"
        log(f"    {sym:8s} | broker={cfg['broker_symbol']:10s} | "
            f"sl_mult={cfg['sl_atr_mult']}×ATR | "
            f"SL={cfg['sl_min']}-{cfg['sl_max']}pts | "
            f"TV={cfg['tick_value_eur']:.5f}€/pt/lot | "
            f"spread_max={cfg['spread_max']}pts | providers={prov}")
    log("=" * 62)

    if not connect_mt5():
        log("Echec connexion MT5 — arrêt.", "ERROR")
        return

    init_performance_file()
    log(f"BOT {BOT_NAME} OPÉRATIONNEL — en attente de signaux.")

    await telegram_client.start()

    while True:
        if not telegram_client.is_connected():
            log("Telegram déconnecté — reconnexion...", "WARNING")
            await telegram_client.connect()

        ensure_connection()

        if daily_loss_exceeded():
            await asyncio.sleep(60)
            continue

        manage_groups()
        update_performance_log()
        await asyncio.sleep(2)


# ==========================================================
# ENTRYPOINT
# ==========================================================
if __name__ == "__main__":
    while True:
        try:
            log(f"Démarrage session Telegram ({BOT_NAME})...")
            with telegram_client:
                telegram_client.loop.run_until_complete(main())
        except Exception as e:
            log(f"Crash détecté : {e}", "ERROR")
            time.sleep(5)
