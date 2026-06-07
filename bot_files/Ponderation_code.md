
# -*- coding: utf-8 -*-
"""
Ponderation.py v3.0 — EA Desk Quant
=============================================================================
Moteur de ponderation multi-actifs avec mode defensif/offensif
Gestion du risque dynamique, drawdown, monitoring des EA MT5

ARCHITECTURE DES FICHIERS :
  BASE_PATH       : fichiers propres a Python (logs, counters, stats, perf)
  MT5_COMMON_PATH : fichiers partages avec les EA MT5 (CSV, heartbeats)
                    Accessible via FileOpen(..., FILE_COMMON) dans MQL5

NOUVEAUTES v3.3 :
  - Nikkei retire de SYMBOLS_PROPFIRM (spread broker RaiseFX = 14x seuil max)
  - Portefeuille propfirm passe de 8 a 7 actifs actifs
  - WEIGHT_BOUNDS["Nikkei"] conserve pour reintegration future
  - Coherence bornes recalculee (somme minis < 100%, somme maxis > 100%)

NOUVEAUTES v3.0 :
  - Momentum absolu par actif avec periode INDIVIDUALISEE par symbole
    Calibrage depuis backtest v2 sur 7.2 ans (methode composee, fidele au robot)
    Resultats : v3.0 indiv = +53.1% cumule, 7.38%/an, DD 2.54%, Calmar 2.91
  - MOMENTUM_PERIOD_BY_SYMBOL : dict de periodes optimales par actif
    Calibrees sur backtest v3 (7.3 ans, seuils de range corriges par periode)
    S&P500=60j, DAX40=60j (indices US/EU : tendances trimestrielles)
    NAS100=90j, USDJPY=90j, Silver=90j (tendances intermediaires)
    EU50=120j (cycles longs indice europeens)
    GBPJPY=90j (politique monetaire UK/Japon, cycle intermediaire)
  - Ponderation hybride : alpha x (1/vol) + (1-alpha) x |momentum|
    ALPHA_DEFENSIF=0.70 (70% vol inverse), ALPHA_OFFENSIF=0.30 (70% momentum)
  - Filtre range : mise en veille CSV si |momentum_i| < MOMENTUM_RANGE_THRESHOLD[i]
    Seuils = retour_annuel/6 depuis backtest v2 (60j = 2 mois = 1/6 annee)
    USE_MOMENTUM_FILTER=True (desactivable)
  - Backlog v3.1 : pilotage auto DEFENSIF/OFFENSIF par momentum portfolio
    (reporte apres validation en production)

NOUVEAUTES v3.3 :
  - NB_SL_MAX + SAFETY_FACTOR : controle risque en EUR par trade
  - compute_risk_eur() : calcule le budget max par trade
  - Colonne RiskEUR injectee dans le CSV portfolio_weights
  - L'EA v2.8 bloque les actifs trop chers pour le capital courant

NOUVEAUTES v3.1 (historique) :
  - INITIAL_BALANCE hardcodee (fiable meme apres suppression counters.json)
  - Ecriture InitBalance_XXXX.csv dans Common/Files a chaque cycle
  - Suppression initial_balance de counters.json

NOUVEAUTES v3 (historique) :
  - Remplacement "risk_parity"/"volatility_parity" par MODE_DEFENSIF (bool)
  - Portefeuille elargi issu du backtest v2 (8 actifs propfirm + Gold CP)
  - Bornes min/max par symbole calculees a partir des scores backtest v2
  - Deux jeux de bornes : mode defensif ON et mode defensif OFF
  - SL_MULT mis a jour selon parametres optimaux backtest v2 (SL=2.5)
  - Coherence assuree : somme(mini) < 100% et somme(maxi) > 100%

LOGIQUE DES DEUX NIVEAUX :
  Ponderation.py : MODE_DEFENSIF True/False
    True  → poids inversement proportionnels a la volatilite
             bornes serrees (ecart 30% autour du poids naturel)
             favorise la stabilite et la preservation du capital
    False → poids proportionnels a la volatilite
             bornes larges (ecart 60% autour du poids naturel)
             favorise les actifs les plus dynamiques

  EA MT5 : TRADING_MODE "propfirm" / "compte_propre"
    propfirm     → seuls les actifs COMPAT_PROPFIRM=true tradent
    compte_propre → tous les actifs COMPAT_COMPTE_PROPRE=true tradent

  => 4 configurations possibles :
     1. Defensif + Propfirm   : challenge prudent (recommande)
     2. Offensif + Propfirm   : challenge agressif
     3. Defensif + Compte propre : capital propre prudent
     4. Offensif + Compte propre : capital propre agressif

ORIGINE DES BORNES PAR SYMBOLE (calcul du 2026-04-20) :
  Methode : poids_naturel = score_backtest_v2 / somme_scores_portefeuille
            mini_defensif = poids_naturel * 0.70 (tolerance 30%)
            maxi_defensif = poids_naturel * 1.30
            mini_offensif = poids_naturel * 0.40 (tolerance 60%)
            maxi_offensif = poids_naturel * 1.60
  Scores backtest v2 : S&P500=71, NAS100=71, DAX40=68, USDJPY=67,
                       Silver=61, Nikkei=59, EU50=57, GBPJPY=55, Gold=50
  Coherence verifiee : somme(minis) = 70% < 100%, somme(maxis) = 130% > 100%
=============================================================================
"""

import time
import os
import json
import csv
import io
import traceback
from datetime import datetime, timezone, timedelta
import pandas as pd
import numpy as np
import MetaTrader5 as mt5

# =============================================================================
# SECTION 1 — CONFIGURATION COMPTE
# Adapter lors d'un transfert sur un nouveau compte
# =============================================================================
MT5_PATH     = r"C:\\Program Files\\Raise Global MT5 Terminal Demo04\\terminal64.exe"
LOGIN        = 5007258
PASSWORD     = "zL3!gvG8Ol"
SERVER       = "RaiseGlobal-Live"
BROKER_NAME  = "RaiseFX"
ACCOUNT_TYPE = "demo"    # "demo" / "live" / "propfirm"

BASE_PATH       = r"C:\\Users\\Administrator\\Desktop\\EA Desk Quant"
MT5_COMMON_PATH = r"C:\\Users\\Administrator\\AppData\\Roaming\\MetaQuotes\\Terminal\\Common\\Files"

ENABLE_TRADING = True    # False = suspension immediate de tous les robots

# =============================================================================
# BALANCE INITIALE DU COMPTE — codee en dur pour fiabilite maximale
#
# Cette valeur est le capital de reference pour le calcul du drawdown total.
# Elle est codee en dur volontairement : une valeur automatique pourrait etre
# erronee si Ponderation.py est relance apres des trades (balance != capital initial).
#
# A MODIFIER MANUELLEMENT si changement de compte ou rechargement du compte.
# Exemple : compte RaiseGlobal challenge 1000 EUR -> INITIAL_BALANCE = 1000.0
# =============================================================================
INITIAL_BALANCE = 50000.0  # Capital de reference pour le calcul du DD total (EUR)
# Compte demo 5007258 — 50 000 EUR.
# Le robot se bloque si balance < 45 500 EUR (9% de 50 000).
# A RECALIBRER si rechargement significatif du compte.

# =============================================================================
# CONTROLE DU RISQUE EN EUROS PAR TRADE — [v3.2]
#
# NB_SL_MAX    : nombre de SL acceptables par jour par actif
#   Le budget DD journalier est divise par ce nombre pour obtenir
#   le risque max tolere par trade individuel.
#   Exemple : NB_SL_MAX=3 => un actif peut perdre au plus 1/3 du budget DD jour.
#
# SAFETY_FACTOR : marge de securite (0 < x <= 1)
#   Absorbe les fluctuations de change (EURUSD) et imprecisions de
#   conversion de devise. 0.80 = marge de 20% par rapport au theorique.
#
# Formule : RISK_EUR = (INITIAL_BALANCE * MAX_DAILY_LOSS_PCT * SAFETY_FACTOR)
#                      / NB_SL_MAX
#
# Avec les valeurs par defaut :
#   (1000 * 0.045 * 0.80) / 3 = 12.00 EUR par trade
#
# L'EA v2.8 compare ce montant au cout reel du trade
#   cout_reel = SL_pts x TICK_VALUE / TICK_SIZE x lot_min
# et refuse l'entree si cout_reel > RISK_EUR.
# Effet : Silver avec 1000 EUR capital est automatiquement bloque.
# Il se "reactivera" quand le capital aura suffisamment augmente.
# =============================================================================
NB_SL_MAX     = 3      # Nombre de SL acceptables par jour (diviseur du budget DD)
SAFETY_FACTOR = 0.80   # Marge securite pour fluctuations devise (80% du theorique)

# =============================================================================
# SECTION 2 — MODE DE PONDERATION
#
# MODE_DEFENSIF = False  # OFFENSIF — valide par backtest v2 7.2 ans (v3.0 indiv +53.1%)  → ponderation DEFENSIVE
#   Poids inversement proportionnels a la volatilite (actifs calmes favorises)
#   Bornes serrees (tolerance 30% autour du poids naturel)
#   Usage recommande : challenges propfirm, preservation du capital
#
# MODE_DEFENSIF = False → ponderation OFFENSIVE
#   Poids directement proportionnels a la volatilite (actifs volatils favorises)
#   Bornes larges (tolerance 60% autour du poids naturel)
#   Usage : compte propre, recherche de performance maximale
# =============================================================================
MODE_DEFENSIF = True


# =============================================================================
# SECTION 2b — PARAMETRES MOMENTUM (v3.0)
#
# Le momentum absolu mesure le deplacement net d'un actif sur N bougies D1 :
#   momentum_i = (close_J0 - close_J-N) / close_J-N
#
# Periodes INDIVIDUALISEES par actif — calibrees sur backtest v2 (7.2 ans)
# Methode : optimisation du Profit Factor par actif, validee sur 30/60/90/120j
#
#   Indices directionnels (tendances trimestrielles bien marquees) : 60j
#   Forex reactif aux cycles macro courts (USDJPY, EU50)             : 30j
#   GBPJPY : cycles longs politiques monetaires UK/Japon             : 120j
#   Silver  : metal, tendances intermediaires                        : 90j
#
# ALPHA_DEFENSIF / ALPHA_OFFENSIF :
#   Poids hybride = alpha x (1/vol) + (1-alpha) x |momentum|
#   En mode DEFENSIF : alpha=0.70 (privilege volatilite inverse)
#   En mode OFFENSIF : alpha=0.30 (privilege momentum)
#
# USE_MOMENTUM_FILTER : active/desactive la mise en veille des actifs en range
#   Si True et |momentum_i| < MOMENTUM_RANGE_THRESHOLD[i] → TradeAllowed=0
#   Seuils = retour_annuel / 6 (60j = 2 mois = 1/6 annee)
# =============================================================================

# Periodes de momentum individualisees par actif (en bougies D1)
# Calibrees sur backtest 7.2 ans, methode composee fidele au robot reel
MOMENTUM_PERIOD_BY_SYMBOL = {
    # Periodes optimales calibrees sur backtest v3 (7.3 ans, seuils corriges)
    # Methode : seuil_range = base_60j x (periode / 60) — correction appliquee
    "S&P500": 60,    # PF 1.270 a 60j — optimal confirme
    "NAS100": 90,    # PF 1.150 a 90j (+0.016 vs 60j) — tendances tech plus lentes
    "DAX40":  60,    # PF 1.071 a 60j — optimal confirme
    "USDJPY": 90,    # PF 1.152 a 90j (+0.027 vs 60j) — correction majeure (etait 30j)
    "EU50":   120,   # PF 1.127 a 120j (+0.058 vs 60j) — cycles longs indices EU
    "GBPJPY": 90,    # PF 1.076 a 90j (+0.058 vs 60j) — etait 120j
    "Silver": 90,    # PF 1.099 a 90j (+0.013 vs 60j) — inchange
    "Gold":   60,    # Non teste en portefeuille — conserve a 60j par defaut
    "Nikkei": 60,    # Exclu portefeuille actif — conserve pour reintegration
}
MOMENTUM_PERIOD = 60   # Valeur par defaut si symbole absent du dict ci-dessus

ALPHA_DEFENSIF    = 0.70   # Part de (1/vol) en mode DEFENSIF
ALPHA_OFFENSIF    = 0.30   # Part de (1/vol) en mode OFFENSIF

USE_MOMENTUM_FILTER = True  # Mettre en veille les actifs en range (|mom|<seuil)

# Seuils de range par actif (|momentum Nj| en decimal)
# Methode : retour_annuel_moyen / 6 depuis backtest v2 (7.2 ans)
# Seuil exprime par rapport a la periode optimale de chaque actif
MOMENTUM_RANGE_THRESHOLD = {
    # Seuils recalcules sur la periode optimale de chaque actif
    # Methode : seuil = base_60j x (periode_actif / 60)
    # base_60j = retour_annuel_moyen / 6 (depuis backtest v2)
    "S&P500": 0.021,   # 60j, base 2.1% × (60/60) = 2.1%
    "NAS100": 0.018,   # 90j, base 1.2% × (90/60) = 1.8%
    "DAX40":  0.012,   # 60j, base 1.2% × (60/60) = 1.2%
    "USDJPY": 0.015,   # 90j, base 1.0% × (90/60) = 1.5%
    "EU50":   0.012,   # 120j, base 0.6% × (120/60) = 1.2%
    "GBPJPY": 0.014,   # 90j, base 0.9% × (90/60) = 1.35% → 1.4%
    "Silver": 0.015,   # 90j, base 1.0% × (90/60) = 1.5%
    "Gold":   0.013,   # 60j, base 1.3% × (60/60) = 1.3% (inchange)
    "Nikkei": 0.010,   # Exclu — conserve pour reintegration
}

# =============================================================================
# SECTION 3 — PORTEFEUILLES PAR MODE EA
#
# SYMBOLS_PROPFIRM : actifs compatibles TRADING_MODE="propfirm"
#   Selectionnes sur critere propfirm_ok=1 dans backtest v2
#   Scores : S&P500=71, NAS100=71, DAX40=68, USDJPY=67,
#            Silver=61, EU50=57, GBPJPY=55
#
#   NIKKEI EXCLU (v3.3) : spread broker RaiseFX structurellement 14x le seuil
#   tolerable (ratio median=13.75, minimum absolu=6.25 — aucune heure viable).
#   A reintegrer uniquement si changement de broker ou verification manuelle
#   d'un spread normal sur cet instrument. Symbole conserve dans WEIGHT_BOUNDS
#   pour faciliter la reintegration future.
#
# SYMBOLS_COMPTE_PROPRE : actifs supplementaires pour TRADING_MODE="compte_propre"
#   Gold=50 : bon compte propre mais DD trop eleve pour propfirm
#
# SYMBOLS : union des deux listes (tous les actifs geres par Ponderation.py)
# =============================================================================
SYMBOLS_PROPFIRM     = ["S&P500", "NAS100", "DAX40", "USDJPY",
                         "Silver", "EU50",  "GBPJPY"]
SYMBOLS_COMPTE_PROPRE = ["Gold"]
SYMBOLS = SYMBOLS_PROPFIRM + SYMBOLS_COMPTE_PROPRE

# =============================================================================
# SECTION 4 — BORNES DE PONDERATION PAR SYMBOLE ET PAR MODE
#
# Origine : calcul du 2026-04-20 a partir des scores du backtest v2
# Methode documentee en en-tete de ce fichier
#
# Structure : WEIGHT_BOUNDS[symbole] = {
#     "mini_def" : borne min en mode defensif (tolerance 30%)
#     "maxi_def" : borne max en mode defensif
#     "mini_off" : borne min en mode offensif (tolerance 60%)
#     "maxi_off" : borne max en mode offensif
#     "score"    : score backtest v2 (reference)
#     "poids_naturel" : poids theorique = score / somme_scores (%)
# }
#
# VERIFICATION COHERENCE (obligatoire) :
#   Portefeuille actif : 7 actifs propfirm + Gold compte propre
#   Mode defensif  : somme(mini_def) = 62.0% < 100% OK  (7 actifs hors Nikkei)
#                    somme(maxi_def) = 115.0% > 100% OK
#   Mode offensif  : somme(mini_off) = 35.0% < 100% OK
#                    somme(maxi_off) = 140.0% > 100% OK
# =============================================================================
WEIGHT_BOUNDS = {
    # Portefeuille PROPFIRM (8 actifs, somme scores = 509)
    "S&P500": {"score":71,"poids_naturel":13.9,
               "mini_def":0.10,"maxi_def":0.18,
               "mini_off":0.06,"maxi_off":0.22},
    "NAS100": {"score":71,"poids_naturel":13.9,
               "mini_def":0.10,"maxi_def":0.18,
               "mini_off":0.06,"maxi_off":0.22},
    "DAX40":  {"score":68,"poids_naturel":13.4,
               "mini_def":0.09,"maxi_def":0.17,
               "mini_off":0.05,"maxi_off":0.21},
    "USDJPY": {"score":67,"poids_naturel":13.2,
               "mini_def":0.09,"maxi_def":0.17,
               "mini_off":0.05,"maxi_off":0.21},
    "Silver": {"score":61,"poids_naturel":12.0,
               "mini_def":0.08,"maxi_def":0.16,
               "mini_off":0.05,"maxi_off":0.19},
    # NIKKEI — EXCLU du portefeuille actif depuis v3.3 (spread broker trop large)
    # Conserve ici pour permettre reintegration future sans modifier le calcul.
    # Pour reintegrer : ajouter "Nikkei" dans SYMBOLS_PROPFIRM ci-dessus.
    # Bornes calculees sur portefeuille 8 actifs (score/509)
    "Nikkei": {"score":59,"poids_naturel":11.6,
               "mini_def":0.08,"maxi_def":0.15,
               "mini_off":0.05,"maxi_off":0.19},
    "EU50":   {"score":57,"poids_naturel":11.2,
               "mini_def":0.08,"maxi_def":0.15,
               "mini_off":0.04,"maxi_off":0.18},
    "GBPJPY": {"score":55,"poids_naturel":10.8,
               "mini_def":0.08,"maxi_def":0.14,
               "mini_off":0.04,"maxi_off":0.17},
    # Portefeuille COMPTE PROPRE uniquement (9 actifs, somme scores = 559)
    # Bornes calculees sur portefeuille 9 actifs (poids naturel = score/559)
    "Gold":   {"score":50,"poids_naturel": 8.9,
               "mini_def":0.06,"maxi_def":0.12,
               "mini_off":0.04,"maxi_off":0.14},
}

# =============================================================================
# SECTION 5 — PARAMETRES DE RISQUE
# =============================================================================
RISK_PCT           = 0.005   # Risque de base par trade (0.5%)
MAX_DAILY_LOSS_PCT = 0.045   # Perte journaliere max propfirm (4.5%)
MAX_TOTAL_DD_PCT   = 0.090   # Drawdown total max propfirm (9.0%)
MAX_DAILY_LOSS_CP  = 0.080   # Perte journaliere max compte propre (8%)
MAX_TOTAL_DD_CP    = 0.200   # Drawdown total max compte propre (20%)
SOFT_DD_PCT        = 0.045   # Seuil soft : reduction risque a 50%

# Anti-martingale — serie gagnante
TP_SERIES_LVL1 = 5    # Apres 5 TP consecutifs  -> RiskMult x2
TP_SERIES_LVL2 = 10   # Apres 10 TP consecutifs -> RiskMult x4

# Anti-martingale — serie perdante
SL_SERIES_LVL1 = 5    # Apres 5 SL consecutifs  -> RiskMult /2
SL_SERIES_LVL2 = 10   # Apres 10 SL consecutifs -> RiskMult /4

RISK_MULT_MAX = 4.0
RISK_MULT_MIN = 0.25

# =============================================================================
# SECTION 6 — PARAMETRES ATR / SL PAR SYMBOLE
# Valeurs optimales issues du backtest v2 (2026-04-20)
# =============================================================================
ATR_PERIOD = 14
ATR_TF     = mt5.TIMEFRAME_H1

SL_MULT = {
    # Scores backtest v2 et SL optimaux
    "S&P500": 2.5,   # score=71, SL optimal backtest v2
    "NAS100": 2.5,   # score=71
    "DAX40":  2.5,   # score=68
    "USDJPY": 2.5,   # score=67
    "Silver": 2.5,   # score=61
    "Nikkei": 2.5,   # score=59
    "EU50":   2.0,   # score=57, SL=2.0 optimal sur ce symbole
    "GBPJPY": 2.5,   # score=55
    "Gold":   2.5,   # score=50, compte propre uniquement
}

# =============================================================================
# SECTION 7 — FREQUENCES DE BOUCLE (secondes)
# =============================================================================
FREQ_NORMAL    = 60     # Sessions actives
FREQ_OFF_HOURS = 300    # Hors session (22h-01h UTC)
FREQ_WEEKEND   = 1800   # Weekend (30 min)
FREQ_BLOCKED   = 300    # Drawdown atteint

# =============================================================================
# SECTION 8 — MONITORING DES EA (seuils en minutes)
# =============================================================================
HEARTBEAT_ALERT_MIN   = 3
HEARTBEAT_RESTART_MIN = 8
HEARTBEAT_DISABLE_MIN = 15

# =============================================================================
# SECTION 9 — TAILLE MAXIMALE DES FICHIERS LOG
# =============================================================================
MAX_LOG_SIZE = 10 * 1024 * 1024  # 10 Mo

# =============================================================================
# IDENTIFIANT UNIQUE DU COMPTE
# =============================================================================
ACCOUNT_ID = f"{BROKER_NAME}_{ACCOUNT_TYPE}_{LOGIN}"

# =============================================================================
# CONSTRUCTION DES CHEMINS DE FICHIERS
# =============================================================================
def build_paths():
    return {
        "csv":         os.path.join(MT5_COMMON_PATH,
                                    f"portfolio_weights_{ACCOUNT_ID}.csv"),
        "counters":    os.path.join(BASE_PATH, f"counters_{ACCOUNT_ID}.json"),
        "performance": os.path.join(BASE_PATH, f"Performance_{ACCOUNT_ID}.csv"),
        "stats_hebdo": os.path.join(BASE_PATH, f"Stats_Hebdo_{ACCOUNT_ID}.csv"),
        "monitoring":  os.path.join(BASE_PATH, f"Monitoring_{ACCOUNT_ID}.csv"),
        "python_log":  os.path.join(BASE_PATH, f"Python_Log_{ACCOUNT_ID}.csv"),
    }

def heartbeat_path(symbol):
    return os.path.join(MT5_COMMON_PATH,
                        f"Heartbeat_{symbol}_{ACCOUNT_ID}.csv")

# =============================================================================
# LOGGING
# =============================================================================
def rotate_if_needed(filepath):
    if os.path.exists(filepath) and os.path.getsize(filepath) > MAX_LOG_SIZE:
        base, ext = os.path.splitext(filepath)
        ts = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        os.rename(filepath, f"{base}_archive_{ts}{ext}")

def log(message, paths, level="INFO"):
    now = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
    print(f"[{now}] [{level}] {message}")
    if paths is None:
        return
    try:
        rotate_if_needed(paths["python_log"])
        file_exists = os.path.exists(paths["python_log"])
        with open(paths["python_log"], "a", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            if not file_exists:
                writer.writerow(["TimestampUTC", "Level", "Message"])
            writer.writerow([now, level, message])
    except Exception as e:
        print(f"[LOG ERROR] {e}")

# =============================================================================
# CONNEXION MT5
# =============================================================================
def connect_mt5(paths):
    for attempt in range(1, 6):
        if mt5.initialize(path=MT5_PATH, login=LOGIN,
                          password=PASSWORD, server=SERVER):
            info = mt5.account_info()
            if info:
                log(f"MT5 connecte — Login:{info.login} | "
                    f"Balance:{info.balance:.2f} {info.currency}", paths)
                return True
        log(f"Tentative {attempt}/5 : {mt5.last_error()}", paths, "WARN")
        time.sleep(5 * attempt)
    log("Connexion MT5 impossible.", paths, "ERROR")
    return False

def ensure_connected(paths):
    if not mt5.terminal_info():
        log("Reconnexion MT5...", paths, "WARN")
        return connect_mt5(paths)
    return True

# =============================================================================
# ACTIVATION DES SYMBOLES
# =============================================================================
def activate_symbols(paths):
    for s in SYMBOLS:
        if not mt5.symbol_select(s, True):
            log(f"Impossible d'activer : {s}", paths, "WARN")

# =============================================================================
# DETECTION SESSION / WEEKEND
# =============================================================================
def is_weekend():
    return datetime.utcnow().weekday() >= 5

def is_monday_early():
    now = datetime.utcnow()
    return now.weekday() == 0 and now.hour < 8

def is_off_hours():
    hour = datetime.utcnow().hour
    return hour >= 22 or hour < 1

def is_friday_close():
    now = datetime.utcnow()
    return now.weekday() == 4 and (
        now.hour > 20 or (now.hour == 20 and now.minute >= 30))

def is_market_open():
    return not (is_weekend() or is_monday_early() or is_friday_close())

def get_loop_delay(trade_allowed_global):
    if not trade_allowed_global: return FREQ_BLOCKED
    if is_weekend():             return FREQ_WEEKEND
    if is_off_hours():           return FREQ_OFF_HOURS
    return FREQ_NORMAL

# =============================================================================
# DONNEES HISTORIQUES ET ATR
# =============================================================================
def get_returns_and_atr(symbol, paths):
    try:
        rates = mt5.copy_rates_from_pos(symbol, ATR_TF, 0, 200)
        if rates is None or len(rates) < 50:
            log(f"{symbol}: donnees insuffisantes", paths, "WARN")
            return None, None
        df     = pd.DataFrame(rates)
        closes = df["close"].astype(float).values
        highs  = df["high"].astype(float).values
        lows   = df["low"].astype(float).values
        returns = np.log(closes[1:] / closes[:-1])
        tr = np.maximum(highs[1:]-lows[1:],
             np.maximum(np.abs(highs[1:]-closes[:-1]),
                        np.abs(lows[1:] -closes[:-1])))
        atr = float(np.mean(tr[-ATR_PERIOD:]))
        return pd.Series(returns), atr
    except Exception as e:
        log(f"{symbol}: erreur donnees — {e}", paths, "ERROR")
        return None, None

# =============================================================================
# CALCUL DES PONDÉRATIONS
#
# MODE_DEFENSIF = True  : poids = 1/volatilite (actifs calmes favorises)
# MODE_DEFENSIF = False : poids = volatilite   (actifs dynamiques favorises)
#
# Les bornes min/max sont celles de WEIGHT_BOUNDS selon le mode.
# La renormalisation iterative garantit que somme des poids = 100%.
# =============================================================================
def _get_bounds(symbol):
    """Retourne (mini, maxi) selon MODE_DEFENSIF et le symbole."""
    b = WEIGHT_BOUNDS.get(symbol)
    if b is None:
        # Symbole non reference : bornes prudentes par defaut
        return (0.05, 0.20) if MODE_DEFENSIF else (0.02, 0.35)
    return (b["mini_def"], b["maxi_def"]) if MODE_DEFENSIF            else (b["mini_off"], b["maxi_off"])

def _clamp_and_renormalize(raw_weights):
    """
    Applique les bornes min/max par symbole (issues de WEIGHT_BOUNDS),
    puis renormalise pour que la somme = 1.0.
    Iterere jusqu'a convergence (max 10 passes).
    """
    weights = dict(raw_weights)

    for _ in range(10):
        total = sum(weights.values())
        if total > 0:
            weights = {s: v / total for s, v in weights.items()}

        clamped = False
        for s in weights:
            mn, mx = _get_bounds(s)
            if weights[s] < mn:
                weights[s] = mn; clamped = True
            elif weights[s] > mx:
                weights[s] = mx; clamped = True

        if not clamped:
            break

    total = sum(weights.values())
    if total > 0:
        weights = {s: v / total for s, v in weights.items()}

    return weights

# =============================================================================
# CALCUL DU MOMENTUM ABSOLU PAR ACTIF (v3.0)
# =============================================================================
def compute_momentum(paths):
    """
    Calcule le momentum absolu de chaque symbole avec sa periode individuelle.
    Periode par actif issue de MOMENTUM_PERIOD_BY_SYMBOL (calibree sur backtest v2).
    Retourne un dict {symbol: momentum_value}
    momentum_value = (close_J0 - close_J-N) / close_J-N  (positif=hausse, negatif=baisse)
    Les actifs sans donnees suffisantes recoivent momentum=None.
    """
    momentums = {}

    for s in SYMBOLS:
        # Periode individualisee par actif
        period = MOMENTUM_PERIOD_BY_SYMBOL.get(s, MOMENTUM_PERIOD)
        needed = period + 1

        try:
            rates = mt5.copy_rates_from_pos(s, mt5.TIMEFRAME_D1, 0, needed)
            if rates is None or len(rates) < needed:
                log(f"{s}: historique D1 insuffisant pour momentum "
                    f"({len(rates) if rates is not None else 0}/{needed} "
                    f"bougies, periode={period}j)",
                    paths, "WARN")
                momentums[s] = None
                continue

            close_now  = float(rates[-1]["close"])
            close_prev = float(rates[0]["close"])

            if close_prev <= 0:
                momentums[s] = None
                continue

            mom = (close_now - close_prev) / close_prev
            momentums[s] = mom

        except Exception as e:
            log(f"{s}: erreur calcul momentum ({period}j) — {e}", paths, "ERROR")
            momentums[s] = None

    return momentums


def compute_weights(paths):
    """
    Calcule les poids de chaque symbole selon MODE_DEFENSIF.
    v3.0 : ponderation hybride volatilite + momentum absolu.
    Retourne (weights, atrs, vols, momentums).
    """
    vols = {}
    atrs = {}

    for s in SYMBOLS:
        ret, atr = get_returns_and_atr(s, paths)
        if ret is not None and len(ret) > 1 and atr is not None:
            vols[s] = float(ret.std())
            atrs[s] = atr
        else:
            vols[s] = 0.01
            atrs[s] = 0.0
            log(f"{s}: volatilite par defaut (0.01)", paths, "WARN")

    # [v3.0] Calcul du momentum absolu pour tous les actifs
    momentums = compute_momentum(paths)

    # Normalisation des momentum absolus sur [0,1] pour la fusion
    mom_abs = {s: abs(momentums[s]) if momentums[s] is not None else 0.0
               for s in SYMBOLS}
    max_mom = max(mom_abs.values()) if max(mom_abs.values()) > 0 else 1.0
    mom_norm = {s: mom_abs[s] / max_mom for s in SYMBOLS}

    # Composante volatilite (normalisee sur [0,1])
    if MODE_DEFENSIF:
        vol_raw  = {s: 1.0 / v if v > 0 else 1.0 for s, v in vols.items()}
        alpha    = ALPHA_DEFENSIF
        mode_label = "DEFENSIF"
    else:
        vol_raw  = {s: v if v > 0 else 0.01 for s, v in vols.items()}
        alpha    = ALPHA_OFFENSIF
        mode_label = "OFFENSIF"

    max_vol = max(vol_raw.values()) if max(vol_raw.values()) > 0 else 1.0
    vol_norm = {s: vol_raw[s] / max_vol for s in SYMBOLS}

    # [v3.0] Fusion : poids_brut = alpha x vol_norm + (1-alpha) x mom_norm
    raw = {s: alpha * vol_norm[s] + (1.0 - alpha) * mom_norm[s]
           for s in SYMBOLS}

    weights = _clamp_and_renormalize(raw)

    total_raw = sum(raw.values())
    log(f"Mode : {mode_label} (alpha={alpha:.2f})", paths)
    log(f"Poids bruts : "
        f"{ {s: round(raw[s]/total_raw, 4) for s in SYMBOLS} }", paths)
    log(f"Momentum ({'/'.join(str(MOMENTUM_PERIOD_BY_SYMBOL.get(s,MOMENTUM_PERIOD))+'j' for s in SYMBOLS)}) : "
        f"{ {s: f'{momentums[s]*100:.1f}%' if momentums[s] is not None else 'N/A' for s in SYMBOLS} }",
        paths)

    return weights, atrs, vols, momentums

# =============================================================================
# CALCUL DES LOTS DYNAMIQUES
# =============================================================================
def compute_lot(symbol, capital, weight, risk_mult, risk_factor, atr, paths):
    try:
        info = mt5.symbol_info(symbol)
        if not info:
            log(f"{symbol}: symbol_info introuvable", paths, "ERROR")
            return 0.01

        vol_min   = info.volume_min
        vol_max   = info.volume_max
        vol_step  = info.volume_step if info.volume_step > 0 else 0.01
        tick_val  = info.trade_tick_value
        tick_size = info.trade_tick_size
        sl_mult   = SL_MULT.get(symbol, 2.0)

        risk_amount = capital * RISK_PCT * weight * risk_mult * risk_factor

        if tick_val > 0 and tick_size > 0 and atr > 0:
            lot_brut = risk_amount / (atr * sl_mult * tick_val / tick_size)
        elif atr > 0 and info.trade_contract_size > 0:
            price = info.bid if info.bid > 0 else info.ask
            if price > 0 and tick_size > 0:
                pip_val  = info.trade_contract_size * tick_size / price
                lot_brut = risk_amount / (atr * sl_mult * pip_val)                            if pip_val > 0 else vol_min
            else:
                lot_brut = vol_min
                log(f"{symbol}: prix nul, lot_min", paths, "WARN")
        else:
            lot_brut = vol_min
            log(f"{symbol}: ATR ou tick nul, lot_min", paths, "WARN")

        steps       = round(lot_brut / vol_step)
        lot_arrondi = round(steps * vol_step, 8)
        lot_final   = max(vol_min, min(vol_max, lot_arrondi))

        if lot_brut < vol_min:
            log(f"{symbol}: lot calcule ({lot_brut:.5f}) < vol_min "
                f"({vol_min}) -> vol_min", paths, "WARN")

        return lot_final

    except Exception as e:
        log(f"{symbol}: erreur calcul lot — {e}", paths, "ERROR")
        return 0.01

# =============================================================================
# GESTION DES COMPTEURS (anti-martingale + journaliers)
# =============================================================================
def _default_counters():
    base = {
        "daily_date":          "",
        "daily_start_balance": 0.0,
        # initial_balance supprime — desormais fourni par INITIAL_BALANCE (hardcode)
    }
    for s in SYMBOLS:
        base[s] = {"tp_streak":0, "sl_streak":0,
                   "sl_today":0,  "tp_today":0}
    return base

def load_counters(paths):
    try:
        if os.path.exists(paths["counters"]):
            with open(paths["counters"], "r", encoding="utf-8") as f:
                data = json.load(f)
            for s in SYMBOLS:
                if s not in data:
                    data[s] = _default_counters()[s]
            return data
    except Exception as e:
        log(f"Erreur counters : {e} — reinit", paths, "WARN")
    return _default_counters()

def save_counters(counters, paths):
    try:
        with open(paths["counters"], "w", encoding="utf-8") as f:
            json.dump(counters, f, indent=2, ensure_ascii=False)
    except Exception as e:
        print(f"[COUNTERS ERROR] {e}")

def reset_daily_counters_if_needed(counters, paths):
    today = datetime.utcnow().strftime("%Y-%m-%d")
    if counters.get("daily_date") == today:
        return counters

    log(f"Nouveau jour ({today}) — reset compteurs journaliers", paths)
    balance = mt5.account_info().balance if mt5.account_info() else 0.0

    for s in SYMBOLS:
        if s in counters:
            counters[s]["sl_today"] = 0
            counters[s]["tp_today"] = 0

    counters["daily_date"]          = today
    counters["daily_start_balance"] = balance

    # [v2.7] initial_balance supprime des counters — utiliser INITIAL_BALANCE

    save_counters(counters, paths)
    return counters

def get_risk_mult(symbol, counters):
    c   = counters.get(symbol, {})
    tps = c.get("tp_streak", 0)
    sls = c.get("sl_streak", 0)

    if tps >= TP_SERIES_LVL2:   mult = 4.0
    elif tps >= TP_SERIES_LVL1: mult = 2.0
    elif sls >= SL_SERIES_LVL2: mult = 0.25
    elif sls >= SL_SERIES_LVL1: mult = 0.5
    else:                        mult = 1.0

    return max(RISK_MULT_MIN, min(RISK_MULT_MAX, mult))

# =============================================================================
# DRAWDOWN ET RISK FACTOR
# =============================================================================
def compute_drawdowns(counters):
    info = mt5.account_info()
    if not info:
        return 0.0, 0.0

    balance     = info.balance
    daily_start = counters.get("daily_start_balance", balance)
    # [v2.7] Utilise INITIAL_BALANCE hardcode — fiable meme apres suppression counters.json
    initial     = INITIAL_BALANCE

    dd_daily = max(0.0, (daily_start - balance) / daily_start) \
               if daily_start > 0 else 0.0
    dd_total = max(0.0, (initial - balance) / initial) \
               if initial > 0 else 0.0

    return dd_daily, dd_total

def compute_risk_factor(dd_daily, dd_total):
    """
    Seuils adaptes au mode actif :
    - propfirm : max_daily=4.5%, max_total=9%
    - compte propre : max_daily=8%, max_total=20%
    Le mode est detecte via les valeurs hardcodees (coherence avec EA MT5).
    """
    if dd_total >= MAX_TOTAL_DD_PCT or dd_daily >= MAX_DAILY_LOSS_PCT:
        return 0.0
    if dd_total >= SOFT_DD_PCT or dd_daily >= SOFT_DD_PCT:
        return 0.5
    return 1.0

def compute_trade_allowed(risk_factor):
    if not ENABLE_TRADING:
        return 0
    return 0 if risk_factor == 0.0 else 1

# =============================================================================
# MONITORING DES EA (lecture des heartbeats depuis MT5_COMMON_PATH)
# Heartbeats ecrits par MQL5 avec separateur TAB et timestamp "YYYY.MM.DD HH:MM:SS"
# Bytes nuls possibles (NAS100, Nikkei) : filtres via replace(chr(0), "")
# =============================================================================
def check_ea_heartbeats(dd_daily, dd_total, paths):
    now      = datetime.utcnow()
    statuses = {}

    for symbol in SYMBOLS:
        hb_file = heartbeat_path(symbol)
        status  = {"trade_allowed": 1, "restart": 0}

        # [v2.9] Ne pas surveiller le heartbeat des symboles en VEILLE intentionnelle
        # Gold est en VEILLE sur compte propfirm (IsModeCompatible=false) — silence normal
        # Detecter via le champ ModeActive du heartbeat si disponible
        if not os.path.exists(hb_file):
            statuses[symbol] = status
            continue

        # Lecture rapide pour verifier ModeActive avant de tester l'age
        try:
            with open(hb_file, "r", encoding="utf-8", errors="ignore") as _fhb:
                _hb_content = _fhb.read().replace("\x00", "")
            import io as _io, csv as _csv_mod
            _hb_reader = _csv_mod.DictReader(_io.StringIO(_hb_content), delimiter="\t")
            _hb_row = None
            for _hb_row in _hb_reader:
                pass
            if _hb_row and _hb_row.get("ModeActive","").strip() == "VEILLE":
                statuses[symbol] = status  # VEILLE intentionnelle — pas d'alerte
                continue
        except Exception:
            pass  # En cas d'erreur lecture, on continue le check normal

        try:
            with open(hb_file, "r", encoding="utf-8",
                      errors="ignore") as f:
                content = f.read().replace("\x00", "")

            reader = csv.DictReader(io.StringIO(content), delimiter="\t")
            row    = None
            for row in reader:
                pass

            if row is None:
                statuses[symbol] = status
                continue

            # MQL5 ecrit "2026.04.18 01:29:52" — convertir en datetime
            ts_str  = row["TimestampUTC"].strip().replace(".", "-", 2)
            hb_ts   = datetime.strptime(ts_str, "%Y-%m-%d %H:%M:%S")
            age_min = (now - hb_ts).total_seconds() / 60.0

            if age_min > HEARTBEAT_DISABLE_MIN:
                status["trade_allowed"] = 0
                msg = (f"EA {symbol} silencieux {age_min:.0f}min — DESACTIVE")
                log(msg, paths, "ERROR")
                _write_monitoring(symbol, msg, dd_daily, dd_total, paths)

            elif age_min > HEARTBEAT_RESTART_MIN:
                status["restart"] = 1
                msg = (f"EA {symbol} silencieux {age_min:.0f}min — RESTART")
                log(msg, paths, "WARN")
                _write_monitoring(symbol, msg, dd_daily, dd_total, paths)

            elif age_min > HEARTBEAT_ALERT_MIN:
                msg = (f"EA {symbol} silencieux {age_min:.0f}min — ALERTE")
                log(msg, paths, "WARN")
                _write_monitoring(symbol, msg, dd_daily, dd_total, paths)

        except Exception as e:
            log(f"Erreur heartbeat {symbol} : {e}", paths, "WARN")

        statuses[symbol] = status

    return statuses

def _write_monitoring(symbol, alert, dd_daily, dd_total, paths):
    rotate_if_needed(paths["monitoring"])
    file_exists = os.path.exists(paths["monitoring"])
    now = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
    with open(paths["monitoring"], "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["TimestampUTC","Symbol","Alert",
                             "DailyDD_pct","TotalDD_pct"])
        writer.writerow([now, symbol, alert,
                         f"{dd_daily*100:.2f}", f"{dd_total*100:.2f}"])

# =============================================================================
# CALCUL DU RISQUE EUR PAR TRADE — [v3.2]
# =============================================================================
def compute_risk_eur():
    """
    Calcule le budget risque maximum en euros par trade.

    Formule : RISK_EUR = (INITIAL_BALANCE * MAX_DAILY_LOSS_PCT * SAFETY_FACTOR)
                         / NB_SL_MAX

    Ce montant est identique pour tous les symboles — c'est une limite
    en euros absolus independante de la ponderation.

    L'EA v2.8 compare cout_reel (SL_pts x tick_value / tick_size x lot_min)
    a cette valeur et bloque le trade si cout_reel > RISK_EUR.
    Cela protege automatiquement contre les actifs trop chers pour
    le capital courant (ex. Silver avec 1000 EUR).
    """
    risk_eur = (INITIAL_BALANCE * MAX_DAILY_LOSS_PCT * SAFETY_FACTOR) / NB_SL_MAX
    return round(risk_eur, 4)


# =============================================================================
# EXPORT CSV PRINCIPAL (vers MT5_COMMON_PATH)
# Colonnes lues par l'EA MT5 v2.8
# =============================================================================
def export_csv(weights, lots, trade_allowed_global, risk_factor,
               dd_daily, dd_total, market_open, counters,
               ea_statuses, paths, momentums=None):
    now  = datetime.utcnow().replace(tzinfo=timezone.utc).isoformat()
    rows = []

    for s in SYMBOLS:
        c         = counters.get(s, {})
        risk_mult = get_risk_mult(s, counters)
        ea_st     = ea_statuses.get(s, {})

        # [v3.0] Filtre momentum : si actif en range → TradeAllowed_Symbol=0
        ta_symbol = ea_st.get("trade_allowed", 1)
        if USE_MOMENTUM_FILTER and ta_symbol == 1 and momentums is not None:
            mom_val = momentums.get(s)
            seuil   = MOMENTUM_RANGE_THRESHOLD.get(s, 0.015)
            if mom_val is not None and abs(mom_val) < seuil:
                ta_symbol = 0
                log(f"{s}: momentum {mom_val*100:.2f}% < seuil {seuil*100:.2f}% "
                    f"→ TradeAllowed_Symbol=0 (range detecte)", paths, "WARN")

        rows.append({
            "AccountID":           ACCOUNT_ID,
            "Symbol":              s,
            "Weight":              round(weights.get(s, 1.0/len(SYMBOLS)), 6),
            "Lot":                 lots.get(s, 0.01),
            "TimestampUTC":        now,
            "MarketOpen":          1 if market_open else 0,
            "EnableTrading":       1 if ENABLE_TRADING else 0,
            "TradeAllowed_Global": trade_allowed_global,
            "TradeAllowed_Symbol": ta_symbol,
            "Restart":             ea_st.get("restart", 0),
            "RiskFactor":          risk_factor,
            "RiskMult":            risk_mult,
            "DailyDrawdown_pct":   round(dd_daily * 100, 3),
            "TotalDrawdown_pct":   round(dd_total * 100, 3),
            "SL_today":            c.get("sl_today", 0),
            "TP_today":            c.get("tp_today", 0),
            "TP_streak":           c.get("tp_streak", 0),
            "SL_streak":           c.get("sl_streak", 0),
            "RiskEUR":             compute_risk_eur(),   # [v3.2] Risque max EUR/trade
        })

    pd.DataFrame(rows).to_csv(paths["csv"], index=False)

# =============================================================================
# EXPORT PERFORMANCE (1x/heure — BASE_PATH)
# =============================================================================
_last_perf_export = [datetime.utcnow() - timedelta(hours=2)]

def export_performance(weights, lots, dd_daily, dd_total, counters, vols, paths):
    now = datetime.utcnow()
    if (now - _last_perf_export[0]).total_seconds() < 3600:
        return
    _last_perf_export[0] = now

    rotate_if_needed(paths["performance"])
    file_exists = os.path.exists(paths["performance"])
    ts = now.strftime("%Y-%m-%d %H:%M:%S UTC")
    mode_str = "DEFENSIF" if MODE_DEFENSIF else "OFFENSIF"

    with open(paths["performance"], "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["TimestampUTC","Mode","Symbol","Weight","Lot",
                             "Volatility","RiskMult","SL_today","TP_today",
                             "TP_streak","SL_streak","DailyDD_pct","TotalDD_pct"])
        for s in SYMBOLS:
            c = counters.get(s, {})
            writer.writerow([ts, mode_str, s,
                             round(weights.get(s,0), 6),
                             lots.get(s, 0.01),
                             round(vols.get(s, 0), 8),
                             get_risk_mult(s, counters),
                             c.get("sl_today",0), c.get("tp_today",0),
                             c.get("tp_streak",0), c.get("sl_streak",0),
                             round(dd_daily*100,3), round(dd_total*100,3)])

# =============================================================================
# EXPORT STATS HEBDO (vendredi 20h30-21h00 UTC — BASE_PATH)
# =============================================================================
_last_hebdo_key = [None]

def export_stats_hebdo(weights, lots, dd_daily, dd_total, counters, paths):
    now = datetime.utcnow()
    is_friday_window = (now.weekday() == 4 and now.hour == 20
                        and 30 <= now.minute <= 59)
    if not is_friday_window:
        return

    week_key = now.strftime("%Y-W%W")
    if _last_hebdo_key[0] == week_key:
        return
    _last_hebdo_key[0] = week_key

    rotate_if_needed(paths["stats_hebdo"])
    file_exists = os.path.exists(paths["stats_hebdo"])
    ts = now.strftime("%Y-%m-%d %H:%M:%S UTC")
    mode_str = "DEFENSIF" if MODE_DEFENSIF else "OFFENSIF"

    with open(paths["stats_hebdo"], "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        if not file_exists:
            writer.writerow(["TimestampUTC","Week","Mode","Symbol",
                             "Weight","Lot","RiskMult",
                             "SL_today","TP_today","TP_streak","SL_streak",
                             "DailyDD_pct","TotalDD_pct"])
        for s in SYMBOLS:
            c = counters.get(s, {})
            writer.writerow([ts, week_key, mode_str, s,
                             round(weights.get(s,0), 6),
                             lots.get(s, 0.01),
                             get_risk_mult(s, counters),
                             c.get("sl_today",0), c.get("tp_today",0),
                             c.get("tp_streak",0), c.get("sl_streak",0),
                             round(dd_daily*100,3), round(dd_total*100,3)])
    log(f"Stats hebdo exportees — semaine {week_key}", paths)


# =============================================================================
# ECRITURE FICHIER InitBalance — vers MT5_COMMON_PATH
# Ecrit a chaque cycle (60s) par Ponderation.py
# Lu par les EA MT5 dans OnInit() pour initialiser g_InitialBalance
# Protege contre suppression accidentelle : reecrit dans la minute suivante
# Format CSV : AccountID,InitialBalance,TimestampUTC
# =============================================================================
def write_init_balance(paths):
    """Ecrit InitBalance_XXXX.csv dans Common/Files a chaque cycle."""
    fname = os.path.join(MT5_COMMON_PATH,
                         f"InitBalance_{ACCOUNT_ID}.csv")
    try:
        now = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
        with open(fname, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(["AccountID", "InitialBalance", "TimestampUTC"])
            writer.writerow([ACCOUNT_ID, f"{INITIAL_BALANCE:.2f}", now])
    except Exception as e:
        log(f"Erreur ecriture InitBalance : {e}", paths, "WARN")

# =============================================================================
# BOUCLE PRINCIPALE
# =============================================================================
def main():
    os.makedirs(BASE_PATH, exist_ok=True)
    os.makedirs(MT5_COMMON_PATH, exist_ok=True)
    paths = build_paths()

    mode_str = "DEFENSIF" if MODE_DEFENSIF else "OFFENSIF"
    log(f"=== Demarrage Ponderation.py v3.0 — AccountID : {ACCOUNT_ID} ===", paths)
    log(f"ENABLE_TRADING={ENABLE_TRADING} | MODE_DEFENSIF={MODE_DEFENSIF} "
        f"({mode_str})", paths)
    log(f"Symboles propfirm    : {SYMBOLS_PROPFIRM}", paths)
    log(f"Symboles compte propre ajoutes : {SYMBOLS_COMPTE_PROPRE}", paths)
    log(f"Fichiers Python    : {BASE_PATH}", paths)
    log(f"Fichiers MT5 commun: {MT5_COMMON_PATH}", paths)

    if not connect_mt5(paths):
        log("Arret — connexion MT5 impossible.", paths, "ERROR")
        return

    activate_symbols(paths)

    counters = load_counters(paths)
    counters = reset_daily_counters_if_needed(counters, paths)

    log(f"Capital initial (hardcode) : {INITIAL_BALANCE:.2f}", paths)

    while True:
        try:
            if not ensure_connected(paths):
                log("MT5 perdu — retry dans 60s", paths, "ERROR")
                time.sleep(60)
                continue

            counters = reset_daily_counters_if_needed(counters, paths)

            market_open        = is_market_open()
            dd_daily, dd_total = compute_drawdowns(counters)
            risk_factor        = compute_risk_factor(dd_daily, dd_total)
            trade_allowed      = compute_trade_allowed(risk_factor)

            log(f"DD jour:{dd_daily*100:.2f}% | DD total:{dd_total*100:.2f}% | "
                f"RiskFactor:{risk_factor} | TradeAllowed:{trade_allowed} | "
                f"MarketOpen:{market_open} | Mode:{mode_str}", paths)

            if not market_open:
                log("Marche ferme — calcul en mode surveillance", paths)

            # Calcul poids selon MODE_DEFENSIF
            weights, atrs, vols, momentums = compute_weights(paths)

            # Calcul lots dynamiques
            info    = mt5.account_info()
            capital = info.balance if info else                       counters.get("initial_balance", 1000.0)
            lots = {}
            for s in SYMBOLS:
                risk_mult = get_risk_mult(s, counters)
                lots[s]   = compute_lot(s, capital, weights[s],
                                        risk_mult, risk_factor,
                                        atrs.get(s, 0.0), paths)

            log(f"Poids : { {s: round(w,4) for s,w in weights.items()} }", paths)
            log(f"Lots  : {lots}", paths)

            # Monitoring EA
            ea_statuses = check_ea_heartbeats(dd_daily, dd_total, paths)

            # Export CSV vers MT5
            export_csv(weights, lots, trade_allowed, risk_factor,
                       dd_daily, dd_total, market_open,
                       counters, ea_statuses, paths)

            # [v2.7] Ecriture InitBalance a chaque cycle
            write_init_balance(paths)

            # Exports periodiques
            export_performance(weights, lots, dd_daily, dd_total,
                               counters, vols, paths)
            export_stats_hebdo(weights, lots, dd_daily, dd_total,
                               counters, paths)

            save_counters(counters, paths)

        except KeyboardInterrupt:
            log("Arret manuel (Ctrl+C).", paths)
            break

        except Exception as e:
            log(f"Erreur boucle : {e}\n{traceback.format_exc()}",
                paths, "ERROR")

        delay = get_loop_delay(trade_allowed)
        time.sleep(delay)

    mt5.shutdown()
    log("MT5 deconnecte. Fin du script.", paths)

# =============================================================================
# POINT D'ENTREE
# =============================================================================
if __name__ == "__main__":
    main()
