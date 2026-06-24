# -*- coding: utf-8 -*-
"""
================================================================================
RANGE BREAKOUT POLYVALENT — Moteur générique de backtest
================================================================================

DESCRIPTION
-----------
Moteur de backtest configurable pour toute stratégie de type "range breakout".
Conçu pour tester différentes configurations sur n'importe quel actif
disposant de données tick Tick Data Suite (GMT+0, NO-DST).

PHILOSOPHIE
-----------
- Un seul fichier, zéro modification de code pour changer de configuration
- Tous les paramètres sont dans la section CONFIG ci-dessous
- Résultats en POINTS (indépendant du broker et de la devise)
- Architecture identique à la V6 (batch annuel, lecture unique, vectorisé)

MODES DE SIGNAL
---------------
SIGNAL_MODE = "FVG"
    Pattern 3 bougies : N (référence), N+1 (displacement), N+2 (confirmation)
    FVG BUY  : N+1 ferme au-dessus du range ET Low(N+2) > High(N)
    FVG SELL : N+1 ferme en-dessous du range ET High(N+2) < Low(N)
    Entrée à la clôture de N+2.

SIGNAL_MODE = "BREAKOUT"
    Entrée dès qu'une bougie ferme au-delà du range.
    BUY  : bougie ferme au-dessus de range_high
    SELL : bougie ferme en-dessous de range_low
    SL   : opposé du range (range_low pour BUY, range_high pour SELL)
    Entrée à la clôture de la bougie de breakout.

GESTION DES HORAIRES
---------------------
Les heures sont saisies en heure LOCALE du marché (RANGE_TIMEZONE).
La conversion UTC est automatique via zoneinfo (Python 3.9+).
Compatible avec tout marché : US (America/New_York), EU (Europe/Berlin),
Londres (Europe/London), Tokyo (Asia/Tokyo), etc.
Le DST est géré automatiquement par zoneinfo — aucune logique manuelle.

En version EA MT5 : utiliser TimeGMT() comme référence et convertir
l'heure locale en GMT via l'offset broker (TimeGMT() - TimeCurrent()).

CLOSE_EOD / TIMEOUT
--------------------
CLOSE_EOD : aucun tick pendant EOD_GAP_MINUTES → clôture au dernier prix mid
TIMEOUT   : trade ouvert depuis MAX_TRADE_DURATION_MIN → clôture au prix mid
Les deux sont exclus du win rate et de la durée moyenne,
mais inclus dans le P&L global et le drawdown.

INDICATEURS CONTEXTUELS
-----------------------
Double snapshot à chaque trade (état à N et état à l'entrée) :
EMA 20/50/100/200 + pentes, RSI 7/14/21, ATR 7/14/21,
Bollinger 20, Aroon 14/25, ratios FVG/ATR et Range/ATR.

SORTIES
-------
CSV trades par année, CSV mensuel, CSV summary global,
fichier log — tous nommés avec RUN_ID et MAGIC_NUMBER.

POINTS DE VIGILANCE
-------------------
1. Timestamps en µs int64 (pandas 2.x datetime64[us]) — .asi8 = µs, .value = ns
2. date.fromtimestamp() non utilisé — conversion via date(1970,1,1) + timedelta
3. DST géré par zoneinfo — ne jamais recoder manuellement
4. CLOSE_EOD et TIMEOUT exclus du win rate et durée moyenne
5. Indicateurs réinitialisés avec LOOKBACK_DAYS jours avant chaque année filtrée
6. MAX_TRADE_DURATION_MIN = 0 → pas de limite de durée
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
from zoneinfo import ZoneInfo

import pandas as pd
import numpy as np

# ==============================================================================
# ► CONFIG — TOUT MODIFIER ICI, JAMAIS DANS LE CODE
# ==============================================================================

# --- Identifiants du run ---
# RUN_ID      : label lisible — décrire la configuration testée
# MAGIC_NUMBER : identifiant unique codé en dur — à changer pour chaque run
#               À reporter dans l'EA MT5 comme magic number des ordres.
RUN_ID       = "v1_nasdaq_fvg_rr3"
MAGIC_NUMBER = "C3D4E5F6"

# --- Fichier de données [MANUEL] ---
FILE_PATH  = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "NASDAQ (US100)"

# --- Horaires du range (heure LOCALE du marché) ---
# RANGE_TIMEZONE : timezone zoneinfo (voir https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
#   Exemples : "America/New_York" (US), "Europe/Berlin" (DAX), "Europe/London" (FTSE)
# RANGE_START_LOCAL / RANGE_END_LOCAL : (heure, minute) en heure locale
#   Le DST est géré automatiquement — indiquer l'heure "économique" du marché.
#   Exemples : ouverture US = (9, 30), ouverture DAX = (9, 0)
RANGE_TIMEZONE    = "America/New_York"
RANGE_START_LOCAL = (9, 30)    # 9h30 heure de New York → 13h30 ou 14h30 UTC
RANGE_END_LOCAL   = (9, 35)    # 9h35 heure de New York → 13h35 ou 14h35 UTC

# --- Signal ---
# SIGNAL_MODE          : "FVG" (pattern 3 bougies) ou "BREAKOUT" (clôture hors range)
# SIGNAL_TIMEFRAME_MIN : timeframe de détection en minutes (1, 5, 15, 30, 60...)
# SIGNAL_WINDOW_MIN    : fenêtre de recherche du signal après RANGE_END (minutes)
SIGNAL_MODE          = "FVG"
SIGNAL_TIMEFRAME_MIN = 1
SIGNAL_WINDOW_MIN    = 90

# --- Direction des trades ---
# "BOTH"      : Buy ET Sell (comportement par défaut)
# "BUY_ONLY"  : uniquement les setups haussiers
# "SELL_ONLY" : uniquement les setups baissiers
TRADE_DIRECTION = "BOTH"

# --- Paramètres stratégie ---
RISK_REWARD              = 3.0    # Ratio TP/SL fixe
MAX_TRADE_DURATION_MIN   = 240    # Durée max d'un trade en minutes (0 = pas de limite)

# --- Filtres de marché ---
MAX_SPREAD_POINTS  = 3.0     # Spread max toléré à l'entrée (points)
MIN_RANGE_POINTS   = 5.0     # Taille minimum du range (points)
EOD_GAP_MINUTES    = 60      # Gap ticks → CLOSE_EOD (minutes)

# --- Filtre années ---
# Liste des années à traiter. [] = toutes les années.
YEARS_FILTER = [2012, 2018, 2020, 2022, 2023, 2024]

# --- Filtre jours de semaine ---
TRADE_MONDAY    = True
TRADE_TUESDAY   = True
TRADE_WEDNESDAY = True
TRADE_THURSDAY  = True
TRADE_FRIDAY    = True
TRADE_SATURDAY  = False
TRADE_SUNDAY    = False
WEEKDAY_ALLOWED = [TRADE_MONDAY, TRADE_TUESDAY, TRADE_WEDNESDAY,
                   TRADE_THURSDAY, TRADE_FRIDAY, TRADE_SATURDAY, TRADE_SUNDAY]

# --- Capital et risque ---
INITIAL_CAPITAL     = 10_000.0
RISK_PERCENT        = 1.0
LOT_MIN             = 0.01
LOT_TOLERANCE_RATIO = 1.5

# --- Indicateurs ---
EMA_PERIODS   = [20, 50, 100, 200]
RSI_PERIODS   = [7, 14, 21]
ATR_PERIODS   = [7, 14, 21]
BB_PERIOD     = 20
BB_STD        = 2.0
AROON_PERIODS = [14, 25]
SLOPE_N       = 5
LOOKBACK_DAYS = 30

# --- Performance ---
CHUNK_SIZE = 500_000
OUTPUT_DIR = os.path.dirname(os.path.abspath(__file__))

# ==============================================================================
# LOGGING
# ==============================================================================

LOG_PATH = os.path.join(OUTPUT_DIR,
    f"rng_{ASSET_NAME.split()[0]}_{RUN_ID}_{MAGIC_NUMBER}.log")

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
# GESTION DES HORAIRES VIA ZONEINFO
# ==============================================================================

_tz        = ZoneInfo(RANGE_TIMEZONE)
_tz_utc    = ZoneInfo("UTC")
_range_cache = {}

def range_bounds_us(d: date) -> tuple[int, int]:
    """
    Retourne (range_start_us, range_end_us) en microsecondes UTC
    pour un jour donné, en tenant compte du DST via zoneinfo.

    Exemple pour RANGE_TIMEZONE = "America/New_York" :
      RANGE_START_LOCAL = (9, 30) → 13:30 UTC en été, 14:30 UTC en hiver
    """
    if d in _range_cache:
        return _range_cache[d]

    # Construire le datetime local avec zoneinfo (DST automatique)
    dt_start_local = datetime(d.year, d.month, d.day,
                              RANGE_START_LOCAL[0], RANGE_START_LOCAL[1],
                              tzinfo=_tz)
    dt_end_local   = datetime(d.year, d.month, d.day,
                              RANGE_END_LOCAL[0],   RANGE_END_LOCAL[1],
                              tzinfo=_tz)

    # Convertir en UTC puis en microsecondes
    rs_us = int(dt_start_local.astimezone(_tz_utc).replace(tzinfo=None)
                .replace(tzinfo=_tz_utc).timestamp() * 1_000_000)
    re_us = int(dt_end_local.astimezone(_tz_utc).replace(tzinfo=None)
                .replace(tzinfo=_tz_utc).timestamp() * 1_000_000)

    _range_cache[d] = (rs_us, re_us)
    return rs_us, re_us

def day_id_to_date(day_id: int) -> date:
    """Conversion robuste day_id → date (indépendante du fuseau local)."""
    return date(1970, 1, 1) + timedelta(days=int(day_id))

# ==============================================================================
# PARSE TIMESTAMP VECTORISÉ
# Format fixe : DD.MM.YYYY HH:MM:SS.mmm (23 caractères)
# ==============================================================================

def parse_timestamps_vectorized(ts_col: pd.Series) -> np.ndarray:
    """
    Parse les timestamps CSV en microsecondes UTC int64.
    Les données Tick Data Suite sont en GMT+0 NO-DST → timestamp = UTC direct.
    """
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
    def __init__(self, period):
        self.a = 2.0/(period+1); self.v = None; self.n = 0; self.p = period
    def update(self, x):
        self.v = x if self.v is None else self.a*x+(1-self.a)*self.v; self.n+=1
    def get(self): return self.v if self.n >= self.p else None

class RSIBuffer:
    def __init__(self, period):
        self.a=1./period; self.ag=self.al=None; self.prev=None; self.n=0; self.p=period
    def update(self, x):
        if self.prev is None: self.prev=x; return
        d=x-self.prev; self.prev=x; g=max(d,0.); l=max(-d,0.)
        if self.ag is None: self.ag=g; self.al=l
        else: self.ag=self.a*g+(1-self.a)*self.ag; self.al=self.a*l+(1-self.a)*self.al
        self.n+=1
    def get(self):
        if self.ag is None or self.n<self.p: return None
        return 100. if self.al==0 else 100.-100./(1.+self.ag/self.al)

class ATRBuffer:
    def __init__(self, period):
        self.a=1./period; self.atr=None; self.pc=None; self.n=0; self.p=period
    def update(self, h, l, c):
        tr=(h-l) if self.pc is None else max(h-l,abs(h-self.pc),abs(l-self.pc))
        self.pc=c; self.atr=tr if self.atr is None else self.a*tr+(1-self.a)*self.atr; self.n+=1
    def get(self): return self.atr if self.n >= self.p else None

class BollingerBuffer:
    def __init__(self, period=20, n_std=2.0):
        self.period=period; self.n_std=n_std; self.buf=deque(maxlen=period)
    def update(self, x): self.buf.append(x)
    def get(self):
        if len(self.buf)<self.period: return None,None,None,None
        arr=np.array(self.buf); mid=arr.mean(); std=arr.std(ddof=1)
        upper=mid+self.n_std*std; lower=mid-self.n_std*std
        return mid, upper, lower, upper-lower
    def get_pct_b(self, price):
        _,_,lower,width=self.get()
        return None if width is None or width==0 else (price-lower)/width

class AroonBuffer:
    def __init__(self, period):
        self.period=period; self.highs=deque(maxlen=period+1); self.lows=deque(maxlen=period+1)
    def update(self, h, l): self.highs.append(h); self.lows.append(l)
    def get(self):
        if len(self.highs)<self.period+1: return None,None
        h=np.array(self.highs); l=np.array(self.lows)
        bsh=self.period-np.argmax(h); bsl=self.period-np.argmin(l)
        return (self.period-bsh)/self.period*100, (self.period-bsl)/self.period*100

class SlopeBuffer:
    def __init__(self, n=5):
        self.n=n; self.buf=deque(maxlen=n); self.x=np.arange(n,dtype=float)
    def update(self, v): self.buf.append(v)
    def get(self):
        if len(self.buf)<self.n: return None
        y=np.array(self.buf); m=y.mean()
        return 0. if m==0 else np.polyfit(self.x,y,1)[0]/m*100

class IndicatorState:
    def __init__(self):
        self.emas   = {p: EMABuffer(p)   for p in EMA_PERIODS}
        self.slopes = {p: SlopeBuffer(SLOPE_N) for p in EMA_PERIODS}
        self.rsis   = {p: RSIBuffer(p)   for p in RSI_PERIODS}
        self.atrs   = {p: ATRBuffer(p)   for p in ATR_PERIODS}
        self.bb     = BollingerBuffer(BB_PERIOD, BB_STD)
        self.aroons = {p: AroonBuffer(p) for p in AROON_PERIODS}

    def update(self, o, h, l, c):
        for p in EMA_PERIODS:
            self.emas[p].update(c)
            v=self.emas[p].get()
            if v is not None: self.slopes[p].update(v)
        for p in RSI_PERIODS:  self.rsis[p].update(c)
        for p in ATR_PERIODS:  self.atrs[p].update(h, l, c)
        self.bb.update(c)
        for p in AROON_PERIODS: self.aroons[p].update(h, l)

    def snapshot(self, price: float, direction: str, suffix: str = '') -> dict:
        s = suffix; snap = {}
        for p in EMA_PERIODS:
            v=self.emas[p].get()
            snap[f'ema{p}{s}']=v
            snap[f'slope_ema{p}{s}']=self.slopes[p].get()
            snap[f'price_above_ema{p}{s}']=(int(price>v) if v is not None else None)
        e200=self.emas[200].get()
        snap[f'trade_with_ema200{s}']=None
        if e200 is not None:
            snap[f'trade_with_ema200{s}']=int(
                (direction=='BUY' and price>e200) or (direction=='SELL' and price<e200))
        for p in RSI_PERIODS: snap[f'rsi{p}{s}']=self.rsis[p].get()
        for p in ATR_PERIODS: snap[f'atr{p}{s}']=self.atrs[p].get()
        bb_mid,bb_up,bb_lo,bb_w=self.bb.get()
        snap[f'bb_width{s}']=round(bb_w,4) if bb_w is not None else None
        snap[f'bb_pct_b{s}']=round(self.bb.get_pct_b(price),4) if bb_w is not None else None
        snap[f'bb_upper{s}']=round(bb_up,4) if bb_up is not None else None
        snap[f'bb_lower{s}']=round(bb_lo,4) if bb_lo is not None else None
        for p in AROON_PERIODS:
            up,dn=self.aroons[p].get()
            snap[f'aroon{p}_up{s}']=round(up,2) if up is not None else None
            snap[f'aroon{p}_down{s}']=round(dn,2) if dn is not None else None
        return snap

# ==============================================================================
# CONSTRUCTION BOUGIES OHLC VECTORISÉE
# ==============================================================================

def build_candles_us(ts_us, ask, bid, timeframe_min):
    """
    Bougies OHLC Ask et Bid depuis arrays numpy µs int64.
    timeframe_min : durée d'une bougie en minutes.
    Retourne (ohlc_ask, ohlc_bid) comme DataFrames indexés Timestamp.
    """
    if len(ts_us) == 0:
        empty = pd.DataFrame(columns=['open','high','low','close'])
        return empty, empty
    tf_us      = timeframe_min * US_PER_MIN
    candles_us = (ts_us // tf_us) * tf_us
    changes    = np.where(np.diff(candles_us) != 0)[0] + 1
    starts     = np.concatenate([[0], changes])
    ends       = np.concatenate([changes, [len(ts_us)]])
    idx        = pd.to_datetime(candles_us[starts], unit='us')
    def mk(arr):
        return pd.DataFrame({
            'open' : arr[starts],
            'high' : np.maximum.reduceat(arr, starts),
            'low'  : np.minimum.reduceat(arr, starts),
            'close': arr[ends-1]
        }, index=idx)
    return mk(ask), mk(bid)

# ==============================================================================
# SIMULATION SL/TP/TIMEOUT VECTORISÉE
# ==============================================================================

def simulate_trade(ask, bid, ts_us, direction, sl, tp,
                   eod_gap_us, max_dur_us, entry_us):
    """
    Simule un trade tick par tick.
    Détecte dans l'ordre : TP, SL, TIMEOUT (durée max), CLOSE_EOD (gap ticks).
    Retourne (result, exit_price, exit_us).
    """
    if direction == 'BUY':
        tp_hit = bid >= tp; sl_hit = bid <= sl
    else:
        tp_hit = ask <= tp; sl_hit = ask >= sl

    # Gap EOD entre ticks consécutifs
    gaps     = np.diff(ts_us)
    gap_idxs = np.where(gaps > eod_gap_us)[0]

    # Timeout : premier tick au-delà de la durée max
    if max_dur_us > 0:
        timeout_idxs = np.where(ts_us - entry_us > max_dur_us)[0]
    else:
        timeout_idxs = np.array([], dtype=np.int64)

    hit       = tp_hit | sl_hit
    i_hit     = int(np.argmax(hit))     if hit.any()             else len(ts_us)
    i_gap     = int(gap_idxs[0])        if len(gap_idxs) > 0     else len(ts_us)
    i_timeout = int(timeout_idxs[0])    if len(timeout_idxs) > 0 else len(ts_us)

    i_first = min(i_hit, i_gap, i_timeout)

    if i_first == len(ts_us):
        # Aucun événement — fin de ticks
        mid = float((ask[-1]+bid[-1])/2)
        return 'CLOSE_EOD', mid, int(ts_us[-1])

    if i_first == i_gap:
        mid = float((ask[i_gap]+bid[i_gap])/2)
        return 'CLOSE_EOD', mid, int(ts_us[i_gap])

    if i_first == i_timeout:
        mid = float((ask[i_timeout]+bid[i_timeout])/2)
        return 'TIMEOUT', mid, int(ts_us[i_timeout])

    # TP ou SL
    result = 'TP' if tp_hit[i_hit] else 'SL'
    return result, (tp if result=='TP' else sl), int(ts_us[i_hit])

# ==============================================================================
# CALCUL DU LOT
# ==============================================================================

def calc_lot(capital, risk_pct, sl_pts, lot_min, tol):
    if sl_pts <= 0: return 0., 'INVALID_SL'
    lot = (capital*risk_pct/100.) / sl_pts
    if lot < lot_min:
        if (lot_min*sl_pts) / (capital*risk_pct/100.) <= tol:
            return lot_min, 'OK_ROUNDED'
        return lot_min, 'LOT_TOO_SMALL'
    return lot, 'OK'

# ==============================================================================
# DÉTECTION DU SIGNAL
# ==============================================================================

def detect_signal_fvg(ca, cb, ts_c, range_h, range_l, indic):
    """
    Détecte un setup FVG dans les bougies de la fenêtre de recherche.
    ca/cb : arrays (N,4) de bougies Ask/Bid (open,high,low,close)
    Retourne (setup_dict, snap_n) ou (None, {})
    """
    snap_n = {}
    for i in range(len(ts_c) - 2):
        indic.update(
            (ca[i,0]+cb[i,0])/2, (ca[i,1]+cb[i,1])/2,
            (ca[i,2]+cb[i,2])/2, (ca[i,3]+cb[i,3])/2
        )
        n_h_ask=ca[i,1];  n_l_bid=cb[i,2]
        n1_h_ask=ca[i+1,1]; n1_l_bid=cb[i+1,2]
        n1_c_ask=ca[i+1,3]; n1_c_bid=cb[i+1,3]
        n2_h_ask=ca[i+2,1]; n2_l_bid=cb[i+2,2]
        n2_c_ask=ca[i+2,3]; n2_c_bid=cb[i+2,3]
        n2_ts=int(ts_c[i+2])
        spread=float(n2_c_ask-n2_c_bid)
        if spread > MAX_SPREAD_POINTS: continue

        if TRADE_DIRECTION in ('BUY_ONLY','BOTH'):
            if n1_c_ask>range_h and n1_h_ask>range_h and n2_l_bid>n_h_ask:
                sl_pts=float(n2_c_ask-n1_l_bid)
                if sl_pts>0:
                    snap_n=indic.snapshot(float(n2_c_ask),'BUY','')
                    return {'direction':'BUY','entry':float(n2_c_ask),
                            'sl':float(n1_l_bid),'spread':spread,
                            'n2_ts':n2_ts,'fvg_size':float(n2_l_bid-n_h_ask)}, snap_n

        if TRADE_DIRECTION in ('SELL_ONLY','BOTH'):
            if n1_c_bid<range_l and n1_l_bid<range_l and n2_h_ask<n_l_bid:
                sl_pts=float(n1_h_ask-n2_c_bid)
                if sl_pts>0:
                    snap_n=indic.snapshot(float(n2_c_bid),'SELL','')
                    return {'direction':'SELL','entry':float(n2_c_bid),
                            'sl':float(n1_h_ask),'spread':spread,
                            'n2_ts':n2_ts,'fvg_size':float(n_l_bid-n2_h_ask)}, snap_n
    return None, {}


def detect_signal_breakout(ca, cb, ts_c, range_h, range_l, indic):
    """
    Détecte un setup BREAKOUT dans les bougies de la fenêtre de recherche.
    Entrée à la clôture de la première bougie fermant au-delà du range.
    SL : opposé du range (range_low pour BUY, range_high pour SELL).
    """
    snap_n = {}
    for i in range(len(ts_c)):
        indic.update(
            (ca[i,0]+cb[i,0])/2, (ca[i,1]+cb[i,1])/2,
            (ca[i,2]+cb[i,2])/2, (ca[i,3]+cb[i,3])/2
        )
        c_ask=ca[i,3]; c_bid=cb[i,3]; c_ts=int(ts_c[i])
        spread=float(ca[i,3]-cb[i,3])
        if spread > MAX_SPREAD_POINTS: continue

        if TRADE_DIRECTION in ('BUY_ONLY','BOTH'):
            if c_ask > range_h:
                sl_pts=float(c_ask-range_l)
                if sl_pts>0:
                    snap_n=indic.snapshot(float(c_ask),'BUY','')
                    return {'direction':'BUY','entry':float(c_ask),
                            'sl':float(range_l),'spread':spread,
                            'n2_ts':c_ts,'fvg_size':0.}, snap_n

        if TRADE_DIRECTION in ('SELL_ONLY','BOTH'):
            if c_bid < range_l:
                sl_pts=float(range_h-c_bid)
                if sl_pts>0:
                    snap_n=indic.snapshot(float(c_bid),'SELL','')
                    return {'direction':'SELL','entry':float(c_bid),
                            'sl':float(range_h),'spread':spread,
                            'n2_ts':c_ts,'fvg_size':0.}, snap_n
    return None, {}

# ==============================================================================
# TRAITEMENT D'UN JOUR
# ==============================================================================

def process_day(ts_us, ask, bid, indic, capital, d):
    rs_us, re_us = range_bounds_us(d)
    end_us = rs_us + SIGNAL_WINDOW_MIN * US_PER_MIN
    tf_us  = SIGNAL_TIMEFRAME_MIN * US_PER_MIN
    eod_gap_us = EOD_GAP_MINUTES * US_PER_MIN
    max_dur_us = MAX_TRADE_DURATION_MIN * US_PER_MIN if MAX_TRADE_DURATION_MIN > 0 else 0

    # ── Range ────────────────────────────────────────────────────────────
    mask_r = (ts_us >= rs_us) & (ts_us < re_us)
    if not mask_r.any():
        return {'date':d,'result':'NO_DATA','capital_after':capital}
    range_h = float(ask[mask_r].max())
    range_l = float(bid[mask_r].min())
    if range_h - range_l < MIN_RANGE_POINTS:
        return {'date':d,'result':'FILTERED_RANGE','capital_after':capital}

    # ── Bougies de la timeframe configurée ──────────────────────────────
    oa, ob   = build_candles_us(ts_us, ask, bid, SIGNAL_TIMEFRAME_MIN)
    oa_idx   = oa.index.asi8
    ob_idx   = ob.index.asi8

    # Mise à jour indicateurs sur bougies AVANT la fenêtre de recherche
    mask_pre = oa_idx < re_us
    ca_pre   = oa.values[mask_pre]
    cb_pre   = ob.values[ob_idx < re_us]
    for i in range(min(len(ca_pre), len(cb_pre))):
        indic.update(
            (ca_pre[i,0]+cb_pre[i,0])/2, (ca_pre[i,1]+cb_pre[i,1])/2,
            (ca_pre[i,2]+cb_pre[i,2])/2, (ca_pre[i,3]+cb_pre[i,3])/2
        )

    # ── Fenêtre de recherche ─────────────────────────────────────────────
    mask_win = (oa_idx >= re_us) & (oa_idx < end_us)
    idx_b    = ob_idx[(ob_idx >= re_us) & (ob_idx < end_us)]
    common_us, ia, ib = np.intersect1d(oa_idx[mask_win], idx_b,
                                        return_indices=True)
    if len(common_us) < (3 if SIGNAL_MODE == 'FVG' else 1):
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    ca    = oa.values[mask_win][ia]
    cb    = ob.values[(ob_idx >= re_us) & (ob_idx < end_us)][ib]
    ts_c  = common_us

    # ── Détection du signal ──────────────────────────────────────────────
    if SIGNAL_MODE == 'FVG':
        setup, snap_n = detect_signal_fvg(ca, cb, ts_c, range_h, range_l, indic)
    else:
        setup, snap_n = detect_signal_breakout(ca, cb, ts_c, range_h, range_l, indic)

    if setup is None:
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    direction = setup['direction']
    entry     = setup['entry']
    sl_price  = setup['sl']
    spread_e  = setup['spread']
    n2_ts     = setup['n2_ts']
    fvg_size  = setup['fvg_size']

    sl_pts   = abs(entry - sl_price)
    tp_price = entry + RISK_REWARD*sl_pts if direction=='BUY' else entry - RISK_REWARD*sl_pts

    # Snapshot N+2 (à l'entrée)
    # Mettre à jour les indicateurs sur les bougies N+1 et N+2 si FVG
    if SIGNAL_MODE == 'FVG' and len(ca) >= 3:
        idx_setup = next((i for i in range(len(ts_c)) if int(ts_c[i]) == n2_ts), None)
        if idx_setup and idx_setup >= 2:
            for j in [idx_setup-1, idx_setup]:
                indic.update(
                    (ca[j,0]+cb[j,0])/2, (ca[j,1]+cb[j,1])/2,
                    (ca[j,2]+cb[j,2])/2, (ca[j,3]+cb[j,3])/2
                )
    snap_n2 = indic.snapshot(entry, direction, '_n2')

    # ── Lot ──────────────────────────────────────────────────────────────
    lot, lot_status = calc_lot(capital, RISK_PERCENT, sl_pts,
                               LOT_MIN, LOT_TOLERANCE_RATIO)
    if lot_status == 'LOT_TOO_SMALL':
        return {'date':d,'result':'LOT_TOO_SMALL','capital_after':capital}

    # ── Simulation ───────────────────────────────────────────────────────
    mask_after = ts_us > n2_ts
    if not mask_after.any():
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    result, exit_price, exit_us = simulate_trade(
        ask[mask_after], bid[mask_after], ts_us[mask_after],
        direction, sl_price, tp_price,
        eod_gap_us, max_dur_us, n2_ts
    )

    # ── P&L ──────────────────────────────────────────────────────────────
    pnl_pts = (exit_price-entry) if direction=='BUY' else (entry-exit_price)
    pnl_pct = pnl_pts / sl_pts * RISK_PERCENT
    capital_new = capital * (1 + pnl_pct/100.)
    duration = (exit_us - n2_ts) / (US_PER_MIN*1.) if exit_us else None

    atr14    = snap_n.get('atr14')
    range_sz = range_h - range_l

    return {
        'date'            : d,
        'year'            : d.year,
        'month'           : d.month,
        'day_of_week'     : d.strftime('%A'),
        'day_of_week_num' : d.weekday()+1,
        'week_of_month'   : (d.day-1)//7+1,
        'direction'       : direction,
        'signal_mode'     : SIGNAL_MODE,
        'entry_time_gmt'  : pd.Timestamp(n2_ts + US_PER_MIN, unit='us'),
        'exit_time_gmt'   : pd.Timestamp(exit_us, unit='us') if exit_us else None,
        'duration_min'    : round(duration,2) if duration else None,
        'entry_price'     : round(entry,4),
        'sl_price'        : round(sl_price,4),
        'tp_price'        : round(tp_price,4),
        'exit_price'      : round(float(exit_price),4),
        'sl_distance_pts' : round(sl_pts,4),
        'pnl_pts'         : round(pnl_pts,4),
        'pnl_pct'         : round(pnl_pct,4),
        'lot_size'        : round(lot,4),
        'lot_status'      : lot_status,
        'capital_before'  : round(capital,4),
        'capital_after'   : round(capital_new,4),
        'result'          : result,
        'close_eod'       : result in ('CLOSE_EOD','TIMEOUT'),
        'range_size_pts'  : round(range_sz,4),
        'fvg_size_pts'    : round(fvg_size,4),
        'spread_entry'    : round(spread_e,4),
        'fvg_atr_ratio'   : round(fvg_size/atr14,4) if atr14 else None,
        'range_atr_ratio' : round(range_sz/atr14,4) if atr14 else None,
        **snap_n,
        **snap_n2,
    }

# ==============================================================================
# MÉTRIQUES
# ==============================================================================

def compute_metrics(trades, label, initial_capital):
    # Trades "propres" : TP ou SL uniquement (hors CLOSE_EOD et TIMEOUT)
    valid = [t for t in trades if t.get('result') in ('TP','SL')]
    all_t = [t for t in trades if t.get('result') in ('TP','SL','CLOSE_EOD','TIMEOUT')]
    n=len(valid); n_all=len(all_t)
    if n_all==0: return {'label':label,'n_trades':0}

    n_tp=sum(1 for t in valid if t['result']=='TP')
    n_sl=n-n_tp
    n_eod=sum(1 for t in all_t if t['result']=='CLOSE_EOD')
    n_tmo=sum(1 for t in all_t if t['result']=='TIMEOUT')

    pnls=[t['pnl_pts'] for t in all_t]
    gp=sum(p for p in [t['pnl_pts'] for t in valid] if p>0)
    gl=sum(abs(p) for p in [t['pnl_pts'] for t in valid] if p<0)
    wr=n_tp/n*100 if n>0 else 0
    pf=gp/gl if gl>0 else float('inf')

    caps=[initial_capital]+[t['capital_after'] for t in all_t]
    peak=caps[0]; maxdd=0.
    for c in caps: peak=max(peak,c); maxdd=max(maxdd,(peak-c)/peak*100)

    streak=ms2=0
    for t in valid:
        streak=streak+1 if t['result']=='SL' else 0; ms2=max(ms2,streak)

    atp=np.mean([t['pnl_pts'] for t in valid if t['result']=='TP']) if n_tp else 0
    asl=np.mean([t['pnl_pts'] for t in valid if t['result']=='SL']) if n_sl else 0
    exp=(wr/100)*atp+(1-wr/100)*asl if n>0 else 0

    dtp=[t['duration_min'] for t in valid if t['result']=='TP' and t.get('duration_min')]
    dsl=[t['duration_min'] for t in valid if t['result']=='SL' and t.get('duration_min')]

    daily={t['date']:t['pnl_pct'] for t in all_t}
    dr=list(daily.values()); sharpe=calmar=None
    if len(dr)>1:
        mu=np.mean(dr); sig=np.std(dr,ddof=1)
        if sig>0: sharpe=round(mu/sig*np.sqrt(252),3)
        if maxdd>0: calmar=round(sum(dr)/maxdd,3)

    return {
        'label':label,'n_trades':n_all,'n_tp':n_tp,'n_sl':n_sl,
        'n_close_eod':n_eod,'n_timeout':n_tmo,
        'win_rate_pct':round(wr,2),
        'net_pts':round(sum(pnls),2),
        'gross_profit_pts':round(gp,2),'gross_loss_pts':round(gl,2),
        'profit_factor':round(pf,3),
        'expectancy_pts':round(exp,2),
        'max_drawdown_pct':round(maxdd,2),
        'max_consec_sl':ms2,
        'sharpe_ratio':sharpe,'calmar_ratio':calmar,
        'avg_dur_tp_min':round(np.mean(dtp),1) if dtp else None,
        'avg_dur_sl_min':round(np.mean(dsl),1) if dsl else None,
        'dur_tp_p95_min':round(np.percentile(dtp,95),1) if len(dtp)>1 else None,
        'dur_sl_p95_min':round(np.percentile(dsl,95),1) if len(dsl)>1 else None,
    }

def print_metrics(m):
    if m['n_trades']==0: log.info(f"    {m['label']} : aucun trade"); return
    log.info(f"\n  ── {m['label']} ──")
    log.info(f"  Trades        : {m['n_trades']}  "
             f"(TP:{m['n_tp']} SL:{m['n_sl']} "
             f"EOD:{m['n_close_eod']} TMO:{m['n_timeout']})")
    log.info(f"  Win rate      : {m['win_rate_pct']}%  (hors EOD/TIMEOUT)")
    log.info(f"  P&L net       : {m['net_pts']:>10.2f} pts")
    log.info(f"  Profit Factor : {m['profit_factor']}")
    log.info(f"  Espérance     : {m['expectancy_pts']:.2f} pts/trade")
    log.info(f"  Drawdown max  : {m['max_drawdown_pct']}%")
    log.info(f"  Sharpe        : {m['sharpe_ratio']}")
    log.info(f"  Calmar        : {m['calmar_ratio']}")
    log.info(f"  SL consécutifs max : {m['max_consec_sl']}")
    if m['avg_dur_tp_min']: log.info(f"  Durée moy TP  : {m['avg_dur_tp_min']} min (p95: {m['dur_tp_p95_min']} min)")
    if m['avg_dur_sl_min']: log.info(f"  Durée moy SL  : {m['avg_dur_sl_min']} min (p95: {m['dur_sl_p95_min']} min)")

# ==============================================================================
# ÉCRITURE ANNUELLE
# ==============================================================================

def write_year(year, trades, summary_rows, initial_capital):
    if not trades: return
    slug   = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','').replace('&','')
    prefix = os.path.join(OUTPUT_DIR, f"rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_{year}")

    pd.DataFrame(trades).to_csv(f"{prefix}_trades.csv", index=False)

    monthly=[]
    for mo in sorted({t['month'] for t in trades}):
        sub=[t for t in trades if t['month']==mo]
        m=compute_metrics(sub,f"{year}-{mo:02d}",initial_capital)
        m['year']=year; m['month']=mo; monthly.append(m)
    pd.DataFrame(monthly).to_csv(f"{prefix}_monthly.csv", index=False)

    m_year=compute_metrics(trades,str(year),initial_capital)
    m_year['year']=year; summary_rows.append(m_year)

    slug2  = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','').replace('&','')
    s_path = os.path.join(OUTPUT_DIR, f"rng_{slug2}_{RUN_ID}_{MAGIC_NUMBER}_summary.csv")
    pd.DataFrame(summary_rows).to_csv(s_path, index=False)

    log.info(f"\n  ── Année {year} — fichiers écrits ──")
    log.info(f"     {prefix}_trades.csv  ({len(trades)} trades)")
    log.info(f"     {prefix}_monthly.csv")
    log.info(f"     {s_path}  (mis à jour)")
    print_metrics(m_year)

# ==============================================================================
# BARRE DE PROGRESSION PAR ANNÉE
# ==============================================================================

class Progress:
    MONTHS=['jan','fév','mar','avr','mai','jun','jul','aoû','sep','oct','nov','déc']
    def __init__(self):
        self.t0=time.time(); self.last=0.; self.year_t0={}
    def update(self, cur_date, n_trades, speed_mbs):
        now=time.time()
        if now-self.last<3.: return
        self.last=now
        if cur_date is None: return
        year=cur_date.year; month=cur_date.month
        if year not in self.year_t0: self.year_t0[year]=now
        pct_year=(month-1)/12*100; bl=24; fi=int(bl*pct_year/100)
        bar='█'*fi+'░'*(bl-fi); mon_str=self.MONTHS[month-1]
        if month>1:
            el_y=now-self.year_t0[year]
            eta_s=el_y/(month-1)*(12-month+1)
            em,es=divmod(int(eta_s),60); eta_str=f"ETA {em}m{es:02d}s"
        else: eta_str=""
        print(f"\r  {year} [{bar}] {mon_str}  {speed_mbs:4.1f} Mo/s  "
              f"{eta_str}  trades:{n_trades}     ", end='', flush=True)
    def done(self):
        el=time.time()-self.t0
        print(f"\r  {'─'*65}", flush=True)
        log.info(f"  ✓ Lecture terminée en {el/60:.1f} min")

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL
# ==============================================================================

def run_backtest():
    log.info("="*70)
    log.info(f" RANGE BREAKOUT POLYVALENT — {ASSET_NAME}")
    log.info(f" Mode : {SIGNAL_MODE}  |  Direction : {TRADE_DIRECTION}  |  "
             f"TF : {SIGNAL_TIMEFRAME_MIN}min  |  R:R {RISK_REWARD}:1")
    log.info(f" Horaire : {RANGE_START_LOCAL[0]:02d}h{RANGE_START_LOCAL[1]:02d}–"
             f"{RANGE_END_LOCAL[0]:02d}h{RANGE_END_LOCAL[1]:02d} {RANGE_TIMEZONE}")
    log.info(f" Fenêtre signal : {SIGNAL_WINDOW_MIN} min  |  "
             f"Durée max trade : {MAX_TRADE_DURATION_MIN} min")
    log.info(f" Fichier : {FILE_PATH}")
    noms=['Lun','Mar','Mer','Jeu','Ven','Sam','Dim']
    jours=[noms[i] for i,v in enumerate(WEEKDAY_ALLOWED) if v]
    log.info(f" Années : {YEARS_FILTER if YEARS_FILTER else 'toutes'}  |  "
             f"Jours : {jours}")
    log.info(f" Filtres — Spread max : {MAX_SPREAD_POINTS} pts  |  "
             f"Range min : {MIN_RANGE_POINTS} pts  |  EOD gap : {EOD_GAP_MINUTES} min")
    log.info(f" RUN_ID : {RUN_ID}  |  MAGIC_NUMBER : {MAGIC_NUMBER}")
    log.info("="*70)

    if not os.path.exists(FILE_PATH):
        log.error(f"Fichier introuvable : {FILE_PATH}"); pause(); sys.exit(1)

    file_size=os.path.getsize(FILE_PATH)
    log.info(f"\n Taille : {file_size/1e9:.2f} Go  |  Chunks : {CHUNK_SIZE:,} lignes\n")

    indic=IndicatorState(); capital=INITIAL_CAPITAL
    cur_year=None; year_trades=[]; summary_rows=[]
    cur_day_id=-1; day_ts=[]; day_ask=[]; day_bid=[]
    lookback_buf=deque(maxlen=LOOKBACK_DAYS)
    cnt_no_data=cnt_filtered=cnt_no_setup=cnt_eod=cnt_tmo=cnt_too_small=0
    prog=Progress(); bytes_read=0; speed_mbs=0.

    log.info(" Lecture unique — progression toutes les 3 secondes\n")

    def flush_day(day_id, ts_list, ask_list, bid_list):
        nonlocal capital, cur_year, year_trades, indic
        nonlocal cnt_no_data,cnt_filtered,cnt_no_setup,cnt_eod,cnt_tmo,cnt_too_small
        if not ts_list or day_id<0: return
        d=day_id_to_date(day_id)
        if not WEEKDAY_ALLOWED[d.weekday()]: return

        ts_arr=np.array(ts_list,dtype=np.int64)
        ask_arr=np.array(ask_list,dtype=np.float32)
        bid_arr=np.array(bid_list,dtype=np.float32)

        if cur_year is None: cur_year=d.year

        if d.year!=cur_year:
            if not YEARS_FILTER or cur_year in YEARS_FILTER:
                write_year(cur_year,year_trades,summary_rows,INITIAL_CAPITAL)
            year_trades=[]; capital=INITIAL_CAPITAL; cur_year=d.year
            indic=IndicatorState()
            for lb_ts,lb_ask,lb_bid in lookback_buf:
                lb_oa,lb_ob=build_candles_us(lb_ts,lb_ask,lb_bid,SIGNAL_TIMEFRAME_MIN)
                ca=lb_oa.values; cb=lb_ob.values
                for i in range(min(len(ca),len(cb))):
                    indic.update((ca[i,0]+cb[i,0])/2,(ca[i,1]+cb[i,1])/2,
                                 (ca[i,2]+cb[i,2])/2,(ca[i,3]+cb[i,3])/2)
            log.info(f"\n  Nouvelle année : {d.year} — "
                     f"indicateurs réinitialisés sur {len(lookback_buf)} jours")

        lookback_buf.append((ts_arr.copy(),ask_arr.copy(),bid_arr.copy()))

        if YEARS_FILTER and d.year not in YEARS_FILTER:
            oa,ob=build_candles_us(ts_arr,ask_arr,bid_arr,SIGNAL_TIMEFRAME_MIN)
            ca=oa.values; cb=ob.values
            for i in range(min(len(ca),len(cb))):
                indic.update((ca[i,0]+cb[i,0])/2,(ca[i,1]+cb[i,1])/2,
                             (ca[i,2]+cb[i,2])/2,(ca[i,3]+cb[i,3])/2)
            return

        res=process_day(ts_arr,ask_arr,bid_arr,indic,capital,d)
        r=res.get('result','')
        if   r=='NO_DATA':        cnt_no_data+=1
        elif r=='FILTERED_RANGE': cnt_filtered+=1
        elif r=='NO_SETUP':       cnt_no_setup+=1
        elif r=='LOT_TOO_SMALL':  cnt_too_small+=1
        elif r in ('TP','SL','CLOSE_EOD','TIMEOUT'):
            capital=res['capital_after']
            if r=='CLOSE_EOD': cnt_eod+=1
            if r=='TIMEOUT':   cnt_tmo+=1
            year_trades.append(res)

    reader=pd.read_csv(FILE_PATH,header=None,names=['timestamp','ask','bid'],
                       usecols=[0,1,2],chunksize=CHUNK_SIZE,
                       dtype={'ask':'float32','bid':'float32'},engine='c')

    for chunk in reader:
        bytes_read+=len(chunk)*33
        el=time.time()-prog.t0; speed_mbs=bytes_read/el/1e6 if el>0 else 0.
        cur_d=day_id_to_date(cur_day_id) if cur_day_id>=0 else None
        prog.update(cur_d,len(year_trades),speed_mbs)

        ts_us=parse_timestamps_vectorized(chunk['timestamp'])
        ask_v=chunk['ask'].values; bid_v=chunk['bid'].values

        day_ids=ts_us//US_PER_DAY
        changes=np.where(np.diff(day_ids)!=0)[0]+1
        seg_s=np.concatenate([[0],changes]); seg_e=np.concatenate([changes,[len(ts_us)]])

        for s,e in zip(seg_s,seg_e):
            sdid=int(day_ids[s])
            if sdid!=cur_day_id:
                flush_day(cur_day_id,day_ts,day_ask,day_bid)
                cur_day_id=sdid; day_ts=[]; day_ask=[]; day_bid=[]
            day_ts.extend(ts_us[s:e].tolist())
            day_ask.extend(ask_v[s:e].tolist())
            day_bid.extend(bid_v[s:e].tolist())

    flush_day(cur_day_id,day_ts,day_ask,day_bid)
    if cur_year and year_trades:
        if not YEARS_FILTER or cur_year in YEARS_FILTER:
            write_year(cur_year,year_trades,summary_rows,INITIAL_CAPITAL)

    prog.done()

    log.info(f"\n{'='*70}\n RÉSULTATS GLOBAUX — {ASSET_NAME}\n{'='*70}")
    all_trades=[]
    slug=ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','').replace('&','')
    for row in summary_rows:
        yr=row.get('year')
        if yr:
            f=os.path.join(OUTPUT_DIR,f"rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_{yr}_trades.csv")
            if os.path.exists(f):
                all_trades.extend(pd.read_csv(f).to_dict('records'))
    if all_trades: print_metrics(compute_metrics(all_trades,"GLOBAL",INITIAL_CAPITAL))
    else: log.info("  Aucun trade.")

    log.info(f"\n Compteurs :")
    log.info(f"   Sans données   : {cnt_no_data}")
    log.info(f"   Filtrés range  : {cnt_filtered}")
    log.info(f"   Sans setup     : {cnt_no_setup}")
    log.info(f"   Lot trop petit : {cnt_too_small}")
    log.info(f"   CLOSE_EOD      : {cnt_eod}")
    log.info(f"   TIMEOUT        : {cnt_tmo}")

    log.info(f"\n Fichiers dans : {OUTPUT_DIR}")
    log.info(f"   rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_XXXX_trades.csv")
    log.info(f"   rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_XXXX_monthly.csv")
    log.info(f"   rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_summary.csv")
    log.info(f"   {LOG_PATH}")
    log.info(f"\n{'='*70}\n Terminé.\n{'='*70}\n")


if __name__ == '__main__':
    try:
        run_backtest()
    except Exception:
        log.error("\n"+"="*70+"\n ERREUR INATTENDUE\n"+"="*70)
        log.error(traceback.format_exc())
    finally:
        pause()

