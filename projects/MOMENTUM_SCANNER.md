
# PROJET : MOMENTUM_SCANNER
_Période: 2026 → en cours | Statut: construction_
_Dernière mise à jour : 2026-06-06_

---

## Objectif
Scanner multi-actifs (411 symboles RaiseFX) pour scorer chaque actif sur 3 critères
(momentum, tendance, structure de marché) + filtres complémentaires (spread, corrélation),
puis exécuter automatiquement des ordres en suivi de tendance via Python pur + API MT5,
sans aucun EA MQL5.

---

## Fichiers produits

| Fichier | Statut | Rôle |
|---|---|---|
| `list_symbols_fast2.py` | ✅ Fonctionnel | Liste ultra-rapide des 411 symboles RaiseFX (métadonnées seules, sans copy_rates) |
| `symbols_raisefx.csv` | ✅ Produit | 411 symboles avec catégorie, sous-catégorie, description, spread_min, digits, visible |
| `extract_eurusd_d1.py` | ✅ Fonctionnel | Extraction historique Daily EURUSD → CSV variations J/J-1 |
| `eurusd_d1_variations.csv` | ✅ Produit | Variations close/close, var_pct_J, var_pct_Jm1, direction |
| `momentum_analyzer.py` | ✅ Fonctionnel | Analyse momentum conditionnel EURUSD (intervalles fixes + quantiles) |
| `momentum_results.csv` | ✅ Produit | Résultats analyse (50 lignes, train+test, fixed+quantile) |
| `get_symbol_specs.py` | ⚠️ v2 en cours | Specs MT5 par symbole : lot_min, tick_value, point_value, ATR H1, risque réel |
| `symbol_specs.csv` | ❌ Vide (bug v1) | Cible : specs complètes pour calcul lot sizing |
| `asset_scanner.py` | 🔄 Codé, non testé | Scanner D1 multi-actifs : scores momentum/tendance/structure + rapport HTML |
| `robot.py` | 📋 Architecture définie | Point d'entrée boucle principale (à coder) |
| `config.py` | 📋 À coder | Paramètres globaux |
| `scanner.py` | 📋 À coder | Calcul scores D1 (intégré dans asset_scanner.py pour l'instant) |
| `signal_h1.py` | 📋 À coder | Confirmation signal H1 |
| `risk_manager.py` | 📋 À coder | Sizing, anti-martingale, circuit breaker, horaires clôture |
| `trade_executor.py` | 📋 À coder | Passage ordres MT5, gestion SL/TP |
| `logger.py` | 📋 À coder | Logs + alertes Telegram |

---

## Architecture technique

```
ROBOT CENTRAL (robot.py) — boucle toutes les minutes
│
├── 23h00 : Scan D1 (après clôture US)
│   └── scanner.py → calcule scores sur tous actifs éligibles
│       → sélectionne TOP N actifs (N à définir)
│       → filtre corrélation (r > 0.80 → garder le mieux scoré)
│       → filtre spread (ratio spread/ATR_H1 > 20% → exclure)
│
├── H+01min : Scan H1 (chaque heure)
│   └── signal_h1.py → confirme signal sur TOP N
│       → ADX H1 > 20 ?
│       → Prix > EMA20 H1 (long) ou < EMA20 H1 (short) ?
│       → Pas de news majeure (filtre calendrier — à définir)
│
├── En continu : gestion positions ouvertes
│   └── trade_executor.py → SL/TP trailing, clôture horaires
│
└── risk_manager.py → transversal, appelé à chaque décision
│
API Python MetaTrader5
│
MT5 Terminal RaiseFX (arrière-plan, aucun graphique requis)
```

**Broker :** RaiseFX (Raise Global MT5 Terminal Demo04)
**Login démo :** 5007258 | **Serveur :** RaiseGlobal-Live
**Capital de référence :** 50 000 EUR

---

## Univers d'actifs

**Inclus :**
- Indices CFD : NAS100, S&P500, DAX40, DJ30, EU50, FTSE100, Nikkei, ASX200
- Matières premières : Gold, Silver, oilWTI, BRENT, Copper
- Forex : GBPJPY, USDJPY, GBPUSD, AUDUSD, USDCAD, EURUSD (référence)
- Actions USA (NYSE/NASDAQ CFD) — clôture avant 21h45
- Actions Europe (Euronext/Xetra CFD) — clôture avant 17h15 été / 16h15 hiver ⚠️
- Crypto : BTCUSD, ETHUSD uniquement

**Exclus (et pourquoi) :**
- Shares Asia : horaires de nuit, complexité 3 fuseaux horaires
- Crypto altcoins (103 paires) : manipulation, liquidité insuffisante
- Indices illiquides (HKG33, China50, IBEX35) : spreads larges, horaires décalés
- Paires forex exotiques (TRY, ZAR, HUF...) : spreads prohibitifs en H1
- ETFs : redondance avec indices

**Idée abandonnée :** analyse momentum conditionnel sur EURUSD Daily (var J-1 → proba J).
Résultats : signal unique à 56.5% p_hausse (intervalle [-0.3%,-0.2%], pval=0.011),
mean-reversion légère sur grands mouvements. EURUSD trop efficient, approche insuffisante seule.

---

## Logique de scoring (D1)

### Score global
```
Score_global = 0.40 × Score_Momentum
             + 0.35 × Score_Tendance
             + 0.25 × Score_Structure
```

### Score Momentum [0, 1]
- ROC(10), ROC(20), ROC(60) : `(close[-1] - close[-N]) / close[-N] × 100`
- Score = sigmoid(mean_ROC × 5)
- Bonus ×1.15 si tous les ROC pointent dans le même sens (plafonné à 1.0)
- Direction = LONG si mean_ROC > 0, SHORT sinon

### Score Tendance [0, 1]
- ADX(14) normalisé : `min(ADX / 60, 1.0)` — seuil utile à 20, fort à 40+
- Pente régression linéaire(60j) normalisée : `min(|pente| / 0.5, 1.0)` (0.5%/j = solide)
- R² de la régression : mesure la linéarité
- `Score_Tend = 0.40 × ADX_score + 0.35 × Pente_score + 0.25 × R²`

### Score Structure [0, 1]
- Efficiency Ratio Kaufman(20) : `|close[-1]-close[-N]| / Σ|variations|` → 1=tendance, 0=bruit
- ATR relatif(14) : `ATR / prix_moyen × 100` — zone idéale 0.3% à 3%
- `Score_Struct = 0.60 × ER + 0.40 × ATR_score`

### Filtres complémentaires
- **Spread/ATR H1** : `spread_pts / ATR_H1 × 100`
  - < 5% : excellent | 5-10% : acceptable | > 20% : éliminatoire
- **Corrélation** : si r > 0.80 sur 60j entre deux actifs → garder le mieux scoré
- **Score ajusté** : `Score_final = Score_global × (1 - ratio_spread)` (à implémenter)
- **Lot minimum faisable** : si `risk_lot_min > risk_cible` → actif bloqué pour cet actif uniquement

### Seuils signal
- Score ≥ 0.65 : signal fort
- Score 0.55-0.65 : signal moyen
- Score < 0.55 : pas de trade
- **Règle :** seuils figés a priori, sans optimisation sur historique (anti-overfitting)

---

## Confirmation signal H1 (signal_h1.py — à coder)

Pour chaque actif du TOP N sélectionné par le scanner D1 :
1. ADX H1 > 20
2. Prix > EMA20 H1 (LONG) ou Prix < EMA20 H1 (SHORT)
3. Pas en zone de résistance/support majeure (à définir)
4. Filtre calendrier économique : exclure les jours de news majeures (API à définir)

---

## Calcul de la taille des lots

**Formule centrale (raisonnement en valeur risquée, pas en % pur) :**
```
Risk_cible_EUR  = Capital × RISK_PCT × RiskMult
                = 50 000 × 0.5% × 1.0 = 250€ de base

SL_pts          = ATR_H1 × SL_MULT  (SL_MULT = 2.5)

Point_value     = tick_value / tick_size  (par lot, par point)

Lot_optimal_raw = Risk_cible_EUR / (SL_pts × Point_value)
Lot_optimal     = max(lot_min, arrondi_inférieur(Lot_optimal_raw, lot_step))

Risk_réel       = Lot_optimal × SL_pts × Point_value
                → doit être ≈ Risk_cible (écart acceptable : ±15%)
```

**Filtre faisabilité :**
```python
risk_lot_min = lot_min × SL_pts × Point_value
if risk_lot_min > Risk_cible → trade BLOQUÉ pour cet actif
   (pas bloqué pour les autres)
```

**Specs MT5 nécessaires par actif :** `volume_min`, `volume_max`, `volume_step`,
`trade_tick_value`, `trade_tick_size`, `trade_contract_size`, `digits`

---

## Gestion du risque (risk_manager.py — à coder)

### Anti-martingale (inspiré de Ponderation.py — validé en production)
```
SL_SERIES_LVL1 = 5  → RiskMult × 0.5  (après 5 SL consécutifs)
SL_SERIES_LVL2 = 10 → RiskMult × 0.25 (après 10 SL consécutifs)
TP_SERIES_LVL1 = 5  → RiskMult × 2.0  (après 5 TP consécutifs)
TP_SERIES_LVL2 = 10 → RiskMult × 4.0
RISK_MULT_MIN   = 0.25  (plancher absolu)
RISK_MULT_MAX   = 4.0   (plafond absolu)
```
- Compteurs **par actif** (un SL sur NAS100 n'affecte pas Gold)
- Quand RiskMult réduit → lot_min peut dépasser risk_cible → trade ignoré pour cet actif (pas bloqué globalement)
- **Pas de circuit breaker manuel** : le robot est autonome, il réduit le risque automatiquement

### Règles globales
- Max 1% du capital par trade (hors RiskMult)
- Max 5% de risque simultané total
- Levier effectif max : 2:1 (indépendamment du levier broker)
- SL/TP systématiques sur chaque position (protection si robot crash)

### Horaires de clôture (heure Paris — détection auto été/hiver via pytz)

| Type d'actif | Clôture robot été | Clôture robot hiver |
|---|---|---|
| Indices EU (DAX40, EU50...) | 21h45 | 21h45 |
| Indices US (NAS100, S&P500, DJ30) | 22h45 | 22h45 |
| **Actions Europe ← PIÈGE** | **17h15** | **16h15** |
| Actions US | 21h45 | 21h45 |
| Forex / Matières premières | 22h45 | 22h45 |
| **Vendredi tous actifs** | **21h00** | **21h00** |

### Résilience opérationnelle
- Reconnexion MT5 automatique si déconnexion
- Watchdog : vérification connexion toutes les 5 minutes
- Alerte Telegram si anomalie (à coder)
- Logs complets de toutes les décisions
- VPS Windows recommandé pour éviter extinction PC
- Jours fériés par marché : table à coder

---

## Ce qui fonctionne

- Connexion MT5 RaiseFX via Python API ✅
- Listing 411 symboles en < 5 secondes (`list_symbols_fast2.py`) ✅
- Extraction historique Daily avec détection automatique du max disponible ✅
- Calcul variations J/J-1 et export CSV (`extract_eurusd_d1.py`) ✅
- Analyse momentum conditionnel avec split train/test, test binomial, rapport HTML ✅
- Architecture complète du scanner codée (`asset_scanner.py`) ✅
- Logique de scoring (3 critères + formules) définie et codée ✅
- Table horaires de clôture définie avec gestion été/hiver ✅

---

## Ce qui ne fonctionne pas / points en suspens

- `get_symbol_specs.py` v1 : CSV vide car crash silencieux (division par zéro sur atr_h1=0) → v2 codée avec écriture au fil de l'eau, non encore testée
- `asset_scanner.py` : codé mais jamais exécuté, résultats non validés
- Filtre spread intégré dans le scanner : codé dans `get_symbol_specs.py` mais pas encore branché dans `asset_scanner.py`
- Filtre corrélation : défini conceptuellement, non codé
- `signal_h1.py` : non codé
- `risk_manager.py` : non codé
- `trade_executor.py` : non codé
- `robot.py` (boucle principale) : non codé
- Filtre calendrier économique (news) : API à choisir, non codé
- Table jours fériés par marché : non codée
- Alertes Telegram : non codées
- Spread moyen H1 (vs spread instantané) : identifié comme plus fiable, non implémenté

---

## ⚠️ Problèmes rencontrés — NE PAS reproduire

1. **`copy_rates_from_pos` avec MAX_BARS = 200 000 sur D1** → erreur `Invalid params`
   MT5 refuse. Pour D1, demander 500 à 6000 bougies max. Utiliser la détection par paliers
   (500, 1000, 1500...) ou directement 3000 (MT5 retourne ce qui est dispo sans crash).

2. **`symbol_select(sym, True)` dans une boucle de 411 symboles** → force le téléchargement
   depuis le serveur broker → ~24s/symbole → 411 symboles = ~2h45. Solution : utiliser
   `symbols_get()` pour les métadonnées (< 5s pour 411), et `symbol_select` uniquement
   pour les actifs qu'on va effectivement trader.

3. **Écriture CSV uniquement en fin de script** → si crash, fichier vide.
   Toujours écrire ligne par ligne avec `f.flush()` immédiat.

4. **Division par zéro sur `atr_h1 = 0`** → crash silencieux.
   Toujours protéger : `ratio = spread/atr if atr > 0 else 999.0`.

5. **Heure de clôture actions Europe** : fermeture à 16h30 Paris en heure d'hiver
   (vs 17h30 en été). Piège vécu sur Ponderation.py. Détection automatique obligatoire via `pytz`.

6. **Raisonnement en % de risque pur** (sans vérifier la valeur en EUR par lot minimum) →
   certains actifs (Silver, petites actions) ont un lot minimum dont le coût SL dépasse
   le budget de risque. Toujours calculer `risk_lot_min = lot_min × SL_pts × point_value`
   et comparer à `Risk_cible_EUR` avant d'envoyer un ordre.

7. **Bougie D1 du jour en cours** : si le scanner tourne avant 23h, la dernière bougie D1
   des actifs US est incomplète. Forcer le scanner après 23h ou ignorer la bougie en cours.

---

## Points de vigilance spécifiques

- **Look-ahead bias** : seuils de scoring figés a priori, jamais optimisés sur l'historique
- **Survivorship bias** : les 411 symboles sont les actifs survivants — historique biaisé vers les gagnants
- **Corrélation NAS100/S&P500** (r~0.95) : deux trades = deux fois le même risque → filtre obligatoire
- **Spread non statique** : élargissement en ouverture de marché et le vendredi après-midi
  → utiliser spread moyen H1 plutôt que spread instantané
- **ADX retardé** : reflète les 2-3 dernières semaines → croiser avec ADX H1 pour confirmation
- **Slippage actions illiquides** : ordres market en fin de session → préférer ordres limit ±0.05%
- **Changement de régime** : score calculé sur ~2 ans mélange des régimes différents
  → envisager un indicateur de régime (VIX ou ATR normalisé 252j) pour adapter les seuils
- **ROC(60) peut masquer un retournement récent** → pondérer ROC(10) > ROC(60)

---

## Statut actuel

**Phase : Construction — aucun test lancé**

Scripts produits et fonctionnels : extraction données, analyse statistique exploratoire, listing symboles.
Scanner codé mais non exécuté. Specs MT5 (lot sizing) : script v2 prêt, non encore testé.
Modules robot (signal H1, risk manager, executor, boucle principale) : à coder.

---

## Prochaines étapes (dans l'ordre)

1. **Tester `get_symbol_specs.py` v2** → valider `symbol_specs.csv` complet
2. **Intégrer filtre spread dans `asset_scanner.py`** depuis `symbol_specs.csv`
3. **Ajouter filtre corrélation** dans le scanner (r > 0.80 → garder le mieux scoré)
4. **Exécuter `asset_scanner.py`** → analyser le classement, calibrer les seuils
5. **Coder `signal_h1.py`** → ADX H1 + EMA20 H1
6. **Coder `risk_manager.py`** → anti-martingale Ponderation.py + table horaires clôture
7. **Coder `trade_executor.py`** → ordres MT5 avec SL/TP systématiques
8. **Assembler `robot.py`** → boucle principale avec scheduling
9. **Paper trading 4-6 semaines** → log de toutes les décisions sans ordres réels
10. **Démo live 4-6 semaines** → ordres réels sur compte démo, sizing 0.5%
11. **Validation** : Sharpe > 1.0, zéro incident clôture manquée → passage live progressif

---

## Références

- Broker : RaiseFX | Login : 5007258 | Serveur : RaiseGlobal-Live
- MT5 path : `C:\Program Files\Raise Global MT5 Terminal Demo04\terminal64.exe`
- Inspiré de : `Ponderation.py` (anti-martingale LVL1/LVL2 validé en production)
- Littérature : Jegadeesh & Titman (1993) momentum actions | Perry Kaufman Efficiency Ratio
