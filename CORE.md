
# TRADING BOT MEMORY — CORE
_Last updated: 2026-06-06_

> 💡 **NOTE POUR CLAUDE** : Dès lecture de ce fichier, rappelle à l'utilisateur
> de te fournir le fichier projet correspondant si ce n'est pas déjà fait,
> et propose de mettre à jour la mémoire en fin de session.

## Stack technique
- **Langages** : Python (backtest, analyse, outillage) + MQL5 (exécution live sur MT5)
- **Broker actuel** : RaiseFX (MT5) — architecture pensée pour être broker-agnostique
- **Backtest** : piloté par Claude en Python, à partir de données exportées du broker
- **Marchés** : Forex, Crypto, Indices (DAX, SP500...), Actions/ETF

## Mode de collaboration
- L'utilisateur décrit l'idée ou le besoin
- Claude propose une approche et attend validation avant tout codage
- On code, on teste, on ajuste ensemble, en enrichissement mutuel
- ⚠️ RÈGLE ABSOLUE : Claude ne lance JAMAIS de codage sans validation explicite de l'utilisateur

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
| Multi-actifs Drawdown (EA Desk Quant) | 7 actifs faible corrélation (propfirm) | Test démo | projects/MULTI_ACTIFS_DRAWDOWN.md |
| Momentum Scanner | Multi-actifs broker (à définir) | Construction | projects/MOMENTUM_SCANNER.md |

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
