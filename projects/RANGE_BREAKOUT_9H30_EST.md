# PROJET — Range Breakout 9h30 EST + Fair Value Gap
_Last updated: 2026-06-24_

## Résumé
Stratégie de scalping sur l'ouverture du marché US (9h30 EST). Entrée sur breakout de la première bougie 5min, confirmé par un Fair Value Gap (FVG) sur chart 1min. Un seul trade par jour, risk/reward fixe 2:1, gestion du lot dynamique à 1% du capital. P&L exprimé en points dans le backtest Python (indépendant du broker), converti en devise dans l'EA MT5.

---

## Statut
| Phase | Statut |
|-------|--------|
| Conception et validation logique | ✅ Terminé |
| Script analyse exploratoire (script 1) | ✅ Terminé — JSON validé le 2026-06-24 |
| Script backtest Python (v4) | ✅ Codé — **en cours d'exécution** sur 28 Go |
| Script micro-trade test | ✅ Codé et exécuté |
| Paramètres v4 mis à jour post-analyse | ✅ Validé le 2026-06-24 |
| Analyse des résultats backtest | ⏳ En attente de la fin du backtest |
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
- **Vérification effectuée** sur les trades 2013 : timestamps d'entrée cohérents avec les fenêtres EST/EDT ✅

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
- La durée **p95** des trades TP/SL servira à calibrer `MAX_TRADE_DURATION` dans l'EA MT5
- Observation sur données 2013-2014 : p95 durée SL autour de 120-155 min → `MAX_TRADE_DURATION` suggéré : **240 min**

---

## Filtres actifs (V1)
| Filtre | Valeur | Source | Rôle |
|--------|--------|--------|------|
| `MAX_SPREAD_POINTS` | **3.0 pts** | Analyse exploratoire — validé 2026-06-24 | p95 spread ouverture × 1.2 |
| `MIN_RANGE_POINTS`  | **5.0 pts** | Analyse exploratoire — option B conservative | Affiner post-backtest via range_size_pts |
| `EOD_GAP_MINUTES`   | **60 min**  | Analyse exploratoire — validé 2026-06-24 | Coupure nocturne ~20h GMT, durée ~105 min |
| `SESSION_END_HOUR`  | **23h GMT** | Confirmé Dukascopy + RaiseFX | Heure fin de session |
| `WEEKDAY_FILTER`    | True | Fixe | Exclure samedi et dimanche |

---

## Variante retest (USE_RETEST_ENTRY)
Paramètre présent dans le code, désactivé en V1.
Logique : si le breakout initial ne crée pas de FVG, on attend un retour dans le range. Si un FVG se forme depuis le range → entrée dans la direction du breakout initial.
À activer et tester une fois les résultats V1 analysés.

---

## Résultats partiels du backtest v4 (en cours)
Premiers résultats disponibles — backtest en cours d'exécution au moment de cette mise à jour.

| Année | Trades | Win rate | P&L net (pts) | Profit Factor | Drawdown max | Sharpe |
|-------|--------|----------|---------------|---------------|--------------|--------|
| 2012 | 133 | 36.8% | +62.91 | 1.255 | 12.5% | 1.15 |
| 2013 | 108 | 31.4% | -132.60 | 0.662 | 19.8% | -0.59 |
| 2014 | 169 | 30.9% | -147.88 | 0.830 | 25.6% | -0.75 |

**Observations préliminaires :**
- Seuil de rentabilité théorique pour R:R 2:1 = **33.3% de win rate**
- 2012 au-dessus du seuil ✅ — 2013/2014 en-dessous ❌
- Durées des trades cohérentes avec du scalping (médiane 13-17 min)
- Quelques trades longs (> 120 min) présents chaque année — normal sur marché peu volatil avec petit SL
- CLOSE_EOD rares et bien gérés

**Bug mineur identifié (timestamp) :**
Le timestamp d'entrée enregistré correspond à l'ouverture de la bougie N+2, pas à sa fermeture (+1 minute). Impact : nul sur les prix et P&L, légère surestimation des durées sur ~7% des trades. À corriger dans la v4.1.

---

## Analyse exploratoire — résultats clés (données 2012-2026)
| Métrique | Global | Train (2012-2021) | Test (2022-2026) |
|----------|--------|-------------------|------------------|
| Range médian 5min | 21.1 pts | 12.9 pts | 52.8 pts |
| Spread médian ouverture | 1.57 pts | 1.31 pts | 1.78 pts |
| Spread p95 ouverture | 2.20 pts | 2.20 pts | 1.92 pts |
| Prix mid moyen | 9 684 pts | 6 390 pts | 17 361 pts |
| Coupure nocturne | ~20h GMT | ~20h GMT | ~20h GMT |
| Durée coupure médiane | 105 min | 105 min | 105 min |

**Alerte walk-forward majeure :** taille du range × 4 entre train et test (+199%) — changement structurel réel du marché lié à la hausse des prix (×3.3). Pas un problème de stratégie — à analyser via `range_size_pts` dans les résultats du backtest.

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

**Note CONTRACT_VALUE** : le backtest Python travaille entièrement en points. La CONTRACT_VALUE sera utilisée uniquement dans l'EA MT5, où `SymbolInfoDouble(SYMBOL_TRADE_TICK_VALUE)` retournera la valeur exacte en temps réel.

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

### Bugs identifiés
| Bug | Impact | Statut |
|-----|--------|--------|
| Timestamp entrée = ouverture N+2 au lieu de fermeture (+1 min) | Nul sur prix/P&L, légère surestimation durée (~7% des trades) | À corriger en v4.1 |
| pandas 2.x : `.asi8` en µs vs `.value` en ns | Corrigé en v4 | ✅ |
| `date.fromtimestamp()` dépendant du fuseau local | Corrigé en v4 | ✅ |

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
1. ⏳ Attendre la fin du backtest v4
2. Analyser les résultats année par année via le summary.csv
3. Identifier les conditions favorables via les indicateurs contextuels (range_size, ATR, EMA200...)
4. Corriger le bug de timestamp (v4.1)
5. Tester la variante retest (`USE_RETEST_ENTRY = True`)
6. Répliquer sur DAX, puis autres actifs
7. Coder l'EA MT5
