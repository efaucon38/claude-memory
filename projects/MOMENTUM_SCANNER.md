
# PROJET : MOMENTUM_SCANNER
_Période: 2026 → en cours | Statut: construction_

## Objectif
Identifier en temps réel les actifs les plus forts disponibles chez le broker via un scoring multi-critères, puis exécuter des trades en suivi de tendance.

## Architecture prévue
- **Scanner** : Python + API MT5 (pas d'EA MQL5)
- **Flux** : Liste d'actifs prédéfinis → Scoring → Sélection → Exécution ordres

## Critères de scoring
1. Momentum
2. Tendance
3. Structure de marché claire
4. Paramètres complémentaires : spread, (à compléter)

## Ce qui fonctionne
- Concept défini et architecture posée

## Ce qui ne fonctionne pas / points en suspens
- Système en construction, aucun test lancé

## Statut actuel
- Phase de construction — aucun backtest ni test démo effectué

## Prochaines étapes
- Définir précisément les formules de scoring pour chaque critère
- Implémenter le scanner
- Lancer les premiers backtests
