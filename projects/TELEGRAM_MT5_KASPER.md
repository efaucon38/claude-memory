
# PROJET : TELEGRAM_MT5_KASPER
_Période: 2026 → en cours | Statut: test démo_

## Objectif
Parser des signaux Telegram d'un provider (Kasper) et exécuter automatiquement les trades correspondants sur MT5 via l'API Python.

## Architecture
- **Fichier principal** : `botTelegramMT5_Kasper_ACTIF.py`
- **Flux** : Signal Telegram → Parser Python → API Python MT5 → Exécution ordre
- **Signal entrant** : actif, sens (BUY/SELL), SL, TP1, TP2, TP3
- **Exécution** : via API Python MT5 (pas d'EA MQL5)

## Ce qui fonctionne
- Parsing des signaux Telegram opérationnel
- Prise des pertes (SL) correctement gérée
- Collecte de données en cours semaine après semaine

## Ce qui ne fonctionne pas / problèmes identifiés
- Mauvaise gestion des gains : les TP (TP1, TP2, TP3) ne sont pas correctement pris
- Résultats globaux insuffisants pour passer en prod

## Statut actuel
- En test sur compte démo
- Phase d'acquisition de données et de peaufinage hebdomadaire

## Prochaines étapes
- Corriger la logique de prise de gains (gestion TP1/TP2/TP3)
- Continuer l'accumulation de données pour valider le système

## Notes
- Le nom de fichier inclut l'actif traité : `botTelegramMT5_Kasper_ACTIF.py`
