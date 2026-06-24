# -*- coding: utf-8 -*-
"""
================================================================================
BACKTEST V6 — STRATÉGIE RANGE BREAKOUT 9H30 EST + FAIR VALUE GAP
================================================================================

DESCRIPTION
-----------
Stratégie de scalping sur l'ouverture du marché US (9h30 EST) :
  1. Range de la première bougie 5min (High Ask / Low Bid, mèches incluses)
  2. Breakout confirmé par un Fair Value Gap (FVG) sur bougies 1min
  3. Market order simulé à la fermeture de la 3e bougie du pattern FVG

RÉSULTATS EN POINTS
-------------------
Tous les P&L sont exprimés en POINTS, pas en devise.
Objectif : valider la stratégie sur données Dukascopy indépendamment
de tout broker. La conversion en devise se fera dans l'EA MT5 live.

ARCHITECTURE
------------
- Lecture unique et séquentielle du fichier CSV
- Traitement par BATCH ANNUEL :
    → écriture CSV après chaque année
    → libération mémoire complète
    → capital réinitialisé à INITIAL_CAPITAL chaque année
- Initialisation des indicateurs avec 30 jours d'historique avant chaque année
- Timestamps en microsecondes int64 (pandas 2.x datetime64[us])
- Dispatch par jour : np.diff vectorisé — zéro boucle Python sur les ticks
- Bougies 1min : np.reduceat vectorisé
- Simulation SL/TP : np.argmax sur array booléen

PARAMÈTRES
----------
Tous les paramètres sont documentés et justifiés en tête de section CONFIG.
Les paramètres issus de l'analyse exploratoire (script 1) sont marqués [ANALYSE].
Les paramètres à valider manuellement sont marqués [MANUEL].

CLOSE_EOD
---------
Si aucun tick ne se produit pendant EOD_GAP_MINUTES après le dernier tick
du trade, le trade est clôturé au dernier prix mid disponible.
  - result = 'CLOSE_EOD', close_eod = True
  - P&L calculé sur ce prix réel (pas zéro)
  - EXCLU du win rate, profit factor, durée moyenne
  - INCLUS dans le P&L global et le drawdown
  - NOTE POUR L'EA MT5 : utiliser la durée p95 des trades TP/SL comme
    MAX_TRADE_DURATION pour éviter les positions overnight.

INDICATEURS CONTEXTUELS
-----------------------
Double snapshot enregistré à chaque trade :
  - Snapshot N   : état du marché AVANT le signal (suffixe par défaut)
  - Snapshot N+2 : état du marché À L'ENTRÉE (suffixe _n2)
Permet d'analyser séparément les conditions pré-signal et les conditions
au moment de l'entrée.

VARIANTE RETEST
---------------
Supprimée en V6 — aucun trade retest détecté sur 15 ans (V5).
La logique de retest a été retirée du code.

POINTS DE VIGILANCE
-------------------
1. Timestamps en µs int64 — ne jamais mélanger avec .value (ns)
2. date.fromtimestamp() non utilisé — conversion via timedelta depuis epoch
3. DST US géré dynamiquement (2e dim. mars → 1er dim. novembre)
4. CLOSE_EOD exclu du win rate et de la durée moyenne
5. Indicateurs initialisés avec 30 jours d'historique avant chaque année
6. Clôture forcée au 31/12 de toute position ouverte en CLOSE_EOD
7. Lot minimum : si lot calculé < LOT_MIN et dépassement > LOT_TOLERANCE_RATIO
   → trade non pris (result = 'LOT_TOO_SMALL')
================================================================================
"""

import os
import sys
import gc
import time
import logging
import traceback
from datetime import date, datetime, timedelta
from collections import deque

import pandas as pd
import numpy as np

# ==============================================================================
# ► CONFIG — À MODIFIER SELON L'ACTIF
# ==============================================================================

# --- Identifiants du run ---
# RUN_ID      : label lisible — modifier pour chaque nouvelle exécution
#               Exemples : "v5_retest", "v5_no_retest", "v5_spread2"
# MAGIC_NUMBER : identifiant unique codé en dur — NE PAS GÉNÉRER AUTOMATIQUEMENT
#               Changer manuellement si on veut isoler un nouveau run.
#               Intégré dans TOUS les noms de fichiers (CSV + log).
#               Permet de retrouver exactement quels exports correspondent
#               à quelle exécution, même après plusieurs runs.
#               À reporter tel quel dans l'EA MT5 comme magic number des ordres
#               (permet de tracer les ordres passés par ce robot spécifiquement).
RUN_ID       = "v6_rr3"
MAGIC_NUMBER = "B2C3D4E5"   # ← Changer manuellement pour chaque nouveau run

# --- Fichier de données [MANUEL] ---
FILE_PATH  = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "NASDAQ (US100)"

# --- Paramètres issus de l'analyse exploratoire [ANALYSE] ---
# Valeurs validées le 2026-06-24 sur données NASDAQ 2012-2026 (28.77 Go)
# Source : analyse_NASDAQ_US100_resume.json
#
# MAX_SPREAD_POINTS : p95 spread ouverture (2.20 pts) + marge → 3.0 pts
#   Filtre les pics anormaux (max observé 20.8 pts) sans impacter les jours normaux
#   p99 global = 2.58 pts, max récent (2022-2026) = 2.81 pts
#
# MIN_RANGE_POINTS : valeur conservative (option B validée)
#   Range médian train 12.93 pts, test 52.78 pts (marché × 3.3 en prix)
#   5 pts laisse le backtest révéler empiriquement le seuil optimal
#   Analyser range_size_pts et range_atr_ratio dans les résultats
#
# EOD_GAP_MINUTES : coupure nocturne à ~20h GMT, durée ~105 min
#   Trades résolus bien avant — 60 min largement suffisant
#
# SESSION_END_HOUR : dernier tick à 23h GMT (confirmé Dukascopy + RaiseFX)
MAX_SPREAD_POINTS  = 3.0    # [ANALYSE] p95 ouverture × 1.2 — validé 2026-06-24
MIN_RANGE_POINTS   = 5.0    # [ANALYSE] Conservative — affiner post-backtest
EOD_GAP_MINUTES    = 60     # [ANALYSE] Coupure nocturne ~20h GMT, durée ~105 min
SESSION_END_HOUR   = 23     # [ANALYSE] Confirmé Dukascopy + RaiseFX

# --- Paramètres stratégie ---
RISK_REWARD        = 3.0    # Ratio TP/SL fixe — 3:1 en V6
# --- Filtre années (backtest partiel) ---
# Liste des années à traiter. Mettre [] pour traiter toutes les années.
# V6 : sélection des années les plus représentatives pour valider R:R 3:1
YEARS_FILTER  = [2012, 2018, 2020, 2022, 2023, 2024]

# --- Filtre jours de semaine ---
# Permet d'exclure certains jours selon l'analyse ou le symbole cible.
# Applicable aussi dans l'EA MT5 avec les mêmes paramètres.
TRADE_MONDAY    = True    # Lundi
TRADE_TUESDAY   = True    # Mardi
TRADE_WEDNESDAY = True    # Mercredi — à désactiver si confirmé négatif
TRADE_THURSDAY  = True    # Jeudi
TRADE_FRIDAY    = True    # Vendredi
TRADE_SATURDAY  = False   # Toujours False sur NASDAQ (marché fermé)
TRADE_SUNDAY    = False   # Toujours False sur NASDAQ (marché fermé)

# Tableau indexé par weekday() Python (0=lundi, 6=dimanche)
# Utilisé dans flush_day() pour filtrer les jours non souhaités
WEEKDAY_ALLOWED = [
    TRADE_MONDAY,    # 0
    TRADE_TUESDAY,   # 1
    TRADE_WEDNESDAY, # 2
    TRADE_THURSDAY,  # 3
    TRADE_FRIDAY,    # 4
    TRADE_SATURDAY,  # 5
    TRADE_SUNDAY,    # 6
]

# --- Paramètres capital ---
INITIAL_CAPITAL    = 10_000.0  # Unités abstraites, réinitialisé chaque année
RISK_PERCENT       = 1.0       # Risque par trade en % du capital
LOT_MIN            = 0.01      # Volume minimal [MANUEL — vérifier chez le broker]
LOT_TOLERANCE_RATIO = 1.5      # Ratio max toléré si lot calculé < LOT_MIN

# --- Indicateurs ---
EMA_PERIODS   = [20, 50, 100, 200]
RSI_PERIODS   = [7, 14, 21]
ATR_PERIODS   = [7, 14, 21]
BB_PERIOD     = 20              # Bollinger Bands
BB_STD        = 2.0
AROON_PERIODS = [14, 25]
SLOPE_N       = 5               # Fenêtre pente EMA
LOOKBACK_DAYS = 30              # Jours d'historique pour init indicateurs

# --- Performance ---
CHUNK_SIZE    = 500_000
OUTPUT_DIR    = os.path.dirname(os.path.abspath(__file__))

# ==============================================================================
# LOGGING
# ==============================================================================

LOG_PATH = os.path.join(OUTPUT_DIR,
    f"backtest_{ASSET_NAME.split()[0]}_{RUN_ID}_{MAGIC_NUMBER}.log")

logging.basicConfig(
    level    = logging.INFO,
    format   = "%(asctime)s  %(levelname)-8s  %(message)s",
    datefmt  = "%Y-%m-%d %H:%M:%S",
    handlers = [
        logging.FileHandler(LOG_PATH, encoding='utf-8', mode='w'),
        logging.StreamHandler(sys.stdout),
    ]
)
log = logging.getLogger()

def pause():
    print("\n" + "="*70)
    print(" Appuyez sur Entrée pour fermer...")
    print("="*70)
    try: input()
    except Exception: pass

sys.excepthook = lambda t, v, tb: (
    log.error("ERREUR FATALE:\n" +
              "".join(__import__('traceback').format_exception(t, v, tb))),
    pause()
)

# ==============================================================================
# CONSTANTES TEMPORELLES (microsecondes — pandas 2.x datetime64[us])
# ==============================================================================

US_PER_MIN = 60_000_000
US_PER_DAY = 86_400_000_000

# ==============================================================================
# GESTION DU TEMPS ET DST US
# ==============================================================================

_dst_cache   = {}
_range_cache = {}

def is_us_dst(d: date) -> bool:
    """True si la date est en heure d'été US (EDT = UTC-4)."""
    if d in _dst_cache: return _dst_cache[d]
    y  = d.year
    ds = date(y, 3, 1 + (6 - date(y, 3, 1).weekday()) % 7 + 7)
    de = date(y, 11, 1 + (6 - date(y, 11, 1).weekday()) % 7)
    _dst_cache[d] = ds <= d < de
    return _dst_cache[d]

def range_start_us(d: date) -> int:
    """Timestamp µs du début de la bougie de range (9h30 EST ou EDT)."""
    if d in _range_cache: return _range_cache[d]
    hour = 13 if is_us_dst(d) else 14   # EDT=UTC-4→13h30 GMT | EST=UTC-5→14h30 GMT
    val  = int(pd.Timestamp(datetime(d.year, d.month, d.day, hour, 30)).value // 1000)
    _range_cache[d] = val
    return val

def day_id_to_date(day_id: int) -> date:
    """Conversion robuste day_id → date (indépendante du fuseau local)."""
    return date(1970, 1, 1) + timedelta(days=int(day_id))

# ==============================================================================
# PARSE TIMESTAMP VECTORISÉ
# Format fixe : DD.MM.YYYY HH:MM:SS.mmm (23 caractères)
# ==============================================================================

def parse_timestamps_vectorized(ts_col: pd.Series) -> np.ndarray:
    s    = ts_col
    year = s.str[6:10].astype(np.int32).values
    mon  = s.str[3:5 ].astype(np.int32).values
    day  = s.str[0:2 ].astype(np.int32).values
    hour = s.str[11:13].astype(np.int32).values
    minu = s.str[14:16].astype(np.int32).values
    sec  = s.str[17:19].astype(np.int32).values
    ms   = s.str[20:23].astype(np.int32).values
    base = pd.to_datetime({
        'year': year, 'month': mon, 'day': day,
        'hour': hour, 'minute': minu, 'second': sec
    }).values.astype('datetime64[us]').astype(np.int64)
    return base + ms.astype(np.int64) * 1000

# ==============================================================================
# INDICATEURS TECHNIQUES — BUFFERS INCRÉMENTAUX
# ==============================================================================

class EMABuffer:
    """EMA incrémentale (ewm span)."""
    def __init__(self, period):
        self.a = 2.0 / (period + 1)
        self.v = None; self.n = 0; self.p = period
    def update(self, x):
        self.v = x if self.v is None else self.a*x + (1-self.a)*self.v
        self.n += 1
    def get(self): return self.v if self.n >= self.p else None

class RSIBuffer:
    """RSI de Wilder."""
    def __init__(self, period):
        self.a = 1./period; self.ag = self.al = None
        self.prev = None; self.n = 0; self.p = period
    def update(self, x):
        if self.prev is None: self.prev = x; return
        d = x - self.prev; self.prev = x
        g = max(d, 0.); l = max(-d, 0.)
        if self.ag is None: self.ag = g; self.al = l
        else:
            self.ag = self.a*g + (1-self.a)*self.ag
            self.al = self.a*l + (1-self.a)*self.al
        self.n += 1
    def get(self):
        if self.ag is None or self.n < self.p: return None
        return 100. if self.al == 0 else 100. - 100./(1. + self.ag/self.al)

class ATRBuffer:
    """ATR incrémental."""
    def __init__(self, period):
        self.a = 1./period; self.atr = None
        self.pc = None; self.n = 0; self.p = period
    def update(self, h, l, c):
        tr = (h-l) if self.pc is None else max(h-l, abs(h-self.pc), abs(l-self.pc))
        self.pc = c
        self.atr = tr if self.atr is None else self.a*tr + (1-self.a)*self.atr
        self.n += 1
    def get(self): return self.atr if self.n >= self.p else None

class BollingerBuffer:
    """Bandes de Bollinger incrémentales."""
    def __init__(self, period=20, n_std=2.0):
        self.period = period; self.n_std = n_std
        self.buf    = deque(maxlen=period)
    def update(self, x): self.buf.append(x)
    def get(self):
        if len(self.buf) < self.period: return None, None, None, None
        arr  = np.array(self.buf)
        mid  = arr.mean()
        std  = arr.std(ddof=1)
        upper = mid + self.n_std * std
        lower = mid - self.n_std * std
        width = upper - lower
        return mid, upper, lower, width
    def get_pct_b(self, price):
        """Position du prix dans la bande : 0=bande basse, 1=bande haute."""
        mid, upper, lower, width = self.get()
        if width is None or width == 0: return None
        return (price - lower) / width

class AroonBuffer:
    """Aroon Up/Down incrémental."""
    def __init__(self, period):
        self.period = period
        self.highs  = deque(maxlen=period+1)
        self.lows   = deque(maxlen=period+1)
    def update(self, h, l): self.highs.append(h); self.lows.append(l)
    def get(self):
        if len(self.highs) < self.period + 1: return None, None
        h_arr = np.array(self.highs)
        l_arr = np.array(self.lows)
        # Indice du plus haut/bas depuis la gauche (le plus récent = droite)
        bars_since_high = self.period - np.argmax(h_arr)
        bars_since_low  = self.period - np.argmin(l_arr)
        aroon_up   = (self.period - bars_since_high) / self.period * 100
        aroon_down = (self.period - bars_since_low)  / self.period * 100
        return aroon_up, aroon_down

class SlopeBuffer:
    """Pente linéaire normalisée sur N dernières valeurs (% par bougie)."""
    def __init__(self, n=5):
        self.n = n; self.buf = deque(maxlen=n); self.x = np.arange(n, dtype=float)
    def update(self, v): self.buf.append(v)
    def get(self):
        if len(self.buf) < self.n: return None
        y = np.array(self.buf); m = y.mean()
        return 0. if m == 0 else np.polyfit(self.x, y, 1)[0] / m * 100

class IndicatorState:
    """Tous les buffers d'indicateurs en un objet."""
    def __init__(self):
        self.emas    = {p: EMABuffer(p)                   for p in EMA_PERIODS}
        self.slopes  = {p: SlopeBuffer(SLOPE_N)           for p in EMA_PERIODS}
        self.rsis    = {p: RSIBuffer(p)                   for p in RSI_PERIODS}
        self.atrs    = {p: ATRBuffer(p)                   for p in ATR_PERIODS}
        self.bb      = BollingerBuffer(BB_PERIOD, BB_STD)
        self.aroons  = {p: AroonBuffer(p)                 for p in AROON_PERIODS}

    def update(self, o, h, l, c):
        """Met à jour tous les indicateurs avec une nouvelle bougie 1min."""
        for p in EMA_PERIODS:
            self.emas[p].update(c)
            v = self.emas[p].get()
            if v is not None: self.slopes[p].update(v)
        for p in RSI_PERIODS:  self.rsis[p].update(c)
        for p in ATR_PERIODS:  self.atrs[p].update(h, l, c)
        self.bb.update(c)
        for p in AROON_PERIODS: self.aroons[p].update(h, l)

    def snapshot(self, price: float, direction: str, suffix: str = '') -> dict:
        """
        Retourne un instantané de tous les indicateurs.
        suffix : '' pour snapshot N, '_n2' pour snapshot N+2.
        """
        s = suffix
        snap = {}

        # EMAs + pentes + position prix
        for p in EMA_PERIODS:
            v = self.emas[p].get()
            snap[f'ema{p}{s}']              = v
            snap[f'slope_ema{p}{s}']        = self.slopes[p].get()
            snap[f'price_above_ema{p}{s}']  = (int(price > v) if v is not None else None)

        # Direction vs EMA200
        e200 = self.emas[200].get()
        snap[f'trade_with_ema200{s}'] = None
        if e200 is not None:
            snap[f'trade_with_ema200{s}'] = int(
                (direction == 'BUY'  and price > e200) or
                (direction == 'SELL' and price < e200)
            )

        # RSIs
        for p in RSI_PERIODS:
            snap[f'rsi{p}{s}'] = self.rsis[p].get()

        # ATRs
        for p in ATR_PERIODS:
            atr_val = self.atrs[p].get()
            snap[f'atr{p}{s}'] = atr_val

        # Ratios ATR (relatifs au ATR14 par défaut)
        atr14 = self.atrs[14].get()
        snap[f'atr14{s}'] = atr14   # doublon explicite pour compatibilité

        # Bollinger Bands
        bb_mid, bb_upper, bb_lower, bb_width = self.bb.get()
        snap[f'bb_width{s}']  = round(bb_width, 4) if bb_width  is not None else None
        snap[f'bb_pct_b{s}']  = round(self.bb.get_pct_b(price), 4) \
                                 if bb_width is not None else None
        snap[f'bb_upper{s}']  = round(bb_upper, 4) if bb_upper  is not None else None
        snap[f'bb_lower{s}']  = round(bb_lower, 4) if bb_lower  is not None else None

        # Aroon
        for p in AROON_PERIODS:
            up, down = self.aroons[p].get()
            snap[f'aroon{p}_up{s}']   = round(up,   2) if up   is not None else None
            snap[f'aroon{p}_down{s}'] = round(down, 2) if down is not None else None

        return snap

# ==============================================================================
# CONSTRUCTION BOUGIES 1MIN VECTORISÉE
# ==============================================================================

def build_1min_candles_us(ts_us, ask, bid):
    """
    Bougies OHLC 1min depuis arrays numpy µs int64.
    Retourne (ohlc_ask, ohlc_bid) comme DataFrames indexés Timestamp.
    """
    if len(ts_us) == 0:
        empty = pd.DataFrame(columns=['open','high','low','close'])
        return empty, empty
    minutes_us = (ts_us // US_PER_MIN) * US_PER_MIN
    changes    = np.where(np.diff(minutes_us) != 0)[0] + 1
    starts     = np.concatenate([[0], changes])
    ends       = np.concatenate([changes, [len(ts_us)]])
    idx        = pd.to_datetime(minutes_us[starts], unit='us')
    def mk(arr):
        return pd.DataFrame({
            'open' : arr[starts],
            'high' : np.maximum.reduceat(arr, starts),
            'low'  : np.minimum.reduceat(arr, starts),
            'close': arr[ends - 1]
        }, index=idx)
    return mk(ask), mk(bid)

# ==============================================================================
# SIMULATION SL/TP VECTORISÉE
# ==============================================================================

def simulate_trade_ticks(ask, bid, ts_us, direction, sl, tp, eod_gap_us):
    """
    Simule l'exécution d'un trade tick par tick.
    Détecte TP, SL, ou clôture sur gap de ticks (CLOSE_EOD).

    eod_gap_us : gap maximum entre ticks en µs avant CLOSE_EOD
    Retourne (result, exit_price_pts, exit_ts_us, last_tick_us)
    """
    if direction == 'BUY':
        tp_hit = bid >= tp; sl_hit = bid <= sl
    else:
        tp_hit = ask <= tp; sl_hit = ask >= sl

    # Détection gap EOD entre ticks consécutifs
    if len(ts_us) > 1:
        gaps    = np.diff(ts_us)
        gap_idx = np.where(gaps > eod_gap_us)[0]
    else:
        gap_idx = np.array([], dtype=np.int64)

    hit = tp_hit | sl_hit
    first_hit_idx = int(np.argmax(hit)) if hit.any() else len(hit)
    first_gap_idx = int(gap_idx[0]) if len(gap_idx) > 0 else len(ts_us)

    if not hit.any() and len(gap_idx) == 0:
        # Fin des ticks sans résolution
        mid_last = float((ask[-1] + bid[-1]) / 2)
        return 'CLOSE_EOD', mid_last, int(ts_us[-1]), int(ts_us[-1])

    if first_gap_idx < first_hit_idx:
        # Gap EOD avant SL/TP
        mid_last = float((ask[first_gap_idx] + bid[first_gap_idx]) / 2)
        return 'CLOSE_EOD', mid_last, int(ts_us[first_gap_idx]), int(ts_us[first_gap_idx])

    # SL ou TP
    i      = first_hit_idx
    result = 'TP' if tp_hit[i] else 'SL'
    exit_p = tp if result == 'TP' else sl
    return result, float(exit_p), int(ts_us[i]), int(ts_us[-1])

# ==============================================================================
# CALCUL DU LOT (en unités abstraites)
# ==============================================================================

def calc_lot(capital, risk_pct, sl_pts, lot_min, tolerance_ratio):
    """
    Calcule la taille du lot en unités abstraites.
    En backtest Python, le lot est exprimé en unités de risque, pas en lots réels.
    Le P&L sera exprimé en points × lot_size.

    Retourne (lot_size, status) où status est 'OK' ou 'LOT_TOO_SMALL'.
    """
    if sl_pts <= 0: return 0., 'INVALID_SL'
    risk_units = capital * risk_pct / 100.
    lot        = risk_units / sl_pts
    if lot < lot_min:
        # Vérifier si on peut arrondir au minimum tolérable
        risk_at_min = lot_min * sl_pts
        overrun     = risk_at_min / risk_units
        if overrun <= tolerance_ratio:
            return lot_min, 'OK_ROUNDED'
        return lot_min, 'LOT_TOO_SMALL'
    return lot, 'OK'

# ==============================================================================
# TRAITEMENT D'UN JOUR
# ==============================================================================

def process_day(ts_us, ask, bid, indic: IndicatorState,
                capital: float, d: date) -> dict:
    """
    Traite un jour de données tick.
    Retourne un dict décrivant le trade ou le statut de la journée.
    """
    rs_us  = range_start_us(d)
    re_us  = rs_us + 5  * US_PER_MIN
    end_us = rs_us + 90 * US_PER_MIN
    eod_gap_us = EOD_GAP_MINUTES * US_PER_MIN

    # ── Range 5min ──────────────────────────────────────────────────────
    mask_r = (ts_us >= rs_us) & (ts_us < re_us)
    if not mask_r.any():
        return {'date': d, 'result': 'NO_DATA', 'capital_after': capital}

    range_h = float(ask[mask_r].max())
    range_l = float(bid[mask_r].min())
    if range_h - range_l < MIN_RANGE_POINTS:
        return {'date': d, 'result': 'FILTERED_RANGE', 'capital_after': capital}

    # ── Bougies 1min ────────────────────────────────────────────────────
    oa, ob  = build_1min_candles_us(ts_us, ask, bid)
    oa_idx  = oa.index.asi8   # µs int64
    ob_idx  = ob.index.asi8

    # Mise à jour indicateurs sur bougies AVANT la fenêtre de recherche
    mask_pre = oa_idx < re_us
    ca_pre   = oa.values[mask_pre]
    cb_pre   = ob.values[mask_pre[:len(ob)]]
    for i in range(min(len(ca_pre), len(cb_pre))):
        indic.update(
            (ca_pre[i,0]+cb_pre[i,0])/2,
            (ca_pre[i,1]+cb_pre[i,1])/2,
            (ca_pre[i,2]+cb_pre[i,2])/2,
            (ca_pre[i,3]+cb_pre[i,3])/2
        )

    # ── Fenêtre de recherche FVG ─────────────────────────────────────────
    mask_win = (oa_idx >= re_us) & (oa_idx < end_us)
    idx_a    = oa_idx[mask_win]
    idx_b    = ob_idx[(ob_idx >= re_us) & (ob_idx < end_us)]
    common_us, ia, ib = np.intersect1d(idx_a, idx_b, return_indices=True)

    if len(common_us) < 3:
        return {'date': d, 'result': 'NO_SETUP', 'capital_after': capital}

    ca    = oa.values[mask_win][ia]
    cb    = ob.values[(ob_idx >= re_us) & (ob_idx < end_us)][ib]
    ts_c  = common_us

    snap_n = {}
    setup  = None

    for i in range(len(common_us) - 2):
        # Mise à jour indicateurs sur bougie N
        indic.update(
            (ca[i,0]+cb[i,0])/2, (ca[i,1]+cb[i,1])/2,
            (ca[i,2]+cb[i,2])/2, (ca[i,3]+cb[i,3])/2
        )

        n_h_ask  = ca[i,  1];  n_l_bid  = cb[i,  2]
        n1_h_ask = ca[i+1,1];  n1_l_bid = cb[i+1,2]
        n1_c_ask = ca[i+1,3];  n1_c_bid = cb[i+1,3]
        n2_h_ask = ca[i+2,1];  n2_l_bid = cb[i+2,2]
        n2_c_ask = ca[i+2,3];  n2_c_bid = cb[i+2,3]
        n2_ts    = int(ts_c[i+2])

        spread = float(n2_c_ask - n2_c_bid)
        if spread > MAX_SPREAD_POINTS: continue

        # FVG BUY : N+1 ferme au-dessus du range, gap entre N et N+2
        if n1_c_ask > range_h and n1_h_ask > range_h and n2_l_bid > n_h_ask:
            sl_pts = float(n2_c_ask - n1_l_bid)
            if sl_pts > 0:
                snap_n = indic.snapshot(float(n2_c_ask), 'BUY', '')
                setup  = ('BUY', float(n2_c_ask), float(n1_l_bid),
                          spread, n2_ts, float(n2_l_bid - n_h_ask))
                break

        # FVG SELL : N+1 ferme en-dessous du range, gap entre N et N+2
        if n1_c_bid < range_l and n1_l_bid < range_l and n2_h_ask < n_l_bid:
            sl_pts = float(n1_h_ask - n2_c_bid)
            if sl_pts > 0:
                snap_n = indic.snapshot(float(n2_c_bid), 'SELL', '')
                setup  = ('SELL', float(n2_c_bid), float(n1_h_ask),
                          spread, n2_ts, float(n_l_bid - n2_h_ask))
                break

    if setup is None:
        return {'date': d, 'result': 'NO_SETUP', 'capital_after': capital}

    direction, entry, sl_price, spread_entry, n2_ts, fvg_size = setup
    sl_pts   = abs(entry - sl_price)
    tp_price = entry + RISK_REWARD*sl_pts if direction == 'BUY' else entry - RISK_REWARD*sl_pts

    # Mise à jour indicateurs sur bougie N+1 et N+2 pour snapshot N+2
    for j in range(-1, 1):   # i+1 et i+2 (i est la dernière valeur du for ci-dessus)
        idx_j = i + 1 + j
        if 0 <= idx_j < len(ca):
            indic.update(
                (ca[idx_j,0]+cb[idx_j,0])/2,
                (ca[idx_j,1]+cb[idx_j,1])/2,
                (ca[idx_j,2]+cb[idx_j,2])/2,
                (ca[idx_j,3]+cb[idx_j,3])/2
            )
    snap_n2 = indic.snapshot(entry, direction, '_n2')

    # ── Calcul lot ───────────────────────────────────────────────────────
    lot, lot_status = calc_lot(capital, RISK_PERCENT, sl_pts,
                               LOT_MIN, LOT_TOLERANCE_RATIO)
    if lot_status == 'LOT_TOO_SMALL':
        return {'date': d, 'result': 'LOT_TOO_SMALL', 'capital_after': capital}

    # ── Simulation SL/TP vectorisée ──────────────────────────────────────
    mask_after = ts_us > n2_ts
    if not mask_after.any():
        return {'date': d, 'result': 'NO_SETUP', 'capital_after': capital}

    result, exit_price, exit_us, last_us = simulate_trade_ticks(
        ask[mask_after], bid[mask_after], ts_us[mask_after],
        direction, sl_price, tp_price, eod_gap_us
    )

    # ── P&L en points ────────────────────────────────────────────────────
    pnl_pts = (exit_price - entry) if direction == 'BUY' else (entry - exit_price)
    pnl_pct = pnl_pts / sl_pts * RISK_PERCENT   # % du capital (±1% si SL/TP exact)
    capital_new = capital * (1 + pnl_pct / 100.)

    duration = (exit_us - n2_ts) / (US_PER_MIN * 1.) if exit_us else None
    range_sz = range_h - range_l
    atr14    = snap_n.get('atr14')

    record = {
        # Identification
        'date'            : d,
        'year'            : d.year,
        'month'           : d.month,
        'day_of_week'     : d.strftime('%A'),
        'day_of_week_num' : d.weekday() + 1,
        'week_of_month'   : (d.day - 1) // 7 + 1,
        # Trade
        'direction'       : direction,
        'entry_time_gmt'  : pd.Timestamp(n2_ts + US_PER_MIN, unit='us'),  # fermeture N+2
        'exit_time_gmt'   : pd.Timestamp(exit_us, unit='us') if exit_us else None,
        'duration_min'    : round(duration, 2) if duration else None,
        # Prix
        'entry_price'     : round(entry, 4),
        'sl_price'        : round(sl_price, 4),
        'tp_price'        : round(tp_price, 4),
        'exit_price'      : round(exit_price, 4),
        # Performance
        'sl_distance_pts' : round(sl_pts, 4),
        'pnl_pts'         : round(pnl_pts, 4),
        'pnl_pct'         : round(pnl_pct, 4),
        'lot_size'        : round(lot, 4),
        'lot_status'      : lot_status,
        'capital_before'  : round(capital, 4),
        'capital_after'   : round(capital_new, 4),
        # Résultat
        'result'          : result,
        'close_eod'       : result == 'CLOSE_EOD',
        # Contexte setup
        'range_size_pts'  : round(range_sz, 4),
        'fvg_size_pts'    : round(fvg_size, 4),
        'spread_entry'    : round(spread_entry, 4),
        'fvg_atr_ratio'   : round(fvg_size / atr14, 4) if atr14 else None,
        'range_atr_ratio' : round(range_sz / atr14, 4) if atr14 else None,
        # Snapshots indicateurs
        **snap_n,    # État avant signal (suffixe '')
        **snap_n2,   # État à l'entrée   (suffixe '_n2')
    }
    return record

# ==============================================================================
# MÉTRIQUES PAR PÉRIODE
# ==============================================================================

def compute_metrics(trades: list, label: str, initial_capital: float) -> dict:
    """Calcule les métriques de performance sur une liste de trades."""
    all_t  = [t for t in trades if t.get('result') in ('TP', 'SL', 'CLOSE_EOD')]
    valid  = [t for t in trades if t.get('result') in ('TP', 'SL')]  # Hors CLOSE_EOD
    n_all  = len(all_t)
    n      = len(valid)

    if n_all == 0:
        return {'label': label, 'n_trades': 0}

    n_tp     = sum(1 for t in valid if t['result'] == 'TP')
    n_sl     = sum(1 for t in valid if t['result'] == 'SL')
    n_eod    = sum(1 for t in all_t if t['result'] == 'CLOSE_EOD')

    # P&L en points (tous trades)
    pnls_pts = [t['pnl_pts'] for t in all_t]
    gp_pts   = sum(p for p in pnls_pts if p > 0)
    gl_pts   = sum(abs(p) for p in pnls_pts if p < 0)
    net_pts  = sum(pnls_pts)

    # Win rate et métriques sur TP/SL uniquement
    wr   = n_tp / n * 100 if n > 0 else 0
    pf   = gp_pts / gl_pts if gl_pts > 0 else float('inf')
    avg_tp_pts = np.mean([t['pnl_pts'] for t in valid if t['result']=='TP']) if n_tp else 0
    avg_sl_pts = np.mean([t['pnl_pts'] for t in valid if t['result']=='SL']) if n_sl else 0
    exp_pts    = (wr/100)*avg_tp_pts + (1-wr/100)*avg_sl_pts if n > 0 else 0

    # Drawdown sur courbe de capital (tous trades)
    caps = [initial_capital] + [t['capital_after'] for t in all_t]
    peak = caps[0]; maxdd = 0.
    for c in caps:
        peak  = max(peak, c)
        maxdd = max(maxdd, (peak - c) / peak * 100)

    # Séries de SL consécutifs
    streak = max_streak = 0
    for t in valid:
        streak = streak + 1 if t['result'] == 'SL' else 0
        max_streak = max(max_streak, streak)

    # Durée (hors CLOSE_EOD)
    dur_tp = [t['duration_min'] for t in valid
              if t['result'] == 'TP' and t.get('duration_min')]
    dur_sl = [t['duration_min'] for t in valid
              if t['result'] == 'SL' and t.get('duration_min')]

    # Sharpe et Calmar (rendements journaliers en %)
    # Jours avec trade : pnl_pct du trade. Jours sans trade : 0.
    daily_pnl = {t['date']: t['pnl_pct'] for t in all_t}
    daily_returns = list(daily_pnl.values())
    sharpe = calmar = None
    if len(daily_returns) > 1:
        mu    = np.mean(daily_returns)
        sigma = np.std(daily_returns, ddof=1)
        if sigma > 0:
            sharpe = round(mu / sigma * np.sqrt(252), 3)
        annual_return = sum(daily_returns)
        if maxdd > 0:
            calmar = round(annual_return / maxdd, 3)

    return {
        'label'               : label,
        'n_trades'            : n_all,
        'n_tp'                : n_tp,
        'n_sl'                : n_sl,
        'n_close_eod'         : n_eod,
        'win_rate_pct'        : round(wr, 2),
        'net_pts'             : round(net_pts, 2),
        'gross_profit_pts'    : round(gp_pts, 2),
        'gross_loss_pts'      : round(gl_pts, 2),
        'profit_factor'       : round(pf, 3),
        'expectancy_pts'      : round(exp_pts, 2),
        'max_drawdown_pct'    : round(maxdd, 2),
        'max_consec_sl'       : max_streak,
        'sharpe_ratio'        : sharpe,
        'calmar_ratio'        : calmar,
        'avg_dur_tp_min'      : round(np.mean(dur_tp), 1) if dur_tp else None,
        'avg_dur_sl_min'      : round(np.mean(dur_sl), 1) if dur_sl else None,
        'dur_tp_p95_min'      : round(np.percentile(dur_tp, 95), 1) if len(dur_tp) > 1 else None,
        'dur_sl_p95_min'      : round(np.percentile(dur_sl, 95), 1) if len(dur_sl) > 1 else None,
    }

def print_metrics(m: dict):
    if m['n_trades'] == 0:
        log.info(f"    {m['label']} : aucun trade"); return
    log.info(f"\n  ── {m['label']} ──")
    log.info(f"  Trades        : {m['n_trades']}  "
             f"(TP:{m['n_tp']} SL:{m['n_sl']} EOD:{m['n_close_eod']})")
    log.info(f"  Win rate      : {m['win_rate_pct']}%  (hors CLOSE_EOD)")
    log.info(f"  P&L net       : {m['net_pts']:>10.2f} pts")
    log.info(f"  Profit Factor : {m['profit_factor']}")
    log.info(f"  Espérance     : {m['expectancy_pts']:.2f} pts/trade")
    log.info(f"  Drawdown max  : {m['max_drawdown_pct']}%")
    log.info(f"  Sharpe        : {m['sharpe_ratio']}")
    log.info(f"  Calmar        : {m['calmar_ratio']}")
    log.info(f"  SL consécutifs max : {m['max_consec_sl']}")
    if m['avg_dur_tp_min']:
        log.info(f"  Durée moy TP  : {m['avg_dur_tp_min']} min "
                 f"(p95: {m['dur_tp_p95_min']} min)")
    if m['avg_dur_sl_min']:
        log.info(f"  Durée moy SL  : {m['avg_dur_sl_min']} min "
                 f"(p95: {m['dur_sl_p95_min']} min)")

# ==============================================================================
# BARRE DE PROGRESSION
# ==============================================================================

class Progress:
    """
    Barre de progression par ANNÉE en cours.
    Affiche le mois courant et la progression dans l'année.
    """
    MONTHS = ['jan','fév','mar','avr','mai','jun',
              'jul','aoû','sep','oct','nov','déc']

    def __init__(self):
        self.t0       = time.time()
        self.last     = 0.
        self.year_t0  = {}    # heure de début de chaque année
        self.year_cur = None

    def update(self, cur_date, n_trades, speed_mbs):
        now = time.time()
        if now - self.last < 3.: return
        self.last = now
        if cur_date is None: return

        year  = cur_date.year
        month = cur_date.month

        # Enregistrer le début de l'année
        if year not in self.year_t0:
            self.year_t0[year] = now

        # Progression dans l'année (basée sur le mois)
        pct_year = (month - 1) / 12 * 100
        bl = 24; fi = int(bl * pct_year / 100)
        bar = '█' * fi + '░' * (bl - fi)

        # Vitesse globale
        el  = now - self.t0
        sp  = speed_mbs

        # ETA année en cours (extrapolation linéaire depuis le mois)
        if month > 1:
            elapsed_year = now - self.year_t0[year]
            eta_year_s   = elapsed_year / (month - 1) * (12 - month + 1)
            em, es = divmod(int(eta_year_s), 60)
            eta_str = f"ETA année {em}m{es:02d}s"
        else:
            eta_str = ""

        mon_str = self.MONTHS[month - 1]
        print(f"\r  {year} [{bar}] {mon_str}  {sp:4.1f} Mo/s  "
              f"{eta_str}  trades:{n_trades}     ",
              end='', flush=True)

    def done(self):
        el = time.time() - self.t0
        print(f"\r  {'─'*65}", flush=True)
        log.info(f"  ✓ Lecture terminée en {el/60:.1f} min")

# ==============================================================================
# ÉCRITURE DES FICHIERS D'UNE ANNÉE
# ==============================================================================

def write_year(year: int, trades: list, summary_rows: list,
               initial_capital: float):
    """Écrit les CSV d'une année et met à jour le fichier summary."""
    if not trades: return

    slug    = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','')
    prefix  = os.path.join(OUTPUT_DIR, f"backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_{year}")

    # CSV trades
    df_trades = pd.DataFrame(trades)
    df_trades.to_csv(f"{prefix}_trades.csv", index=False)

    # CSV mensuel
    monthly = []
    for mo in sorted({t['month'] for t in trades}):
        sub = [t for t in trades if t['month'] == mo]
        m   = compute_metrics(sub, f"{year}-{mo:02d}", initial_capital)
        m['year'] = year; m['month'] = mo
        monthly.append(m)
    pd.DataFrame(monthly).to_csv(f"{prefix}_monthly.csv", index=False)

    # Métriques annuelles
    m_year = compute_metrics(trades, str(year), initial_capital)
    m_year['year'] = year
    summary_rows.append(m_year)

    # Mise à jour summary global (écrit à chaque année)
    summary_path = os.path.join(
        OUTPUT_DIR, f"backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_summary.csv")
    pd.DataFrame(summary_rows).to_csv(summary_path, index=False)

    log.info(f"\n  ── Année {year} — fichiers écrits ──")
    log.info(f"     {prefix}_trades.csv  ({len(trades)} trades)")
    log.info(f"     {prefix}_monthly.csv")
    log.info(f"     {summary_path}  (mis à jour)")
    print_metrics(m_year)

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL
# ==============================================================================

def run_backtest():
    log.info("="*70)
    log.info(f" BACKTEST V6 — {ASSET_NAME}  (R:R {RISK_REWARD}:1)")
    log.info(f" Capital : {INITIAL_CAPITAL:,.0f} pts/an  |  "
             f"Risque : {RISK_PERCENT}%  |  R:R {RISK_REWARD}:1")
    log.info(f" Fichier : {FILE_PATH}")
    noms = ['Lun','Mar','Mer','Jeu','Ven','Sam','Dim']
    jours_actifs = [noms[i] for i, v in enumerate(WEEKDAY_ALLOWED) if v]
    log.info(f" Années filtrées : {YEARS_FILTER if YEARS_FILTER else 'toutes'}")
    log.info(f" Jours actifs    : {jours_actifs}")
    log.info(f" Filtres — Spread max : {MAX_SPREAD_POINTS} pts  |  "
             f"Range min : {MIN_RANGE_POINTS} pts")
    log.info(f" EOD gap : {EOD_GAP_MINUTES} min")
    log.info(f" Paramètres validés le 2026-06-24 sur données NASDAQ 2012-2026")
    log.info(f" RUN_ID      : {RUN_ID}")
    log.info(f" MAGIC_NUMBER: {MAGIC_NUMBER}")
    log.info("="*70)

    if not os.path.exists(FILE_PATH):
        log.error(f"Fichier introuvable : {FILE_PATH}")
        pause(); sys.exit(1)

    file_size = os.path.getsize(FILE_PATH)
    log.info(f"\n Taille : {file_size/1e9:.2f} Go  |  "
             f"Chunks : {CHUNK_SIZE:,} lignes\n")

    # ── État global ──────────────────────────────────────────────────────
    indic        = IndicatorState()
    capital      = INITIAL_CAPITAL
    cur_year     = None
    year_trades  = []
    summary_rows = []

    # Buffers journaliers
    cur_day_id   = -1
    day_ts       = []
    day_ask      = []
    day_bid      = []

    # Buffers lookback (30 derniers jours pour init indicateurs)
    lookback_buf = deque(maxlen=LOOKBACK_DAYS)

    # Compteurs
    cnt_no_data = cnt_filtered = cnt_no_setup = cnt_eod = cnt_too_small = 0

    prog       = Progress()
    bytes_read = 0
    speed_mbs  = 0.

    log.info(" Lecture unique — progression toutes les 3 secondes\n")

    def flush_day(day_id, ts_list, ask_list, bid_list):
        """Traite le buffer d'un jour complet."""
        nonlocal capital, cur_year, year_trades, indic
        nonlocal cnt_no_data, cnt_filtered, cnt_no_setup, cnt_eod, cnt_too_small

        if not ts_list or day_id < 0: return
        d = day_id_to_date(day_id)

        # Filtre jour de semaine (utilise WEEKDAY_ALLOWED — 7 booléens)
        if not WEEKDAY_ALLOWED[d.weekday()]: return

        ts_arr  = np.array(ts_list,  dtype=np.int64)
        ask_arr = np.array(ask_list, dtype=np.float32)
        bid_arr = np.array(bid_list, dtype=np.float32)

        # ── Passage à une nouvelle année ────────────────────────────────
        if cur_year is None:
            cur_year = d.year

        if d.year != cur_year:
            # Écrire l'année écoulée — seulement si elle est dans YEARS_FILTER
            if not YEARS_FILTER or cur_year in YEARS_FILTER:
                write_year(cur_year, year_trades, summary_rows, INITIAL_CAPITAL)

            # Réinitialiser pour la nouvelle année
            year_trades = []
            capital     = INITIAL_CAPITAL
            cur_year    = d.year

            # Réinitialiser les indicateurs et les ré-alimenter
            # avec le lookback des jours précédents (toujours, même années filtrées)
            indic = IndicatorState()
            for lb_ts, lb_ask, lb_bid in lookback_buf:
                lb_oa, lb_ob = build_1min_candles_us(lb_ts, lb_ask, lb_bid)
                ca = lb_oa.values; cb = lb_ob.values
                for i in range(min(len(ca), len(cb))):
                    indic.update(
                        (ca[i,0]+cb[i,0])/2, (ca[i,1]+cb[i,1])/2,
                        (ca[i,2]+cb[i,2])/2, (ca[i,3]+cb[i,3])/2
                    )
            log.info(f"\n  Nouvelle année : {d.year} — "
                     f"indicateurs réinitialisés sur {len(lookback_buf)} jours")

        # Stocker dans le lookback pour l'année suivante (toujours)
        lookback_buf.append((ts_arr.copy(), ask_arr.copy(), bid_arr.copy()))

        # ── Traitement du jour ───────────────────────────────────────────
        # Si l'année n'est pas dans YEARS_FILTER : mise à jour indicateurs
        # uniquement, pas de trade enregistré
        if YEARS_FILTER and d.year not in YEARS_FILTER:
            # Mettre à jour les indicateurs sans enregistrer de trades
            oa, ob = build_1min_candles_us(ts_arr, ask_arr, bid_arr)
            ca = oa.values; cb = ob.values
            for i in range(min(len(ca), len(cb))):
                indic.update(
                    (ca[i,0]+cb[i,0])/2, (ca[i,1]+cb[i,1])/2,
                    (ca[i,2]+cb[i,2])/2, (ca[i,3]+cb[i,3])/2
                )
            return

        # Clôture forcée au 31/12 : le flag est géré via le passage d'année
        res = process_day(ts_arr, ask_arr, bid_arr, indic, capital, d)
        r   = res.get('result', '')

        if   r == 'NO_DATA':        cnt_no_data   += 1
        elif r == 'FILTERED_RANGE': cnt_filtered  += 1
        elif r == 'NO_SETUP':       cnt_no_setup  += 1
        elif r == 'LOT_TOO_SMALL':  cnt_too_small += 1
        elif r in ('TP', 'SL', 'CLOSE_EOD'):
            capital = res['capital_after']
            if r == 'CLOSE_EOD': cnt_eod += 1
            year_trades.append(res)

    # ── Lecture par chunks ───────────────────────────────────────────────
    reader = pd.read_csv(
        FILE_PATH, header=None,
        names=['timestamp', 'ask', 'bid'],
        usecols=[0, 1, 2],
        chunksize=CHUNK_SIZE,
        dtype={'ask': 'float32', 'bid': 'float32'},
        engine='c'
    )

    # Taille moyenne réelle par ligne (calculée une fois sur le fichier)
    # Format : "DD.MM.YYYY HH:MM:SS.mmm,ask,bid\n" ≈ 33 octets en moyenne
    avg_bytes_per_line = file_size // max(1, file_size // 33)

    for chunk in reader:
        bytes_read += len(chunk) * 33
        elapsed_total = time.time() - prog.t0
        speed_mbs = bytes_read / elapsed_total / 1e6 if elapsed_total > 0 else 0.
        cur_date_disp = day_id_to_date(cur_day_id) if cur_day_id >= 0 else None
        prog.update(cur_date_disp, len(year_trades), speed_mbs)

        ts_us    = parse_timestamps_vectorized(chunk['timestamp'])
        ask_vals = chunk['ask'].values
        bid_vals = chunk['bid'].values

        # Dispatch vectorisé par jour
        day_ids    = ts_us // US_PER_DAY
        changes    = np.where(np.diff(day_ids) != 0)[0] + 1
        seg_starts = np.concatenate([[0], changes])
        seg_ends   = np.concatenate([changes, [len(ts_us)]])

        for s, e in zip(seg_starts, seg_ends):
            seg_day_id = int(day_ids[s])
            if seg_day_id != cur_day_id:
                flush_day(cur_day_id, day_ts, day_ask, day_bid)
                cur_day_id = seg_day_id
                day_ts     = []
                day_ask    = []
                day_bid    = []
            day_ts.extend(ts_us[s:e].tolist())
            day_ask.extend(ask_vals[s:e].tolist())
            day_bid.extend(bid_vals[s:e].tolist())

    # Dernier jour et dernière année
    flush_day(cur_day_id, day_ts, day_ask, day_bid)
    if cur_year and year_trades:
        if not YEARS_FILTER or cur_year in YEARS_FILTER:
            write_year(cur_year, year_trades, summary_rows, INITIAL_CAPITAL)

    prog.done()

    # ── Résumé global ────────────────────────────────────────────────────
    log.info(f"\n{'='*70}")
    log.info(f" RÉSULTATS GLOBAUX — {ASSET_NAME}")
    log.info(f"{'='*70}")

    all_trades = []
    for row in summary_rows:
        # Recharger les trades depuis les CSV pour les métriques globales
        # (ils ont été libérés de la mémoire après chaque année)
        year = row.get('year')
        if year:
            slug = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','')
            f    = os.path.join(OUTPUT_DIR, f"backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_{year}_trades.csv")
            if os.path.exists(f):
                df = pd.read_csv(f)
                all_trades.extend(df.to_dict('records'))

    if all_trades:
        m_global = compute_metrics(all_trades, "GLOBAL", INITIAL_CAPITAL)
        print_metrics(m_global)
    else:
        log.info("  Aucun trade — vérifiez les paramètres.")

    log.info(f"\n Compteurs :")
    log.info(f"   Sans données   : {cnt_no_data}")
    log.info(f"   Filtrés range  : {cnt_filtered}")
    log.info(f"   Sans setup     : {cnt_no_setup}")
    log.info(f"   Lot trop petit : {cnt_too_small}")
    log.info(f"   CLOSE_EOD      : {cnt_eod}")

    slug = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','')
    log.info(f"\n Fichiers générés dans : {OUTPUT_DIR}")
    log.info(f"   backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_XXXX_trades.csv")
    log.info(f"   backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_XXXX_monthly.csv")
    log.info(f"   backtest_{slug}_{RUN_ID}_{MAGIC_NUMBER}_summary.csv")
    log.info(f"   {LOG_PATH}")
    log.info(f"\n{'='*70}")
    log.info(f" Backtest terminé.")
    log.info(f"{'='*70}\n")


if __name__ == '__main__':
    try:
        run_backtest()
    except Exception:
        log.error("\n" + "="*70)
        log.error(" ERREUR INATTENDUE")
        log.error("="*70)
        log.error(traceback.format_exc())
    finally:
        pause()

