# -*- coding: utf-8 -*-
"""
================================================================================
SCRIPT 1 — ANALYSE EXPLORATOIRE DES DONNÉES TICK
Stratégie Range Breakout 9h30 EST + Fair Value Gap
================================================================================

OBJECTIF
--------
Analyser les données tick d'un actif pour proposer des valeurs informées
pour les paramètres du backtest. Ce script ne trade pas — il observe et mesure.

RÉSULTATS
---------
Ce script produit un rapport complet permettant de décider collectivement :
  - MAX_SPREAD_POINTS  : seuil de spread à l'ouverture US
  - MIN_RANGE_POINTS   : taille minimale du range 5min
  - CONTRACT_VALUE     : valeur empirique d'un point pour 1 lot
  - EOD_GAP_MINUTES    : durée de coupure nocturne typique
  - SESSION_END_GMT    : heure de fin de session réelle dans les données

WALK-FORWARD
------------
Les statistiques sont calculées sur 70% des jours (période d'entraînement)
puis vérifiées sur les 30% restants (période de test).
Une alerte est émise si les distributions diffèrent significativement.

POLYVALENCE
-----------
Changer FILE_PATH + ASSET_NAME pour analyser un autre actif.
Toute la logique s'adapte automatiquement.

FORMAT DES DONNÉES
------------------
CSV Tick Data Suite, GMT+0, NO-DST, sans header.
Colonnes : timestamp | ask | bid | vol_ask | vol_bid
Format   : DD.MM.YYYY HH:MM:SS.mmm (23 caractères fixes)

PARAMÈTRES À RENSEIGNER
------------------------
Après lecture de ce rapport, les paramètres suivants devront être validés
avant de lancer le backtest :
  - MAX_SPREAD_POINTS        (calculé ici)
  - MIN_RANGE_POINTS         (calculé ici)
  - CONTRACT_VALUE           (calculé ici — à confirmer via trade test)
  - EOD_GAP_MINUTES          (calculé ici)
  - SESSION_END_GMT          (calculé ici)
  - COMMISSION_PER_LOT       (non calculable — vérifier chez le broker)
  - SYMBOL_MT5_NAME          (à confirmer dans MT5)
  - REFERENCE_CAPITAL        (décision du trader)
  - LOT_TOLERANCE_RATIO      (décision du trader)
================================================================================
"""

import os
import sys
import time
import json
import logging
import traceback
from datetime import date, datetime, timedelta
from collections import defaultdict

import pandas as pd
import numpy as np

# ==============================================================================
# CONFIG — À MODIFIER SELON L'ACTIF
# ==============================================================================

FILE_PATH  = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "NASDAQ (US100)"

# Heure de range 9h30 EST — gérée dynamiquement (DST US)
# EST = UTC-5 → 14h30 GMT | EDT = UTC-4 → 13h30 GMT

CHUNK_SIZE   = 500_000
OUTPUT_DIR   = os.path.dirname(os.path.abspath(__file__))

# ==============================================================================
# CONSTANTES TEMPORELLES (microsecondes — pandas 2.x datetime64[us])
# ==============================================================================

US_PER_MIN = 60_000_000
US_PER_DAY = 86_400_000_000

# ==============================================================================
# LOGGING
# ==============================================================================

LOG_PATH = os.path.join(OUTPUT_DIR, f"analyse_exploratoire_{ASSET_NAME.split()[0]}.log")

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
    log.error("ERREUR FATALE:\n" + "".join(__import__('traceback').format_exception(t,v,tb))),
    pause()
)

# ==============================================================================
# GESTION DU TEMPS ET DST US
# ==============================================================================

_dst_cache   = {}
_range_cache = {}

def is_us_dst(d: date) -> bool:
    if d in _dst_cache: return _dst_cache[d]
    y = d.year
    ds = date(y, 3, 1 + (6 - date(y,3,1).weekday()) % 7 + 7)
    de = date(y, 11, 1 + (6 - date(y,11,1).weekday()) % 7)
    _dst_cache[d] = ds <= d < de
    return _dst_cache[d]

def range_start_us(d: date) -> int:
    if d in _range_cache: return _range_cache[d]
    hour = 13 if is_us_dst(d) else 14
    dt   = datetime(d.year, d.month, d.day, hour, 30, 0)
    val  = int(pd.Timestamp(dt).value // 1000)
    _range_cache[d] = val
    return val

def day_id_to_date(day_id: int) -> date:
    """Conversion robuste day_id → date (pas de dépendance au fuseau local)."""
    return date(1970, 1, 1) + timedelta(days=day_id)

# ==============================================================================
# PARSE TIMESTAMP VECTORISÉ
# Format fixe DD.MM.YYYY HH:MM:SS.mmm
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
        'year':year,'month':mon,'day':day,
        'hour':hour,'minute':minu,'second':sec
    }).values.astype('datetime64[us]').astype(np.int64)
    return base + ms.astype(np.int64) * 1000

# ==============================================================================
# ANALYSE D'UN JOUR
# ==============================================================================

def analyse_day(ts_us, ask, bid, d: date) -> dict | None:
    """
    Analyse un jour de données tick.
    Retourne un dict de métriques ou None si données insuffisantes.
    """
    rs_us = range_start_us(d)
    re_us = rs_us + 5 * US_PER_MIN

    # ── Spread global sur la journée ─────────────────────────────────
    spreads_all = (ask - bid).astype(np.float32)

    # ── Spread à l'ouverture US (fenêtre 14h20–14h50 GMT) ───────────
    # On prend ±20 min autour de 9h30 EST pour avoir une fenêtre représentative
    window_start = rs_us - 10 * US_PER_MIN
    window_end   = rs_us + 20 * US_PER_MIN
    mask_open    = (ts_us >= window_start) & (ts_us < window_end)
    spreads_open = spreads_all[mask_open]

    # ── Range 5min ───────────────────────────────────────────────────
    mask_range = (ts_us >= rs_us) & (ts_us < re_us)
    if not mask_range.any():
        return None   # Pas de données à l'ouverture US

    range_h    = float(ask[mask_range].max())
    range_l    = float(bid[mask_range].min())
    range_size = range_h - range_l
    n_ticks_range = int(mask_range.sum())

    # ── Détection de la coupure nocturne (EOD gap) ───────────────────
    # On cherche le plus grand gap entre ticks consécutifs sur la journée
    if len(ts_us) > 1:
        gaps_us  = np.diff(ts_us)
        max_gap  = float(gaps_us.max()) / US_PER_MIN   # en minutes
        # Position du gap max
        gap_idx  = int(np.argmax(gaps_us))
        gap_hour = int(ts_us[gap_idx] // 3_600_000_000) % 24
    else:
        max_gap  = 0.0
        gap_hour = -1

    # ── CONTRACT_VALUE empirique ─────────────────────────────────────
    # On mesure la variation de prix mid sur des intervalles de 1 minute
    # La valeur du point = variation_prix / variation_theorique
    # Pour un CFD Mode 2 : contract_value ≈ prix_mid_moyen × point
    # On utilise le prix mid moyen à l'ouverture comme référence
    mid_open = float((ask[mask_range].mean() + bid[mask_range].mean()) / 2)
    # Mode 2 MT5 : tick_size = point = 0.01, contract_size = 1
    # contract_value = prix_mid × contract_size × point / tick_size
    # = prix_mid × 1 × 0.01 / 0.01 = prix_mid
    # MAIS nos données montrent 0.88$/pt → il y a un facteur de normalisation
    # On mesure empiriquement : contract_value_empirique = tick_value / volume_min / tick_size * point
    # Avec tick_value calculé depuis le prix : tick_value = prix × contract_size × tick_size
    # Donc : cv = prix × contract_size × tick_size / volume_min / tick_size * point
    #           = prix × contract_size × point / volume_min
    #           = prix × 1 × 0.01 / 0.01 = prix
    # Ce qui donne ~29500 — trop élevé. La valeur réelle (0.88) suggère un facteur /~33000
    # → On stocke le prix mid et on laisse la calibration se faire après analyse
    mid_price = mid_open

    # ── Heure du dernier tick ────────────────────────────────────────
    last_tick_hour   = int(ts_us[-1] // 3_600_000_000) % 24
    last_tick_minute = int((ts_us[-1] % 3_600_000_000) // US_PER_MIN)

    return {
        'date'               : d,
        'year'               : d.year,
        'month'              : d.month,
        'weekday'            : d.weekday(),   # 0=lun, 4=ven
        'n_ticks'            : len(ts_us),
        'n_ticks_range'      : n_ticks_range,
        'range_size_pts'     : round(range_size, 2),
        'range_h'            : round(range_h, 2),
        'range_l'            : round(range_l, 2),
        'mid_price_open'     : round(mid_price, 2),
        # Spread global
        'spread_day_mean'    : round(float(spreads_all.mean()), 4) if len(spreads_all) else None,
        'spread_day_median'  : round(float(np.median(spreads_all)), 4) if len(spreads_all) else None,
        'spread_day_p95'     : round(float(np.percentile(spreads_all, 95)), 4) if len(spreads_all) else None,
        'spread_day_max'     : round(float(spreads_all.max()), 4) if len(spreads_all) else None,
        # Spread ouverture US
        'spread_open_mean'   : round(float(spreads_open.mean()), 4) if len(spreads_open) else None,
        'spread_open_median' : round(float(np.median(spreads_open)), 4) if len(spreads_open) else None,
        'spread_open_p95'    : round(float(np.percentile(spreads_open, 95)), 4) if len(spreads_open) else None,
        'spread_open_max'    : round(float(spreads_open.max()), 4) if len(spreads_open) else None,
        # EOD gap
        'max_gap_min'        : round(max_gap, 1),
        'max_gap_hour_gmt'   : gap_hour,
        # Session
        'last_tick_hour_gmt' : last_tick_hour,
        'last_tick_min_gmt'  : last_tick_minute,
        # DST
        'is_dst'             : is_us_dst(d),
    }

# ==============================================================================
# BARRE DE PROGRESSION
# ==============================================================================

class Progress:
    def __init__(self, total):
        self.total = total; self.t0 = time.time(); self.last = 0.
    def update(self, br, extra=""):
        now = time.time()
        if now - self.last < 3.: return
        self.last = now
        pct = min(br/self.total*100, 100.) if self.total > 0 else 0
        el  = now - self.t0
        sp  = br/el/1e6 if el > 0 else 0
        eta = int((self.total-br)/(br/el)) if br > 0 else 0
        em, es = divmod(eta, 60)
        bl = 28; fi = int(bl*pct/100)
        print(f"\r  [{'█'*fi+'░'*(bl-fi)}] {pct:5.1f}%  {sp:5.1f} Mo/s  "
              f"ETA {em}m{es:02d}s  {extra}     ", end='', flush=True)
    def done(self, n_days):
        el = time.time() - self.t0
        print(f"\r  {'─'*65}", flush=True)
        print(f"  ✓ Terminé en {el/60:.1f} min  |  {n_days} jours analysés")

# ==============================================================================
# ANALYSE STATISTIQUE ET WALK-FORWARD
# ==============================================================================

def compute_stats(days: list, label: str) -> dict:
    """Calcule les statistiques descriptives sur une liste de jours."""
    if not days:
        return {'label': label, 'n_days': 0}

    df = pd.DataFrame(days)
    df = df[df['n_ticks_range'] > 0].copy()   # Exclure jours sans données
    n  = len(df)

    def pct(col, p):
        v = df[col].dropna()
        return round(float(np.percentile(v, p)), 4) if len(v) else None

    def stats(col):
        v = df[col].dropna()
        if len(v) == 0: return {}
        return {
            'mean'  : round(float(v.mean()), 4),
            'median': round(float(v.median()), 4),
            'p10'   : round(float(v.quantile(0.10)), 4),
            'p25'   : round(float(v.quantile(0.25)), 4),
            'p75'   : round(float(v.quantile(0.75)), 4),
            'p90'   : round(float(v.quantile(0.90)), 4),
            'p95'   : round(float(v.quantile(0.95)), 4),
            'p99'   : round(float(v.quantile(0.99)), 4),
            'max'   : round(float(v.max()), 4),
            'min'   : round(float(v.min()), 4),
        }

    # Jours sans données (pas de ticks à l'ouverture US)
    n_no_data  = sum(1 for d in days if d is None or d.get('n_ticks_range', 0) == 0)

    # Distribution des gaps (coupure nocturne)
    gaps = df['max_gap_min'].dropna()
    gaps_significant = gaps[gaps > 30]

    return {
        'label'         : label,
        'n_days_total'  : len(days),
        'n_days_valid'  : n,
        'n_days_no_data': n_no_data,
        'date_start'    : str(df['date'].min()),
        'date_end'      : str(df['date'].max()),

        'range_size'    : stats('range_size_pts'),
        'spread_open'   : stats('spread_open_mean'),
        'spread_day'    : stats('spread_day_mean'),
        'mid_price'     : stats('mid_price_open'),

        'eod_gap': {
            'pct_days_with_gap_gt30min' : round(len(gaps_significant) / n * 100, 1) if n > 0 else 0,
            'typical_gap_hour_gmt'      : int(df['max_gap_hour_gmt'].mode().iloc[0]) if n > 0 else None,
            'gap_duration_median_min'   : round(float(gaps_significant.median()), 1) if len(gaps_significant) else 0,
        },

        'session_end': {
            'last_tick_hour_mode'   : int(df['last_tick_hour_gmt'].mode().iloc[0]) if n > 0 else None,
            'last_tick_hour_p5'     : pct('last_tick_hour_gmt', 5),
            'last_tick_hour_p95'    : pct('last_tick_hour_gmt', 95),
        },

        'dst_pct': round(df['is_dst'].mean() * 100, 1) if n > 0 else 0,
    }


def suggest_params(train_stats: dict, test_stats: dict) -> dict:
    """
    Propose des valeurs de paramètres basées sur les statistiques
    et vérifie la stabilité entre train et test.
    """
    suggestions = {}
    warnings    = []

    # MAX_SPREAD_POINTS : p95 du spread à l'ouverture + marge 20%
    sp_train = train_stats['spread_open'].get('p95')
    sp_test  = test_stats['spread_open'].get('p95')
    if sp_train:
        suggested_spread = round(sp_train * 1.2, 2)
        suggestions['MAX_SPREAD_POINTS'] = {
            'valeur_proposee' : suggested_spread,
            'base'            : f"p95 spread ouverture × 1.2 = {sp_train} × 1.2",
            'train_p95'       : sp_train,
            'test_p95'        : sp_test,
        }
        if sp_test and abs(sp_test - sp_train) / sp_train > 0.3:
            warnings.append(
                f"⚠ ALERTE WALK-FORWARD — MAX_SPREAD_POINTS : "
                f"p95 train={sp_train} vs test={sp_test} "
                f"(écart {abs(sp_test-sp_train)/sp_train*100:.0f}% > 30%)"
            )

    # MIN_RANGE_POINTS : p10 de la taille du range (exclure les jours anormaux)
    rng_train = train_stats['range_size'].get('p10')
    rng_test  = test_stats['range_size'].get('p10')
    if rng_train:
        suggested_range = round(rng_train * 0.8, 1)
        suggestions['MIN_RANGE_POINTS'] = {
            'valeur_proposee' : suggested_range,
            'base'            : f"p10 range × 0.8 = {rng_train} × 0.8",
            'train_p10'       : rng_train,
            'test_p10'        : rng_test,
        }
        if rng_test and abs(rng_test - rng_train) / rng_train > 0.3:
            warnings.append(
                f"⚠ ALERTE WALK-FORWARD — MIN_RANGE_POINTS : "
                f"p10 train={rng_train} vs test={rng_test} "
                f"(écart {abs(rng_test-rng_train)/rng_train*100:.0f}% > 30%)"
            )

    # EOD_GAP_MINUTES
    gap_train = train_stats['eod_gap'].get('gap_duration_median_min', 0)
    if gap_train > 0:
        suggested_gap = max(30, round(gap_train * 0.5, 0))
    else:
        suggested_gap = 30
    suggestions['EOD_GAP_MINUTES'] = {
        'valeur_proposee' : int(suggested_gap),
        'base'            : f"50% de la durée médiane de coupure ({gap_train} min)",
        'pct_jours_avec_coupure' : train_stats['eod_gap'].get('pct_days_with_gap_gt30min'),
    }

    # SESSION_END_GMT
    last_h = train_stats['session_end'].get('last_tick_hour_mode')
    suggestions['SESSION_END_GMT'] = {
        'valeur_proposee' : last_h,
        'base'            : "heure du dernier tick (mode statistique)",
    }

    # CONTRACT_VALUE — note méthodologique
    mid_mean = train_stats['mid_price'].get('mean')
    suggestions['CONTRACT_VALUE'] = {
        'valeur_theorique_mt5' : 0.880693,
        'note'                 : (
            "Valeur issue du script get_symbol_params.py sur RaiseFX Demo04. "
            "Les données Tick Data Suite (Dukascopy) peuvent donner une valeur "
            "légèrement différente. A confirmer via trade test en démo. "
            f"Prix mid moyen à l'ouverture sur la période : {mid_mean}"
        ),
    }

    return suggestions, warnings


def check_walk_forward_stability(train_stats: dict, test_stats: dict) -> list:
    """Vérifie la stabilité des distributions entre train et test."""
    alerts = []

    pairs = [
        ('range_size',   'mean',  0.25, "Taille range"),
        ('spread_open',  'mean',  0.30, "Spread ouverture"),
        ('spread_day',   'mean',  0.30, "Spread journalier"),
    ]
    for key, metric, threshold, label in pairs:
        v_train = train_stats.get(key, {}).get(metric)
        v_test  = test_stats.get(key, {}).get(metric)
        if v_train and v_test and v_train > 0:
            ecart = abs(v_test - v_train) / v_train
            if ecart > threshold:
                alerts.append(
                    f"⚠ ALERTE WALK-FORWARD — {label} : "
                    f"train={v_train:.4f} test={v_test:.4f} "
                    f"écart={ecart*100:.0f}% > {threshold*100:.0f}%"
                )
    return alerts

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL
# ==============================================================================

def run_analysis():
    log.info("="*70)
    log.info(f" ANALYSE EXPLORATOIRE — {ASSET_NAME}")
    log.info(f" Fichier : {FILE_PATH}")
    log.info("="*70)

    if not os.path.exists(FILE_PATH):
        log.error(f"Fichier introuvable : {FILE_PATH}")
        pause(); sys.exit(1)

    file_size  = os.path.getsize(FILE_PATH)
    log.info(f" Taille : {file_size/1e9:.2f} Go  |  Chunks : {CHUNK_SIZE:,} lignes")
    log.info(" Lecture unique — progression toutes les 3 secondes\n")

    # Buffers journaliers
    cur_day_id = -1
    day_ts     = []
    day_ask    = []
    day_bid    = []
    all_days   = []

    bytes_read = 0
    prog       = Progress(file_size)

    reader = pd.read_csv(
        FILE_PATH, header=None,
        names=['timestamp','ask','bid'],
        usecols=[0,1,2],
        chunksize=CHUNK_SIZE,
        dtype={'ask':'float32','bid':'float32'},
        engine='c'
    )

    def flush_day(day_id, ts_list, ask_list, bid_list):
        if not ts_list or day_id < 0: return
        d = day_id_to_date(day_id)
        if d.weekday() >= 5: return   # Exclure week-end

        ts_arr  = np.array(ts_list,  dtype=np.int64)
        ask_arr = np.array(ask_list, dtype=np.float32)
        bid_arr = np.array(bid_list, dtype=np.float32)

        result = analyse_day(ts_arr, ask_arr, bid_arr, d)
        if result is not None:
            all_days.append(result)

    for chunk in reader:
        bytes_read += len(chunk) * 28
        prog.update(bytes_read,
                    extra=f"| {day_id_to_date(cur_day_id) if cur_day_id>=0 else '...'} "
                          f"| jours:{len(all_days)}")

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
                day_ts  = []
                day_ask = []
                day_bid = []
            day_ts.extend(ts_us[s:e].tolist())
            day_ask.extend(ask_vals[s:e].tolist())
            day_bid.extend(bid_vals[s:e].tolist())

    flush_day(cur_day_id, day_ts, day_ask, day_bid)
    prog.done(len(all_days))

    # ── Walk-forward 70/30 ───────────────────────────────────────────
    n_total  = len(all_days)
    n_train  = int(n_total * 0.70)
    train    = all_days[:n_train]
    test     = all_days[n_train:]

    log.info(f"\n Jours analysés : {n_total}")
    log.info(f" Période train  : {all_days[0]['date']} → {all_days[n_train-1]['date']} ({n_train} jours)")
    log.info(f" Période test   : {all_days[n_train]['date']} → {all_days[-1]['date']} ({len(test)} jours)")

    train_stats = compute_stats(train, "TRAIN (70%)")
    test_stats  = compute_stats(test,  "TEST (30%)")
    full_stats  = compute_stats(all_days, "GLOBAL")

    # ── Alertes walk-forward ─────────────────────────────────────────
    wf_alerts = check_walk_forward_stability(train_stats, test_stats)

    # ── Propositions de paramètres ───────────────────────────────────
    suggestions, param_warnings = suggest_params(train_stats, test_stats)
    all_alerts = wf_alerts + param_warnings

    # ── Affichage résultats ──────────────────────────────────────────
    log.info(f"\n{'='*70}")
    log.info(f" STATISTIQUES GLOBALES — {ASSET_NAME}")
    log.info(f"{'='*70}")

    def print_stats(stats, label):
        log.info(f"\n  ── {label} ({stats.get('n_days_valid',0)} jours valides) ──")
        log.info(f"  Période        : {stats.get('date_start')} → {stats.get('date_end')}")
        log.info(f"  Jours sans données : {stats.get('n_days_no_data',0)}")
        log.info(f"  Jours en DST (EDT) : {stats.get('dst_pct',0)}%")

        log.info(f"\n  RANGE 5MIN (points)")
        r = stats.get('range_size', {})
        log.info(f"    Médiane={r.get('median')}  Moyenne={r.get('mean')}  "
                 f"p10={r.get('p10')}  p25={r.get('p25')}  "
                 f"p75={r.get('p75')}  p95={r.get('p95')}  Max={r.get('max')}")

        log.info(f"\n  SPREAD À L'OUVERTURE US (points)")
        s = stats.get('spread_open', {})
        log.info(f"    Médiane={s.get('median')}  Moyenne={s.get('mean')}  "
                 f"p75={s.get('p75')}  p90={s.get('p90')}  "
                 f"p95={s.get('p95')}  p99={s.get('p99')}  Max={s.get('max')}")

        log.info(f"\n  SPREAD JOURNALIER MOYEN (points)")
        sd = stats.get('spread_day', {})
        log.info(f"    Médiane={sd.get('median')}  Moyenne={sd.get('mean')}  "
                 f"p95={sd.get('p95')}  Max={sd.get('max')}")

        log.info(f"\n  PRIX MID À L'OUVERTURE US")
        m = stats.get('mid_price', {})
        log.info(f"    Min={m.get('min')}  Médiane={m.get('median')}  "
                 f"Moyenne={m.get('mean')}  Max={m.get('max')}")

        log.info(f"\n  COUPURE NOCTURNE (EOD GAP)")
        g = stats.get('eod_gap', {})
        log.info(f"    Jours avec gap > 30min : {g.get('pct_days_with_gap_gt30min')}%")
        log.info(f"    Heure typique du gap   : {g.get('typical_gap_hour_gmt')}h GMT")
        log.info(f"    Durée médiane du gap   : {g.get('gap_duration_median_min')} min")

        log.info(f"\n  FIN DE SESSION (dernier tick)")
        se = stats.get('session_end', {})
        log.info(f"    Heure mode : {se.get('last_tick_hour_mode')}h GMT")
        log.info(f"    p5–p95     : {se.get('last_tick_hour_p5')}h – "
                 f"{se.get('last_tick_hour_p95')}h GMT")

    print_stats(full_stats,  "GLOBAL")
    print_stats(train_stats, "TRAIN 70%")
    print_stats(test_stats,  "TEST 30%")

    # ── Alertes ──────────────────────────────────────────────────────
    log.info(f"\n{'='*70}")
    log.info(f" ALERTES WALK-FORWARD")
    log.info(f"{'='*70}")
    if all_alerts:
        for a in all_alerts:
            log.warning(f"  {a}")
    else:
        log.info("  ✓ Aucune alerte — distributions stables entre train et test")

    # ── Paramètres proposés ──────────────────────────────────────────
    log.info(f"\n{'='*70}")
    log.info(f" PARAMÈTRES PROPOSÉS")
    log.info(f" (basés sur la période d'entraînement 70%)")
    log.info(f"{'='*70}")
    for param, info_p in suggestions.items():
        val = info_p.get('valeur_proposee') or info_p.get('valeur_theorique_mt5')
        log.info(f"\n  {param} = {val}")
        for k, v in info_p.items():
            if k != 'valeur_proposee':
                log.info(f"    {k} : {v}")

    log.info(f"\n{'='*70}")
    log.info(f" PARAMÈTRES À RENSEIGNER MANUELLEMENT")
    log.info(f" (non calculables depuis les données tick)")
    log.info(f"{'='*70}")
    manual = {
        'COMMISSION_PER_LOT_ROUNDTRIP' : 'A vérifier chez le broker (0 si intégré dans le spread)',
        'SYMBOL_MT5_NAME'              : 'A confirmer dans MT5 (ex: NAS100, US100, NASDAQ)',
        'REFERENCE_CAPITAL'            : 'Capital de référence du trader (ex: 10000)',
        'LOT_TOLERANCE_RATIO'          : 'Ratio max toléré si lot calculé < lot minimum (ex: 1.5)',
        'RISK_PERCENT'                 : 'Risque par trade en % du capital (défaut: 1.0)',
        'RISK_REWARD'                  : 'Ratio TP/SL (défaut: 2.0)',
        'CONTRACT_VALUE'               : 'Valeur d\'1 point pour 1 lot — confirmer via trade test',
    }
    for param, note in manual.items():
        log.info(f"  {param} : {note}")

    # ── Export CSV et JSON ───────────────────────────────────────────
    slug      = ASSET_NAME.replace(' ','_').replace('(','').replace(')','').replace('/','')
    prefix    = os.path.join(OUTPUT_DIR, f"analyse_{slug}")

    # CSV détail par jour
    df_days = pd.DataFrame(all_days)
    df_days.to_csv(f"{prefix}_jours.csv", index=False)

    # CSV stats par année
    yearly_stats = []
    for y in sorted(df_days['year'].unique()):
        sub  = [d for d in all_days if d['year'] == y]
        st   = compute_stats(sub, str(y))
        row  = {'year': y, 'n_days': st['n_days_valid']}
        for key in ['range_size','spread_open','spread_day']:
            for metric in ['mean','median','p95']:
                v = st.get(key, {}).get(metric)
                row[f'{key}_{metric}'] = v
        yearly_stats.append(row)
    pd.DataFrame(yearly_stats).to_csv(f"{prefix}_annuel.csv", index=False)

    # JSON résumé complet
    summary = {
        'meta': {
            'generated_at' : datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            'asset'        : ASSET_NAME,
            'file'         : FILE_PATH,
            'n_days_total' : n_total,
            'n_train'      : n_train,
            'n_test'       : len(test),
        },
        'stats_global'  : {k: v for k, v in full_stats.items() if k != 'label'},
        'stats_train'   : {k: v for k, v in train_stats.items() if k != 'label'},
        'stats_test'    : {k: v for k, v in test_stats.items() if k != 'label'},
        'walk_forward_alerts' : all_alerts,
        'parametres_proposes' : suggestions,
        'parametres_manuels'  : manual,
    }
    json_path = f"{prefix}_resume.json"
    with open(json_path, 'w', encoding='utf-8') as f:
        json.dump(summary, f, indent=2, ensure_ascii=False, default=str)

    log.info(f"\n{'─'*70}")
    log.info(f" Fichiers exportés :")
    log.info(f"   {prefix}_jours.csv    (détail par jour)")
    log.info(f"   {prefix}_annuel.csv   (stats par année)")
    log.info(f"   {prefix}_resume.json  (résumé complet)")
    log.info(f"   {LOG_PATH}")
    log.info(f"\n{'='*70}")
    log.info(f" Analyse terminée. Transmettez le fichier JSON pour validation")
    log.info(f" des paramètres avant lancement du backtest.")
    log.info(f"{'='*70}\n")


if __name__ == '__main__':
    try:
        run_analysis()
    except Exception:
        log.error("\n" + "="*70)
        log.error(" ERREUR INATTENDUE")
        log.error("="*70)
        log.error(traceback.format_exc())
    finally:
        pause()
