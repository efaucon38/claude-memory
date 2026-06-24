# PROJET — Range Breakout 9h30 EST + Fair Value Gap
_Last updated: 2026-06-24_

## Résumé
Stratégie de scalping sur l'ouverture du marché US (9h30 EST).
Entrée sur breakout de la première bougie 5min, confirmé par un Fair Value Gap (FVG) sur chart 1min.
Un seul trade par jour, risk/reward fixe 2:1, gestion du lot dynamique à 1% du capital.

## Statut
| Phase | Statut |
|-------|--------|
| Conception et validation logique | ✅ Terminé |
| Script Python backtest (v3) | ✅ Codé, en cours d'exécution |
| Analyse des résultats backtest | ⏳ En attente des résultats |
| Optimisation des filtres | ⏳ En attente des résultats |
| Codage EA MQL5 | 🔜 À venir |
| Tests forward sur RaiseFX | 🔜 À venir |

## Actif testé en premier
- **NASDAQ (US100)** — données 2011-10-01 → 2026-05-31
- Fichier : `C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv`
- Taille : ~28.77 Go

## Logique de la stratégie

### Timing
- Session : lundi–vendredi uniquement
- Bougie de range : 9h30–9h35 EST
  - Hiver (EST = UTC-5) : 14h30–14h35 GMT
  - Été (EDT = UTC-4)   : 13h30–13h35 GMT
  - Gestion DST US automatique dans le code (2e dimanche de mars → 1er dimanche de novembre)

### Construction du range
- High = max(Ask) sur la bougie 5min — mèches incluses
- Low  = min(Bid) sur la bougie 5min — mèches incluses

### Signal FVG (pattern 3 bougies 1min)
| Bougie | Rôle |
|--------|------|
| N      | Référence (borne du gap) |
| N+1    | Displacement : ferme au-delà du range |
| N+2    | Confirmation du FVG |

- **FVG BUY**  : `N+1 close Ask > range_h` ET `Low(N+2 bid) > High(N ask)`
- **FVG SELL** : `N+1 close Bid < range_l` ET `High(N+2 ask) < Low(N bid)`

### Entrée
- BUY  : Ask de clôture de N+2 (market order)
- SELL : Bid de clôture de N+2 (market order)

### Stop Loss
- BUY  : Low(N+1) mèche incluse — prix Bid
- SELL : High(N+1) mèche incluse — prix Ask

### Take Profit
- 2× la distance Entry–SL (R:R 2:1 fixe)

### Gestion
- 1 trade max par jour
- SL touché → fin de journée, pas de nouveau trade
- Risque : 1% du capital courant (paramétrable)
- Lot calculé dynamiquement : `Risque$ / (SL_pts × CONTRACT_VALUE)`

## Filtres actifs (V1)
| Filtre | Valeur | Rôle |
|--------|--------|------|
| `MAX_SPREAD_POINTS` | 4.0 pts | Exclure les entrées en spread anormal |
| `MIN_RANGE_POINTS`  | 10.0 pts | Exclure les jours sans volatilité (fériés, etc.) |
| `WEEKDAY_FILTER`    | True | Exclure samedi et dimanche |

## Variante retest (USE_RETEST_ENTRY)
Paramètre présent dans le code, désactivé en V1.
Logique : si le breakout initial ne crée pas de FVG, on attend un retour dans le range.
Si un FVG se forme depuis le range → entrée dans la direction du breakout initial.
À activer et tester une fois les résultats V1 analysés.

## Architecture technique du script Python

### Fichier
- Script : `bot_files/Robot_range_breakout_9h30_EST_Code.md`
- Script analyse exploratoire : `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_Code.md`
- Script micro trade de test :  `bot_files/Robot_range_breakout_9h30_EST_micro_trade_Code.md`
- Version actuelle : v3 (fully vectorized)

### Points techniques clés
- Lecture unique et séquentielle du fichier CSV (un seul passage, aucun rechargement)
- Parse timestamp vectorisé : extraction numérique directe sur strings (format fixe 23 chars)
- Dispatch par jour : `np.diff` sur array de day_ids — zéro boucle Python sur les ticks
- Bougies 1min : `np.maximum.reduceat` / `np.minimum.reduceat`
- Timestamps en **microsecondes int64** (pandas 2.x, datetime64[us])
- Indicateurs incrémentaux : EMABuffer, RSIBuffer, ATRBuffer, SlopeBuffer
- Simulation SL/TP : `np.argmax` sur array booléen (vectorisé)
- Performance estimée : 15–30 min pour 28 Go de données tick

### Bug corrigé en cours de développement
pandas 2.x retourne `datetime64[us]` et non `datetime64[ns]`.
`.asi8` donne des **microsecondes**, mais `pd.Timestamp.value` donne des nanosecondes.
Solution : tout convertir explicitement en µs via `.astype('datetime64[us]').astype(np.int64)`.
Ne jamais mélanger `.asi8` (µs) et `.value` (ns) dans la même comparaison.

## Indicateurs contextuels enregistrés à chaque trade
| Indicateur | Détail |
|------------|--------|
| EMA 20/50/100/200 | Sur bougies 1min mid price |
| Pente EMA (slope) | Régression linéaire sur 5 dernières valeurs, normalisée en %/bougie |
| Price above EMA | Booléen : prix > EMA à l'entrée |
| RSI 7/14/21 | Sur bougies 1min mid price |
| ATR 14 | Sur bougies 1min mid price |
| FVG/ATR ratio | Taille du FVG relative à la volatilité ambiante |
| Range/ATR ratio | Taille du range relative à la volatilité ambiante |
| Trade with EMA200 | Booléen : trade dans le sens de la tendance longue |
| Spread à l'entrée | En points |
| Jour de la semaine | 1=lundi … 5=vendredi |
| Semaine dans le mois | 1 à 5 |

## Sorties du backtest
| Fichier | Contenu |
|---------|---------|
| `backtest_NASDAQ_US100_trades.csv`  | Détail de chaque trade + 40+ colonnes contextuelles |
| `backtest_NASDAQ_US100_yearly.csv`  | Métriques agrégées par année |
| `backtest_NASDAQ_US100_monthly.csv` | Métriques agrégées par mois |
| `backtest_NASDAQ_US100_spread.csv`  | Distribution du spread par heure GMT |

## Métriques calculées
Win rate, profit net, profit factor, espérance par trade, drawdown max,
séries de SL consécutifs max, durée moyenne TP vs SL — global + par année + par mois.

## Prochaines étapes
1. Analyser les résultats du backtest NASDAQ (résultats attendus)
2. Calibrer `MAX_SPREAD_POINTS` via l'analyse spread par heure
3. Analyser les indicateurs contextuels pour identifier les conditions favorables/défavorables
4. Tester la variante retest (`USE_RETEST_ENTRY = True`)
5. Répliquer le backtest sur DAX, puis autres actifs
6. Coder l'EA MQL5 à partir de la logique validée en Python
