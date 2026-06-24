# PROJET — Range Breakout Polyvalent
_Last updated: 2026-06-24_

## Résumé
Moteur générique de backtest pour toute stratégie de type "range breakout".
Conçu pour tester différentes configurations sur n'importe quel actif
disposant de données tick Tick Data Suite (GMT+0, NO-DST).

Dérive du projet Range Breakout 9h30 EST + FVG (NASDAQ), dont il généralise
la logique à tout actif, timezone, mode de signal et timeframe, avec toutefois une différence sur l'entrée (sortie de range simple) en mode "BREAKOUT" (entrée après FGV en mode "FVG").

---

## Statut
| Phase | Statut |
|-------|--------|
| Script Python polyvalent | ✅ Opérationnel — testé en modes FVG et BREAKOUT |
| Backtest NASDAQ (référence) | ✅ Effectué en V4/V5/V6 — résultats disponibles |
| Analyse exploratoire SP500 | ⏳ En cours d'exécution |
| Backtest SP500 | 🔜 En attente des paramètres de l'analyse exploratoire |
| Backtest US30 (Dow Jones) | 🔜 À planifier |
| Backtest XAUUSD (Or) | 🔜 À planifier |
| Backtest DAX (ouverture EU) | 🔜 À planifier |
| Codage EA MT5 | 🔜 À venir |

---

## Fichiers
| Fichier | Rôle |
|---------|------|
| `bot_files/Robot_range_breakout_polyvalent_code.md` | Code source Python du moteur |
| `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_code.md` | Script d'analyse exploratoire (à adapter par actif) |

---

## Guide d'utilisation — Version Python

### Étape 1 — Analyse exploratoire (obligatoire pour chaque nouvel actif)

Avant tout backtest, lancer `analyse_exploratoire.py` sur les données tick
de l'actif cible. Ce script produit un fichier JSON avec les valeurs
recommandées pour les paramètres clés.

```bash
python analyse_exploratoire_SP500.py
```

**Ce qu'il faut adapter dans le script d'analyse :**
- `FILE_PATH` → chemin vers le fichier CSV de l'actif
- `ASSET_NAME` → nom lisible de l'actif

**Ce que l'analyse fournit :**
- `MAX_SPREAD_POINTS` : p95 du spread à l'ouverture × 1.2
- `MIN_RANGE_POINTS` : p10 du range × 0.8 (valeur conservative)
- `EOD_GAP_MINUTES` : 50% de la durée médiane de coupure nocturne
- Distribution du spread par heure GMT (pour vérifier la liquidité à l'heure cible)
- Alertes walk-forward si les distributions train/test diffèrent > 30%

---

### Étape 2 — Configuration du moteur

Ouvrir `range_breakout_polyvalent.py` et modifier **uniquement la section CONFIG** :

```python
# --- Identifiants du run ---
RUN_ID       = "v1_sp500_fvg_rr3"   # label lisible de la configuration
MAGIC_NUMBER = "D4E5F6G7"            # changer à chaque nouveau run

# --- Fichier de données ---
FILE_PATH  = r"C:\...\USA_500_Index_GMT+0_NO-DST ticks.csv"
ASSET_NAME = "S&P 500 (US500)"

# --- Horaires ---
RANGE_TIMEZONE    = "America/New_York"  # timezone zoneinfo
RANGE_START_LOCAL = (9, 30)             # heure locale du marché
RANGE_END_LOCAL   = (9, 35)

# --- Signal ---
SIGNAL_MODE          = "FVG"      # "FVG" ou "BREAKOUT"
SIGNAL_TIMEFRAME_MIN = 1          # timeframe d'analyse en minutes
SIGNAL_WINDOW_MIN    = 90         # fenêtre temporelle de recherche après le range, en minutes

# --- Direction ---
TRADE_DIRECTION = "BOTH"          # "BOTH", "BUY_ONLY", "SELL_ONLY"

# --- Stratégie ---
RISK_REWARD            = 3.0
MAX_TRADE_DURATION_MIN = 240      # 0 = pas de limite

# --- Filtres ---
MAX_SPREAD_POINTS = X.X   # issu de l'analyse exploratoire
MIN_RANGE_POINTS  = X.X   # issu de l'analyse exploratoire
EOD_GAP_MINUTES   = X     # issu de l'analyse exploratoire
YEARS_FILTER      = []    # [] = toutes les années
```

---

### Étape 3 — Lancement

```bash
python range_breakout_polyvalent.py
```

**Suivi en temps réel :**
La barre de progression affiche l'année en cours, le mois, la vitesse
de lecture et l'ETA pour l'année. Les résultats de chaque année sont
écrits sur disque immédiatement — pas besoin d'attendre la fin.

**Fichiers générés :**
rng_ASSET_RUN_ID_MAGIC_XXXX_trades.csv    ← détail par trade + indicateurs

rng_ASSET_RUN_ID_MAGIC_XXXX_monthly.csv   ← métriques par mois

rng_ASSET_RUN_ID_MAGIC_MAGIC_summary.csv  ← 1 ligne par année (mis à jour en continu)

rng_ASSET_RUN_ID_MAGIC.log                ← log complet

---

### Workflow de déploiement sur un nouvel actif

Récupérer les données tick (Tick Data Suite, GMT+0, NO-DST)
Adapter et lancer analyse_exploratoire.py
Valider les paramètres avec Claude (JSON de l'analyse)
Configurer range_breakout_polyvalent.py
Lancer le backtest
Analyser les résultats (summary.csv + trades.csv)
Itérer sur les paramètres si nécessaire
Coder l'EA MT5 avec les paramètres validés

---

### Paramètres timezone disponibles (exemples)

| Marché | RANGE_TIMEZONE | Ouverture locale |
|--------|----------------|-----------------|
| NASDAQ / SP500 / Dow | `America/New_York` | 9h30 |
| DAX / CAC40 | `Europe/Berlin` | 9h00 |
| FTSE 100 | `Europe/London` | 8h00 |
| Nikkei | `Asia/Tokyo` | 9h00 |
| Or / Pétrole (ouverture US) | `America/New_York` | 9h30 |

Le DST est géré automatiquement pour toutes ces timezones via `zoneinfo`.
**Ne jamais recoder le DST manuellement.**

---

## Points de vigilance pour la transformation en EA MT5

### 1. Gestion du temps — point le plus critique

**En Python :** `zoneinfo` convertit automatiquement l'heure locale en UTC.
Les données Tick Data Suite sont en GMT+0 = UTC → comparaison directe.

**En MT5 :** pas de `zoneinfo`. La conversion doit être faite manuellement.

```mql5
// Référence temporelle : toujours TimeGMT(), jamais TimeCurrent()
datetime gmt_now = TimeGMT();

// Offset broker = différence entre heure broker et GMT
int broker_offset = (int)(TimeCurrent() - TimeGMT());

// Heure de début du range en GMT
// Exemple NASDAQ 9h30 New York :
// Hiver (EST) : 14h30 GMT | Été (EDT) : 13h30 GMT
// Implémenter la détection DST US en MQL5 :
// DST US : 2e dim. mars → 1er dim. novembre
bool is_dst = IsDST_US(gmt_now);
int range_start_gmt_hour = is_dst ? 13 : 14;
int range_start_gmt_min  = 30;
```

**Fonction `IsDST_US()` à implémenter en MQL5** — c'est un point clé
qui devra être codé et testé rigoureusement. Se référer à
`library/time_issues_and_trading.md`.

---

### 2. MAGIC_NUMBER → magic number des ordres MT5

Le `MAGIC_NUMBER` du script Python doit être reporté tel quel dans l'EA MT5
comme magic number des ordres. Cela permet de tracer exactement quels ordres
ont été passés par quel robot, et de filtrer les ordres dans le journal.

```mql5
int MAGIC = 0xC3D4E5F6;  // même valeur que dans le script Python
request.magic = MAGIC;
```

---

### 3. Mode SIGNAL_MODE

**FVG en MT5 :** utiliser `iOpen()`, `iHigh()`, `iLow()`, `iClose()` sur
le timeframe configuré (SIGNAL_TIMEFRAME_MIN). Le pattern 3 bougies
s'applique identiquement. Attention aux index MT5 : bougie 0 = en cours,
bougie 1 = dernière fermée.

**BREAKOUT en MT5 :** plus simple — surveiller la clôture de chaque bougie
et comparer avec range_high / range_low.

---

### 4. TRADE_DIRECTION

En MT5 :
```mql5
input string TradeDirection = "BOTH";  // "BOTH", "BUY_ONLY", "SELL_ONLY"
```
Filtrer dans `OnTick()` avant d'envoyer l'ordre.

---

### 5. MAX_TRADE_DURATION_MIN

En MT5, implémenter dans `OnTick()` :
```mql5
if(PositionSelect(Symbol())) {
   datetime open_time = (datetime)PositionGetInteger(POSITION_TIME);
   if((TimeGMT() - open_time) > MAX_TRADE_DURATION_MIN * 60) {
      // Clôturer la position
   }
}
```

---

### 6. Filtres jours de semaine

```mql5
input bool TradeMonday    = true;
input bool TradeTuesday   = true;
input bool TradeWednesday = true;
input bool TradeThursday  = true;
input bool TradeFriday    = true;

MqlDateTime dt; TimeGMT(dt);
bool day_allowed = false;
switch(dt.day_of_week) {
   case 1: day_allowed = TradeMonday;    break;
   case 2: day_allowed = TradeTuesday;   break;
   case 3: day_allowed = TradeWednesday; break;
   case 4: day_allowed = TradeThursday;  break;
   case 5: day_allowed = TradeFriday;    break;
}
```

---

### 7. Calcul du lot en MT5

Ne pas utiliser de `CONTRACT_VALUE` fixe. Utiliser :
```mql5
double tick_value = SymbolInfoDouble(Symbol(), SYMBOL_TRADE_TICK_VALUE);
double tick_size  = SymbolInfoDouble(Symbol(), SYMBOL_TRADE_TICK_SIZE);
double point      = SymbolInfoDouble(Symbol(), SYMBOL_POINT);

double sl_distance_pts = MathAbs(entry - sl) / point;
double risk_amount     = AccountBalance() * RiskPercent / 100.0;
double lot = risk_amount / (sl_distance_pts * tick_value / tick_size * point);
lot = MathFloor(lot / SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_STEP))
      * SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_STEP);
lot = MathMax(lot, SymbolInfoDouble(Symbol(), SYMBOL_VOLUME_MIN));
```

---

## Résultats de référence (NASDAQ, données 2012-2026)

| Version | R:R | Mode | Années+ | P&L total | Win rate |
|---------|-----|------|---------|-----------|---------|
| V4 | 2:1 | FVG | 3/15 | -5 432 pts | 29.8% |
| V5 | 2:1 | FVG + retest | 3/15 | -5 432 pts | 29.8% |
| V6 (en cours) | 3:1 | FVG | — | — | — |

**Observation clé :** avec R:R 2:1, win rate global 29.8% < seuil rentabilité 33.3%.
Simulation R:R 3:1 (borne haute) : 12/15 années positives.
Seuil de rentabilité théorique pour R:R 3:1 = **25.0%** → validé sur les données.

---

## Prochaines étapes
1. ⏳ Attendre les résultats du backtest V6 (R:R 3:1, 6 années sélectionnées)
2. ⏳ Attendre les résultats de l'analyse exploratoire SP500
3. Lancer le backtest SP500 avec `range_breakout_polyvalent.py`
4. Comparer NASDAQ vs SP500 sur les mêmes années
5. Tester mode BREAKOUT vs FVG sur NASDAQ
6. Tester `TRADE_DIRECTION = "SELL_ONLY"` (meilleur WR observé en V4)
7. Coder l'EA MT5 générique
