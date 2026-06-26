# ======================================================================
# Run 1/3 — Trailing factor=2.0, BE=0.0 (trailing immédiat)
# ======================================================================

# -*- coding: utf-8 -*-
"""
================================================================================
RANGE BREAKOUT POLYVALENT V8 — Trailing ATR sur données tick
================================================================================

DESCRIPTION
-----------
Moteur de backtest range breakout avec sortie dynamique TRAILING ATR.
Basé sur l'architecture V6 (TICKS uniquement) — simulation de sortie
vectorisée sur bougies 1min reconstruites depuis les ticks.

SORTIE TRAILING ATR
-------------------
Pas de TP fixe. Le SL remonte automatiquement derrière le prix.

Phase 1 — SL fixe :
    Tant que le prix n'a pas atteint BE_FACTOR × sl_distance en faveur,
    le SL reste au niveau initial. Évite les sorties prématurées sur bruit.
    BE_FACTOR = 0.0 → trailing immédiat dès l'entrée (pas de phase fixe)
    BE_FACTOR = 0.5 → trailing activé après 0.5 × SL en faveur
    BE_FACTOR = 1.0 → trailing activé après break-even complet

Phase 2 — Trailing :
    Le SL suit le prix extrême atteint avec une distance de
    TRAILING_ATR_FACTOR × ATR14(1min).
    ATR14 calculé sur les bougies 1min depuis l'entrée du trade.

Résultats possibles :
    'TP_TRAIL' : sortie sur trailing SL après BE atteint → trade gagnant
    'SL'       : sortie sur SL fixe avant BE atteint → trade perdant
    'CLOSE_EOD': gap de ticks > EOD_GAP_MINUTES → clôture forcée
    'TIMEOUT'  : durée > MAX_TRADE_DURATION_MIN → clôture forcée

PARAMÈTRES TRAILING
-------------------
TRAILING_ATR_FACTOR :
    2.0 → trailing confortable, bon compromis bruit/capture
    3.0 → trailing large, laisse courir les grands mouvements
BE_FACTOR :
    0.0 → trailing immédiat
    0.5 → activation après 0.5 × SL en faveur
    1.0 → break-even complet

SIMULATION VECTORISÉE
---------------------
La sortie trailing est simulée sur bougies 1min reconstruites depuis
les ticks (np.maximum.reduceat) — pas de boucle Python sur les ticks.
ATR14 calculé bougie par bougie. Sortie détectée sur le LOW/HIGH de
la bougie qui touche le trailing SL (précision bougie, pas tick exact).
Biais minimal et constant — n'affecte pas les conclusions comparatives.

COMPARATIF V8
-------------
Run 1 : factor=2.0, BE=0.0 (trailing immédiat, distance confortable)
Run 2 : factor=3.0, BE=0.0 (trailing large, laisse vraiment courir)
Run 3 : factor=2.0, BE=0.5 (trailing activé après 0.5×SL en faveur)
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
# ► CONFIG — À MODIFIER ICI UNIQUEMENT
# ==============================================================================

# --- Identifiants du run ---
RUN_ID       = "v8_trail_f2_be0"
MAGIC_NUMBER = "K1L2M3N4"   # ← Changer manuellement pour chaque nouveau run

# --- Fichier de données (ticks Tick Data Suite) ---
FILE_PATH  = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "NASDAQ (US100)"

# --- Horaires du range ---
RANGE_TIMEZONE    = "America/New_York"
RANGE_START_LOCAL = (9, 30)
RANGE_END_LOCAL   = (9, 35)

# --- Signal ---
SIGNAL_MODE          = "FVG"
SIGNAL_TIMEFRAME_MIN = 1
SIGNAL_WINDOW_MIN    = 90

# --- Direction ---
TRADE_DIRECTION = "BOTH"

# --- Paramètres trailing ---
# TRAILING_ATR_FACTOR : distance du trailing = N × ATR14(1min)
TRAILING_ATR_FACTOR = 2.0

# BE_FACTOR : fraction du SL à atteindre avant activation du trailing
# 0.0 = trailing immédiat | 0.5 = demi-SL | 1.0 = break-even complet
BE_FACTOR = 0.0

# --- Autres paramètres stratégie ---
MAX_TRADE_DURATION_MIN = 240   # Durée max en minutes (0 = pas de limite)

# --- Filtres ---
MAX_SPREAD_POINTS = 3.0
MIN_RANGE_POINTS  = 5.0
EOD_GAP_MINUTES   = 60

# --- Filtre années ---
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

# --- Indicateurs contextuels ---
EMA_PERIODS   = [20, 50, 100, 200]
RSI_PERIODS   = [7, 14, 21]
ATR_PERIODS   = [7, 14, 21]
BB_PERIOD     = 20
BB_STD        = 2.0
AROON_PERIODS = [14, 25]
SLOPE_N       = 5
LOOKBACK_DAYS = 30

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
# CONSTANTES TEMPORELLES
# ==============================================================================

US_PER_MIN = 60_000_000
US_PER_DAY = 86_400_000_000

# ==============================================================================
# GESTION DES HORAIRES
# ==============================================================================

_tz        = ZoneInfo(RANGE_TIMEZONE)
_tz_utc    = ZoneInfo("UTC")
_range_cache = {}

def range_bounds_us(d: date) -> tuple:
    if d in _range_cache: return _range_cache[d]
    def to_us(h, m):
        dt = datetime(d.year, d.month, d.day, h, m, tzinfo=_tz)
        return int(dt.astimezone(_tz_utc).replace(tzinfo=None)
                     .replace(tzinfo=_tz_utc).timestamp() * 1_000_000)
    _range_cache[d] = (to_us(*RANGE_START_LOCAL), to_us(*RANGE_END_LOCAL))
    return _range_cache[d]

def day_id_to_date(day_id: int) -> date:
    return date(1970, 1, 1) + timedelta(days=int(day_id))

# ==============================================================================
# PARSE TIMESTAMP VECTORISÉ (format DD.MM.YYYY HH:MM:SS.mmm)
# ==============================================================================

def parse_timestamps(ts_col: pd.Series) -> np.ndarray:
    s = ts_col
    base = pd.to_datetime({
        'year'  : s.str[6:10].astype(np.int32),
        'month' : s.str[3:5 ].astype(np.int32),
        'day'   : s.str[0:2 ].astype(np.int32),
        'hour'  : s.str[11:13].astype(np.int32),
        'minute': s.str[14:16].astype(np.int32),
        'second': s.str[17:19].astype(np.int32),
    }).values.astype('datetime64[us]').astype(np.int64)
    return base + s.str[20:23].astype(np.int32).values.astype(np.int64) * 1000

# ==============================================================================
# INDICATEURS TECHNIQUES
# ==============================================================================

class EMABuffer:
    def __init__(self, p):
        self.a=2./(p+1); self.v=None; self.n=0; self.p=p
    def update(self, x):
        self.v = x if self.v is None else self.a*x+(1-self.a)*self.v; self.n+=1
    def get(self): return self.v if self.n>=self.p else None

class RSIBuffer:
    def __init__(self, p):
        self.a=1./p; self.ag=self.al=None; self.prev=None; self.n=0; self.p=p
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
    def __init__(self, p):
        self.a=1./p; self.atr=None; self.pc=None; self.n=0; self.p=p
    def update(self, h, l, c):
        tr=(h-l) if self.pc is None else max(h-l,abs(h-self.pc),abs(l-self.pc))
        self.pc=c; self.atr=tr if self.atr is None else self.a*tr+(1-self.a)*self.atr; self.n+=1
    def get(self): return self.atr if self.n>=self.p else None

class BollingerBuffer:
    def __init__(self, p=20, n=2.):
        self.p=p; self.n=n; self.buf=deque(maxlen=p)
    def update(self, x): self.buf.append(x)
    def get(self):
        if len(self.buf)<self.p: return None,None,None,None
        a=np.array(self.buf); m=a.mean(); s=a.std(ddof=1)
        u=m+self.n*s; lo=m-self.n*s; return m,u,lo,u-lo
    def get_pct_b(self, price):
        _,_,lo,w=self.get()
        return None if w is None or w==0 else (price-lo)/w

class AroonBuffer:
    def __init__(self, p):
        self.p=p; self.hi=deque(maxlen=p+1); self.lo=deque(maxlen=p+1)
    def update(self, h, l): self.hi.append(h); self.lo.append(l)
    def get(self):
        if len(self.hi)<self.p+1: return None,None
        h=np.array(self.hi); l=np.array(self.lo)
        bsh=self.p-np.argmax(h); bsl=self.p-np.argmin(l)
        return (self.p-bsh)/self.p*100,(self.p-bsl)/self.p*100

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
        self.emas  ={p:EMABuffer(p)        for p in EMA_PERIODS}
        self.slopes={p:SlopeBuffer(SLOPE_N) for p in EMA_PERIODS}
        self.rsis  ={p:RSIBuffer(p)         for p in RSI_PERIODS}
        self.atrs  ={p:ATRBuffer(p)         for p in ATR_PERIODS}
        self.bb    =BollingerBuffer(BB_PERIOD,BB_STD)
        self.aroons={p:AroonBuffer(p)        for p in AROON_PERIODS}

    def update(self, o, h, l, c):
        for p in EMA_PERIODS:
            self.emas[p].update(c)
            v=self.emas[p].get()
            if v is not None: self.slopes[p].update(v)
        for p in RSI_PERIODS: self.rsis[p].update(c)
        for p in ATR_PERIODS: self.atrs[p].update(h,l,c)
        self.bb.update(c)
        for p in AROON_PERIODS: self.aroons[p].update(h,l)

    def snapshot(self, price, direction, suffix=''):
        s=suffix; snap={}
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
        bb_m,bb_u,bb_l,bb_w=self.bb.get()
        snap[f'bb_width{s}']=round(bb_w,4) if bb_w is not None else None
        snap[f'bb_pct_b{s}']=round(self.bb.get_pct_b(price),4) if bb_w is not None else None
        snap[f'bb_upper{s}']=round(bb_u,4) if bb_u is not None else None
        snap[f'bb_lower{s}']=round(bb_l,4) if bb_l is not None else None
        for p in AROON_PERIODS:
            up,dn=self.aroons[p].get()
            snap[f'aroon{p}_up{s}']=round(up,2) if up is not None else None
            snap[f'aroon{p}_down{s}']=round(dn,2) if dn is not None else None
        return snap

# ==============================================================================
# CONSTRUCTION BOUGIES 1MIN VECTORISÉE
# ==============================================================================

def build_1min_candles(ts_us, ask, bid):
    if len(ts_us)==0:
        e=pd.DataFrame(columns=['open','high','low','close'])
        return e,e
    min_us=(ts_us//US_PER_MIN)*US_PER_MIN
    changes=np.where(np.diff(min_us)!=0)[0]+1
    starts=np.concatenate([[0],changes]); ends=np.concatenate([changes,[len(ts_us)]])
    idx=pd.to_datetime(min_us[starts],unit='us')
    def mk(arr):
        return pd.DataFrame({
            'open':arr[starts],'high':np.maximum.reduceat(arr,starts),
            'low':np.minimum.reduceat(arr,starts),'close':arr[ends-1]
        },index=idx)
    return mk(ask),mk(bid)

# ==============================================================================
# SIMULATION TRAILING ATR — VECTORISÉE SUR BOUGIES 1MIN
# ==============================================================================

def simulate_trailing(ts_us, ask, bid, direction, sl_fixed,
                      entry_price, sl_distance_pts,
                      eod_gap_us, max_dur_us, entry_us):
    """
    Simulation TRAILING ATR vectorisée sur bougies 1min reconstruites.

    Détecte la sortie sur le LOW/HIGH de la bougie qui touche le trailing SL.
    ATR14 calculé bougie par bougie (Wilder) depuis l'entrée.
    Précision : bougie 1min (biais minimal, constant entre runs).

    Retourne (result, exit_price, exit_us)
    """
    n = len(ts_us)
    if n == 0:
        return 'CLOSE_EOD', float((ask[-1]+bid[-1])/2) if len(ask) else entry_price, entry_us

    # ── Reconstruction bougies 1min ──────────────────────────────────
    mid    = (ask.astype(np.float64) + bid.astype(np.float64)) / 2
    min_us = (ts_us // US_PER_MIN) * US_PER_MIN
    changes= np.where(np.diff(min_us) != 0)[0] + 1
    starts = np.concatenate([[0], changes])
    ends   = np.concatenate([changes, [n]])
    ts_c   = ts_us[starts]          # timestamp de chaque bougie
    highs  = np.maximum.reduceat(mid, starts)
    lows   = np.minimum.reduceat(mid, starts)
    closes = mid[ends - 1]
    nc     = len(ts_c)

    if nc == 0:
        return 'CLOSE_EOD', float(mid[-1]), int(ts_us[-1])

    # ── Vérification gap EOD et timeout par bougie ───────────────────
    # (on vérifie au niveau des ticks pour la précision)
    tick_gaps    = np.diff(ts_us)
    gap_tick_idx = np.where(tick_gaps > eod_gap_us)[0]
    if max_dur_us > 0:
        tmo_tick_idx = np.where(ts_us - entry_us > max_dur_us)[0]
    else:
        tmo_tick_idx = np.array([], dtype=np.int64)

    # Convertir en indices de bougie
    i_gap_tick = int(gap_tick_idx[0]) if len(gap_tick_idx) > 0 else n
    i_tmo_tick = int(tmo_tick_idx[0]) if len(tmo_tick_idx) > 0 else n

    # Bougie correspondante
    i_gap_c = int(np.searchsorted(starts, i_gap_tick, side='right')) - 1
    i_tmo_c = int(np.searchsorted(starts, i_tmo_tick, side='right')) - 1
    i_gap_c = max(0, min(i_gap_c, nc-1))
    i_tmo_c = max(0, min(i_tmo_c, nc-1))

    # ── Simulation bougie par bougie ─────────────────────────────────
    sl_current     = sl_fixed
    be_reached     = (BE_FACTOR == 0.0)
    price_extremum = entry_price
    atr            = 0.
    k              = 1. / 14.
    prev_c         = entry_price

    for i in range(nc):
        # Vérifier EOD gap et timeout avant de traiter la bougie
        if i >= i_gap_c and i_gap_c < nc - 1:
            eod_price = float(mid[i_gap_tick] if i_gap_tick < n else mid[-1])
            return 'CLOSE_EOD', eod_price, int(ts_us[min(i_gap_tick, n-1)])
        if i >= i_tmo_c and i_tmo_c < nc - 1:
            tmo_price = float(mid[i_tmo_tick] if i_tmo_tick < n else mid[-1])
            return 'TIMEOUT', tmo_price, int(ts_us[min(i_tmo_tick, n-1)])

        # Calcul ATR14 (Wilder) incrémental
        tr  = max(highs[i]-lows[i], abs(highs[i]-prev_c), abs(lows[i]-prev_c))
        atr = tr if i == 0 else k*tr + (1-k)*atr
        prev_c = closes[i]

        # Phase 1 : vérifier BE
        if not be_reached:
            be_threshold = BE_FACTOR * sl_distance_pts
            if direction=='BUY'  and highs[i] >= entry_price + be_threshold:
                be_reached = True; price_extremum = highs[i]
            elif direction=='SELL' and lows[i] <= entry_price - be_threshold:
                be_reached = True; price_extremum = lows[i]

        # Phase 2 : mettre à jour le trailing SL
        if be_reached and atr > 0:
            trail_dist = TRAILING_ATR_FACTOR * atr
            if direction == 'BUY':
                price_extremum = max(price_extremum, highs[i])
                sl_trail   = price_extremum - trail_dist
                sl_current = max(sl_current, sl_trail)
            else:
                price_extremum = min(price_extremum, lows[i])
                sl_trail   = price_extremum + trail_dist
                sl_current = min(sl_current, sl_trail)

        # Vérifier si le SL est touché sur cette bougie
        # TP_TRAIL : sortie trailing ET prix de sortie favorable (au-dessus entry pour BUY)
        # SL       : sortie en perte (trailing ou fixe)
        if direction=='BUY' and lows[i] <= sl_current:
            result = 'TP_TRAIL' if sl_current > entry_price else 'SL'
            return result, float(sl_current), int(ts_c[i])
        if direction=='SELL' and highs[i] >= sl_current:
            result = 'TP_TRAIL' if sl_current < entry_price else 'SL'
            return result, float(sl_current), int(ts_c[i])

    return 'CLOSE_EOD', float(mid[-1]), int(ts_us[-1])

# ==============================================================================
# CALCUL DU LOT
# ==============================================================================

def calc_lot(capital, risk_pct, sl_pts, lot_min, tol):
    if sl_pts <= 0: return 0., 'INVALID_SL'
    lot = (capital*risk_pct/100.) / sl_pts
    if lot < lot_min:
        if (lot_min*sl_pts)/(capital*risk_pct/100.) <= tol: return lot_min,'OK_ROUNDED'
        return lot_min,'LOT_TOO_SMALL'
    return lot,'OK'

# ==============================================================================
# TRAITEMENT D'UN JOUR
# ==============================================================================

def process_day(ts_us, ask, bid, indic, capital, d):
    rs_us, re_us = range_bounds_us(d)
    end_us     = rs_us + SIGNAL_WINDOW_MIN * US_PER_MIN
    eod_gap_us = EOD_GAP_MINUTES * US_PER_MIN
    max_dur_us = MAX_TRADE_DURATION_MIN * US_PER_MIN if MAX_TRADE_DURATION_MIN > 0 else 0

    # ── Range 5min ───────────────────────────────────────────────────
    mask_r = (ts_us >= rs_us) & (ts_us < re_us)
    if not mask_r.any():
        return {'date':d,'result':'NO_DATA','capital_after':capital}
    range_h = float(ask[mask_r].max())
    range_l = float(bid[mask_r].min())
    if range_h - range_l < MIN_RANGE_POINTS:
        return {'date':d,'result':'FILTERED_RANGE','capital_after':capital}

    # ── Bougies 1min ─────────────────────────────────────────────────
    oa, ob = build_1min_candles(ts_us, ask, bid)
    oa_idx = oa.index.asi8; ob_idx = ob.index.asi8

    # Mise à jour indicateurs avant la fenêtre
    mask_pre = oa_idx < re_us
    ca_pre = oa.values[mask_pre]; cb_pre = ob.values[ob_idx < re_us]
    for i in range(min(len(ca_pre),len(cb_pre))):
        indic.update((ca_pre[i,0]+cb_pre[i,0])/2,(ca_pre[i,1]+cb_pre[i,1])/2,
                     (ca_pre[i,2]+cb_pre[i,2])/2,(ca_pre[i,3]+cb_pre[i,3])/2)

    # ── Fenêtre de recherche FVG ─────────────────────────────────────
    mask_win = (oa_idx >= re_us) & (oa_idx < end_us)
    idx_b    = ob_idx[(ob_idx >= re_us) & (ob_idx < end_us)]
    common_us,ia,ib = np.intersect1d(oa_idx[mask_win],idx_b,return_indices=True)
    if len(common_us) < 3:
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    ca   = oa.values[mask_win][ia]; cb = ob.values[(ob_idx>=re_us)&(ob_idx<end_us)][ib]
    ts_c = common_us
    snap_n={}; setup=None

    for i in range(len(common_us)-2):
        indic.update((ca[i,0]+cb[i,0])/2,(ca[i,1]+cb[i,1])/2,
                     (ca[i,2]+cb[i,2])/2,(ca[i,3]+cb[i,3])/2)
        n_h_ask=ca[i,1]; n_l_bid=cb[i,2]
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
                    setup=('BUY',float(n2_c_ask),float(n1_l_bid),spread,n2_ts,float(n2_l_bid-n_h_ask)); break

        if TRADE_DIRECTION in ('SELL_ONLY','BOTH'):
            if n1_c_bid<range_l and n1_l_bid<range_l and n2_h_ask<n_l_bid:
                sl_pts=float(n1_h_ask-n2_c_bid)
                if sl_pts>0:
                    snap_n=indic.snapshot(float(n2_c_bid),'SELL','')
                    setup=('SELL',float(n2_c_bid),float(n1_h_ask),spread,n2_ts,float(n_l_bid-n2_h_ask)); break

    if setup is None:
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    direction,entry,sl_price,spread_e,n2_ts,fvg_size = setup
    sl_pts   = abs(entry-sl_price)

    # Snapshot N+2
    if i+1 < len(ca):
        indic.update((ca[i+1,0]+cb[i+1,0])/2,(ca[i+1,1]+cb[i+1,1])/2,
                     (ca[i+1,2]+cb[i+1,2])/2,(ca[i+1,3]+cb[i+1,3])/2)
    if i+2 < len(ca):
        indic.update((ca[i+2,0]+cb[i+2,0])/2,(ca[i+2,1]+cb[i+2,1])/2,
                     (ca[i+2,2]+cb[i+2,2])/2,(ca[i+2,3]+cb[i+2,3])/2)
    snap_n2 = indic.snapshot(entry, direction, '_n2')

    lot,lot_status = calc_lot(capital,RISK_PERCENT,sl_pts,LOT_MIN,LOT_TOLERANCE_RATIO)
    if lot_status=='LOT_TOO_SMALL':
        return {'date':d,'result':'LOT_TOO_SMALL','capital_after':capital}

    mask_after = ts_us > n2_ts
    if not mask_after.any():
        return {'date':d,'result':'NO_SETUP','capital_after':capital}

    result,exit_price,exit_us = simulate_trailing(
        ts_us[mask_after], ask[mask_after], bid[mask_after],
        direction, sl_price, entry, sl_pts,
        eod_gap_us, max_dur_us, n2_ts
    )

    pnl_pts = (exit_price-entry) if direction=='BUY' else (entry-exit_price)
    pnl_pct = pnl_pts/sl_pts*RISK_PERCENT
    capital_new = capital*(1+pnl_pct/100.)
    duration = (exit_us-n2_ts)/(US_PER_MIN*1.) if exit_us else None
    atr14=snap_n.get('atr14'); range_sz=range_h-range_l

    return {
        'date':d,'year':d.year,'month':d.month,
        'day_of_week':d.strftime('%A'),'day_of_week_num':d.weekday()+1,
        'week_of_month':(d.day-1)//7+1,
        'direction':direction,
        'trailing_factor':TRAILING_ATR_FACTOR,'be_factor':BE_FACTOR,
        'entry_time_gmt':pd.Timestamp(n2_ts+US_PER_MIN,unit='us'),
        'exit_time_gmt':pd.Timestamp(exit_us,unit='us') if exit_us else None,
        'duration_min':round(duration,2) if duration else None,
        'entry_price':round(entry,4),'sl_price':round(sl_price,4),
        'tp_price':round(entry+(3*sl_pts if direction=='BUY' else -3*sl_pts),4),
        'exit_price':round(float(exit_price),4),
        'sl_distance_pts':round(sl_pts,4),
        'pnl_pts':round(pnl_pts,4),'pnl_pct':round(pnl_pct,4),
        'lot_size':round(lot,4),'lot_status':lot_status,
        'capital_before':round(capital,4),'capital_after':round(capital_new,4),
        'result':result,'close_eod':result in ('CLOSE_EOD','TIMEOUT'),
        'range_size_pts':round(range_sz,4),'fvg_size_pts':round(fvg_size,4),
        'spread_entry':round(spread_e,4),
        'fvg_atr_ratio':round(fvg_size/atr14,4) if atr14 else None,
        'range_atr_ratio':round(range_sz/atr14,4) if atr14 else None,
        **snap_n,**snap_n2,
    }

# ==============================================================================
# MÉTRIQUES
# ==============================================================================

def compute_metrics(trades, label, initial_capital):
    valid=[t for t in trades if t.get('result') in ('SL','TP_TRAIL')]
    all_t=[t for t in trades if t.get('result') in ('SL','TP_TRAIL','CLOSE_EOD','TIMEOUT')]
    n=len(valid); n_all=len(all_t)
    if n_all==0: return {'label':label,'n_trades':0}
    n_tp=sum(1 for t in valid if t['result']=='TP_TRAIL')
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
    streak=ms=0
    for t in valid:
        streak=streak+1 if t['result']=='SL' else 0; ms=max(ms,streak)
    atp=np.mean([t['pnl_pts'] for t in valid if t['result']=='TP_TRAIL']) if n_tp else 0
    asl=np.mean([t['pnl_pts'] for t in valid if t['result']=='SL']) if n_sl else 0
    exp=(wr/100)*atp+(1-wr/100)*asl if n>0 else 0
    dtp=[t['duration_min'] for t in valid if t['result']=='TP_TRAIL' and t.get('duration_min')]
    dsl=[t['duration_min'] for t in valid if t['result']=='SL' and t.get('duration_min')]
    dr=list({t['date']:t['pnl_pct'] for t in all_t}.values())
    sharpe=calmar=None
    if len(dr)>1:
        mu=np.mean(dr); sig=np.std(dr,ddof=1)
        if sig>0: sharpe=round(mu/sig*np.sqrt(252),3)
        if maxdd>0: calmar=round(sum(dr)/maxdd,3)
    return {
        'label':label,'n_trades':n_all,'n_tp_trail':n_tp,'n_sl':n_sl,
        'n_close_eod':n_eod,'n_timeout':n_tmo,
        'win_rate_pct':round(wr,2),'net_pts':round(sum(pnls),2),
        'gross_profit_pts':round(gp,2),'gross_loss_pts':round(gl,2),
        'profit_factor':round(pf,3),'expectancy_pts':round(exp,2),
        'max_drawdown_pct':round(maxdd,2),'max_consec_sl':ms,
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
             f"(TRAIL:{m['n_tp_trail']} SL:{m['n_sl']} "
             f"EOD:{m['n_close_eod']} TMO:{m['n_timeout']})")
    log.info(f"  Win rate      : {m['win_rate_pct']}%")
    log.info(f"  P&L net       : {m['net_pts']:>10.2f} pts")
    log.info(f"  Profit Factor : {m['profit_factor']}")
    log.info(f"  Espérance     : {m['expectancy_pts']:.2f} pts/trade")
    log.info(f"  Drawdown max  : {m['max_drawdown_pct']}%")
    log.info(f"  Sharpe        : {m['sharpe_ratio']}")
    log.info(f"  SL consécutifs max : {m['max_consec_sl']}")
    if m['avg_dur_tp_min']: log.info(f"  Durée moy TRAIL: {m['avg_dur_tp_min']} min (p95: {m['dur_tp_p95_min']} min)")
    if m['avg_dur_sl_min']: log.info(f"  Durée moy SL   : {m['avg_dur_sl_min']} min (p95: {m['dur_sl_p95_min']} min)")

# ==============================================================================
# ÉCRITURE ANNUELLE
# ==============================================================================

def write_year(year, trades, summary_rows, initial_capital):
    if not trades: return
    slug=ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','').replace('&','')
    prefix=os.path.join(OUTPUT_DIR,f"rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_{year}")
    pd.DataFrame(trades).to_csv(f"{prefix}_trades.csv",index=False)
    monthly=[]
    for mo in sorted({t['month'] for t in trades}):
        sub=[t for t in trades if t['month']==mo]
        m=compute_metrics(sub,f"{year}-{mo:02d}",initial_capital)
        m['year']=year; m['month']=mo; monthly.append(m)
    pd.DataFrame(monthly).to_csv(f"{prefix}_monthly.csv",index=False)
    m_year=compute_metrics(trades,str(year),initial_capital)
    m_year['year']=year; summary_rows.append(m_year)
    s_path=os.path.join(OUTPUT_DIR,f"rng_{slug}_{RUN_ID}_{MAGIC_NUMBER}_summary.csv")
    pd.DataFrame(summary_rows).to_csv(s_path,index=False)
    log.info(f"\n  ── Année {year} — fichiers écrits ──")
    log.info(f"     {prefix}_trades.csv  ({len(trades)} trades)")
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
        pct=(month-1)/12*100; bl=24; fi=int(bl*pct/100)
        bar='█'*fi+'░'*(bl-fi)
        if month>1:
            el=now-self.year_t0[year]
            eta=int(el/(month-1)*(12-month+1)); em,es=divmod(eta,60)
            eta_str=f"ETA {em}m{es:02d}s"
        else: eta_str=""
        print(f"\r  {year} [{bar}] {self.MONTHS[month-1]}  "
              f"{speed_mbs:4.1f} Mo/s  {eta_str}  trades:{n_trades}     ",
              end='',flush=True)
    def done(self):
        el=time.time()-self.t0
        print(f"\r  {'─'*65}",flush=True)
        log.info(f"  ✓ Terminé en {el/60:.1f} min")

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL
# ==============================================================================

def run_backtest():
    log.info("="*70)
    log.info(f" RANGE BREAKOUT V8 — {ASSET_NAME}")
    log.info(f" Mode signal  : {SIGNAL_MODE}  |  Direction : {TRADE_DIRECTION}")
    log.info(f" Mode sortie  : TRAILING ATR (factor={TRAILING_ATR_FACTOR}, BE={BE_FACTOR}×SL)")
    log.info(f" Données      : TICKS — précision maximale (simulation sur bougies 1min)")
    log.info(f" Fichier      : {FILE_PATH}")
    noms=['Lun','Mar','Mer','Jeu','Ven','Sam','Dim']
    jours=[noms[i] for i,v in enumerate(WEEKDAY_ALLOWED) if v]
    log.info(f" Années       : {YEARS_FILTER if YEARS_FILTER else 'toutes'}  |  Jours : {jours}")
    log.info(f" Filtres      : Spread max {MAX_SPREAD_POINTS} pts  |  Range min {MIN_RANGE_POINTS} pts  |  EOD gap {EOD_GAP_MINUTES} min")
    log.info(f" RUN_ID       : {RUN_ID}  |  MAGIC_NUMBER : {MAGIC_NUMBER}")
    log.info("="*70)

    if not os.path.exists(FILE_PATH):
        log.error(f"Fichier introuvable : {FILE_PATH}"); pause(); sys.exit(1)

    file_size=os.path.getsize(FILE_PATH)
    log.info(f"\n Taille : {file_size/1e9:.2f} Go  |  Chunks : {CHUNK_SIZE:,} lignes\n")

    indic=IndicatorState(); capital=INITIAL_CAPITAL
    cur_year=None; year_trades=[]; summary_rows=[]
    cur_day_id=-1; day_ts=[]; day_ask=[]; day_bid=[]
    lookback_buf=deque(maxlen=LOOKBACK_DAYS)
    cnt_no_data=cnt_filtered=cnt_no_setup=cnt_too_small=cnt_eod=cnt_tmo=0
    prog=Progress(); bytes_read=0; speed_mbs=0.

    log.info(" Lecture unique — progression toutes les 3 secondes\n")

    def flush_day(day_id, ts_list, ask_list, bid_list):
        nonlocal capital,cur_year,year_trades,indic
        nonlocal cnt_no_data,cnt_filtered,cnt_no_setup,cnt_too_small,cnt_eod,cnt_tmo
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
                lb_oa,lb_ob=build_1min_candles(lb_ts,lb_ask,lb_bid)
                ca=lb_oa.values; cb=lb_ob.values
                for i in range(min(len(ca),len(cb))):
                    indic.update((ca[i,0]+cb[i,0])/2,(ca[i,1]+cb[i,1])/2,
                                 (ca[i,2]+cb[i,2])/2,(ca[i,3]+cb[i,3])/2)
            log.info(f"\n  Nouvelle année : {d.year} — "
                     f"indicateurs réinitialisés sur {len(lookback_buf)} jours")
        lookback_buf.append((ts_arr.copy(),ask_arr.copy(),bid_arr.copy()))
        if YEARS_FILTER and d.year not in YEARS_FILTER:
            oa,ob=build_1min_candles(ts_arr,ask_arr,bid_arr)
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
        elif r in ('SL','TP_TRAIL','CLOSE_EOD','TIMEOUT'):
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
        ts_us=parse_timestamps(chunk['timestamp'])
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
            if os.path.exists(f): all_trades.extend(pd.read_csv(f).to_dict('records'))
    if all_trades: print_metrics(compute_metrics(all_trades,"GLOBAL",INITIAL_CAPITAL))
    else: log.info("  Aucun trade.")
    log.info(f"\n Compteurs :")
    log.info(f"   Sans données   : {cnt_no_data}")
    log.info(f"   Filtrés range  : {cnt_filtered}")
    log.info(f"   Sans setup     : {cnt_no_setup}")
    log.info(f"   Lot trop petit : {cnt_too_small}")
    log.info(f"   CLOSE_EOD      : {cnt_eod}")
    log.info(f"   TIMEOUT        : {cnt_tmo}")
    log.info(f"\n{'='*70}\n Terminé.\n{'='*70}\n")

if __name__=='__main__':
    try: run_backtest()
    except Exception:
        log.error("\n"+"="*70+"\n ERREUR INATTENDUE\n"+"="*70)
        log.error(traceback.format_exc())
    finally: pause()
