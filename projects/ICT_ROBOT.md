# PROJET : ICT_ROBOT
_Période: 2026 → en cours | Statut: construction_
_Créé : 2026-06-16 | Dernière mise à jour : 2026-06-16_

---

## Objectif
Robot de trading ICT 100% autonome, générique par actif, déployable sur MT5.
Symétrique Bullish/Bearish. Backtestable via MT5 Strategy Tester et Python.
Concepts clés à implémenter : Order Blocks (avec FVG, BOS, displacement), gestion multi-OB, filtrage par session, analyse multi-timeframe (HTF→LTF), Fibonacci. Claude positionné comme expert ICT + Python/MT5 sur ce projet.

## Statut : 🟡 En cours — phase de conception (spec v1.1 figée)

## Stack
- **Exécution live** : Expert Advisor MQL5 sur MT5 (RaiseFX)
- **Backtest Python** : CSV ticks Tick Data Suite → reconstruction OHLCV multi-TF
- **Backtest complémentaire** : export données RaiseFX via bibliothèque MetaTrader5
- **Marchés cibles** : Forex, Indices (DAX, SP500), Or (XAUUSD), Crypto

---

## Spécification v1.1

### MODULE 1 — Biais Directionnel (HTF)
**Timeframes** : D1 (macro) + H4 (opérationnel)

**Logique combinée HH/HL + MSS :**
- Structure des swings : Bullish si HH+HL / Bearish si LH+LL sur H4
- MSS H4 prime en cas de conflit :
  - MSS Bullish : cassure + clôture au-dessus du dernier LH significatif
  - MSS Bearish : cassure + clôture en-dessous du dernier HL significatif
- Alignement D1 obligatoire — si conflit D1 vs H4 : NEUTRAL, pas de trade

**Output** : `BIAS = BULLISH | BEARISH | NEUTRAL`

---

### MODULE 2 — Identification des PD Arrays (MTF)
**Timeframes** : H1 (identification) + H4 (validation zone)

**Définition stricte ICT du OB valide :**

Bullish OB :
- Dernière bougie bearish avant un displacement bullish
- Displacement = mouvement impulsif cassant un swing H1 récent
- FVG obligatoire dans le corps du displacement
- OB non mitigé (clôture sous le Low = invalidation)
- Zone : [Low, High] de la bougie bearish

Bearish OB :
- Dernière bougie bullish avant un displacement bearish
- Displacement = mouvement impulsif cassant un swing H1 récent
- FVG obligatoire dans le corps du displacement
- OB non mitigé (clôture au-dessus du High = invalidation)
- Zone : [Low, High] de la bougie bullish

**Score de confluence (min requis : 2) :**
| Confluence | Bonus |
|---|---|
| OB + FVG alignés | +2 |
| Zone alignée avec niveau D1 | +1 |
| Volume displacement > SMA20 | +1 |
| Session Londres active | +1 |
| Session New York active | +2 |

**Output** : liste des OBs actifs triés par score + proximité prix

---

### MODULE 3 — Entrée de Précision (LTF)
**Timeframes** : M15 (MSS strict) + M5 (optionnel)

**Conditions d'entrée — toutes requises :**
1. BIAS HTF = direction du trade
2. Prix dans zone OB MTF ciblé (score ≥ 2)
3. MSS M15 strict : cassure + clôture au-dessus/dessous du dernier swing M15
4. Session active (London ou NY)
5. Marché ouvert (SYMBOL_TRADE_MODE vérifié)
6. Aucune news High Impact dans fenêtre ±15 min (calendrier MT5 intégré)

**Prix d'entrée** : ordre limit au 50% de l'OB (midpoint)
**Expiration** : annulation si pas de retest dans les 3 bougies M15 suivant le MSS

---

### MODULE 4 — Risk Management
**Stop Loss :**
- Long  : 1 pip sous le Low de l'OB ciblé
- Short : 1 pip au-dessus du High de l'OB ciblé
- SL fixe jusqu'à TP1 atteint

**Taille de position :**
- Risque = 1% du capital (solde équité) — conforme règle globale
- Lot = (Capital × 0.01) / (SL en pips × valeur du pip)
- Formule à valider explicitement avant implémentation

**TP1 — Partiel 50% :**
- Prochain liquidity pool en direction du trade (EQH/EQL ou swing HTF)
- À l'atteinte : fermeture 50% de la position + SL déplacé au breakeven

**Trailing dynamique (50% restant) :**
- Activation après TP1
- Suivi sur dernier OB LTF M15 reformé en direction du trade
- Fallback : trailing ATR(14) × 1.5 si aucun OB LTF disponible
- Objectif : capturer les grandes extensions sans TP2 fixe

---

### MODULE 5 — Filtres de Qualité

**Sessions (horaires UTC, configurables) :**
- London Kill Zone : 07:00 – 10:00
- New York Kill Zone : 13:00 – 16:00
- Hors session : aucune nouvelle entrée (trades ouverts gérés normalement)

**Filtre marché ouvert :**
- Vérification `SYMBOL_TRADE_MODE` avant toute entrée
- Bloque toute tentative d'exécution si marché fermé (férié, week-end, hors horaires)

**Calendrier économique :**
- Source : calendrier intégré MT5 (pas d'API externe)
- Filtre : événements High Impact uniquement
- Fenêtre : ±15 min autour de la news
- Comportement : entrées bloquées / trades ouverts conservés (SL protège)

**Clôture vendredi :**
- Fermeture immédiate de TOUTES les positions à `Friday_Close_Hour` (défaut : 21:00 UTC)
- Sans condition, quelle que soit la situation du trade

---

### MODULE 6 — Backtesting

**Python (source primaire) :**
- Données ticks : fichiers CSV Tick Data Suite
- Reconstruction OHLCV multi-TF depuis les ticks (M5, M15, H1, H4, D1)
- Source complémentaire : export RaiseFX via bibliothèque MetaTrader5
- Métriques : Net Profit, Max Drawdown, Profit Factor, Sharpe, Win Rate, Avg RR
- Visualisation trades : matplotlib / plotly

**MT5 Strategy Tester :**
- Expert Advisor MQL5 natif
- 1 instance = 1 actif (robot générique)
- Paramètres exposés via `input` pour optimisation

---

### Paramètres configurables (inputs)

| Paramètre | Défaut | Description |
|---|---|---|
| `HTF_Bias_TF` | H4 | Timeframe biais principal |
| `OB_TF` | H1 | Timeframe détection OBs |
| `Entry_TF` | M15 | Timeframe MSS entrée |
| `Risk_Percent` | 1.0 | Risque par trade (%) |
| `Min_OB_Score` | 2 | Score minimum OB valide |
| `SL_Buffer_Pips` | 1 | Buffer sous/sur l'OB |
| `London_Start` | 07:00 | Début session Londres (UTC) |
| `London_End` | 10:00 | Fin session Londres (UTC) |
| `NY_Start` | 13:00 | Début session New York (UTC) |
| `NY_End` | 16:00 | Fin session New York (UTC) |
| `Friday_Close_Hour` | 21:00 | Heure clôture forcée vendredi (UTC) |
| `Close_All_Friday` | true | Activer clôture vendredi soir |
| `Check_Market_Open` | true | Bloquer si marché fermé |
| `News_Filter` | true | Filtrage calendrier éco MT5 |
| `News_Window_Min` | 15 | Fenêtre news (minutes) |
| `Trail_ATR_Period` | 14 | Période ATR trailing fallback |
| `Trail_ATR_Multi` | 1.5 | Multiplicateur ATR trailing |

---

## Plan de développement

| Phase | Contenu | Statut |
|---|---|---|
| Phase 1 | Fondations Python : récupération données, détection swings, OBs + FVG | ⬜ À faire |
| Phase 2 | Logique trading Python : biais HTF, signaux, backtesting + métriques | ⬜ À faire |
| Phase 3 | Portage MQL5 : EA MT5, validation croisée Python ↔ MQL5, optimisation | ⬜ À faire |
| Phase 4 | Intégrations : calendrier MT5, gestion sessions, clôture vendredi | ⬜ À faire |

## Décisions actées
- MSS LTF : strict (cassure + clôture) — pas d'approche souple
- Clôture vendredi : immédiate, sans condition
- Calendrier éco : intégré MT5 (pas d'API externe)
- Mode : 100% autonome
- TP2 : pas de niveau fixe — trailing dynamique uniquement
- Score sessions : London +1 / New York +2
