
# PROJET : MULTI_ACTIFS_DRAWDOWN
_Période: 2026 → en cours | Statut: test démo_

## Objectif
Système hyper robuste multi-actifs destiné aux comptes propfirm, avec gestion stricte du drawdown.

## Architecture
- **Cerveau** : `Ponderation.py` (Python) — supervision globale, calcul des lots, coordination
- **EA** : `EA_FX_Universal` (MQL5) — exécution des trades sur chaque graphique MT5
- **Communication** : fichiers complémentaires échangés entre EA et cerveau
- **Actifs** : 8 actifs sélectionnés pour leur faible corrélation inter-actifs

## Logique clé
- Taille des lots calculée par le cerveau Python selon plusieurs règles :
  - Pondération inversement proportionnelle à la volatilité (favorise la stabilité)
  - Règles de drawdown global
- EA universel : reconnaît automatiquement l'actif sur lequel il est déployé et applique les règles spécifiques correspondantes
- Coordination centralisée via le cerveau Python

## Ce qui fonctionne
- Architecture cerveau/EA opérationnelle
- Sélection des 8 actifs faiblement corrélés effectuée
- Pondération par volatilité implémentée

## Ce qui ne fonctionne pas / points en suspens
- Encore en phase de test — résultats à consolider

## Statut actuel
- En test sur compte démo

## Prochaines étapes
- Valider les résultats sur démo avant passage en propfirm

## Notes
- Priorité absolue : robustesse et protection du capital (usage propfirm)
