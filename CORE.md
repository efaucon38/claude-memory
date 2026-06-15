
# TRADING BOT MEMORY — CORE
_Last updated: 2026-06-16_

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

## Projets (index)

| Projet | Marché | Statut | Fichier |
|--------|--------|--------|---------|
| Telegram MT5 Kasper | Multi-actifs (signaux provider) | Test démo | projects/TELEGRAM_MT5_KASPER.md |
| EA Desk Quant | 8 actifs (propfirm 7 + Gold) | **Test démo — 2 comptes actifs, v3.0 OFFENSIF** | projects/MULTI_ACTIFS_DRAWDOWN.md |
| Momentum Scanner | Multi-actifs broker (à définir) | Construction | projects/MOMENTUM_SCANNER.md |
| Fourier | EURUSD / USDJPY / NAS100 (recherche exploratoire) | **Clos — verdict négatif** | projects/FOURIER.md |
| ICT_ROBOT | Multi-actifs | Construction | projects/ICT_ROBOT.md |

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
  Concepts clés à implémenter : Order Blocks (avec FVG, BOS, displacement), 
  gestion multi-OB, filtrage par session, analyse multi-timeframe (HTF→LTF).
  Claude positionné comme expert ICT + Python/MT5 sur ce projet.
