## Projet : UFUNDED_OVERNIGHT_PORTFOLIO
_Démarré : 2026-06-17_ — _Dernière mise à jour : 2026-06-17_

### Contexte
- Plateforme : UFUNDED (ufunded.com) — environnement simulé, invite-only
- Compte : Commander — $90,000 de capital fictif
- Capital overnight disponible : $18,000 (20% × $90,000)
- **Maximum Drawdown absolu : $4,000 — STATIC, calculé sur le capital INITIAL du compte ($90,000/$86,000 plancher), confirmé par l'account manager : ne se recalcule JAMAIS après un payout, reste ancré sur le capital de départ pour toute la durée du compte**
- Profit Split : 75%
- Payout minimum : $500 / Payout maximum : $9,000/mois
- Levier : 1:1 confirmé sur actions/ETF (1:5 sur forex/métaux/indices, hors scope) — vérifié sur la liste réelle des actifs UFUNDED
- Interface : TradingView intégré — pas d'API externe possible, pas de fractionnement d'actions (lot minimum = 1, testé sur DUK et sur un ETF)
- Exécution : manuelle dans l'interface UFUNDED, ordres au marché

### Conditions de trading
- Commission : $0.007/action, minimum $1.25/exécution
- Frais min par aller-retour : $2.50
- Seuil de rentabilité commissions : ≥ 179 actions/trade (aucune de nos 13 lignes ne l'atteint avec $18,000 — accepté, coût négligeable : $16.25 total à l'achat)
- SL/TP disponibles nativement sur la plateforme (Stop-Loss : clic droit position → Protection Orders ; Take-Profit : ordre limite). **Pas de trailing stop natif.**
- Heures de marché : 9h30 – 16h00 ET

### Workflow validé
1. Analyse et allocation calculées en Python (hors plateforme, pas d'accès réseau direct pour Claude → scripts fournis à exécuter en local par l'utilisateur)
2. Exécution manuelle des trades dans l'interface UFUNDED (ordres au marché)
3. Suivi théorique via script de copilotage local (positions saisies manuellement + Yahoo Finance), CSV renvoyés à Claude pour interprétation
4. Recalage périodique sur la réalité du compte via captures d'écran UFUNDED si besoin

### ✅ Univers d'actifs UFUNDED — confirmé et catégorisé
- Liste complète obtenue (920 lignes) : `ufunded.com/markets`, export CSV fourni par l'utilisateur
- 876 lignes Actions/ETF (1:1) + 44 lignes Forex/Métaux/Indices (1:5)
- Catégorisation : 779 Single Stock, 36 Leveraged ETF (exclus), 30 Sector/Thematic ETF, 13 Crypto proxy (exclus), 7 Broad Index ETF, 5 Volatility products (exclus), 4 Other ETF, 2 Royalty/Specialty Trust, 1 Bond ETF (TLT)
- Fichiers : `universe_equities_etf_tagged.csv`, `universe_fx_metals_indices.csv`

### ✅ Méthodologie d'allocation — figée
**Style retenu : Mix défensif** (pas de système dynamique multi-régimes — jugé trop chronophage et source de faux signaux). Architecture finale : **Core stable + garde-fou mécanique sur drawdown réalisé** (pas de prédiction de régime de marché).

**Pipeline de sélection (à rejouer à chaque payout/remaniement) :**
1. Filtre quantitatif sur tout l'univers actions/ETF éligible (yfinance, 5 ans d'historique) : volatilité annualisée ≤25%, beta vs SPY ≤0.6, gap overnight max ≤10%, market cap ≥20Md$
2. Score composite (z-scores pondérés : vol 0.4 / beta 0.4 / gap max 0.2), plafond 4 titres/secteur
3. Décorrélation gloutonne : seuil de corrélation max 0.70 entre lignes retenues
4. Cible : 12 lignes + 1 ETF "joker" pour absorber le cash résiduel d'arrondi

**Deux allocations comparées : A (Pure défensive, 4 secteurs) vs B (Mix, 7 secteurs) → B retenue** (meilleur rendement, volatilité et drawdown sur le backtest 5 ans, malgré l'inclusion de Financial Services/Real Estate jugés a priori plus risqués).

**Pondération : inverse-volatilité** (testé vs équipondérée vs plafonnée à 15% — écarts négligeables sur cette sélection, les volatilités étant proches).

### ✅ Allocation B retenue — 12 lignes + 1 joker
| Secteur dominant | Tickers |
|---|---|
| Utilities | DUK |
| Healthcare | JNJ, NVS |
| Consumer Defensive | KO, PG, PEP |
| Financial Services | CME, CBOE |
| Consumer Cyclical | MCD |
| Industrials | WM |
| Real Estate | O |
| Communication Services | VZ |
| **Joker (absorbe cash résiduel)** | **XLP** (Consumer Staples Select Sector SPDR) — alternative testée : XLU |

Fichiers : `final_selection_B.csv`, `sizing_allocation_B_FINAL_with_joker.csv`

### ✅ Sizing réel exécuté (prix UFUNDED du 17/06/2026)
| Ticker | Prix Ask | Actions | Valeur |
|---|---|---|---|
| DUK | 123.79 | 12 | 1,485.48 |
| JNJ | 233.55 | 7 | 1,634.85 |
| KO | 79.87 | 21 | 1,677.27 |
| PG | 150.38 | 10 | 1,503.80 |
| CME | 252.48 | 5 | 1,262.40 |
| PEP | 141.51 | 10 | 1,415.10 |
| MCD | 284.16 | 5 | 1,420.80 |
| NVS | 151.07 | 9 | 1,359.63 |
| WM | 216.21 | 6 | 1,297.26 |
| O | 60.64 | 24 | 1,455.36 |
| VZ | 45.87 | 28 | 1,284.36 |
| CBOE | 255.57 | 4 | 1,022.28 |
| XLP (joker) | 83.64 | 14 | 1,170.96 |

Total investi : $17,989.55 — Cash résiduel : $10.45 (0.06%) — Commission totale à l'achat : $16.25

### ✅ Gestion du risque — règles figées
- **Stop-Loss catastrophe : -20% du prix d'entrée, sur les 13 lignes sans exception.** Pas de Take-Profit par ligne (contre-productif pour un cœur défensif, validé par simulation). Pas de trailing stop (non dispo nativement, et son rôle est de toute façon repris par le SL large + le palier de drawdown global).
- **Pas de rebalancement périodique actif** (décision finale de l'utilisateur, pour minimiser le temps de gestion — l'écart de performance vs rebalancement mensuel est faible : ~$1,221 sur 5 ans dans le backtest, jugé acceptable face au gain de simplicité)
- **Surveillance du drawdown global** : référence = capital initial du compte ($90,000), seuil fatal -$4,000, **jamais recalculé même après payout** (confirmé account manager)
- Le SL individuel ne protège PAS contre un drawdown généralisé du portefeuille (tous les titres baissent ensemble) — son rôle est limité à un accident idiosyncratique sur une ligne isolée

### ✅ Mécanisme de Payout — règles figées
- Phase d'accumulation initiale : aucun retrait avant que la poche atteigne **$20,000** (matelas de sécurité de $2,000)
- Une fois le matelas atteint : à chaque franchissement de **$22,000**, **payout de $2,000**
- **Au moment du payout : liquidation intégrale de toutes les lignes**, retrait du cash, puis **on rejoue tout le pipeline d'analyse** (filtre quantitatif + décorrélation) sur les $20,000 restants → nouvelle allocation potentiellement différente (le "joker" peut changer, ex: XLU au lieu de XLP selon les circonstances du moment)
- Backtest 5 ans (sur allocation B figée, sans remaniement réel simulé — limite connue) : 4 payouts, $8,000 retirés au total, drawdown max -$254 (très loin du seuil fatal)

### 🛠️ Script de copilotage — livré
- Dossier `copilotage/` : `suivi_portefeuille.py` + `mes_positions.csv` (modèle)
- Fonctionnement : l'utilisateur renseigne ticker/nb_shares/entry_price/entry_date dans le CSV, lance le script en local (accès réseau requis, Claude n'en a pas), récupère les prix actuels via yfinance
- Sorties : détail par ligne (P&L $/%, distance au SL) + synthèse globale de la poche
- **Limite explicite assumée dans le script** : ne mesure que la poche actions/ETF, PAS le drawdown du compte global à $90,000 (qui peut inclure d'autres activités hors de cette poche) — recoupement avec captures d'écran UFUNDED recommandé en cas de doute

### 📁 Scripts Python produits (tous testés, syntaxiquement valides)
- `step1_clean_universe.py` — nettoyage/catégorisation univers UFUNDED
- `step2_quant_filter_TEST.py` / `step2_quant_filter_FULL.py` — calcul métriques quantitatives (vol, beta, gaps overnight, secteur) via yfinance
- `step3a_scoring_preselection.py` — scoring composite + plafond sectoriel
- `step3b_correlation_matrix.py` — matrice de corrélation
- `step3c_decorrelate_selection.py` — sélection finale par décorrélation gloutonne
- `step4_simulate_weightings.py` / `step4b_simulate_with_sl.py` / `step4c_simulate_buyhold_sl.py` — simulations de performance (pondérations, SL, rebalancement vs buy-and-hold)
- `step5a_count_threshold_triggers.py` — comptage fréquence déclenchement seuils drawdown
- `step5b_recalcul_drawdown_ufunded.py` — recalcul drawdown méthode "static" (vs capital initial fixe)
- `step5c_simulate_payouts.py` / `step5d_compare_rebalance_vs_buyhold_payouts.py` — simulation mécanisme payout
- `step6_sizing_real_shares.py` — sizing réel en actions entières + commissions
- `copilotage/suivi_portefeuille.py` — script de suivi théorique post-exécution

### ⚠️ Points en suspens / limites connues
- Le coût en commissions du scénario "rebalancement mensuel" n'a jamais été chiffré précisément (suspecté significatif, mais le choix "sans rebalancement" a été acté avant ce calcul — non bloquant puisque la décision finale ne nécessitait plus cette donnée)
- Le remaniement complet à chaque payout n'a pas pu être simulé de façon rétrospectivement fidèle (impossible de "rejouer" le pipeline avec les données de marché telles qu'elles étaient à chaque date passée) — le backtest des payouts utilise l'allocation B figée comme proxy
- Swap overnight toujours non quantifié précisément par titre (mentionné comme variable dans le User Agreement, jamais creusé en détail — risque mineur, non bloquant à ce stade)
- Statut au 17/06/2026 : allocation définie et sizing calculé, **ordres en cours de passage par l'utilisateur** (12 lignes principales puis XLP en fonction du cash réellement disponible après exécution)
