# TRADING BOT MEMORY — CORE
_Last updated: 2026-06-25_

💡 NOTE POUR CLAUDE : tu te positionnes en tant qu'expert du trading en général maîtrisant l'ensemble des concepts associés (y compris SMC), et du trading algorithmique en Python et EA MT5. Dès lecture de ce fichier, rappelle à l'utilisateur de te fournir le fichier projet correspondant si ce n'est pas déjà fait, et propose de mettre à jour la mémoire en fin de session.

## Stack technique
- **Langages** : Python (backtest, analyse, outillage) + MQL5 (exécution live sur MT5)
- **Broker actuel** : RaiseFX (MT5) pour tests — architecture pensée pour être broker-agnostique
- **Backtest** : piloté par Claude en Python, à partir de données exportées du broker ou Tick Data Suite (stockées localement)
- **Marchés** : Forex, Crypto, Indices (DAX, SP500...), Actions/ETF

## Outils et environnement
- **PC local** : Windows 10 Pro — développement, backtests MT5, Claude Code
- **VPS** : Windows 10 — exécution live MT5, Ponderation.py, backtests Python
- **Claude Code** : installé et opérationnel dans VS Code sur PC local
- Accès direct aux fichiers locaux sans upload
- Mémoire auto-chargée via CLAUDE.md à chaque session
- Clone local du repo mémoire : `C:\TradingBots\claude-memory\`
- Privilégier des méthodes qui économisent la consommation de tokens à chaque fois que cela est possible
- **Tick Data Suite** : données ticks Dukascopy
  - Chemin local : `C:\Users\ericf\Documents\Trading algo\Tick Data Suite\`
  - Format fichiers : `AAAA-MM-JJ - AAAA-MM-JJ - SYMBOLE_GMT+0_NO-DST ticks.csv`
  - Format données : `JJ.MM.AAAA HH:MM:SS.mmm, Bid, Ask, VolBid, VolAsk` (GMT+0, sans DST)
  - ⚠️ Toujours vérifier première/dernière ligne avant tout backtest
  - Workflow obligatoire : télécharger d'abord, exporter les ticks ensuite
  - script de vérification des dates (première et dernière ligne) : bot_files/Tick_Data_Suite_chek_file_dates_code.md 
- **Statut actuel des fichiers** :

| Symbole | Période réelle | Statut | Nom fichier |
|---------|---------------|--------|--------|
| EURUSD  | 01.2005→05.2026 | ✅ Utilisable (6.96 Go) - ticks réels - GMT+0 sans DST | `2005-01-02 - 2026-05-31 - EURUSD_GMT+0_NO-DST ticks.csv` |
| EURUSD  | 05.2003→05.2026 | ✅ Utilisable (8 Mo) - bougies H1 précalculées - GMT+0 sans DST | `2003-05-04 - 2026-05-31 - EURUSD_GMT+0_NO-DST H1_H1.csv` |
| GBPJPY  | 01.2020→05.2026 | ✅ Utilisable (10.26 Go) - ticks réels - GMT+0 sans DST | `2005-01-02 - 2026-05-31 - GBPJPY_GMT+0_NO-DST ticks.csv` |
| USDJPY  | 01.2020→05.2026 | ✅ Utilisable (8.0 Go) - ticks réels - GMT+0 sans DST | `2005-01-02 - 2026-05-31 - USDJPY_GMT+0_NO-DST ticks.csv` |
| USDJPY  | 01.2020→02.2026 | ✅ Utilisable (8.0 Mo) - bougies H1 précalculées - GMT+0 sans DST | `2003-05-04 - 2026-05-31 - USDJPY_GMT+0_NO-DST H1_H1.csv` |
| XAUUSD  | 05.2003→05.2026 | ✅ Utilisable (23.0 Go) - ticks réels - fichier extrait 2015-2026 disponible - GMT+0 sans DST | `2003-05-05 - 2026-06-23 - XAUUSD_GMT+0_NO-DST ticks.csv` |
| XAUUSD extrait | 01.2018→05.2026 | ✅ Utilisable (20.4 Go) - ticks réels - créé via extract_years.py - GMT+0 sans DST | `2017-12-11 - 2026-06-05 - XAUUSD_GMT+0_NO-DST ticks.csv` |
| XAGUSD | 01.2005→05.2026 | ✅ Utilisable (5.5 Go) - ticks réels - GMT+0 sans DST | `2005-01-02 - 2026-05-31 - XAGUSD_GMT+0_NO-DST ticks` |
| USA_100 M1 | 10.2011→05.2026 | ✅ Utilisable (0.25 Go) - bougies M1 précalculées - GMT+0 sans DST | `2011-09-18 - 2026-06-07 - USA_500_Index_GMT+0_NO-DST M1_M1.csv` |
| USA_100 | 10.2011→05.2026 | ✅ Utilisable (28.1 Go) - ticks réels - GMT+0 sans DST | `2011-10-01 - 2026-05-31 - USA_100_Technical_Index_GMT+0_NO-DST ticks.csv` |
| USA_100 | 2026-06-15→2026-06-19 | ✅ Utilisable (92 Mo) - ticks réels - petit échantillon pour test de robot - GMT+0 sans DST | `2026-06-15 - 2026-06-19 - USA_100_Technical_Index_GMT+0_NO-DST ticks pour test.csv` |
| USA_500 | 10.2011→05.2026 | ✅ Utilisable (7.9 Go) - ticks réels - GMT+0 sans DST  | `2011-10-01 - 2026-05-31 - USA_500_Index_GMT+0_NO-DST ticks.csv` |
| USA_500 | 10.2011→05.2026 | ✅ Utilisable (0.2 Go) - bougies M1 précalculées - GMT+0 sans DST | `2011-09-18 - 2026-06-07 - USA_500_Index_GMT+0_NO-DST M1_M1.csv` |
| AUDCAD | 01.2016→06.2026 | ✅ En cours de chargement - ticks réels - GMT+0 sans DST | `2016-01-03 - 2026-06-23 - AUDCAD_GMT+0_NO-DST ticks.csv` |


## Mode de collaboration (interface Claude — programmation Python/MQL5)
- L'utilisateur décrit l'idée ou le besoin
- Claude propose une approche et attend validation avant tout codage
- On code, on teste, on ajuste ensemble, en enrichissement mutuel
- ⚠️ RÈGLE ABSOLUE : Claude ne lance JAMAIS de codage sans validation explicite de l'utilisateur
- Claude rappelle régulièrement à l'utilisateur de mettre à jout les fichiers mémoires (CORE.md + fichiers spécifiques au projet traité

## Règles d'autonomie — Claude Code
Ces règles sont spécifiques à Claude Code. Elles remplacent la règle ci-dessus dans ce contexte.

🔴 **Validation requise (uniquement)** :
- Avant de lancer la rédaction d'un script de robot/stratégie de trading — on fait le tour du sujet ensemble d'abord
- Avant de livrer une version finalisée — Claude doit toujours proposer une analyse critique approfondie pour identifier d'éventuels points oubliés ou mal traités
- Actions destructives ou difficiles à inverser — suppression, écrasement, git reset --hard, commit/push

🟢🟡 **Tout le reste en autonomie** — du moment qu'on se resynchronise avant la finalisation.

**Aucun accès au VPS** : cloisonnement total sauf configuration explicite future.

**Récupération des données** : sources multiples (MT5 API, Tick Data Suite, Yahoo Finance). L'interface claude.ai classique n'a pas d'accès réseau direct — scripts à exécuter localement.

## Risk Management
- Risque max par trade : **1% du capital** par défaut
- Toute dérogation décidée explicitement par l'utilisateur
- Priorité : prudence et préservation du capital avant performance

## Conventions de code
- Séparer clairement : logique de signal / calcul de lot / exécution ordre
- Toujours documenter les paramètres clés en tête de fichier
- Nommage des stratégies : `ACTIF_TIMEFRAME_LOGIQUE`
- **RUN_ID + MAGIC_NUMBER** : tout script de backtest doit avoir ces deux identifiants codés en dur, intégrés dans tous les noms de fichiers (CSV + log). Le MAGIC_NUMBER est reporté dans l'EA MT5 comme magic number des ordres.

## Conventions Python backtest (tick data)
- Données tick : CSV Tick Data Suite, GMT+0, NO-DST, format `DD.MM.YYYY HH:MM:SS.mmm`
- Colonnes : timestamp | ask | bid | vol_ask | vol_bid (sans header)
- Timestamps en **microsecondes int64** (pandas 2.x → `datetime64[us]`, `.asi8` est en µs)
- ⚠️ Ne jamais mélanger `.asi8` (µs) et `pd.Timestamp.value` (ns)
- Parse vectorisé : extraction numérique directe sur strings de longueur fixe
- Dispatch par jour : `np.diff` vectorisé — zéro boucle Python sur les ticks
- Bougies OHLC : `np.maximum.reduceat` / `np.minimum.reduceat`
- Simulation SL/TP : `np.argmax` sur array booléen
- Indicateurs : buffers incrémentaux, accès `.values` directs (jamais `.loc` en boucle)
- Architecture obligatoire multi-Go : lecture unique et séquentielle, chunks de 500k lignes
- **Gestion timezone** : utiliser `zoneinfo` (Python 3.9+) — ne jamais recoder le DST manuellement
- **YEARS_FILTER** : toujours prévoir un filtre d'années dans les scripts d'analyse et de backtest pour gérer les gros fichiers (ex: Gold 22 Go)
- **Extraction de sous-période** : utiliser `extract_years.py` pour créer un fichier réduit avant analyse sur les gros fichiers historiques

## ⚠️ Erreurs récurrentes — NE JAMAIS reproduire
1. **Codage sans autorisation** : Claude ne produit aucun code sans validation préalable
2. **Gestion des fuseaux horaires** : toujours clarifier et documenter l'écart UTC/heure broker
3. **Calcul de la taille des lots** : ne jamais confondre % du capital et valeur absolue
4. **Drawdown "trailing" vs "static"** : toujours vérifier la règle exacte du broker
5. **Hypothèses de simulation non explicitées** : annoncer tous les paramètres structurants AVANT
6. **Walk-forward obligatoire** : séparer 70% entraînement / 30% test — jamais de look-ahead bias
7. **État du marché en parallèle** : toujours calculer des métriques de régime de marché
8. **Spécificités broker par actif** : toujours vérifier tick value, spread, marge, horaires
9. **pandas 2.x datetime64[us]** : `.asi8` = µs, `.value` = ns — ne jamais mélanger
10. **Boucle Python sur les ticks** : toujours vectoriser avec numpy
11. **Console de commande** : toujours paramétrer pour qu'elle reste ouverte même en cas de problème, avec journalisation fichier + console
12. **Scripts longs** : toujours intégrer une barre de progression (par année en cours, pas globale) avec vitesse et ETA

## Projets (index)
| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | `projects/TELEGRAM_MT5_KASPER.md` |
| EA Desk Quant | 8 actifs (propfirm 7 + Gold) | Test démo — v3.0 OFFENSIF | `projects/MULTI_ACTIFS_DRAWDOWN.md` |
| Momentum Scanner | Multi-actifs broker (à définir) | Construction | `projects/MOMENTUM_SCANNER.md` |
| Fourier | EURUSD / USDJPY / NAS100 | Clos — verdict négatif | `projects/FOURIER.md` |
| ICT_ROBOT | Multi-actifs | Construction | `projects/ICT_ROBOT.md` |
| UFUNDED_OVERNIGHT_PORTFOLIO | US Stocks / ETF | Allocation et sizing finalisés | `projects/UFUNDED.md` |
| Range Breakout 9h30 EST + FVG | NASDAQ (US100) | Backtests V4/V5/V6 terminés — résultats mitigés | `projects/RANGE_BREAKOUT_9H30_EST.md` |
| Range Breakout Polyvalent V8 | NASDAQ | ✅ Terminé — 3 runs trailing analysés | `projects/RANGE_BREAKOUT_POLYVALENT.md` |
| Carte d'identité d'un symbol | Multi-actifs | en cours d'élaboration | `projects/CARTE_IDENTITE_SYMBOL.md` |
| EA MT5 AUDCAD | AUDCAD | déclinaison opérationnelle du projet "Carte d'identité d'un symbol", à construire | `` |

## Dernières versions codées (index)
| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs | Test démo | `bot_files/botTelegram_Kasper_code.md` |
| EA Desk Quant — Ponderation v3.0 OFFENSIF | 7 actifs + Gold | Test démo | `bot_files/Ponderation_code.md` |
| EA Desk Quant — EA_FX_Universal v2.9 | 7 actifs | Test démo | `bot_files/EA_FX_Universal_code.md` |
| Range Breakout 9h30 EST — Python v4 | NASDAQ | Référence résultats | `bot_files/Robot_range_breakout_9h30_EST_Code.md` |
| Range Breakout Polyvalent — Python V8 | NASDAQ | En cours d'exécution (3 runs trailing) | `bot_files/Robot_range_breakout_polyvalent_code.md` |
| extract_years.py | Outil utilitaire | Opérationnel | `bot_files/extract_years_code.md` |
| chek_file_dates.py | Outil utilitaire - vérifie les dates de début et de fin d'un fichier CSV de données | Opérationnel | `bot_files/Tick_Data_Suite_chek_file_dates_code.md` |

## Bibliothèque (index)
| Sujet | Fichier |
|-------|---------|
| Gestion du temps en trading | `library/time_issues_and_trading.md` |

## Notes transversales
- **tick_value Silver** : valeur encodée en dur — toujours vérifier les spécifications broker réelles
- **Fuseau horaire** : tout le système doit fonctionner en UTC côté code
- **Spread réel broker** : toujours vérifier avant d'inclure un actif (ex: Nikkei — 10 026 refus en 16 jours)
- **Backtest cohérence timeframe/paramètres** : SMA 30 sur H1 ≠ SMA 30 sur D1
- **Backtest méthode composée obligatoire** : recalculer les lots sur le capital courant à chaque trade
- **Paradoxe portefeuille** : un actif excellent isolément peut dégrader le portefeuille global
- **Risque réel vs théorique** : vérifier le coût réel en EUR avant déploiement
- **INITIAL_BALANCE** : toujours coder en dur et documenter
- **Vérification données avant backtest** : contrôler première et dernière ligne de chaque fichier source
- **Seuils de range momentum** : ajuster selon la période réelle
- **MODE_DEFENSIF** : bug structurel — vérifier `grep -n "^MODE_DEFENSIF"` après déploiement
- **Performances non additives en portefeuille** : raisonner en contribution pondérée
- **Projet ICT Robot** : Order Blocks (FVG, BOS, displacement), Fibonacci, multi-OB, filtrage session, multi-timeframe HTF→LTF
- **Fractionnement d'actions** : testé négatif sur UFUNDED (lot minimum = 1)
- **SL individuel vs garde-fou portefeuille** : distinguer protection idiosyncratique vs choc généralisé
- **Range Breakout Polyvalent** : moteur générique. Gestion timezone via `zoneinfo`. Deux modes : FVG et BREAKOUT. Résultats NASDAQ/SP500 décevants en configuration de base (WR < 25% sur R:R 3:1). Gold en cours de test. Script `extract_years.py` disponible pour découper les gros fichiers historiques.
- **extract_years.py** : outil utilitaire générique pour extraire une plage d'années depuis un fichier tick CSV. Lecture ligne par ligne en streaming, comparaison rapide sur positions 6:10 du timestamp. Stocké avec les scripts Range Breakout Polyvalent pendant la phase de développement. À terme → dossier Outils/Utils.
- **Conclusions intermédiaires Range Breakout** : sur NASDAQ et SP500, win rate systématiquement sous le seuil de rentabilité (25% pour R:R 3:1, 33.3% pour R:R 2:1). Vendredi seul jour profitable sur NASDAQ (WR 26.4%). Mercredi pire jour sur les deux actifs. SP500 BUY meilleur que SELL. La stratégie est plus efficace sur les marchés volatils (ranges > 30 pts) — le Gold est le prochain actif à tester.
- **Conventions de mesure backtests** : toujours exprimer les performances en % du capital (pas en points de prix). Ne jamais sommer des % annuels calculés sur des capitaux réinitialisés — utiliser la moyenne annuelle comme indicateur synthétique. Les points de prix sont réservés aux paramètres techniques (SL, range, spread).
- **Biais M1 pour simulation trailing** : le mode M1 (bougies précalculées) est incompatible avec la détection FVG (nécessite bid/ask séparés). La simulation trailing V8 reste en mode TICKS avec reconstruction vectorisée des bougies 1min. La détection au close de bougie introduit un biais minimal et constant entre runs.
- **Bug TP_TRAIL V8** : corrigé — TP_TRAIL attribué uniquement si le prix de sortie est favorable par rapport au prix d'entrée. Avant correction, tous les trades sortaient en TP_TRAIL (WR 100% avec P&L négatif).
- **Sharpe inadapté aux stratégies non-quotidiennes** : le ratio de Sharpe pénalise les jours sans trade (rendement 0). Pour cette stratégie (~1 trade/jour max), préférer l'espérance et le profit factor comme métriques principales.
- **Note de synthèse** : à rédiger en français pour le groupe d'algo trading. Graphiques en % du capital si on ne peut pas calculer la vraie valeur en monnaie fiat (euros ou dollars selon la devise du compte). Pas de mélange points/pourcentages. Structure : contexte → méthodologie → stratégie → résultats par actif → comparatif modes de sortie → conclusions.
  **Range Breakout V8 — conclusions** : le trailing stop ATR n'améliore pas structurellement la stratégie FVG sur le NASDAQ. Meilleure configuration : factor=3, BE=0 (moyenne −11,5%/an vs −23,6% en TP fixe). Seule 2024 est spectaculairement positive (+53,1%) grâce à la forte directionnalité du NASDAQ. Le problème est structurel — le signal FVG génère trop peu de vrais mouvements persistants. Prochaine étape : filtres additionnels (vendredi uniquement, MIN_RANGE plus élevé).
- **Note de synthèse** : ✅ Rédigée en français, exportée en PDF (juin 2026). Disponible sous `notes/Note_synthese_Range_Breakout_FINALE.pdf`.

## 💡 Idées de stratégies (backlog)
_ne pas hésiter à aller consulter le site internet SSRN https://www.ssrn.com/ssrn/ qui regorge d'informations utiles pour identifier des stratégies éprouvées_

- **Carte d'identité d'un actif** : système pour caractériser un actif (régime/phase à l'instant t).  ✅ promu en projet actif — voir projets/CARTE_IDENTITE_SYMBOL.md (méthodologie complète en cours de construction, premier cas d'application AUDCAD) ;
- **Couplage VIX** : corrélation décalée avec le VIX. À tester en walk-forward 70/30
- **Stratégie "F14"** : trend-following → mean-reversion → breakout → EA MT5 unique avec filtres de régime
- **Gestionnaire de risque adaptatif** : augmenter le risque en phase favorable, réduire en phase défavorable
- **Performance multi-timeframes** : combinaison fractale, entrées/sorties sur timeframes différentes
- **Corrélation indicateurs ↔ mouvement bougie suivante** : valider en walk-forward
- **Algo DCA intelligent** : pente SMA200 positive comme condition d'entrée
- **Facteurs macro-économiques** : lien mesurable avec évolution des indices
- **Données publiques en direct** : Yahoo Finance pour force relative entre actifs
- **Risk-on / Risk-off** : cumuler performances pour lisser la courbe d'equity
- **Variantes pour multi-comptes prop firms** : éviter d'être identifié comme copie de signal
- **Stratégie "Dow"** : points hauts/bas pour optimiser le timing d'entrée
- **Mean Reversion** : déterminer dans quelles conditions on a un système de range horizontal établi, puis les conditions dans lesquelles à partir de quel écart vis-à-vis de la moyenne un trade pris lorsque le prix est en train de s'écarter de la moyenne revient à coup sûr à la moyenne
- **Bougies** : à partir d'un FVG, calculer la probabilité qu'il revienne dans les 50% du FVG, ce qui permettrait de passer des trades

## 📌 Tâches en attente (projets existants)
- **EA_FX_Universal** : retester avec Tick Data Suite, interrupteur module gestion risque, augmenter niveau de risque
- **Range Breakout Polyvalent** : attendre résultats analyse exploratoire Gold → lancer backtest Gold → décider de la suite (filtres additionnels, autres actifs, EA MT5)
