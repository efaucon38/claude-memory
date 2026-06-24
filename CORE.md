# TRADING BOT MEMORY — CORE
_Last updated: 2026-06-24_

💡 NOTE POUR CLAUDE : Dès lecture de ce fichier, rappelle à l'utilisateur de te fournir le fichier projet correspondant si ce n'est pas déjà fait, et propose de mettre à jour la mémoire en fin de session.

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
- **Tick Data Suite** : données ticks Dukascopy
  - Chemin local : `C:\Users\ericf\Documents\Trading algo\Tick Data Suite\`
  - Format fichiers : `AAAA-MM-JJ - AAAA-MM-JJ - SYMBOLE_GMT+0_NO-DST ticks.csv`
  - Format données : `JJ.MM.AAAA HH:MM:SS.mmm, Bid, Ask, VolBid, VolAsk` (GMT+0, sans DST)
  - ⚠️ Toujours vérifier première/dernière ligne avant tout backtest (nom de fichier ≠ contenu réel — expérience EURUSD/XAUUSD/USA_100)
  - Workflow obligatoire : télécharger d'abord, exporter les ticks ensuite
- **Statut actuel des fichiers** :

| Symbole | Période réelle | Statut |
|---------|---------------|--------|
| EURUSD  | 01.2020→02.2026 | ✅ Utilisable |
| GBPJPY  | 01.2020→02.2026 | ✅ Utilisable |
| USDJPY  | 01.2020→02.2026 | ✅ Utilisable |
| XAGUSD  | 01.2020→02.2026 | ✅ Utilisable |
| XAUUSD  | tronqué oct.2021 | 🔴 À relancer |
| USA_100 | 2011-10-01→2026-05-31 | ✅ Utilisable (28.77 Go) |

## Mode de collaboration (interface Claude — programmation Python/MQL5)
- L'utilisateur décrit l'idée ou le besoin
- Claude propose une approche et attend validation avant tout codage
- On code, on teste, on ajuste ensemble, en enrichissement mutuel
- ⚠️ RÈGLE ABSOLUE : Claude ne lance JAMAIS de codage sans validation explicite de l'utilisateur

## Règles d'autonomie — Claude Code
Ces règles sont spécifiques à Claude Code (accès direct aux fichiers locaux, exécution de commandes). Elles remplacent la règle ci-dessus dans ce contexte, qui visait la rédaction de scripts via l'interface Claude classique.

🔴 **Validation requise (uniquement)** :
- Avant de lancer la rédaction d'un script de robot/stratégie de trading (ex: Ponderation.py, EA MT5) — on fait le tour du sujet ensemble d'abord
- Avant de livrer une version finalisée d'un tel script — on se met d'accord au préalable sur les paramètres principaux, notamment ceux qui touchent au risque réel (% risque, taille de lot, etc.), même si des itérations/tests ont eu lieu en autonomie entre les deux
- Actions destructives ou difficiles à inverser — suppression, écrasement, git reset --hard, commit/push

🟢🟡 **Tout le reste en autonomie** (lecture, recherche, analyse, vérification de données, tests de paramètres en backtest/itération, édition de fichiers de mémoire/doc, etc.) — du moment qu'on se resynchronise avant la finalisation.

Note pour Claude : suggérer à l'utilisateur de changer de mode d'autonomie quand cela semble pertinent pour la tâche en cours.

**Aucun accès au VPS** : Claude Code n'a et n'aura aucun accès direct au VPS (pas de SSH, pas d'identifiants, pas d'outil réseau) sauf configuration explicite future par l'utilisateur. Les deux environnements restent cloisonnés.

**Récupération des données** : Les sources de données potentielles sont multiples : données du broker disponible via l'API MT5, données Tick Data Suite disponibles sur ordinateur local après téléchargement, données publiques accessible via Yahoo Finance. Leur utilisation dépendra de la nature du projet. L'outil d'exécution de code de l'interface claude.ai classique n'a pas d'accès réseau direct (confirmé sur le projet UFUNDED — pip/yfinance inutilisables dans ce bac à sable claude.ai spécifique). Claude y conserve cependant un accès web via ses outils de recherche/navigation (web_search, web_fetch) pour consulter de la documentation ou des pages publiques. Pour tout script nécessitant un téléchargement de données de marché en direct dans ce contexte, le script doit être exécuté localement par l'utilisateur (ou via Claude Code), qui renvoie les résultats (CSV) pour interprétation.

## Risk Management
- Risque max par trade : **1% du capital** par défaut
- Toute dérogation à cette règle doit être décidée explicitement par l'utilisateur
- Priorité : prudence et préservation du capital avant performance

## Conventions de code
- Séparer clairement : logique de signal / calcul de lot / exécution ordre
- Toujours documenter les paramètres clés en tête de fichier
- Nommage des stratégies : `ACTIF_TIMEFRAME_LOGIQUE` (ex: EURUSD_H1_EMA_CROSS)

## Conventions Python backtest (tick data)
- Données tick : CSV Tick Data Suite, GMT+0, NO-DST, format `DD.MM.YYYY HH:MM:SS.mmm`
- Colonnes : timestamp | ask | bid | vol_ask | vol_bid (sans header)
- Timestamps en **microsecondes int64** (pandas 2.x → `datetime64[us]`, `.asi8` est en µs)
- ⚠️ Ne jamais mélanger `.asi8` (µs) et `pd.Timestamp.value` (ns) dans la même comparaison
- Parse vectorisé : extraction numérique directe sur strings de longueur fixe (plus rapide que strptime)
- Dispatch par jour : `np.diff` vectorisé sur array de day_ids — **zéro boucle Python sur les ticks**
- Bougies 1min : `np.maximum.reduceat` / `np.minimum.reduceat`
- Simulation SL/TP : `np.argmax` sur array booléen
- Indicateurs : buffers incrémentaux (EMABuffer, RSIBuffer, ATRBuffer, SlopeBuffer), accès `.values` directs (jamais `.loc` en boucle)
- Architecture obligatoire pour fichiers multi-Go : lecture unique et séquentielle, chunks de 500k lignes

## ⚠️ Erreurs récurrentes — NE JAMAIS reproduire
1. **Codage sans autorisation** : Claude ne produit aucun code sans validation préalable explicite de l'utilisateur
2. **Gestion des fuseaux horaires** : toujours clarifier et documenter l'écart UTC / heure broker en début de projet, ne jamais supposer qu'ils sont identiques
3. **Calcul de la taille des lots** : ne jamais confondre % du capital et valeur absolue — toujours expliciter la formule utilisée et la faire valider
4. **Drawdown "trailing" vs "static"** : ne jamais supposer qu'un drawdown se calcule par rapport au plus-haut glissant — beaucoup de prop firms (dont UFUNDED, confirmé par account manager) utilisent un drawdown STATIQUE, ancré sur le capital initial. Toujours vérifier/faire confirmer la règle exacte du broker/plateforme avant de bâtir une simulation de risque.
5. **Hypothèses de simulation non explicitées** : tout paramètre structurant d'un backtest (fréquence de rebalancement, fractionnement des actions, prise en compte des commissions...) doit être annoncé clairement AVANT de lancer la simulation.
6. **Walk-forward obligatoire** : tout backtest doit séparer le jeu de données en 70% (entraînement) et 30% (test). Jamais de look-ahead bias.
7. **État du marché en parallèle du backtest** : toujours calculer des métriques de régime de marché (volatilité, tendance, range...) pour identifier a posteriori dans quelles phases la stratégie performe.
8. **Spécificités broker par actif** : toujours vérifier les spécificités réelles du broker pour chaque actif cible (tick value, spread, marge, horaires) avant déploiement.
9. **pandas 2.x datetime64[us]** : `.asi8` retourne des microsecondes, pas des nanosecondes — ne jamais mélanger les unités dans une même comparaison temporelle.
10. **Boucle Python sur les ticks** : toujours vectoriser avec numpy — une boucle `for` sur des millions de ticks est rédhibitoire en performance (facteur ×1000 vs vectorisé).

## Projets (index)
| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | `projects/TELEGRAM_MT5_KASPER.md` |
| EA Desk Quant | 8 actifs (propfirm 7 + Gold) | Test démo — 2 comptes actifs, v3.0 OFFENSIF | `projects/MULTI_ACTIFS_DRAWDOWN.md` |
| Momentum Scanner | Multi-actifs broker (à définir) | Construction | `projects/MOMENTUM_SCANNER.md` |
| Fourier | EURUSD / USDJPY / NAS100 (recherche exploratoire) | Clos — verdict négatif | `projects/FOURIER.md` |
| ICT_ROBOT | Multi-actifs | Construction | `projects/ICT_ROBOT.md` |
| UFUNDED_OVERNIGHT_PORTFOLIO | US Stocks / ETF | Allocation et sizing finalisés | `projects/UFUNDED.md` |
| **Range Breakout 9h30 EST + FVG** | **NASDAQ (US100), extensible** | **Backtest Python v3 en cours d'exécution** | **`projects/RANGE_BREAKOUT_9H30_EST.md`** |

## Dernières versions codées (index)
| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | `bot_files/botTelegram_Kasper_code.md` |
| EA Desk Quant — Ponderation v3.0 OFFENSIF | 7 actifs faible corrélation + Gold | 2 comptes actifs en test démo | `bot_files/Ponderation_code.md` |
| EA Desk Quant — EA_FX_Universal v2.9 | 7 actifs faible corrélation (propfirm) | Test démo | `bot_files/EA_FX_Universal_code.md` |
| **Range Breakout 9h30 EST — script Python v3** | **NASDAQ (US100)** | **Backtest en cours** | **`bot_files/Robot_range_breakout_9h30_EST_Code.md`** |

## Bibliothèque (index)
| Sujet | Fichier |
|-------|---------|
| Gestion du temps en trading | `library/time_issues_and_trading.md` |

## Notes transversales
- **tick_value Silver** : valeur encodée en dur — toujours vérifier les spécifications broker réelles avant tout nouveau déploiement ou ajout d'actif
- **Fuseau horaire** : tout le système doit fonctionner en UTC côté code. Toujours documenter l'écart UTC/heure broker en début de projet.
- **Spread réel broker** : toujours vérifier le spread réel en production avant d'inclure un actif, même si le backtest est excellent (ex: Nikkei — 10 026 refus en 16 jours sur RaiseFX)
- **Backtest : cohérence timeframe/paramètres** : les paramètres doivent correspondre au timeframe du backtest — une SMA 30 sur H1 ≠ SMA 30 sur D1
- **Backtest : méthode composée obligatoire** : toujours recalculer les lots sur le capital courant à chaque trade — la méthode additive sous-estime la capitalisation de +7 à +10 pts sur 7 ans
- **Paradoxe portefeuille** : un actif excellent isolément peut dégrader le portefeuille global (ex: Gold). Toujours backtester au niveau portefeuille, pas uniquement par actif
- **Risque réel vs théorique** : toujours vérifier le coût réel d'un trade en EUR avant déploiement (Silver/Gold peuvent être 7×-14× le risque théorique si mal calibrés)
- **INITIAL_BALANCE** : toujours la coder en dur et la documenter — une valeur automatique peut être erronée après des trades ou un rechargement de compte
- **Vérification données avant backtest** : toujours contrôler première et dernière ligne de chaque fichier source avant utilisation. Claude Code peut automatiser ce contrôle sur les fichiers locaux.
- **Seuils de range momentum** : toujours ajuster le seuil selon la période réelle (seuil = base_60j × période/60). Un seuil fixe biaise les conclusions sur les périodes courtes/longues
- **MODE_DEFENSIF** : attention au bug structurel — la valeur effective est sur une ligne seule distincte des commentaires. Toujours vérifier `grep -n "^MODE_DEFENSIF"` après déploiement
- **Performances non additives en portefeuille** : un actif à +7.7%/an isolément peut contribuer seulement +0.7%/an en portefeuille. Toujours raisonner en contribution pondérée
- **Projet ICT Robot** (démarré 2026-06-16) : objectif = robot ICT optimal en Python/MT5. Concepts clés : Order Blocks (avec FVG, BOS, displacement), Fibonacci, gestion multi-OB, filtrage par session, analyse multi-timeframe (HTF→LTF).
- **Fractionnement d'actions** : ne jamais supposer qu'une plateforme permet l'achat de fractions. Testé négatif sur UFUNDED (lot minimum = 1, sur actions ET ETF).
- **SL individuel vs garde-fou portefeuille** : bien distinguer le rôle d'un SL par ligne (protection idiosyncratique) de celui d'un palier de drawdown portefeuille (protection contre choc généralisé). Un SL trop serré peut être contre-productif sur une stratégie cœur stable — toujours tester la sensibilité à plusieurs seuils.
- **Range Breakout 9h30 EST** : backtest Python v3 fully vectorized. Architecture lecture unique obligatoire pour fichiers tick multi-Go. DST US géré dans le code (pas dans les données). Script polyvalent : changer `FILE_PATH`, `ASSET_NAME` et `CONTRACT_VALUE` suffit pour tester un autre actif.

## 💡 Idées de stratégies (backlog)
_(inchangé — conservé tel quel)_

- **Carte d'identité d'un actif** : système pour caractériser un actif (régime/phase à l'instant t) de façon synthétique.
- **Couplage VIX** : tester si certains actifs présentent une corrélation décalée (lead/lag) avec le VIX. À tester en walk-forward 70/30.
- **Stratégie "F14"** (programme multi-étapes) : trend-following → mean-reversion → breakout → combinaison dans un EA MT5 unique avec filtres de régime.
- **Gestionnaire de risque adaptatif** : augmenter le risque en phase favorable (enchaînement de TPs) et le réduire en phase défavorable.
- **Performance multi-timeframes** : tester une même stratégie sur plusieurs timeframes, explorer la combinaison fractale et les entrées/sorties sur timeframes différentes.
- **Corrélation indicateurs ↔ mouvement bougie suivante** : analyser un large éventail d'indicateurs et identifier ceux corrélés au mouvement de la bougie suivante. Valider en walk-forward.
- **Algo DCA intelligent** : ne prendre position que si la pente SMA200 est positive ; sortir quand la pente s'inverse.
- **Facteurs macro-économiques** : lien entre facteurs macro mesurables et évolution des indices boursiers, à différentes timeframes.
- **Données publiques en direct** : récupération via Yahoo Finance pour classement de force relative entre actifs.
- **Risk-on / Risk-off** : identifier actifs performants en risk-on et risk-off, cumuler leurs performances pour lisser la courbe d'equity.
- **Variantes de stratégies pour multi-comptes prop firms** : construire des versions différentes apparentées pour éviter d'être identifié comme copie de signal.
- **Stratégie "Dow"** : sur actifs en tendance avec structure claire, identifier points hauts/bas pour optimiser le timing d'entrée.

## 📌 Tâches en attente (projets existants)
- **EA_FX_Universal** : retester avec les données Tick Data Suite, en ajoutant un interrupteur sur le module de gestion du risque (désapprouvé par Léo), et en augmentant le niveau de risque pour observer l'impact sur le drawdown quotidien et global.
