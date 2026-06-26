# PROJET — Range Breakout Polyvalent
_Last updated: 2026-06-26_

## Résumé
Moteur générique de backtest pour toute stratégie de type "range breakout".
Conçu pour tester différentes configurations sur n'importe quel actif
disposant de données tick Tick Data Suite (GMT+0, NO-DST).

Dérive du projet Range Breakout 9h30 EST + FVG (NASDAQ), dont il généralise
la logique à tout actif, timezone, mode de signal et timeframe.

---

## Statut
| Phase | Statut |
|-------|--------|
| Script Python polyvalent (FVG + BREAKOUT) | ✅ Opérationnel |
| Script extract_years.py | ✅ Opérationnel |
| Script analyse_exploratoire | ✅ Opérationnel (NASDAQ, SP500, Gold) |
| Backtest NASDAQ V4/V5/V6 (TP fixe) | ✅ Analysé |
| Backtest SP500 v1 (TP fixe R:R 3:1) | ✅ Analysé |
| Backtest Gold v1 (TP fixe R:R 3:1) | ✅ Analysé |
| Backtest V8 NASDAQ (Trailing ATR, 3 configs) | ⏳ En cours |
| Note de synthèse pour groupe algo trading | ⏳ En attente résultats V8 |
| Codage EA MT5 | 🔜 À venir si résultats concluants |

---

## Actifs testés et paramètres validés

### NASDAQ (US100)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 3.0 pts | Analyse exploratoire 2026-06-24 |
| `MIN_RANGE_POINTS` | 5.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| `RANGE_START_LOCAL` | (9, 30) | — |
| Fichier ticks | `2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv` | 28.77 Go |
| Fichier M1 | `2011-09-18 - 2026-06-07 - USA_100_Technical_Index_GMT+0_NO-DST M1_M1.csv` | 0.25 Go |

### S&P 500 (US500)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 1.0 pt | Analyse exploratoire 2026-06-25 |
| `MIN_RANGE_POINTS` | 2.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| Fichier ticks | `2011-10-01 - 2026-05-31 - USA_500_Index_GMT+0_NO-DST ticks.csv` | 8.06 Go |

### Gold (XAUUSD)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 0.75 pts | Analyse exploratoire 2026-06-26 |
| `MIN_RANGE_POINTS` | 2.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 30 min | Coupure 20h GMT, ~60 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| Fichier ticks source | `2003-05-05 - 2026-05-31 - XAUUSD_GMT+0_NO-DST ticks.csv` | 22.97 Go |
| Fichier ticks extrait | `2015-01-01 - 2026-12-31 - XAUUSD_GMT+0_NO-DST ticks.csv` | ~10 Go |

---

## Résultats des backtests — TP fixe R:R 3:1, mode FVG

### Seuil de rentabilité théorique
- R:R 2:1 → win rate minimum = **33.3%**
- R:R 3:1 → win rate minimum = **25.0%**

### NASDAQ — 6 années sélectionnées
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2012 | 130 | 28.5% | +0.35% ✅ | 17.4% |
| 2018 | 249 | 18.9% | -9.8% ❌ | — |
| 2020 | 249 | 20.5% | -8.6% ❌ | — |
| 2022 | 255 | 20.8% | -7.2% ❌ | — |
| 2023 | 251 | 19.5% | -11.2% ❌ | — |
| 2024 | 248 | 27.4% | +13.1% ✅ | — |
| **Moyenne** | — | **22.1%** | **-2.2%/an** | — |

### S&P 500 — 6 années sélectionnées
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2012 | 6 | 33.3% | +0.1% ✅ | — |
| 2018 | 242 | 17.4% | -2.5% ❌ | — |
| 2020 | 236 | 17.4% | -4.1% ❌ | — |
| 2022 | 248 | 21.4% | -0.7% ❌ | — |
| 2023 | 241 | 20.7% | -1.6% ❌ | — |
| 2024 | 240 | 23.8% | +0.8% ✅ | — |
| **Moyenne** | — | **20.2%** | **-1.3%/an** | — |

### Gold — 4 années disponibles
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2022 | 230 | 17.5% | -51.7% ❌ | 47.2% |
| 2023 | 220 | 24.0% | -2.0% ❌ | 16.7% |
| 2024 | 240 | 20.2% | -35.2% ❌ | 39.7% |
| 2025 | 235 | 19.6% | -41.4% ❌ | 44.0% |
| **Moyenne** | — | **20.3%** | **-32.6%/an** | — |

### Conclusions intermédiaires
- **Aucun actif n'est rentable** en configuration de base (FVG, R:R 3:1)
- **SP500 = actif le plus stable** — pertes limitées, drawdowns faibles
- **NASDAQ = seul actif avec une année vraiment rentable** (2024 : +13.1%)
- **Gold = pire actif** — drawdowns catastrophiques, WR systématiquement sous 25%
- Win rate systématiquement sous le seuil de 25% — problème structurel
- La stratégie FVG fonctionne mieux sur les marchés **volatils et directionnels**
- **Mercredi = pire jour** sur tous les actifs
- **SELL légèrement meilleur que BUY** sur SP500 et Gold

---

## Backtest V8 — Trailing ATR (en cours)

### Configuration
- Signal : FVG identique aux versions précédentes
- Sortie : trailing ATR vectorisé sur bougies 1min reconstruites depuis les ticks
- Actif : NASDAQ, 6 années [2012, 2018, 2020, 2022, 2023, 2024]

### 3 runs en parallèle
| Run | Factor | BE | MAGIC | Interprétation |
|-----|--------|----|-------|----------------|
| V8-1 | 2.0 | 0.0 | K1L2M3N4 | Trailing immédiat, distance confortable |
| V8-2 | 3.0 | 0.0 | L2M3N4O5 | Trailing large, laisse courir |
| V8-3 | 2.0 | 0.5 | M3N4O5P6 | Trailing activé après 0.5×SL |

### Résultat partiel 2012
| Run | Trades | WR | P&L % capital |
|-----|--------|-----|---------------|
| V8-1 (f=2, BE=0) | 133 | 33.1% | +0.03% |
| V8-3 (f=2, BE=0.5) | 133 | 77.4% | +0.32% |

### Points de vigilance V8
- `TP_TRAIL` = sortie trailing **avec prix favorable** (corrigé — bug initial : WR 100% avec P&L négatif)
- Simulation vectorisée sur bougies 1min — pas de boucle tick par tick
- Biais minimal : sortie détectée au LOW/HIGH de la bougie, pas au tick exact
- Mode M1 abandonné : incompatible avec détection FVG (pas de bid/ask séparés)

---

## Fichiers disponibles
| Fichier | Rôle |
|---------|------|
| `bot_files/Robot_range_breakout_polyvalent_code.md` | Code source moteur polyvalent (FVG + BREAKOUT, TP fixe) |
| `bot_files/Robot_range_breakout_polyvalent_V8_code.md` | Code source V8 (Trailing ATR) |
| `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_code.md` | Script analyse exploratoire |
| `bot_files/extract_years_code.md` | Outil extraction plage d'années |

---

## Workflow de déploiement sur un nouvel actif
1. Si fichier > 10 Go : extraire la période utile via `extract_years.py`
2. Lancer `analyse_exploratoire.py` (avec `YEARS_FILTER` si nécessaire)
3. Valider les paramètres avec Claude (JSON de l'analyse)
4. Configurer `range_breakout_polyvalent.py`
5. Lancer le backtest
6. Analyser les résultats
7. Tester le trailing V8 si résultats prometteurs
8. Coder l'EA MT5 avec les paramètres validés

---

## Conventions de mesure (IMPORTANT)
- **Performance = % du capital** uniquement — jamais de points de prix
- **Ne pas sommer les % annuels** — capitaux réinitialisés chaque année
- **Indicateur synthétique = moyenne annuelle** des P&L %
- **Sharpe inadapté** à cette stratégie (pénalise les jours sans trade)
- **Préférer** : win rate, espérance, profit factor, drawdown max

---

## Prochaines étapes
1. ⏳ Attendre les résultats complets V8 (3 runs NASDAQ)
2. Rédiger la note de synthèse en français pour le groupe algo trading
3. Décider de la suite selon résultats V8 :
   - Si trailing concluant → tester sur SP500 et Gold
   - Si toujours décevant → explorer d'autres heures d'ouverture ou actifs
4. Coder l'EA MT5 si résultats suffisamment positifs
