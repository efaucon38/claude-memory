# PROJET — Range Breakout Polyvalent
_Last updated: 2026-06-27_

## Résumé
Moteur générique de backtest pour toute stratégie de type "range breakout".
Testé sur NASDAQ (US100), S&P 500 (US500) et Or (XAUUSD), avec deux modes
de sortie : TP fixe R:R 3:1 et Trailing Stop ATR (V8).

---

## Statut
| Phase | Statut |
|-------|--------|
| Script Python polyvalent (FVG + BREAKOUT, TP fixe) | ✅ Opérationnel |
| Script V8 (Trailing ATR, TICKS, vectorisé) | ✅ Opérationnel |
| Script extract_years.py | ✅ Opérationnel |
| Script analyse_exploratoire (NASDAQ, SP500, Gold) | ✅ Opérationnel |
| Backtest NASDAQ V4/V5/V6 (TP fixe) | ✅ Analysé |
| Backtest SP500 v1 (TP fixe R:R 3:1) | ✅ Analysé |
| Backtest Gold v1 (TP fixe R:R 3:1) | ✅ Analysé |
| Backtest V8 NASDAQ — 3 configs trailing | ✅ Analysé |
| Note de synthèse PDF (français) | ✅ Rédigée — juin 2026 |
| Codage EA MT5 | 🔜 En attente de résultats plus concluants |

---

## Actifs testés et paramètres validés

### NASDAQ (US100)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 3.0 pts | Analyse exploratoire |
| `MIN_RANGE_POINTS` | 5.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| Fichier ticks | `2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv` | 28.77 Go |
| Fichier M1 | `2011-09-18 - 2026-06-07 - USA_100_Technical_Index_GMT+0_NO-DST M1_M1.csv` | 0.25 Go |

### S&P 500 (US500)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 1.0 pt | Analyse exploratoire |
| `MIN_RANGE_POINTS` | 2.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 60 min | Coupure 20h GMT, ~105 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| Fichier ticks | `2011-10-01 - 2026-05-31 - USA_500_Index_GMT+0_NO-DST ticks.csv` | 8.06 Go |

### Gold (XAUUSD)
| Paramètre | Valeur | Source |
|-----------|--------|--------|
| `MAX_SPREAD_POINTS` | 0.75 pts | Analyse exploratoire |
| `MIN_RANGE_POINTS` | 2.0 pts | Conservative |
| `EOD_GAP_MINUTES` | 30 min | Coupure 20h GMT, ~60 min |
| `RANGE_TIMEZONE` | America/New_York | — |
| Fichier ticks extrait | `2015-01-01 - 2026-12-31 - XAUUSD_GMT+0_NO-DST ticks.csv` | ~10 Go |
| ⚠ Données réelles | Début décembre 2017 (malgré le nom du fichier) | vérifier avec check_file_dates.py |

---

## Résultats — TP fixe R:R 3:1, signal FVG 9h30 EST

### Seuil de rentabilité
- R:R 3:1 → win rate minimum = **25.0%**

### NASDAQ — 6 années [2012, 2018, 2020, 2022, 2023, 2024]
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2012 | 130 | 28.5% | +0.35% ✅ | 17.4% |
| 2018 | 249 | 18.9% | -9.80% ❌ | 32.1% |
| 2020 | 249 | 20.5% | -8.60% ❌ | 28.7% |
| 2022 | 255 | 20.8% | -7.20% ❌ | 25.3% |
| 2023 | 251 | 19.5% | -11.20% ❌ | 34.2% |
| 2024 | 248 | 27.4% | +13.10% ✅ | 12.8% |
| **Moyenne** | — | **22.1%** | **-2.2%/an** | — |

### S&P 500 — 6 années [2012, 2018, 2020, 2022, 2023, 2024]
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2012 | 6 | 33.3% | +0.10% ✅ | 2.1% |
| 2018 | 242 | 17.4% | -2.50% ❌ | 10.3% |
| 2020 | 236 | 17.4% | -4.10% ❌ | 14.7% |
| 2022 | 248 | 21.4% | -0.70% ❌ | 8.2% |
| 2023 | 241 | 20.7% | -1.60% ❌ | 9.8% |
| 2024 | 240 | 23.8% | +0.80% ✅ | 7.1% |
| **Moyenne** | — | **20.2%** | **-1.3%/an** | — |

### Gold — 4 années [2022, 2023, 2024, 2025]
| Année | Trades | WR | P&L % capital | DD max |
|-------|--------|-----|---------------|--------|
| 2022 | 230 | 17.5% | -51.7% ❌ | 47.2% |
| 2023 | 220 | 24.0% | -2.0% ❌ | 16.7% |
| 2024 | 240 | 20.2% | -35.2% ❌ | 39.7% |
| 2025 | 235 | 19.6% | -41.4% ❌ | 44.0% |
| **Moyenne** | — | **20.3%** | **-32.6%/an** | — |

---

## Résultats — Trailing Stop ATR V8 (NASDAQ uniquement)

### Paramètres du trailing
**TRAILING_ATR_FACTOR** : distance du trailing = N × ATR14 (bougies 1 min)
- Factor 2 → trailing serré, sort souvent sur micro-reculs
- Factor 3 → trailing large, laisse courir les grandes tendances

**BE_FACTOR** : fraction du SL à atteindre avant activation du trailing
- BE=0.0 → trailing immédiat dès l'entrée
- BE=0.5 → trailing activé après 0.5 × SL en faveur
- BE=1.0 → trailing activé au break-even complet

### Comparatif 4 configurations — NASDAQ [2012, 2018, 2020, 2022, 2023, 2024]
| Configuration | 2012 | 2018 | 2020 | 2022 | 2023 | 2024 | Moy/an |
|---------------|------|------|------|------|------|------|--------|
| TP fixe R:R 3:1 | +0.4% | -9.8% | -8.6% | -7.2% | -11.2% | +13.1% | **-2.2%** |
| Trail. f=2, BE=0 | -10.2% | -37.4% | -39.0% | -49.1% | -47.3% | -11.6% | **-32.4%** |
| Trail. f=3, BE=0 | +12.2% | -10.2% | -37.3% | -38.0% | -49.1% | +53.1% | **-11.5%** |
| Trail. f=2, BE=0.5 | -5.0% | -41.8% | -49.2% | -53.8% | -56.0% | -5.7% | **-35.2%** |

### Conclusions V8
- **Meilleure config trailing : factor=3, BE=0** (−11.5%/an) — seule à battre le TP fixe en moyenne
- **2024 exceptionnelle** avec trailing f=3 : +53.1% grâce à la forte directionnalité du NASDAQ
- **BE=0.5 contre-productif** : réduit le ratio gain/perte (1.29x) sans améliorer suffisamment le WR
- **Problème structurel** : le signal FVG génère trop peu de mouvements directionnels persistants
- Le trailing n'est pas la solution miracle — les filtres additionnels sont la piste prioritaire

---

## Sélection des années testées — justification

6 années sélectionnées pour couvrir des régimes de marché contrastés :

| Année | Régime | Justification |
|-------|--------|---------------|
| 2012 | Bull market calme | Référence basse volatilité |
| 2018 | Forte volatilité, corrections | Teste la robustesse en conditions adverses |
| 2020 | Crise Covid, volatilité extrême | Choc exogène majeur |
| 2022 | Bear market, inflation | Marché baissier prolongé |
| 2023 | Reprise progressive | Transition — ni haussier ni baissier |
| 2024 | Bull market fort, NASDAQ directionnel | Conditions optimales |

---

## Conventions de mesure (IMPORTANT)
- **Performance = % du capital** uniquement — jamais de points de prix
- **Ne jamais sommer les % annuels** — capitaux réinitialisés chaque année
- **Indicateur synthétique = moyenne annuelle** des P&L %
- **Sharpe inadapté** — pénalise les jours sans trade, préférer espérance et profit factor

---

## Pistes d'amélioration identifiées
- **Vendredi uniquement sur NASDAQ** — seul jour systématiquement au-dessus de 25%
- **Exclure le mercredi** — pire jour sur tous les actifs
- **MIN_RANGE plus élevé sur SP500** — les petits ranges génèrent de mauvais FVG
- **Tester trailing f=3 sur SP500 et Gold**
- **Explorer d'autres fenêtres temporelles** pour le signal FVG

---

## Workflow de déploiement sur un nouvel actif
1. Si fichier > 10 Go : extraire via `extract_years.py`
2. Vérifier les dates réelles avec `check_file_dates.py`
3. Lancer `analyse_exploratoire.py` (avec `YEARS_FILTER`)
4. Valider les paramètres avec Claude (JSON de l'analyse)
5. Configurer et lancer `range_breakout_polyvalent.py`
6. Si résultats mitigés : tester `range_breakout_v8.py` (trailing)
7. Coder l'EA MT5 si résultats suffisamment positifs

---

## Fichiers disponibles
| Fichier | Rôle |
|---------|------|
| `bot_files/Robot_range_breakout_polyvalent_code.md` | Moteur polyvalent FVG/BREAKOUT, TP fixe |
| `bot_files/Robot_range_breakout_polyvalent_V8_code.md` | V8 — Trailing ATR, TICKS uniquement |
| `bot_files/Robot_range_breakout_9h30_EST_analyse_exploratoire_code.md` | Script analyse exploratoire |
| `bot_files/extract_years_code.md` | Extraction plage d'années depuis gros fichier |
| `bot_files/check_file_dates_code.md` | Vérification dates première/dernière ligne |
| `notes/Note_synthese_Range_Breakout_FINALE.pdf` | Note de synthèse PDF — juin 2026 |

---

## Prochaines étapes
1. Tester les filtres additionnels identifiés (vendredi, MIN_RANGE, mercredi exclu)
2. Tester trailing f=3 sur SP500 et Gold si les filtres améliorent le NASDAQ
3. Décider du déploiement EA MT5 sur la base des résultats filtrés
