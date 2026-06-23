"""
================================================================================
BACKTEST — STRATÉGIE RANGE BREAKOUT + FAIR VALUE GAP (FVG)
Version 3 — Fully vectorized, zero Python dispatch loop
================================================================================

DESCRIPTION
-----------
Stratégie de scalping sur l'ouverture US (9h30 EST) :
  1. Range de la première bougie 5min (High Ask / Low Bid, mèches incluses)
  2. Breakout confirmé par un Fair Value Gap (FVG) sur bougies 1min
  3. Market order à la fermeture de la 3e bougie du pattern FVG

POLYVALENCE
-----------
Changer FILE_PATH + ASSET_NAME + CONTRACT_VALUE pour un autre actif.
Toute la logique fonctionne identiquement (DAX, EURUSD, Gold, etc.)

FORMAT DES DONNÉES
------------------
CSV Tick Data Suite, GMT+0, NO-DST, sans header.
Colonnes : timestamp | ask | bid | vol_ask | vol_bid
Format   : DD.MM.YYYY HH:MM:SS.mmm (toujours 23 caractères, longueur fixe)

LOGIQUE DE TRADING
------------------
- Session    : lundi–vendredi uniquement
- Timing     : 9h30–9h35 EST = 14h30 GMT (EST) / 13h30 GMT (EDT)
               Changement d'heure US géré automatiquement.
- Range      : High Ask / Low Bid de la bougie 5min (mèches incluses)
- FVG BUY    : Low(N+2 bid) > High(N ask) — N+1 ferme au-dessus du range
- FVG SELL   : High(N+2 ask) < Low(N bid) — N+1 ferme en-dessous du range
- Entrée BUY : Ask de clôture de N+2
- Entrée SELL: Bid de clôture de N+2
- SL BUY     : Low N+1 mèche incluse (Bid)
- SL SELL    : High N+1 mèche incluse (Ask)
- TP         : Entry ± 2 × distance SL (R:R 2:1 fixe)
- Exécution  : simulation vectorisée (np.argmax sur array booléen)
- Gestion    : 1 trade/jour max — SL touché = fin de journée

ARCHITECTURE V3 — PERFORMANCE MAXIMALE
----------------------------------------
- Lecture unique et séquentielle du fichier
- Parse timestamp vectorisé (extraction numérique directe, évite strptime)
- Dispatch par jour : np.diff vectorisé, ZERO boucle Python sur les ticks
- Bougies 1min : np.reduceat vectorisé
- Indicateurs : accès .values directs, pas de .loc
- Simulation SL/TP : np.argmax sur arrays booléens
- Estimation : ~15-30 min pour 28 Go de données

VARIANTE RETEST (USE_RETEST_ENTRY)
-----------------------------------
Paramètre réservé pour activation future (V2 de la stratégie).

INDICATEURS CONTEXTUELS (enregistrés à chaque trade)
-----------------------------------------------------
EMA 20/50/100/200, pentes, RSI 7/14/21, ATR 14,
ratios FVG/ATR et Range/ATR, direction vs EMA200,
jour de semaine, semaine du mois, spread à l'entrée.

SORTIES
-------
Console     : progression toutes les 3s + résumés global et par année
CSV trades  : détail complet + contexte marché (40+ colonnes)
CSV annuel  : métriques par année
CSV mensuel : métriques par mois
CSV spread  : distribution spread par heure GMT (calibration filtre)
================================================================================
"""

import os, sys, time
from datetime import date, datetime, timedelta
from collections import deque

import pandas as pd
import numpy as np

# ==============================================================================
# ► CONFIG — À MODIFIER SELON L'ACTIF
# ==============================================================================

FILE_PATH    = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME   = "NASDAQ (US100)"

INITIAL_CAPITAL = 10_000.0   # Capital de départ ($)
RISK_PERCENT    = 1.0        # Risque par trade (% du capital courant)
CONTRACT_VALUE  = 20.0       # Valeur d'1 point pour 1 lot ($)
                             # NQ futures : 20$/pt | DAX CFD : ~25€/pt

USE_RETEST_ENTRY = False     # Variante retest — réservé pour V2
RISK_REWARD      = 2.0       # Ratio TP/SL fixe

MAX_SPREAD_POINTS = 4.0      # Spread max toléré à l'entrée (points)
MIN_RANGE_POINTS  = 10.0     # Range 5min minimum (filtre jours fériés/anormaux)
WEEKDAY_FILTER    = True     # Exclure samedi (5) et dimanche (6)

EMA_PERIODS  = [20, 50, 100, 200]
RSI_PERIODS  = [7, 14, 21]
ATR_PERIOD   = 14

CHUNK_SIZE   = 500_000       # Lignes par chunk
OUTPUT_DIR   = os.path.dirname(os.path.abspath(__file__))

# ==============================================================================
# CONSTANTES TEMPORELLES (tout en microsecondes — pandas 2.x datetime64[us])
# ==============================================================================
US_PER_MIN  = 60_000_000
US_PER_DAY  = 86_400_000_000

# ==============================================================================
# PARSE TIMESTAMP VECTORISÉ
# Format fixe : DD.MM.YYYY HH:MM:SS.mmm (23 caractères)
# Positions    : 0123456789012345678901 2
# ==============================================================================

def parse_timestamps_vectorized(ts_col: pd.Series) -> np.ndarray:
    """
    Parse les timestamps en microsecondes int64 depuis l'epoch Unix.
    Méthode : extraction numérique directe sur strings de longueur fixe.
    ~25% plus rapide que pd.to_datetime avec format explicite.
    """
    s = ts_col
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

    return base + ms.astype(np.int64) * 1000   # microsecondes

# ==============================================================================
# GESTION DU TEMPS ET DST US
# ==============================================================================

_dst_cache   = {}
_range_cache = {}

def is_us_dst(d: date) -> bool:
    """True si la date est en heure d'été US (EDT = UTC-4)."""
    if d in _dst_cache:
        return _dst_cache[d]
    y = d.year
    ds = date(y, 3, 1 + (6 - date(y,3,1).weekday()) % 7 + 7)
    de = date(y, 11, 1 + (6 - date(y,11,1).weekday()) % 7)
    _dst_cache[d] = ds <= d < de
    return _dst_cache[d]

def range_start_us(d: date) -> int:
    """Timestamp µs du début de la bougie de range (9h30 EST ou EDT)."""
    if d in _range_cache:
        return _range_cache[d]
    hour = 13 if is_us_dst(d) else 14
    dt   = datetime(d.year, d.month, d.day, hour, 30, 0)
    val  = int(pd.Timestamp(dt).value // 1000)
    _range_cache[d] = val
    return val

# ==============================================================================
# INDICATEURS TECHNIQUES — BUFFERS INCRÉMENTAUX
# ==============================================================================

class EMABuffer:
    def __init__(self, period):
        self.a = 2.0 / (period + 1)
        self.v = None; self.n = 0; self.p = period
    def update(self, x):
        self.v = x if self.v is None else self.a*x + (1-self.a)*self.v
        self.n += 1
    def get(self): return self.v if self.n >= self.p else None

class RSIBuffer:
    def __init__(self, period):
        self.a = 1.0/period; self.ag = self.al = None
        self.prev = None; self.n = 0; self.p = period
    def update(self, x):
        if self.prev is None: self.prev=x; return
        d=x-self.prev; self.prev=x
        g=max(d,0.); l=max(-d,0.)
        if self.ag is None: self.ag=g; self.al=l
        else:
            self.ag=self.a*g+(1-self.a)*self.ag
            self.al=self.a*l+(1-self.a)*self.al
        self.n+=1
    def get(self):
        if self.ag is None or self.n<self.p: return None
        return 100. if self.al==0 else 100.-100./(1.+self.ag/self.al)

class ATRBuffer:
    def __init__(self, period):
        self.a=1./period; self.atr=None; self.pc=None; self.n=0; self.p=period
    def update(self, h, l, c):
        tr=(h-l) if self.pc is None else max(h-l,abs(h-self.pc),abs(l-self.pc))
        self.pc=c
        self.atr=tr if self.atr is None else self.a*tr+(1-self.a)*self.atr
        self.n+=1
    def get(self): return self.atr if self.n>=self.p else None

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
        self.slopes = {p: SlopeBuffer(5) for p in EMA_PERIODS}
        self.rsis   = {p: RSIBuffer(p)   for p in RSI_PERIODS}
        self.atr    = ATRBuffer(ATR_PERIOD)

    def update(self, o, h, l, c):
        for p in EMA_PERIODS:
            self.emas[p].update(c)
            v=self.emas[p].get()
            if v is not None: self.slopes[p].update(v)
        for p in RSI_PERIODS: self.rsis[p].update(c)
        self.atr.update(h,l,c)

    def snapshot(self, price, direction):
        snap={}
        for p in EMA_PERIODS:
            v=self.emas[p].get()
            snap[f'ema{p}']=v
            snap[f'slope_ema{p}']=self.slopes[p].get()
            snap[f'price_above_ema{p}']=(int(price>v) if v is not None else None)
        for p in RSI_PERIODS: snap[f'rsi{p}']=self.rsis[p].get()
        snap['atr14']=self.atr.get()
        e200=self.emas[200].get()
        snap['trade_with_ema200']=None
        if e200 is not None:
            snap['trade_with_ema200']=int(
                (direction=='BUY' and price>e200) or
                (direction=='SELL' and price<e200))
        return snap

# ==============================================================================
# CONSTRUCTION BOUGIES 1MIN (vectorisée, timestamps en µs int64)
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
            'close': arr[ends-1]
        }, index=idx)

    return mk(ask), mk(bid)

# ==============================================================================
# SIMULATION SL/TP VECTORISÉE
# ==============================================================================

def simulate_trade(ask, bid, ts_us, direction, sl, tp):
    if direction == 'BUY':
        tp_hit = bid >= tp; sl_hit = bid <= sl
    else:
        tp_hit = ask <= tp; sl_hit = ask >= sl
    hit = tp_hit | sl_hit
    if not hit.any():
        return 'OPEN_EOD', None, None
    i = np.argmax(hit)
    r = 'TP' if tp_hit[i] else 'SL'
    return r, (tp if r=='TP' else sl), int(ts_us[i])

# ==============================================================================
# CALCUL DU LOT
# ==============================================================================

def calc_lot(capital, risk_pct, sl_pts, cv):
    return 0. if sl_pts<=0 else (capital*risk_pct/100.)/(sl_pts*cv)

# ==============================================================================
# TRAITEMENT D'UN JOUR
# ==============================================================================

def process_day(ts_us, ask, bid, indic, capital, d):
    """Traite les ticks d'un jour. ts_us en µs int64."""
    rs_us  = range_start_us(d)
    re_us  = rs_us + 5  * US_PER_MIN
    end_us = rs_us + 90 * US_PER_MIN

    # ── Range 5min ──────────────────────────────────────────────────────
    mask_r = (ts_us >= rs_us) & (ts_us < re_us)
    if not mask_r.any():
        return {'result': 'NO_DATA'}
    range_h = float(ask[mask_r].max())
    range_l = float(bid[mask_r].min())
    if range_h - range_l < MIN_RANGE_POINTS:
        return {'result': 'FILTERED_RANGE'}

    # ── Bougies 1min ────────────────────────────────────────────────────
    oa, ob = build_1min_candles_us(ts_us, ask, bid)
    if len(oa) == 0:
        return {'result': 'NO_DATA'}

    # Accès par .values pour éviter .loc (beaucoup plus rapide)
    oa_idx = oa.index.asi8           # µs int64
    ob_idx = ob.index.asi8

    # ── Mise à jour indicateurs — bougies AVANT la fenêtre de recherche ─
    # Trouver indices communs et < re_us via numpy
    mask_pre_a = oa_idx < re_us
    mask_pre_b = ob_idx < re_us
    ca_pre = oa.values[mask_pre_a]   # shape (N,4)
    cb_pre = ob.values[mask_pre_b]
    n_pre  = min(len(ca_pre), len(cb_pre))
    for i in range(n_pre):
        indic.update(
            (ca_pre[i,0]+cb_pre[i,0])/2,
            (ca_pre[i,1]+cb_pre[i,1])/2,
            (ca_pre[i,2]+cb_pre[i,2])/2,
            (ca_pre[i,3]+cb_pre[i,3])/2
        )

    # ── Fenêtre de recherche FVG ─────────────────────────────────────────
    mask_win_a = (oa_idx >= re_us) & (oa_idx < end_us)
    mask_win_b = (ob_idx >= re_us) & (ob_idx < end_us)

    # Intersection des index (numpy)
    idx_a = oa_idx[mask_win_a]
    idx_b = ob_idx[mask_win_b]
    common_us, ia, ib = np.intersect1d(idx_a, idx_b, return_indices=True)

    if len(common_us) < 3:
        return {'result': 'NO_SETUP'}

    ca = oa.values[mask_win_a][ia]
    cb = ob.values[mask_win_b][ib]
    ts_c = common_us   # µs int64

    setup = None
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
        n2_us_val= int(ts_c[i+2])

        spread = float(n2_c_ask - n2_c_bid)
        if spread > MAX_SPREAD_POINTS:
            continue

        if n1_c_ask > range_h and n1_h_ask > range_h and n2_l_bid > n_h_ask:
            sl_pts = float(n2_c_ask - n1_l_bid)
            if sl_pts > 0:
                setup = ('BUY', float(n2_c_ask), float(n1_l_bid),
                         spread, n2_us_val, float(n2_l_bid - n_h_ask))
                break

        if n1_c_bid < range_l and n1_l_bid < range_l and n2_h_ask < n_l_bid:
            sl_pts = float(n1_h_ask - n2_c_bid)
            if sl_pts > 0:
                setup = ('SELL', float(n2_c_bid), float(n1_h_ask),
                         spread, n2_us_val, float(n_l_bid - n2_h_ask))
                break

    if setup is None:
        return {'result': 'NO_SETUP'}

    direction, entry, sl_price, spread_entry, n2_us_val, fvg_size = setup
    sl_pts   = abs(entry - sl_price)
    tp_price = entry + RISK_REWARD*sl_pts if direction=='BUY' else entry - RISK_REWARD*sl_pts

    # ── Simulation SL/TP vectorisée ──────────────────────────────────────
    mask_after = ts_us > n2_us_val
    if not mask_after.any():
        return {'result': 'NO_SETUP'}

    result, exit_price, exit_us = simulate_trade(
        ask[mask_after], bid[mask_after], ts_us[mask_after],
        direction, sl_price, tp_price
    )

    lot = calc_lot(capital, RISK_PERCENT, sl_pts, CONTRACT_VALUE)
    if exit_price is None: exit_price = entry
    pnl_pts = (exit_price-entry) if direction=='BUY' else (entry-exit_price)
    pnl_usd = pnl_pts * CONTRACT_VALUE * lot
    duration = (exit_us - n2_us_val) / (1_000_000*60.) if exit_us else None

    snap    = indic.snapshot(entry, direction)
    atr_val = snap.get('atr14')
    range_sz= range_h - range_l

    return {
        'date'           : d,
        'year'           : d.year,
        'month'          : d.month,
        'direction'      : direction,
        'entry_time_gmt' : pd.Timestamp(n2_us_val, unit='us'),
        'exit_time_gmt'  : pd.Timestamp(exit_us, unit='us') if exit_us else None,
        'duration_min'   : round(duration,2) if duration else None,
        'entry_price'    : round(entry,4),
        'sl_price'       : round(sl_price,4),
        'tp_price'       : round(tp_price,4),
        'exit_price'     : round(exit_price,4),
        'sl_distance_pts': round(sl_pts,4),
        'lot_size'       : round(lot,4),
        'pnl_pts'        : round(pnl_pts,4),
        'pnl_dollars'    : round(pnl_usd,2),
        'capital_before' : round(capital,2),
        'capital_after'  : round(capital+pnl_usd,2),
        'result'         : result,
        'range_size_pts' : round(range_sz,4),
        'fvg_size_pts'   : round(fvg_size,4),
        'spread_entry'   : round(spread_entry,4),
        'day_of_week'    : d.strftime('%A'),
        'day_of_week_num': d.weekday()+1,
        'week_of_month'  : (d.day-1)//7+1,
        'fvg_atr_ratio'  : round(fvg_size/atr_val,4) if atr_val else None,
        'range_atr_ratio': round(range_sz/atr_val,4) if atr_val else None,
        **snap
    }

# ==============================================================================
# MÉTRIQUES
# ==============================================================================

def compute_metrics(trades, label="Global"):
    valid=[t for t in trades if t.get('result') in ('TP','SL')]
    n=len(valid)
    if n==0: return {'label':label,'n_trades':0}
    n_tp=sum(1 for t in valid if t['result']=='TP'); n_sl=n-n_tp
    pnls=[t['pnl_dollars'] for t in valid]
    gp=sum(p for p in pnls if p>0); gl=sum(abs(p) for p in pnls if p<0)
    wr=n_tp/n*100; pf=gp/gl if gl>0 else float('inf')
    caps=[t['capital_after'] for t in valid]
    peak=caps[0]; maxdd=0.
    for c in caps: peak=max(peak,c); maxdd=max(maxdd,(peak-c)/peak*100)
    streak=ms2=0
    for t in valid:
        streak=streak+1 if t['result']=='SL' else 0; ms2=max(ms2,streak)
    atp=np.mean([t['pnl_dollars'] for t in valid if t['result']=='TP']) if n_tp else 0
    asl=np.mean([t['pnl_dollars'] for t in valid if t['result']=='SL']) if n_sl else 0
    exp=(wr/100)*atp+(1-wr/100)*asl
    dtp=[t['duration_min'] for t in valid if t['result']=='TP' and t.get('duration_min')]
    dsl=[t['duration_min'] for t in valid if t['result']=='SL' and t.get('duration_min')]
    return {
        'label':label,'n_trades':n,'n_tp':n_tp,'n_sl':n_sl,
        'win_rate_pct':round(wr,2),
        'net_profit_$':round(sum(pnls),2),
        'gross_profit_$':round(gp,2),'gross_loss_$':round(gl,2),
        'profit_factor':round(pf,3),
        'expectancy_$/trade':round(exp,2),
        'max_drawdown_pct':round(maxdd,2),
        'max_consec_sl':ms2,
        'avg_dur_tp_min':round(np.mean(dtp),1) if dtp else None,
        'avg_dur_sl_min':round(np.mean(dsl),1) if dsl else None,
    }

def print_metrics(m):
    if m['n_trades']==0: print(f"    {m['label']} : aucun trade"); return
    print(f"\n  ── {m['label']} ──")
    print(f"  Trades        : {m['n_trades']}  (TP:{m['n_tp']} SL:{m['n_sl']})")
    print(f"  Win rate      : {m['win_rate_pct']}%")
    print(f"  Profit net    : ${m['net_profit_$']:>12,.2f}")
    print(f"  Profit Factor : {m['profit_factor']}")
    print(f"  Espérance     : ${m['expectancy_$/trade']:.2f}/trade")
    print(f"  Drawdown max  : {m['max_drawdown_pct']}%")
    print(f"  SL consécutifs max : {m['max_consec_sl']}")
    if m['avg_dur_tp_min']: print(f"  Durée moy TP  : {m['avg_dur_tp_min']} min")
    if m['avg_dur_sl_min']: print(f"  Durée moy SL  : {m['avg_dur_sl_min']} min")

# ==============================================================================
# BARRE DE PROGRESSION
# ==============================================================================

class Progress:
    def __init__(self, total):
        self.total=total; self.t0=time.time(); self.last=0.
    def update(self, br, extra=""):
        now=time.time()
        if now-self.last<3.: return
        self.last=now
        pct=min(br/self.total*100,100.) if self.total>0 else 0
        el=now-self.t0; sp=br/el/1e6 if el>0 else 0
        eta=int((self.total-br)/(br/el)) if br>0 else 0
        em,es=divmod(eta,60); bl=28; fi=int(bl*pct/100)
        print(f"\r  [{'█'*fi+'░'*(bl-fi)}] {pct:5.1f}%  {sp:5.1f} Mo/s  "
              f"ETA {em}m{es:02d}s  {extra}     ",end='',flush=True)
    def done(self, n):
        el=time.time()-self.t0
        print(f"\r  {'─'*65}",flush=True)
        print(f"  ✓ Terminé en {el/60:.1f} min  |  {n} trades détectés")

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL
# ==============================================================================

def run_backtest():
    print("="*70)
    print(f" BACKTEST — {ASSET_NAME}  (v3)")
    print(f" Capital : ${INITIAL_CAPITAL:,.0f}  |  Risque : {RISK_PERCENT}%  |  R:R {RISK_REWARD}:1")
    print(f" Fichier : {FILE_PATH}")
    print(f" Variante retest : {'OUI' if USE_RETEST_ENTRY else 'NON'}")
    print(f" Filtres — Spread max : {MAX_SPREAD_POINTS} pts  |  Range min : {MIN_RANGE_POINTS} pts")
    print("="*70)

    if not os.path.exists(FILE_PATH):
        print(f"\n[ERREUR] Fichier introuvable :\n  {FILE_PATH}"); sys.exit(1)

    file_size = os.path.getsize(FILE_PATH)
    print(f"\n Taille fichier : {file_size/1e9:.2f} Go  |  Chunks : {CHUNK_SIZE:,} lignes\n")

    capital    = INITIAL_CAPITAL
    all_trades = []
    indic      = IndicatorState()

    # Buffer journalier — stocké en arrays pré-alloués pour éviter append
    cur_day_id = -1
    day_ts     = []
    day_ask    = []
    day_bid    = []

    spread_rows = []
    cnt_no_data = cnt_filtered = cnt_no_setup = 0
    prog        = Progress(file_size)
    bytes_read  = 0

    print(" Lecture unique — progression toutes les 3 secondes\n")

    def flush_day(day_id, ts_list, ask_list, bid_list):
        """Traite le buffer d'un jour complet."""
        nonlocal capital, cnt_no_data, cnt_filtered, cnt_no_setup
        if not ts_list or day_id < 0: return
        d = date.fromtimestamp(day_id * 86400)   # date depuis day_id
        if WEEKDAY_FILTER and d.weekday() >= 5: return

        res = process_day(
            np.array(ts_list,  dtype=np.int64),
            np.array(ask_list, dtype=np.float32),
            np.array(bid_list, dtype=np.float32),
            indic, capital, d
        )
        r = res.get('result','')
        if   r == 'NO_DATA':         cnt_no_data  += 1
        elif r == 'FILTERED_RANGE':  cnt_filtered += 1
        elif r == 'NO_SETUP':        cnt_no_setup += 1
        elif r in ('TP','SL','OPEN_EOD'):
            capital = res['capital_after']
            all_trades.append(res)

    reader = pd.read_csv(
        FILE_PATH, header=None,
        names=['timestamp','ask','bid'],
        usecols=[0,1,2],
        chunksize=CHUNK_SIZE,
        dtype={'ask':'float32','bid':'float32'},
        engine='c'
    )

    for chunk in reader:
        bytes_read += len(chunk) * 28
        prog.update(bytes_read,
                    extra=f"| {date.fromtimestamp(cur_day_id*86400) if cur_day_id>=0 else '...'} "
                          f"| trades:{len(all_trades)}")

        # Parse timestamp vectorisé
        ts_us    = parse_timestamps_vectorized(chunk['timestamp'])
        ask_vals = chunk['ask'].values
        bid_vals = chunk['bid'].values

        # Spread sample 1/500
        for i in range(0, len(chunk), 500):
            h = int(ts_us[i] // 3_600_000_000) % 24
            spread_rows.append({'hour': h, 'spread': float(ask_vals[i]-bid_vals[i])})

        # Dispatch vectorisé par jour : ZERO boucle Python sur les ticks
        day_ids  = ts_us // US_PER_DAY           # ID entier du jour (µs/86400µs)
        changes  = np.where(np.diff(day_ids) != 0)[0] + 1
        seg_starts = np.concatenate([[0], changes])
        seg_ends   = np.concatenate([changes, [len(ts_us)]])

        for s, e in zip(seg_starts, seg_ends):
            seg_day_id = int(day_ids[s])
            if seg_day_id != cur_day_id:
                flush_day(cur_day_id, day_ts, day_ask, day_bid)
                cur_day_id = seg_day_id
                day_ts  = []
                day_ask = []
                day_bid = []
            day_ts.extend(ts_us[s:e].tolist())
            day_ask.extend(ask_vals[s:e].tolist())
            day_bid.extend(bid_vals[s:e].tolist())

    flush_day(cur_day_id, day_ts, day_ask, day_bid)
    prog.done(len(all_trades))

    # ── Résultats ────────────────────────────────────────────────────────
    print(f"\n{'='*70}\n RÉSULTATS\n{'='*70}")
    print(f"  Trades         : {len(all_trades)}")
    print(f"  Sans données   : {cnt_no_data}")
    print(f"  Filtrés range  : {cnt_filtered}")
    print(f"  Sans setup     : {cnt_no_setup}")
    print(f"  Capital final  : ${capital:,.2f}")
    print(f"  Rendement      : {(capital/INITIAL_CAPITAL-1)*100:.1f}%")

    if not all_trades:
        print("\n  Aucun trade — vérifiez les paramètres."); return

    print_metrics(compute_metrics(all_trades,"GLOBAL"))

    print(f"\n{'='*70}\n RÉSULTATS PAR ANNÉE\n{'='*70}")
    years=sorted({t['year'] for t in all_trades}); yearly=[]
    for y in years:
        sub=[t for t in all_trades if t['year']==y]
        m=compute_metrics(sub,str(y)); print_metrics(m)
        m['year']=y; yearly.append(m)

    monthly=[]
    for y,mo in sorted({(t['year'],t['month']) for t in all_trades}):
        sub=[t for t in all_trades if t['year']==y and t['month']==mo]
        m=compute_metrics(sub,f"{y}-{mo:02d}"); m['year']=y; m['month']=mo
        monthly.append(m)

    if spread_rows:
        sdf=pd.DataFrame(spread_rows)
        sagg=sdf.groupby('hour')['spread'].agg(
            count='count',mean='mean',median='median',
            p95=lambda x:x.quantile(0.95),
            p99=lambda x:x.quantile(0.99),max='max').round(4)
        print(f"\n{'='*70}\n SPREAD PAR HEURE GMT\n{'='*70}")
        print(sagg.to_string())
    else: sagg=pd.DataFrame()

    slug=(ASSET_NAME.replace(' ','_').replace('(','')
          .replace(')','').replace('/',''))
    prefix=os.path.join(OUTPUT_DIR,f"backtest_{slug}")
    pd.DataFrame(all_trades).to_csv(f"{prefix}_trades.csv", index=False)
    pd.DataFrame(yearly).to_csv(f"{prefix}_yearly.csv",     index=False)
    pd.DataFrame(monthly).to_csv(f"{prefix}_monthly.csv",   index=False)
    if not sagg.empty: sagg.to_csv(f"{prefix}_spread.csv")

    print(f"\n Fichiers : {OUTPUT_DIR}")
    print(f"   backtest_{slug}_trades/yearly/monthly/spread .csv")
    print(f"\n{'='*70}\n Backtest terminé.\n{'='*70}\n")

if __name__ == '__main__':
    run_backtest()

