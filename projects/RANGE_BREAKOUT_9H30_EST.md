# PROJET — Range Breakout 9h30 EST + Fair Value Gap
_Last updated: 2026-06-24_

## Résumé
Stratégie de scalping sur l'ouverture du marché US (9h30 EST). Entrée sur breakout de la première bougie 5min, confirmé par un Fair Value Gap (FVG) sur chart 1min. Un seul trade par jour, risk/reward fixe 2:1, gestion du lot dynamique à 1% du capital. P&L exprimé en points dans le backtest Python (indépendant du broker), converti en devise dans l'EA MT5.

---

## Statut
| Phase | Statut |
|-------|--------|
| Conception et validation logique | ✅ Terminé |
| Script analyse exploratoire (script 1) | ✅ Codé — en cours d'exécution sur 28 Go |
| Script backtest Python (v4) | ✅ Codé — en attente des paramètres de l'analyse exploratoire |
| Script micro-trade test | ✅ Codé et exécuté |
| Mise à jour paramètres v4 post-analyse | ⏳ En attente du JSON de l'analyse exploratoire |
| Analyse des résultats backtest | ⏳ En attente |
| Optimisation des filtres | ⏳ En attente |
| Codage EA MQL5 | 🔜 À venir |
| Tests forward sur RaiseFX | 🔜 À venir |

---

## Actif testé en premier
- **NASDAQ (US100)** — données 2011-10-01 → 2026-05-31
- Fichier : `C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv`
- Taille : ~28.77 Go
- Source : Tick Data Suite (Dukascopy), GMT+0, NO-DST

---

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
- Lot calculé dynamiquement en points (backtest Python) ou via `SYMBOL_TRADE_TICK_VALUE` (EA MT5)
- Capital réinitialisé à 10 000 unités abstraites à chaque début d'année (backtest)

### Gestion CLOSE_EOD
- Si aucun tick pendant `EOD_GAP_MINUTES` après le dernier tick du trade → clôture forcée
- `result = 'CLOSE_EOD'`, `close_eod = True`
- P&L calculé sur le dernier prix mid disponible (réel, pas zéro)
- **EXCLU** du win rate, profit factor, durée moyenne
- **INCLUS** dans le P&L global et le drawdown
- La durée p95 des trades TP/SL servira à calibrer `MAX_TRADE_DURATION` dans l'EA MT5

---

## Filtres actifs (V1)
| Filtre | Valeur actuelle | Source | Rôle |
|--------|----------------|--------|------|
| `MAX_SPREAD_POINTS` | 4.0 pts | ⚠ Provisoire — à mettre à jour post-analyse exploratoire | Exclure les entrées en spread anormal |
| `MIN_RANGE_POINTS`  | 10.0 pts | ⚠ Provisoire — à mettre à jour post-analyse exploratoire | Exclure les jours sans volatilité |
| `EOD_GAP_MINUTES`   | 30 min | ⚠ Provisoire — à mettre à jour post-analyse exploratoire | Détection coupure nocturne |
| `SESSION_END_HOUR`  | 23h GMT | ⚠ Provisoire — à mettre à jour post-analyse exploratoire | Heure fin de session |
| `WEEKDAY_FILTER`    | True | Fixe | Exclure samedi et dimanche |

---

## Variante retest (USE_RETEST_ENTRY)
Paramètre présent dans le code, désactivé en V1.
Logique : si le breakout initial ne crée pas de FVG, on attend un retour dans le range. Si un FVG se forme depuis le range → entrée dans la direction du breakout initial.
À activer et tester une fois les résultats V1 analysés.

---

## Spécifications broker NAS100 — RaiseFX Demo04
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| Login | 5007258 | Demo04 |
| Serveur | RaiseGlobal-Live | MT5 |
| Symbole MT5 | NAS100 | MT5 |
| Type | CFD Mode 2 | MT5 |
| Digits | 2 | MT5 |
| Point / Tick size | 0.01 | MT5 |
| Tick value (0.01 lot) | 0.008807 USD | MT5 |
| CONTRACT_VALUE théorique | 0.8807 USD/point/lot | Calculé via MT5 API |
| CONTRACT_VALUE mesurée | 0.4651 USD/point/lot | Micro-trade test (compte EUR) |
| Écart théorique/mesuré | ~47% | Probablement lié à la conversion EUR/USD |
| Commissions | 0 (intégrées dans le spread) | Micro-trade test |
| Sessions (GMT) | Lun–Ven 01:05–23:55 | Capture d'écran MT5 |
| Swap long | -5.0 USD/lot/jour | MT5 |
| Swap short | -5.0 USD/lot/jour | MT5 |
| Volume min | 0.01 lot | MT5 |
| Volume step | 0.01 lot | MT5 |

**Note CONTRACT_VALUE** : le backtest Python travaille entièrement en points (pas de conversion devise). La CONTRACT_VALUE sera utilisée uniquement dans l'EA MT5, où `SymbolInfoDouble(SYMBOL_TRADE_TICK_VALUE)` retournera la valeur exacte en temps réel.

---

## Architecture technique

### Scripts disponibles
| Script | Rôle | Fichier |
|--------|------|---------|
| Analyse exploratoire | Script 1 — analyse des données tick, propose les paramètres | `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_code.md` |
| Backtest Python v4 | Script 2 — backtest complet par batch annuel | `bot_files/Robot_range_breakout_9h30_EST_backtest_Python_v4_code.md` |
| Get symbol params | Récupère les spécifications du symbole depuis MT5 | `bot_files/Robot_range_breakout_9h30_EST_get_symbol_params_code.md` |
| Micro-trade test | Mesure empirique de la CONTRACT_VALUE | `bot_files/Robot_range_breakout_9h30_EST_micro_trade_code.md` |

### Workflow de déploiement sur un nouvel actif
1. Lancer `analyse_exploratoire.py` sur les données tick de l'actif
2. Transmettre le JSON de résultats pour validation collaborative des paramètres
3. Mettre à jour les paramètres `[ANALYSE]` dans `backtest_range_breakout_v4.py`
4. Lancer le backtest v4
5. Analyser les résultats, calibrer les filtres
6. Coder l'EA MT5 avec les paramètres validés
7. Tester en démo forward sur RaiseFX

### Points techniques clés (v4)
- Lecture unique et séquentielle du fichier CSV
- Traitement par **batch annuel** : écriture CSV + libération mémoire après chaque année
- Capital réinitialisé à 10 000 unités abstraites par an
- Indicateurs réinitialisés avec 30 jours de lookback au 1er janvier
- Parse timestamp vectorisé (format fixe 23 chars)
- Dispatch par jour : `np.diff` vectorisé — zéro boucle Python sur les ticks
- Bougies 1min : `np.maximum.reduceat` / `np.minimum.reduceat`
- Timestamps en **microsecondes int64** (pandas 2.x, `datetime64[us]`)
- Double snapshot indicateurs : état à N (avant signal) et à N+2 (à l'entrée)
- CLOSE_EOD sur gap de ticks > `EOD_GAP_MINUTES`
- Console qui reste ouverte + logging fichier + console
- **P&L en points uniquement** — pas de conversion devise dans le backtest

### Bug corrigé (pandas 2.x)
`pandas 2.x` retourne `datetime64[us]` et non `datetime64[ns]`. `.asi8` donne des **microsecondes**, mais `pd.Timestamp.value` donne des nanosecondes. Solution : tout convertir via `.astype('datetime64[us]').astype(np.int64)`. Ne jamais mélanger `.asi8` (µs) et `.value` (ns).

---

## Indicateurs contextuels enregistrés à chaque trade (v4)
Double snapshot : suffixe `''` pour l'état à N, suffixe `'_n2'` pour l'état à N+2.

| Indicateur | Détail |
|------------|--------|
| EMA 20/50/100/200 + pentes | Sur bougies 1min mid price |
| Price above EMA | Booléen : prix > EMA à l'entrée |
| Trade with EMA200 | Booléen : trade dans le sens de la tendance longue |
| RSI 7/14/21 | Sur bougies 1min mid price |
| ATR 7/14/21 | Sur bougies 1min mid price |
| Bollinger 20 | Largeur de bande, %B, bandes haute/basse |
| Aroon 14/25 | Aroon Up et Down |
| FVG/ATR ratio | Taille du FVG relative à la volatilité |
| Range/ATR ratio | Taille du range relative à la volatilité |
| Spread à l'entrée | En points |
| Jour de la semaine | 1=lundi … 5=vendredi |
| Semaine dans le mois | 1 à 5 |

---

## Sorties du backtest v4
| Fichier | Contenu |
|---------|---------|
| `backtest_NASDAQ_US100_XXXX_trades.csv` | Détail de chaque trade + indicateurs contextuels (par année) |
| `backtest_NASDAQ_US100_XXXX_monthly.csv` | Métriques agrégées par mois (par année) |
| `backtest_NASDAQ_US100_summary.csv` | 1 ligne par année — mis à jour en continu pendant le backtest |

## Métriques calculées
Win rate (hors CLOSE_EOD), P&L net en points, profit factor, espérance par trade, drawdown max, séries de SL consécutifs max, Sharpe ratio, Calmar ratio, durée moyenne TP/SL + p95, compteurs NO_DATA / FILTERED_RANGE / NO_SETUP / LOT_TOO_SMALL / CLOSE_EOD.

---

## Prochaines étapes
1. ⏳ Attendre la fin de l'analyse exploratoire (~15h30 heure locale)
2. Valider les paramètres proposés par l'analyse (MAX_SPREAD, MIN_RANGE, EOD_GAP, SESSION_END)
3. Mettre à jour les paramètres `[ANALYSE]` dans `backtest_range_breakout_v4.py`
4. Lancer le backtest v4 complet
5. Analyser les résultats année par année
6. Tester la variante retest (`USE_RETEST_ENTRY = True`)
7. Répliquer sur DAX, puis autres actifs
8. Coder l'EA MQL5
