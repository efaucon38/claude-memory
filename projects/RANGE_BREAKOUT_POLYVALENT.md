# PROJET — Range Breakout Polyvalent
_Last updated: 2026-06-25_

## Résumé
Moteur générique de backtest pour toute stratégie de type "range breakout". Conçu pour tester différentes configurations sur n'importe quel actif disposant de données tick Tick Data Suite (GMT+0, NO-DST).

Dérive du projet Range Breakout 9h30 EST + FVG (NASDAQ), dont il généralise la logique à tout actif, timezone, mode de signal et timeframe. Deux modes de signal : FVG (pattern 3 bougies avec gap) et BREAKOUT (clôture hors range).

---

## Statut
| Phase | Statut |
|-------|--------|
| Script Python polyvalent | ✅ Opérationnel — testé en modes FVG et BREAKOUT |
| Script extract_years.py | ✅ Opérationnel — découpe les gros fichiers historiques |
| Backtest NASDAQ V4/V5/V6 | ✅ Terminé — résultats analysés |
| Analyse exploratoire SP500 | ✅ Terminée |
| Backtest SP500 (6 années) | ✅ Terminé — résultats analysés |
| Analyse exploratoire Gold | ⏳ En cours — fichier extrait 2015-2026 en création |
| Backtest Gold | 🔜 En attente de l'analyse exploratoire |
| Backtest US30 (Dow Jones) | 🔜 À planifier |
| Backtest DAX (ouverture EU) | 🔜 À planifier |
| Codage EA MT5 | 🔜 À venir |

---

## Fichiers
| Fichier | Rôle |
|---------|------|
| `bot_files/Robot_range_breakout_polyvalent_code.md` | Code source Python du moteur |
| `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_code.md` | Script d'analyse exploratoire (à adapter par actif) |
| `bot_files/extract_years_code.md` | Outil utilitaire — extraction d'une plage d'années depuis un gros fichier tick |

---

## Résultats des backtests (R:R 3:1, mode FVG, 6 années sélectionnées)

### NASDAQ (US100) — V6
| Année | Trades | Win rate | P&L net (pts) |
|-------|--------|----------|---------------|
| 2012 | 130 | 28.5% | +34.5 ✅ |
| 2018 | 249 | 18.9% | -982.4 ❌ |
| 2020 | 249 | 20.5% | -861.1 ❌ |
| 2022 | 255 | 20.8% | -717.9 ❌ |
| 2023 | 251 | 19.5% | -1117.8 ❌ |
| 2024 | 248 | 27.4% | +1306.6 ✅ |
| **TOTAL** | **1 382** | **22.1%** | **-2 338.1** |

### S&P 500 (US500) — v1_sp500_fvg_rr3
| Année | Trades | Win rate | P&L net (pts) |
|-------|--------|----------|---------------|
| 2012 | 6 | 33.3% | +10.3 ✅ |
| 2018 | 242 | 17.4% | -254.3 ❌ |
| 2020 | 236 | 17.4% | -405.8 ❌ |
| 2022 | 248 | 21.4% | -65.0 ❌ |
| 2023 | 241 | 20.7% | -159.8 ❌ |
| 2024 | 240 | 23.8% | +84.5 ✅ |
| **TOTAL** | **1 213** | **20.2%** | **-790.1** |

### Comparatif NASDAQ vs SP500
- **SP500 perd 3× moins** que le NASDAQ (-790 vs -2 338 pts) malgré un WR légèrement inférieur — effet des SL plus petits en valeur absolue
- Les **deux actifs sont sous le seuil de rentabilité** (25% pour R:R 3:1)
- **SP500 2012 : seulement 6 trades** — range trop petit (médiane 2.8 pts, MIN_RANGE = 2 pts), marché trop calme

### Conclusions intermédiaires
- Win rate systématiquement sous 25% sur R:R 3:1 — problème structurel sur ces deux actifs
- **Vendredi = seul jour profitable sur NASDAQ** (WR 26.4%, P&L +399 pts) — piste intéressante
- **Mercredi = pire jour sur les deux actifs** (NASDAQ 21.9%, SP500 17.6%)
- **SP500 BUY meilleur que SELL** (-368 vs -806 pts) — inverse attendu
- **SP500 range optimal Q3 (7-9 pts)** — quasi à l'équilibre (-19 pts), les très petits et très grands ranges sont mauvais
- La stratégie est plus efficace sur les marchés **volatils** (ranges > 30 pts sur NASDAQ en 2024)
- **Gold = prochain actif à tester** — forte réactivité à 9h30 EST, volatilité élevée, FVG souvent nets

---

## Paramètres validés par actif

### NASDAQ (US100)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 3.0 pts | Analyse exploratoire 2026-06-24 |
| `MIN_RANGE_POINTS` | 5.0 pts | Conservative — à affiner |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `SESSION_END_HOUR` | 23h GMT | Confirmé |
| `RANGE_TIMEZONE` | America/New_York | — |
| `RANGE_START_LOCAL` | (9, 30) | — |
| `RANGE_END_LOCAL` | (9, 35) | — |

### S&P 500 (US500)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 1.0 pt | Analyse exploratoire 2026-06-25 — p95=0.655 |
| `MIN_RANGE_POINTS` | 2.0 pts | Conservative — affiner (Q3 optimal = 7-9 pts) |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `SESSION_END_HOUR` | 23h GMT | Confirmé |
| `RANGE_TIMEZONE` | America/New_York | — |
| `RANGE_START_LOCAL` | (9, 30) | — |
| `RANGE_END_LOCAL` | (9, 35) | — |

### Gold (XAUUSD) — en cours
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| Tous paramètres | ⏳ En attente | Analyse exploratoire en cours (2015-2025) |
| Fichier source | `2015-01-01 - 2026-12-31 - XAUUSD_GMT+0_NO-DST ticks.csv` | Extrait via extract_years.py |

---

## Guide d'utilisation — Version Python

### Étape 0 — Gros fichiers historiques (> 10 Go)
Pour les actifs avec un long historique (Gold 22 Go, etc.), extraire d'abord une sous-période :

```bash
python extract_years.py
```

Configurer `YEAR_START`, `YEAR_END` et `FILE_PATH` dans le script. Le fichier extrait est sauvegardé dans le même dossier que la source avec un nom normalisé. Placer ensuite le fichier extrait dans le dossier Tick Data Suite.

### Étape 1 — Analyse exploratoire (obligatoire pour chaque nouvel actif)

```bash
python analyse_exploratoire_ACTIF.py
```

**À adapter dans le script :**
- `FILE_PATH` → chemin vers le fichier CSV
- `ASSET_NAME` → nom lisible
- `YEARS_FILTER` → liste d'années si analyse partielle ([] = toutes)

**Ce que l'analyse fournit :**
- `MAX_SPREAD_POINTS` : p95 spread ouverture × 1.2
- `MIN_RANGE_POINTS` : p10 range × 0.8 (conservative)
- `EOD_GAP_MINUTES` : 50% durée médiane coupure nocturne
- Alertes walk-forward si distributions train/test diffèrent > 30%

### Étape 2 — Configuration du moteur

Modifier uniquement la section CONFIG de `range_breakout_polyvalent.py` :

```python
RUN_ID       = "v1_gold_fvg_rr3"
MAGIC_NUMBER = "E5F6G7H8"       # Changer à chaque nouveau run — reporter dans l'EA MT5

FILE_PATH  = r"C:\...\2015-01-01 - 2026-12-31 - XAUUSD_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "Gold (XAUUSD)"

RANGE_TIMEZONE    = "America/New_York"
RANGE_START_LOCAL = (9, 30)
RANGE_END_LOCAL   = (9, 35)

SIGNAL_MODE          = "FVG"     # "FVG" ou "BREAKOUT"
SIGNAL_TIMEFRAME_MIN = 1
SIGNAL_WINDOW_MIN    = 90
TRADE_DIRECTION      = "BOTH"    # "BOTH", "BUY_ONLY", "SELL_ONLY"
RISK_REWARD          = 3.0
MAX_TRADE_DURATION_MIN = 240

MAX_SPREAD_POINTS = X.X   # issu de l'analyse exploratoire
MIN_RANGE_POINTS  = X.X   # issu de l'analyse exploratoire
EOD_GAP_MINUTES   = X
YEARS_FILTER      = []    # [] = toutes les années
```

### Étape 3 — Lancement

```bash
python range_breakout_polyvalent.py
```

La barre de progression affiche l'année en cours, le mois, la vitesse et l'ETA par année. Les résultats sont écrits sur disque après chaque année.

**Fichiers générés :**

rng_ASSET_RUN_ID_MAGIC_XXXX_trades.csv    ← détail par trade + indicateurs contextuels

rng_ASSET_RUN_ID_MAGIC_XXXX_monthly.csv   ← métriques par mois

rng_ASSET_RUN_ID_MAGIC_summary.csv        ← 1 ligne par année (mis à jour en continu)

rng_ASSET_RUN_ID_MAGIC.log               ← log complet

---

## Workflow de déploiement sur un nouvel actif

1. Récupérer les données tick (Tick Data Suite, GMT+0, NO-DST)
2. Si fichier > 10 Go : extraire la période utile via extract_years.py
3. Adapter et lancer analyse_exploratoire.py (avec YEARS_FILTER si nécessaire)
4. Valider les paramètres avec Claude (JSON de l'analyse)
5. Configurer range_breakout_polyvalent.py
6. Lancer le backtest
7. Analyser les résultats (summary.csv + trades.csv)
8. Itérer sur les paramètres si nécessaire
9. Coder l'EA MT5 avec les paramètres validés

---

## Paramètres timezone disponibles (exemples)

| Marché | RANGE_TIMEZONE | Ouverture locale |
|--------|----------------|-----------------|
| NASDAQ / SP500 / Dow / Gold | `America/New_York` | 9h30 |
| DAX / CAC40 | `Europe/Berlin` | 9h00 |
| FTSE 100 | `Europe/London` | 8h00 |
| Nikkei | `Asia/Tokyo` | 9h00 |

Le DST est géré automatiquement via `zoneinfo`. **Ne jamais recoder le DST manuellement.**

---

## Points de vigilance pour la transformation en EA MT5

### 1. Gestion du temps — point le plus critique
En Python : `zoneinfo` convertit automatiquement l'heure locale en UTC.
En MT5 : utiliser `TimeGMT()` comme référence, implémenter `IsDST_US()` manuellement.

```mql5
datetime gmt_now = TimeGMT();
bool is_dst = IsDST_US(gmt_now);
int range_start_gmt_hour = is_dst ? 13 : 14;
int range_start_gmt_min  = 30;
```

### 2. MAGIC_NUMBER → magic number des ordres MT5
```mql5
int MAGIC = 0xE5F6G7H8;  // même valeur que dans le script Python
request.magic = MAGIC;
```

### 3. SIGNAL_MODE en MT5
- **FVG** : `iOpen()`, `iHigh()`, `iLow()`, `iClose()` sur SIGNAL_TIMEFRAME_MIN. Bougie 0 = en cours, bougie 1 = dernière fermée.
- **BREAKOUT** : surveiller la clôture de chaque bougie vs range_high/range_low.

### 4. TRADE_DIRECTION
```mql5
input string TradeDirection = "BOTH";  // "BOTH", "BUY_ONLY", "SELL_ONLY"
```

### 5. MAX_TRADE_DURATION_MIN
```mql5
if(PositionSelect(Symbol())) {
   datetime open_time = (datetime)PositionGetInteger(POSITION_TIME);
   if((TimeGMT() - open_time) > MAX_TRADE_DURATION_MIN * 60) {
      // Clôturer la position
   }
}
```

### 6. Filtres jours de semaine
```mql5
input bool TradeMonday = true; /* ... */
MqlDateTime dt; TimeGMT(dt);
// Filtrer selon dt.day_of_week
```

### 7. Calcul du lot en MT5
```mql5
double tick_value = SymbolInfoDouble(Symbol(), SYMBOL_TRADE_TICK_VALUE);
double tick_size  = SymbolInfoDouble(Symbol(), SYMBOL_TRADE_TICK_SIZE);
double point      = SymbolInfoDouble(Symbol(), SYMBOL_POINT);
double sl_pts     = MathAbs(entry - sl) / point;
double risk       = AccountBalance() * RiskPercent / 100.0;
double lot        = risk / (sl_pts * tick_value / tick_size * point);
lot = MathFloor(lot / SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_STEP))
      * SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_STEP);
lot = MathMax(lot, SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_MIN));
```

---

## Prochaines étapes
1. ⏳ Attendre fin extraction Gold (extract_years.py en cours)
2. Lancer analyse_exploratoire_Gold.py sur le fichier extrait 2015-2026
3. Valider les paramètres Gold avec Claude
4. Lancer le backtest Gold avec range_breakout_polyvalent.py
5. Analyser et comparer NASDAQ / SP500 / Gold
6. Tester filtres additionnels identifiés (vendredi NASDAQ, MIN_RANGE SP500 > 7 pts, BUY_ONLY SP500)
7. Décider de la suite (notamment : codage EA MT5)
