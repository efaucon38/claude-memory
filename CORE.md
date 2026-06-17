# TRADING BOT MEMORY — CORE
_Last updated: 2026-06-17_

> 💡 **NOTE POUR CLAUDE** : Dès lecture de ce fichier, rappelle à l'utilisateur
> de te fournir le fichier projet correspondant si ce n'est pas déjà fait,
> et propose de mettre à jour la mémoire en fin de session.

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
  - Clone local du repo mémoire : C:\TradingBots\claude-memory\
- **Tick Data Suite** : données ticks Dukascopy
  - Chemin local : C:\Users\ericf\Documents\Trading algo\Tick Data Suite\
  - Format fichiers : AAAA-MM-JJ - AAAA-MM-JJ - SYMBOLE_GMT+0_NO-DST ticks.csv
  - Format données : JJ.MM.AAAA HH:MM:SS.mmm, Bid, Ask, VolBid, VolAsk (GMT+0, sans DST)
  - ⚠️ Toujours vérifier première/dernière ligne avant tout backtest
    (nom de fichier ≠ contenu réel — expérience EURUSD/XAUUSD/USA_100)
  - Workflow obligatoire : télécharger d'abord, exporter les ticks ensuite
  - Statut actuel des fichiers :
    | Symbole | Période réelle   | Statut       |
    |---------|------------------|--------------|
    | EURUSD  | 01.2020→02.2026  | ✅ Utilisable |
    | GBPJPY  | 01.2020→02.2026  | ✅ Utilisable |
    | USDJPY  | 01.2020→02.2026  | ✅ Utilisable |
    | XAGUSD  | 01.2020→02.2026  | ✅ Utilisable |
    | XAUUSD  | tronqué oct.2021 | 🔴 À relancer |
    | USA_100 | en cours export  | ⏳ Attendre   |

## Mode de collaboration (interface Claude — programmation Python/MQL5)
- L'utilisateur décrit l'idée ou le besoin
- Claude propose une approche et attend validation avant tout codage
- On code, on teste, on ajuste ensemble, en enrichissement mutuel
- ⚠️ RÈGLE ABSOLUE : Claude ne lance JAMAIS de codage sans validation explicite de l'utilisateur

## Règles d'autonomie — Claude Code
> Ces règles sont spécifiques à Claude Code (accès direct aux fichiers locaux,
> exécution de commandes). Elles remplacent la règle ci-dessus dans ce contexte,
> qui visait la rédaction de scripts via l'interface Claude classique.

**🔴 Validation requise (uniquement) :**
1. Avant de lancer la rédaction d'un script de robot/stratégie de trading
   (ex: Ponderation.py, EA MT5) — on fait le tour du sujet ensemble d'abord
2. Avant de livrer une version finalisée d'un tel script — on se met d'accord
   au préalable sur les paramètres principaux, notamment ceux qui touchent
   au risque réel (% risque, taille de lot, etc.), même si des itérations/tests
   ont eu lieu en autonomie entre les deux
3. Actions destructives ou difficiles à inverser — suppression, écrasement,
   `git reset --hard`, commit/push

**🟢🟡 Tout le reste en autonomie** (lecture, recherche, analyse, vérification
de données, tests de paramètres en backtest/itération, édition de fichiers de
mémoire/doc, etc.) — du moment qu'on se resynchronise avant la finalisation.

**Note pour Claude** : suggérer à l'utilisateur de changer de mode d'autonomie
quand cela semble pertinent pour la tâche en cours (ex: proposer plus de cadrage
si un sujet devient sensible, ou plus de liberté si une tâche est répétitive
et sans enjeu).

**Aucun accès au VPS** : Claude Code n'a et n'aura aucun accès direct au VPS
(pas de SSH, pas d'identifiants, pas d'outil réseau) sauf configuration
explicite future par l'utilisateur. Les deux environnements restent cloisonnés.

**Récupération des données** : Les sources de données potentielles sont multiples : données du broker disponible via l'API MT5, données Tick Data Suite disponibles sur ordinateur local après téléchargement, données publiques accessible via Yahoo Finance. Leur utilisation dépendra de la nature du projet. L'outil d'exécution de code de l'interface claude.ai classique n'a pas d'accès réseau direct (confirmé sur le projet UFUNDED — pip/yfinance inutilisables dans ce bac à sable claude.ai spécifique). Claude y conserve cependant un accès web via ses outils de recherche/navigation (web_search, web_fetch) pour consulter de la documentation ou des pages publiques. Pour tout script nécessitant un téléchargement de données de marché en direct dans ce contexte, le script doit être exécuté localement par l'utilisateur (ou via Claude Code, qui s'exécute sur la machine locale et n'a pas cette restriction), qui renvoie les résultats (CSV) pour interprétation. 

## Risk Management
- Risque max par trade : **1% du capital** par défaut
- Toute dérogation à cette règle doit être décidée explicitement par l'utilisateur
- Priorité : prudence et préservation du capital avant performance

## Conventions de code
- Séparer clairement : logique de signal / calcul de lot / exécution ordre
- Toujours documenter les paramètres clés en tête de fichier
- Nommage des stratégies : `ACTIF_TIMEFRAME_LOGIQUE` (ex: EURUSD_H1_EMA_CROSS)

## ⚠️ Erreurs récurrentes — NE JAMAIS reproduire
1. **Codage sans autorisation** : Claude ne produit aucun code sans validation préalable explicite de l'utilisateur
2. **Gestion des fuseaux horaires** : toujours clarifier et documenter l'écart UTC / heure broker en début de projet, ne jamais supposer qu'ils sont identiques
3. **Calcul de la taille des lots** : ne jamais confondre % du capital et valeur absolue — toujours expliciter la formule utilisée et la faire valider
4. **Drawdown "trailing" vs "static"** : ne jamais supposer qu'un drawdown se calcule par rapport au plus-haut glissant (convention par défaut en gestion de portefeuille classique) — beaucoup de prop firms (dont UFUNDED, confirmé par account manager) utilisent un drawdown STATIQUE, ancré sur le capital initial, qui ne bouge jamais même après profits ou payouts. Toujours vérifier/faire confirmer la règle exacte du broker/plateforme avant de bâtir une simulation de risque.
5. **Hypothèses de simulation non explicitées** : tout paramètre structurant d'un backtest (fréquence de rebalancement, fractionnement des actions, prise en compte des commissions...) doit être annoncé clairement AVANT de lancer la simulation, pas découvert a posteriori suite à une question de l'utilisateur (erreur commise sur le projet UFUNDED : poids simulés en % continus sans vérifier d'abord la faisabilité en lots entiers).
6. **Walk-forward obligatoire** : tout backtest doit séparer le jeu de données en 70% (entraînement/paramétrage) et 30% (test sur données inconnues). Jamais de look-ahead bias. Le paramétrage validé sur les 70% doit être rejoué tel quel sur les 30% restants pour juger de la robustesse réelle.
7. **État du marché en parallèle du backtest** : pour tout backtest, calculer en parallèle des métriques caractérisant le régime de marché du moment (volatilité, tendance, range...) afin d'identifier a posteriori dans quelles phases la stratégie est performante ou non — pas seulement le P&L brut.
8. **Spécificités broker par actif** : toujours vérifier les spécificités réelles du broker pour chaque actif cible (tick value, spread, marge, horaires) avant déploiement — ne jamais supposer une valeur générique.

## Projets (index)

| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | projects/TELEGRAM_MT5_KASPER.md |
| EA Desk Quant | 8 actifs (propfirm 7 + Gold) | **Test démo — 2 comptes actifs, v3.0 OFFENSIF** | projects/MULTI_ACTIFS_DRAWDOWN.md |
| Momentum Scanner | Multi-actifs broker (à définir) | Construction | projects/MOMENTUM_SCANNER.md |
| Fourier | EURUSD / USDJPY / NAS100 (recherche exploratoire) | **Clos — verdict négatif** | projects/FOURIER.md |
| ICT_ROBOT | Multi-actifs | Construction | projects/ICT_ROBOT.md |
| UFUNDED_OVERNIGHT_PORTFOLIO | US Stocks / ETF | Allocation et sizing finalisés — ordres prévus pour être passés prochainement (marché fermé pour le moment) | projects/UFUNDED.md |

## Dernières versions codées (index)

| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | bot_files/botTelegram_Kasper_code.md |
| EA Desk Quant — Ponderation v3.0 OFFENSIF, périodes corrigées | 7 actifs faible corrélation (propfirm) + Gold (compte propre) | 2 comptes actifs en test démo | bot_files/Ponderation_code.md |
| EA Desk Quant — EA_FX_Universal v2.9 | 7 actifs faible corrélation (propfirm) | Test démo | bot_files/EA_FX_Universal_code.md |

## Bibliothèque (index)

| Sujet | Fichier |
|--------|--------|
| Gestion du temps en trading | library/time_issues_and_trading.md |

## Notes transversales
- **tick_value Silver** : valeur encodée en dur — toujours vérifier les spécifications
  broker réelles avant tout nouveau déploiement ou ajout d'actif (erreur reproduite
  sur les deux projets Telegram et EA Desk Quant)
- **Fuseau horaire** : tout le système doit fonctionner en UTC côté code.
  Toujours documenter l'écart UTC/heure broker en début de projet.
- **Spread réel broker** : toujours vérifier le spread réel en production avant
  d'inclure un actif, même si le backtest est excellent (ex: Nikkei — 10 026 refus
  en 16 jours sur RaiseFX)
- **Backtest : cohérence timeframe/paramètres** : les paramètres doivent correspondre
  au timeframe du backtest — une SMA 30 sur H1 ≠ SMA 30 sur D1 (erreur EA Desk Quant)
- **Backtest : méthode composée obligatoire** : toujours recalculer les lots sur le
  capital courant à chaque trade — la méthode additive sous-estime la capitalisation
  de +7 à +10 pts sur 7 ans
- **Paradoxe portefeuille** : un actif excellent isolément peut dégrader le portefeuille
  global (ex: Gold). Toujours backtester au niveau portefeuille, pas uniquement par actif
- **Risque réel vs théorique** : toujours vérifier le coût réel d'un trade en EUR
  avant déploiement (Silver/Gold peuvent être 7×-14× le risque théorique si mal calibrés)
- **INITIAL_BALANCE** : toujours la coder en dur et la documenter — une valeur
  automatique peut être erronée après des trades ou un rechargement de compte
- **Vérification données avant backtest** : toujours contrôler première
  et dernière ligne de chaque fichier source avant utilisation.
  Claude Code peut automatiser ce contrôle sur les fichiers locaux.
- **Seuils de range momentum** : toujours ajuster le seuil selon la période réelle
  (`seuil = base_60j × période/60`). Un seuil fixe biaise les conclusions sur les
  périodes courtes/longues (erreur corrigée en backtest v3 EA Desk Quant)
- **MODE_DEFENSIF** : attention au bug structurel — la valeur effective est sur une
  ligne seule (`MODE_DEFENSIF = True/False`) distincte des commentaires. Toujours
  vérifier `grep -n "^MODE_DEFENSIF"` après déploiement d'un nouveau fichier
- **Performances non additives en portefeuille** : un actif à +7.7%/an isolément
  peut contribuer seulement +0.7%/an en portefeuille (Gold : 9% × 7.7%). Toujours
  raisonner en contribution pondérée, pas en performance individuelle
- **Projet ICT Robot** (démarré 2026-06-16) : objectif = robot ICT optimal en Python/MT5.
  Concepts clés à implémenter : Order Blocks (avec FVG, BOS, displacement), niveaux 
  de Fibonacci comme points d'entrée potentiels, gestion multi-OB, filtrage par
  session, analyse multi-timeframe (HTF→LTF). Claude positionné comme expert ICT
  + Python/MT5 sur ce projet.
- **Fractionnement d'actions à vérifier systématiquement** : ne jamais supposer qu'une plateforme permet l'achat de fractions d'actions/ETF. Testé négatif sur UFUNDED (lot minimum = 1, sur actions ET ETF) — contrairement aux brokers retail US classiques (Fidelity, Schwab...). Le cash résiduel d'arrondi lié aux lots entiers peut peser plus lourd que les commissions elles-mêmes sur un portefeuille multi-lignes à capital limité.
- **SL individuel vs garde-fou portefeuille** : bien distinguer le rôle d'un Stop-Loss par ligne (protection contre un accident idiosyncratique isolé) de celui d'un palier de drawdown au niveau du portefeuille global (protection contre un choc de marché généralisé, où toutes les lignes baissent ensemble et où le SL individuel n'aide pas). Un SL trop serré peut être contre-productif sur une stratégie cœur stable/buy-and-hold, car il coupe des positions saines sur du simple bruit cyclique — toujours tester la sensibilité à plusieurs seuils avant de figer (cf. projet UFUNDED : -8% contre-productif dans 7 cas/8 sur backtest 5 ans, -20% retenu).

## 💡 Idées de stratégies (backlog)
> Idées capturées au fil des réflexions, non développées à ce jour. À revisiter
> quand un créneau de travail se libère ou qu'un projet en cours s'y prête.

1. **Carte d'identité d'un actif** : système pour caractériser un actif (régime/phase à l'instant t) de façon synthétique. Voir module 5.2 ~min 53 pour un descriptif des phases de référence.

2. **Couplage VIX** : comprendre le calcul du VIX et tester si certains actifs présentent une corrélation décalée (lead/lag) avec lui. À tester en walk-forward 70/30.

3. **Stratégie "F14" (programme multi-étapes)** :
   - Construire une stratégie trend-following (un ou plusieurs indicateurs, entrées/sorties éventuellement sur timeframes différents), la faire tourner sur un actif au choix, en mesurant une batterie de paramètres de détection de régime (Aroon, ADX, volatilité...)
   - Classer la performance selon ces paramètres
   - Ajouter des filtres de régime pour ne garder la stratégie active que dans les conditions jugées optimales, puis valider en walk-forward sur les 30% non utilisés
   - Répéter la démarche avec une stratégie mean-reversion (range), puis une stratégie breakout, sur le même actif
   - Explorer d'autres grandes familles de stratégies pertinentes
   - Vérifier la corrélation des résultats entre les différentes stratégies
   - Combiner l'ensemble dans un seul EA MT5 (valable pour un actif donné) et backtester

4. **Gestionnaire de risque adaptatif** : pour une stratégie donnée, mesurer l'impact d'un système augmentant le risque en phase favorable (enchaînement de TPs) et le réduisant en phase défavorable (enchaînement de SLs).

5. **Performance multi-timeframes** : tester une même stratégie sur plusieurs timeframes pour comparer. Explorer la combinaison fractale (ex: D1/H4/H1 ou H4/H1/M15) et la possibilité de prendre les entrées sur une timeframe et les sorties sur une autre (finesse d'exécution).

6. **Corrélation indicateurs ↔ mouvement de la bougie suivante** : pour un actif et une timeframe donnés, analyser un large éventail d'indicateurs et identifier ceux corrélés au mouvement de la bougie suivante. Valider les hypothèses en walk-forward.

7. **Algo DCA intelligent — améliorations** :
   - Ne prendre position que si la pente de la SMA200 est positive (période de calcul de la pente à définir — 10 ? — et seuil minimum à déterminer ; identifier aussi le moment de la journée où les pentes les plus fortes apparaissent)
   - Sortir quand la pente s'inverse
   - Étudier la possibilité de vendre (pas seulement arrêter d'acheter) quand les critères s'inversent

8. **Facteurs macro-économiques** : creuser le lien entre facteurs macro mesurables et évolution des indices boursiers, à différentes timeframes (y compris très amples). Chercher des corrélations exploitables.

9. **Données publiques en direct** : explorer la récupération de données en direct via des sources publiques (Yahoo Finance — CAC40, S&P500, Nasdaq100...) dans une optique de classement de force relative entre actifs.

10. **Risk-on / Risk-off** : identifier des actifs performants en risk-on et d'autres en risk-off, cumuler leurs performances dans le temps pour lisser la courbe d'equity. Question ouverte : comment définir de façon mesurable qu'on est en phase risk-on ou risk-off ?

11. **Variantes de stratégies pour multi-comptes prop firms** : construire des versions différentes mais apparentées d'une même stratégie (indicateurs d'entrée/sortie variés, mixés) pour déployer sur plusieurs comptes (ex: FTMO) sans risquer d'être identifié comme copie de signal.

12. **Stratégie "Dow"** : sur actifs en tendance avec structure claire, identifier points hauts/bas pour optimiser le timing d'entrée.

## 📌 Tâches en attente (projets existants)
- **EA_FX_Universal** : retester avec les données Tick Data Suite, en ajoutant un interrupteur sur le module de gestion du risque (désapprouvé par Léo), et en augmentant le niveau de risque pour observer l'impact sur le drawdown quotidien et global.
