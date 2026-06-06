# PROJET : MULTI_ACTIFS_DRAWDOWN (EA Desk Quant)
_Période: 2026 → en cours | Statut: test démo | Versions: Ponderation.py v3.3 / EA v2.9_

## Objectif
Système multi-actifs hyper robuste destiné aux comptes propfirm, avec gestion stricte
du drawdown. Un cerveau Python supervise des EA MQL5 universels déployés sur chaque
graphique MT5. Validé sur 7.3 ans de backtest.

## Fichiers
- `Ponderation_{LOGIN}.py` — cerveau Python (login dans le nom du fichier)
- `EA_FX_Universal.mq5` — EA MQL5 universel (un seul fichier pour tous les actifs)
- `portfolio_weights_RaiseFX_{TYPE}_{LOGIN}.csv` — canal de communication Python→EA
- `InitBalance_{AccountID}.csv` — balance initiale écrite par Python, lue par EA
- `Heartbeat_{SYMBOLE}_{AccountID}.csv` — heartbeat par EA, lu par Python
- `EA_Journal_{AccountID}.csv` — journal de trades par EA
- `counters_{AccountID}.json` — état persistant Python (streaks, drawdowns)

## Architecture
```
Ponderation.py (cycle 60s)
    → connexion MT5 : balance, volatilité (ATR), positions
    → calcul drawdown journalier/total → RiskFactor (1.0/0.5/0.0)
    → calcul momentum par actif (période individualisée)
    → pondération hybride : 0.30×(1/vol) + 0.70×|momentum| (mode OFFENSIF)
    → filtre range : |momentum| < seuil → TradeAllowed_Symbol=0
    → dimensionnement lots + calcul RiskEUR
    → écriture portfolio_weights CSV + InitBalance CSV
    → monitoring heartbeats EA

EA_FX_Universal (OnTimer 60s + OnTick)
    → lecture CSV portfolio_weights (poids, lot, autorisations)
    → filtres entrée (vendredi, session, news, spread, gap, liquidité, ATR)
    → signal tendance multi-timeframe (H1+H4+D1)
    → gestion position : BE → trailing stop
    → heartbeat + journal CSV
```

## AccountID
Format : `{BROKER}_{TYPE}_{LOGIN}` — ex : `RaiseFX_demo_5007258`
Apparaît dans tous les noms de fichiers CSV → isolation totale entre comptes.

## Comptes déployés
| Compte   | Mode          | Capital      | INITIAL_BALANCE | Actifs          |
|----------|---------------|--------------|-----------------|-----------------|
| 5005661  | compte_propre | ~1 117 EUR   | 1 000 EUR       | 8 (Gold actif)  |
| 5007258  | propfirm      | 50 000 EUR   | 50 000 EUR      | 7 (Gold veille) |

## Portefeuille actif (propfirm — 7 actifs)
| Actif   | Score BT | PF opt. | Période mom. | Seuil range | Session (UTC) | EMA_D1 |
|---------|----------|---------|--------------|-------------|---------------|--------|
| S&P500  | 71       | 1.270   | 60j          | 2.1%        | 13h-21h       | 20     |
| NAS100  | 71       | 1.150   | 90j          | 1.8%        | 13h-21h       | 20     |
| DAX40   | 68       | 1.071   | 60j          | 1.2%        | 07h-15h       | 20     |
| USDJPY  | 67       | 1.152   | 90j          | 1.5%        | 24h/24        | 50     |
| Silver  | 61       | 1.099   | 90j          | 1.5%        | 12h-20h       | 50     |
| EU50    | 57       | 1.127   | 120j         | 1.2%        | 00h-15h       | 50     |
| GBPJPY  | 55       | 1.076   | 90j          | 1.4%        | 07h-17h       | 50     |

**Exclus :**
- Nikkei : spread RaiseFX 14× le seuil (10 026 refus en 16 jours) — conservé dans WEIGHT_BOUNDS pour réintégration future
- Gold : VEILLE auto en propfirm. En compte propre : non recommandé (dégrade le portefeuille malgré +7.7%/an individuel)
- FTSE100, EURJPY, AUDJPY, GBPUSD, AUDUSD : scores trop faibles (exclus backtest)

## Résultats backtest v3 (7.3 ans, méthode composée)
| Configuration           | Ret. cumulé | Ret./an | PF    | DD max | Sharpe | Calmar |
|-------------------------|-------------|---------|-------|--------|--------|--------|
| v2.9-DEF 60j (baseline) | 43.5%       | 5.60%   | 1.111 | 3.53%  | 1.601  | 1.588  |
| v2.9-OFF 60j            | 44.8%       | 5.78%   | 1.113 | 3.63%  | 1.624  | 1.595  |
| v3.0-DEF 60j uniforme   | 46.3%       | 5.50%   | 1.120 | 2.31%  | 1.725  | 2.383  |
| v3.0-OFF 60j uniforme   | 48.0%       | 5.70%   | 1.124 | 2.83%  | 1.754  | 2.010  |
| v3.0-DEF individualisé  | 54.2%       | 7.76%   | 1.140 | 3.07%  | 1.999  | 2.529  |

→ La version individualisée (seuils corrigés) domine sur tous les critères.

Résultats complets (backtest v3, 60j) :
| Config                  | Groupe         | Gold | Trades | PF    | Ret./an | DD max | Calmar |
|-------------------------|----------------|------|--------|-------|---------|--------|--------|
| v2.9-DEF_mom60j         | Propfirm 7     | Non  | 16 981 | 1.111 | 5.60%   | 3.53%  | 1.588  |
| v2.9-OFF_mom60j         | Propfirm 7     | Non  | 16 981 | 1.113 | 5.78%   | 3.63%  | 1.595  |
| v3.0-DEF_mom60j         | Propfirm 7     | Non  | 15 034 | 1.120 | 5.50%   | 2.31%  | 2.383  |
| v3.0-OFF_mom60j         | Propfirm 7     | Non  | 15 034 | 1.124 | 5.70%   | 2.83%  | 2.010  |
| v3.0-DEF_Gold_mom60j    | Compte propre 8| Oui  | 16 867 | 1.119 | 5.32%   | 2.87%  | 1.854  |
| v3.0-OFF_Gold_mom60j    | Compte propre 8| Oui  | 16 867 | 1.125 | 5.60%   | 3.42%  | 1.636  |

## Logique de signal (EA — multi-timeframe)
Tendance confirmée sur 4 niveaux simultanés :
- H1 : prix > SMA_fast > SMA_slow
- H4 : prix > SMA_H4 (50 périodes)
- D1 : prix > EMA_D1 (20 sur indices US/DAX, 50 sur autres)
→ BUY si tout aligné haussier, SELL si tout aligné baissier, sinon pas de trade

## Gestion de position (EA)
1. Ouverture : SL = 2.5×ATR H1 (2.0× pour EU50)
2. BE : SL déplacé au prix d'entrée + spread (PnL neutre garanti)
3. Trailing stop : actif après BE, déplacement min = TRAIL_MIN_MOVE×ATR (0.10 par défaut)

## Calcul des lots (Python → EA)
```
lot = (capital × RISK_PCT × weight × risk_mult × risk_factor) / (SL_pts × ATR)
```
- RISK_PCT = 0.5% de base
- weight = poids du portefeuille (volatilité inverse + momentum)
- risk_mult = anti-martingale (×2 après 5 TP, ÷2 après 5 SL consécutifs, max ×4 / min ×0.25)
- risk_factor = 1.0 / 0.5 / 0.0 selon niveau de drawdown
- Plafond lot : max 2% capital. Plafond SL : max 1% capital par trade (v2.9)
- RiskEUR : budget max en EUR par trade injecté dans CSV → EA bloque si coût réel > RiskEUR

## Pondération des poids (Python)
- **Mode offensif** (actif) : 30% inverse volatilité + 70% momentum absolu
- **Mode défensif** : 70% inverse volatilité + 30% momentum absolu
- Bornes min/max par actif issues des scores backtest (tolérance 30% en défensif, 60% en offensif)
- Momentum individualisé par actif : S&P500/DAX40=60j, NAS100/USDJPY/Silver/GBPJPY=90j, EU50=120j
- Filtre range : si |momentum| < seuil ajusté → EA mis en veille (TradeAllowed_Symbol=0)
- Formule correction seuil : `seuil_effectif = seuil_base_60j × (période/60)`

## Sécurités propfirm
| Règle                     | Propfirm                    | Compte propre |
|---------------------------|-----------------------------|---------------|
| Perte journalière max     | 4.5%                        | 8.0%          |
| Drawdown total max        | 9.0%                        | 20.0%         |
| Objectif profit Phase 1   | 10%                         | —             |
| Consistency rule          | Meilleur jour < 35% total   | —             |
| Gold                      | VEILLE automatique          | Actif         |
| Max SL/jour/actif         | 3 → cooldown 60 min         | idem          |
| Fermeture vendredi EU     | 14h30 UTC                   | idem          |
| Fermeture vendredi global | 20h15 UTC                   | idem          |
| Filtre news               | ±30 min high importance     | idem          |
| Filtre gap lundi          | gap > 2×ATR → pas de trade  | idem          |
| Filtre liquidité          | tick volume moy. H1 ≥ 10    | idem          |
| CSV fraîcheur             | max 300s (fallback lot déf.)| idem          |

## Points clés des backtests
- **Méthode composée** : lots recalculés sur capital courant à chaque trade (fidèle au robot réel). Écart méthode additive vs composée : +7 à +10 pts sur 7 ans.
- **Correction seuils de range** : `seuil_effectif = seuil_base_60j × (période/60)` — sans cette correction, les périodes courtes semblaient meilleures qu'elles ne l'étaient réellement. A modifié les conclusions pour 4 actifs sur 7 (USDJPY 30j→90j, EU50 30j→120j, NAS100 60j→90j, GBPJPY 120j→90j).
- **Paradoxe Gold** : excellent isolément (+7.7%/an) mais dégrade le portefeuille. Contribution effective ~9%×7.7%=0.7%/an seulement, et déplace du capital d'actifs plus efficaces. Score propfirm=NON en conséquence directe.
- **Backtest invalide Yahoo Finance** : paramètres H1 (SMA=30h) appliqués sur D1 (SMA=30j) → 15× moins de trades. Toujours vérifier la cohérence timeframe/paramètres.
- **Historique limitant** : USDJPY depuis 2019-02-28 (actif le plus récent sur RaiseFX).
- **Récupération historique MT5** : `copy_rates_from_pos(symbol, H1, 0, 50000)` — méthode validée sur RaiseFX.

## ⚠️ Problèmes rencontrés — NE PAS reproduire
1. **Blocage DD total (compte 5005661)** : pertes massives Silver/Gold en 2 jours, pas de mécanisme de reprise → reset INITIAL_BALANCE nécessaire. Cause : risque réel 7×-14× le prévu sur ces actifs.
2. **Silver/Gold risque disproportionné** : SL en points × valeur point mal évaluée → filtres RiskEUR (v2.8) et plafond MAX_RISK_PCT_PER_TRADE=1% (v2.9) ajoutés.
3. **Nikkei inutilisable en production** : 10 026 refus spread en 16 jours malgré bon score backtest. Toujours vérifier le spread réel broker avant d'inclure un actif.
4. **Sessions DAX40/EU50 incorrectes** : configurées jusqu'à 17h alors que marché ferme à 15h30 → corrigé 07h-15h UTC en v2.9.
5. **Trailing Silver 30+ modify/2h** : PositionModify envoyé à chaque tick même pour fraction de point → TRAIL_MIN_MOVE=0.10 ATR ajouté en v2.9.
6. **Alertes heartbeat Gold 4200/jour** : Gold en VEILLE n'émet pas de heartbeat → Python interprétait le silence comme une panne. Solution : lire le champ ModeActive avant d'alerter.
7. **Seuils range sans correction de période** : voir point backtests ci-dessus — a faussé les périodes optimales de 4 actifs.
8. **Boucle vendredi sur indices EU** : positions réouvertes après fermeture car flag non persistant → g_FridayCloseStarted ajouté en v2.9.

## Points de vigilance spécifiques
- **INITIAL_BALANCE hardcodée** : à modifier manuellement si changement ou rechargement significatif de compte
- **Fuseau horaire** : tout le système en UTC (`TimeGMT()` côté EA, `datetime.utcnow()` côté Python). Ne jamais utiliser l'heure locale.
- **Nommage fichiers** : le login est dans tous les noms CSV → mettre à jour CSV_FILENAME dans les paramètres EA si changement de compte
- **Silver tick_value** : comme pour le projet Telegram, toujours vérifier les spécifications broker réelles avant tout nouveau déploiement

## Checklist déploiement nouveau compte
1. Adapter LOGIN, PASSWORD, MT5_PATH, INITIAL_BALANCE dans Ponderation.py
2. Vérifier MOMENTUM_PERIOD_BY_SYMBOL et MOMENTUM_RANGE_THRESHOLD
3. Créer `counters_{AccountID}.json` avec `daily_start_balance` = capital du compte
4. Lancer Ponderation.py → vérifier AccountID et capital initial dans les logs
5. Attacher EA_FX_Universal sur chaque graphique H1 → vérifier Gold en VEILLE si propfirm
6. Attendre 2 cycles → vérifier DD jour=0.00% et DD total=0.00%

## Backlog v3.1 (prévu)
- Pilotage auto DEFENSIF/OFFENSIF par momentum portfolio global : Σ(|mom_i| × poids_i)
- Veille combinée : |momentum| < seuil ET sl_streak ≥ 2
- Mécanisme de récupération DD (DD_RECOVERY_THRESHOLD)
- Données H1 période longue — exploration Dukascopy
- Validation out-of-sample : 2019-2022 vs 2023-2026
- EMA vs SMA : non testé — à reconsidérer après rebacktest

## Historique des versions
**Ponderation.py :**
- v3.1 : INITIAL_BALANCE hardcodée, écriture InitBalance CSV
- v3.2 : RiskEUR — budget max en EUR par trade, colonne injectée dans CSV
- v3.3 : Nikkei exclu (spread 14×), Gold VEILLE auto propfirm, portefeuille 7 actifs
- v3.0 : momentum individualisé (MOMENTUM_PERIOD_BY_SYMBOL), pondération hybride, filtre range, MODE_DEFENSIF=False
- v3.0 rev. : périodes recalibrées (backtest v3, seuils corrigés) — NAS100=90j, USDJPY=90j, EU50=120j, GBPJPY=90j

**EA_FX_Universal :**
- v2.5 : BE, fermeture vendredi, filtre magic, plafond lot
- v2.6 : solidité signal, gap lundi, ATR breaker, liquidité, cooldown échec ordre
- v2.7 : MAGIC_NUMBER en premier paramètre, lecture InitBalance depuis Python
- v2.8 : RiskEUR depuis CSV, filtre coût réel, correction BE + spread (PnL neutre garanti)
- v2.8.1 : Nikkei exclu, Gold VEILLE auto propfirm, seuils historique assouplis
- v2.9 : sessions DAX40/EU50 07h-15h UTC, g_FridayCloseStarted, plafond SL 1% capital, TRAIL_MIN_MOVE=0.10 ATR, EMA_D1 50→20 sur S&P500/NAS100/DAX40
