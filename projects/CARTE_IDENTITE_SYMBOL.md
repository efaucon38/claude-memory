# CARTE D'IDENTITÉ DE SYMBOLE — Méthodologie de caractérisation d'actif

Last updated: 2026-07-02
Statut : Architecture validée — développement à démarrer
Lié à : CORE.md (projet actif), backlog "Carte d'identité d'un actif" (promu)

💡 NOTE POUR CLAUDE CODE : Ce document trace la démarche complète, y compris le raisonnement qui a mené aux décisions (pas seulement les décisions finales), car il servira de base à une note de synthèse pour un groupe de trading algorithmique. Avant de coder quoi que ce soit, valider avec l'utilisateur — voir règles d'autonomie dans CORE.md.

---

## 1. Objectif du projet

Avant de concevoir un EA MT5 sur un symbole donné, produire une **caractérisation statistique rigoureuse** de ce symbole à partir de données tick réelles sur longue période (10+ ans), afin de :

1. Déterminer objectivement la **famille de stratégie** la plus adaptée (trend-following, mean-reversion, ou architecture mixte/adaptative) — plutôt que de partir d'hypothèses générales ou de choix par défaut
2. **Calibrer** les indicateurs et paramètres de l'EA (fenêtres, seuils, filtres horaires) sur les caractéristiques réelles mesurées du symbole, plutôt que sur des valeurs par défaut génériques (14, 20...)
3. Permettre la **comparaison objective** entre symboles (est-ce qu'AUDCAD se comporte différemment d'USDJPY ?) et entre périodes pour un même symbole (détection de dérive de régime)

**Principe fondateur** : la démarche est **générique et réutilisable sur n'importe quel actif** (paire forex, action, indice, métal...). Aucune logique spécifique à un symbole donné ne doit être codée en dur dans le pipeline — le symbole est un paramètre d'entrée, jamais une hypothèse de conception.

**Premier cas d'application** : AUDCAD (fichier tick en cours de chargement au 2026-07-02). USDJPY servira de test de charge intermédiaire pendant l'attente, car déjà disponible et plus volumineux (utile pour valider la robustesse du pipeline sur gros volumes avant le run officiel).

---

## 2. Fondements académiques

La démarche s'appuie sur un corpus de recherche identifié et vérifié en plusieurs passes (une première synthèse incomplète a été corrigée après relecture critique — voir §2.5). Quatre blocs complémentaires :

### 2.1 Stylized facts généraux
Référence : **Cont, R. (2001), "Empirical properties of asset returns: stylized facts and statistical issues", Quantitative Finance**. Cadre de référence universel sur les propriétés statistiques des rendements d'actifs financiers : distribution à queues épaisses (fat tails), absence d'autocorrélation linéaire significative à moyen terme, volatility clustering, effet de levier (moins pertinent pour le forex).

### 2.2 Stylized facts spécifiques au forex intraday
Référence : **Guillaume, D.M., Dacorogna, M.M., Davé, R.R., Müller, U.A., Olsen, R.B., Pictet, O.V. (1997), "From the bird's eye to the microscope: A survey of new stylized facts of the intra-daily foreign exchange markets", Finance and Stochastics, 1, 95-129**.
Point méthodologique important issu de cet article : les patterns saisonniers intrajournaliers introduisent un biais fort dans le calcul de statistiques simples sur données intraday — nécessité de désaisonnaliser avant toute comparaison entre heures/jours (dummies saisonnières, Fourier flexible, ou changement d'échelle de temps). Egalement source de l'autocorrélation négative de premier ordre documentée à très haute fréquence (Goodhart 1989), qui disparaît une fois le processus de formation du prix terminé.

### 2.3 Lois d'échelle et temps intrinsèque (Directional Change)
Références :
- **Müller, U.A., Dacorogna, M.M., Olsen, R.B., Pictet, O.V., Schwarz, M., Morgenegg, C. (1990), "Statistical study of foreign exchange rates, empirical evidence of a price change scaling law, and intraday analysis", Journal of Banking & Finance, 14(6), 1189-1208**. Article fondateur : le changement absolu moyen des log-prix suit une loi d'échelle empirique par rapport à l'intervalle de temps de mesure, vérifiée même si les distributions diffèrent selon la taille de l'intervalle.
- **Glattfelder, J.B., Dupuis, A., Olsen, R.B. (2011)** et travaux ultérieurs (Golub, Glattfelder, Olsen 2016-2022 ; Tsang et al. 2010-2024) sur le cadre **Directional Change (DC) / temps intrinsèque**.

Principe du DC : au lieu d'échantillonner à intervalles de temps fixes, on décompose le prix en événements de retournement directionnel déclenchés par un seuil δ (en %) par rapport au dernier extremum, chaque mouvement se décomposant en un événement DC et un overshoot qui suit. Deux lois d'échelle empiriques répliquées sur de nombreuses paires forex : le nombre d'événements DC suit une loi de puissance par rapport au seuil (N(δ) ∝ δ⁻ᵇ), et la longueur moyenne de l'overshoot est approximativement proportionnelle au seuil. Ce cadre permet de décomposer la dynamique d'un marché en liquidité (overshoots) et volatilité (fréquence des DC), et s'applique nativement aux ticks sans échantillonnage arbitraire.

Intérêt direct pour l'EA : contrairement au Hurst/VRT (voir §2.4), l'algorithme DC est **intrinsèquement causal** — à chaque tick, comparaison au dernier extremum connu, détection dès franchissement du seuil. Aucune fenêtre glissante lourde, coût de calcul quasi nul. Recalculable nativement en MQL5, contrairement au ZigZag natif MT5 qui fonctionne sur un nombre de barres (depth/deviation/backstep) et non sur un seuil de variation en %, calculé en temps intrinsèque — ce n'est pas un proxy fidèle.

### 2.4 Outils de classification quantitative trend vs mean-reversion
- **Exposant de Hurst (H)** : mesure la mémoire longue d'une série via la variance du log-prix en fonction du décalage temporel. H < 0.5 → mean-reversion ; H = 0.5 → marche aléatoire ; H > 0.5 → tendanciel.
- **Variance Ratio Test (VRT)** : test complémentaire vérifiant si la variance croît linéairement avec le temps (marche aléatoire) ou non (mean-reversion). Bonne pratique : combiner plusieurs tests (ADF, Phillips-Perron, KPSS, VRT, Hurst) pour validation croisée.
- Application pratique documentée : trader en momentum sur les actifs globalement tendanciels (H > 0.5) et en mean-reversion sur les actifs globalement mean-reverting (H < 0.5), en utilisant le Hurst comme classificateur.

### 2.5 Détection de régime multi-actifs (pour la comparaison inter-symboles)
Approches de clustering non-supervisé (k-means, hiérarchique, DBSCAN) appliquées à des métriques de régime, avec des études ayant appliqué cette logique à des paniers multi-actifs incluant des paires de devises — transposable pour comparer AUDCAD à d'autres paires ou classes d'actifs.

### 2.6 Note méthodologique sur la recherche elle-même
Une première synthèse académique (première passe de recherche) a omis le bloc 2.3 (lois d'échelle / Directional Change), qui s'est avéré être l'apport le plus directement actionnable pour la démarche. Une relecture critique explicite a permis de le retrouver. **Enseignement à conserver** : sur ce type de projet de recherche méthodologique, prévoir systématiquement une passe de vérification/complétude avant de considérer la base bibliographique comme close.

---

## 3. Pipeline de caractérisation — 5 étapes

```
Ticks bruts (10 ans)
      ↓
Nettoyage et validation
      ↓
Rééchantillonnage multi-échelle (temps physique + temps intrinsèque)
      ↓
Calcul des métriques (4 blocs)
      ↓
Carte d'identité du symbole (rapport + comparaison inter-actifs)
```

### 3.1 Ticks bruts
Source : Tick Data Suite (Dukascopy), format GMT+0 NO-DST — voir CORE.md pour le format exact des fichiers.

### 3.2 Nettoyage et validation
Suppression des doublons, filtrage des ticks aberrants (spikes impossibles), **conservation du spread bid/ask tick par tick** (perdu si on ne travaille qu'avec des barres OHLC — nécessaire pour caractériser la liquidité réelle par heure/session).

### 3.3 Rééchantillonnage multi-échelle
Deux logiques en parallèle, calculées séparément sur les mêmes ticks nettoyés :
- **Temps physique** : barres M30/H1 classiques (celles que l'EA utilisera pour trader)
- **Temps intrinsèque** : événements Directional Change à plusieurs seuils δ (ex : 0.1%, 0.2%, 0.5% — à affiner empiriquement), calculés directement sur les ticks

Objectif de la comparaison : juger si M30/H1 sont des échelles pertinentes pour capter la dynamique réelle du symbole, plutôt que de les considérer comme acquises.

### 3.4 Calcul des métriques — 4 blocs

| Bloc | Contenu | Sorties | Lecture |
|---|---|---|---|
| **A. Distribution** | Stylized facts généraux (§2.1) | Skewness, kurtosis, VaR empirique 95/99%, test de normalité (Jarque-Bera), quantiles des rendements | Une kurtosis élevée → mouvements extrêmes plus fréquents que la loi normale → influence le dimensionnement SL/risque par trade |
| **B. Directionnalité (régime)** | Hurst/VRT (§2.4) + scaling DC (§2.3) | Hurst exponent + VRT en fenêtre glissante dans le temps (pas une valeur unique sur 10 ans), nombre d'événements DC par seuil, longueur moyenne des overshoots | H > 0.5 stable → biais trend-following justifié ; H < 0.5 → biais mean-reversion ; H qui oscille beaucoup → régime instable → architecture EA avec filtre de régime adaptatif nécessaire (voir §5) |
| **C. Volatilité et clustering** | Stylized facts (§2.1) | ATR normalisé (ATR/prix) par période, autocorrélation de la volatilité (ACF des rendements²), clusters de régimes de volatilité | Persistance de volatilité mesurée → utile pour filtre d'entrée en trade et pour calibrer la période des indicateurs de volatilité (ATR, StdDev) |
| **D. Saisonnalité (désaisonnalisée)** | Guillaume et al. 1997 (§2.2) | ATR moyen et spread moyen par heure UTC, par jour de semaine, avec correction de désaisonnalisation | Identification objective des meilleures fenêtres horaires pour trader le symbole, basée sur les données réelles plutôt que sur des généralités de marché |

**Point de vigilance transversal** : les métriques des blocs A-C doivent être calculées **par sous-périodes** (par exemple par année) et pas seulement agrégées sur 10 ans — un symbole peut avoir une dynamique différente en début et fin de période. C'est cette même logique de fenêtre glissante qui alimente le calcul du Hurst en régime "instable ou stable" (bloc B).

### 3.5 Sortie finale — la carte d'identité
- Rapport structuré (JSON, exploitable programmatiquement) + version HTML/Markdown (lecture humaine)
- Visualisations : distribution des rendements, évolution du Hurst dans le temps, heatmap ATR par heure/jour, scaling law DC
- Fiche de synthèse qualitative résumant les 4 blocs pour le symbole analysé

---

## 4. Du diagnostic académique aux indicateurs MT5 — méthodologie de sélection

### 4.1 Question initiale et deux approches envisagées

**Approche 1 (écartée)** : chercher des indicateurs MT5 natifs (ADX, ZigZag, Bollinger Bands...) comme proxys des métriques académiques, puis valider empiriquement leur fidélité (corrélation entre l'indicateur MT5 et la métrique académique calculée sur la même période, par exemple ADX vs Hurst en fenêtre glissante).

**Approche 2 (retenue)** : recalculer directement les vraies métriques identifiées par la recherche (Hurst, VRT, DC...) plutôt que de chercher des proxys approximatifs, en MQL5 quand c'est pertinent.

**Verdict** : l'approche 2 est meilleure — elle élimine le risque de fidélité imparfaite d'un proxy — **à condition d'accepter qu'elle déplace la difficulté**, du problème "cet indicateur MT5 approxime-t-il bien la métrique académique ?" vers "cette métrique académique, recalculée en continu, est-elle un signal stable et exploitable en temps réel ?". La réponse à cette seconde question varie fortement selon le bloc (voir §4.2).

### 4.2 Analyse bloc par bloc : simplicité de recalcul en direct (MQL5)

| Bloc | Recalcul en direct (MQL5) | Justification |
|---|---|---|
| **Directional Change** (bloc B, volet scaling) | ✅ Simple — recommandé | Algorithme intrinsèquement causal (comparaison au dernier extremum, tick par tick), coût de calcul quasi nul, naturellement adapté au temps réel |
| **Distribution** (bloc A) | ✅ Simple, mais peu d'intérêt | ATR et StdDev natifs sont déjà des estimateurs proches de ce qu'on veut mesurer ici — recalculer une vraie kurtosis/VaR en glissant est faisable mais l'écart de valeur ajoutée est faible |
| **Saisonnalité** (bloc D) | N/A — pas un recalcul | Paramètre figé, calibré une fois offline, appliqué comme filtre horaire simple (`if heure dans [X,Y]`) — aucune complexité de recalcul temps réel |
| **Hurst / VRT** (bloc B, volet régime) | ⚠️ Complexe — déconseillé en direct | Pas de complexité de code (boucles/arithmétique standard), mais complexité **conceptuelle** : ces estimateurs sont conçus pour un diagnostic ponctuel sur grand échantillon, pas pour un recalcul bar-par-bar. Recalculés en fenêtre glissante fine, ils sont statistiquement beaucoup plus bruyants/instables qu'un ADX (conçu et lissé pour cet usage précis) |

### 4.3 Rôle réel du Hurst/VRT — clarification importante

Distinction essentielle entre deux niveaux de question :
- *"Ce symbole se comporte-t-il en tendance ou en range, structurellement ?"* → question **stratégique**, répondue par le Hurst/VRT
- *"À cet instant précis, dois-je entrer en trend-following ou en mean-reversion ?"* → question **tactique**, répondue par des indicateurs rapides (ADX, etc.) déjà calibrés

**Le Hurst/VRT répond exclusivement à la première.** Sa valeur ajoutée est **architecturale**, pas un signal de trading :
- Détermine quelle famille de stratégie a une chance d'avoir un edge sur le symbole (avant même de parler d'indicateurs)
- Si le régime est stable dans le temps (Hurst stable sur les fenêtres glissantes de la carte d'identité) → un seul EA figé suffit
- Si le régime change souvent → architecture à bascule entre plusieurs modes déjà calibrés, ou portefeuille de plusieurs EA complémentaires

**Ce qui est injecté dans l'EA n'est jamais la valeur du Hurst elle-même** (pas de "Hurst = 0.42" en paramètre), mais le résultat de son interprétation : une décision de conception (architecture mean-reversion vs trend vs bascule adaptative). Les paramètres numériques concrets utilisés en live (seuil δ du DC, fenêtre ATR, plage horaire) proviennent des autres blocs (volatilité, saisonnalité), pas du Hurst directement.

**Raffinement possible (non prioritaire pour la v1)** : si on veut exploiter le Hurst en quasi temps réel, la bonne approche n'est pas un recalcul à chaque bougie M30/H1, mais un recalcul à très basse fréquence (ex : hebdomadaire) comme interrupteur lent entre deux modes déjà calibrés de l'EA — à explorer une fois les résultats concrets obtenus sur AUDCAD.

---

## 5. Architecture technique reliant l'analyse offline et l'EA live

```
Analyse Python (offline)
  10 ans de ticks, périodique (ex: annuel)
      ↓
Carte d'identité calibrée
  Seuils DC, fenêtres, filtres horaires
      ↓
EA MT5 (temps réel)
  DC natif, ATR/ADX calibrés, filtre horaire

↻ recalibrage périodique si dérive de régime détectée
```

### 5.1 Principe des trois étages

**Étage 1 — Python, offline, périodique** : l'analyse complète (10 ans de ticks, stylized facts, Hurst/VRT en glissant, scaling DC, saisonnalité) tourne une fois, puis est **rejouée périodiquement** (ex : annuellement, ou dès dégradation de performance constatée en live) pour vérifier la stabilité structurelle du symbole.

**Étage 2 — La carte d'identité produit une config, pas un flux temps réel** : le résultat se traduit en un jeu de paramètres figés (seuil δ du DC, fenêtre ATR calibrée sur la mémoire de volatilité mesurée, heures de trading autorisées, biais de régime dominant) — littéralement un fichier de configuration consommé par l'EA, pas un module de calcul embarqué.

**Étage 3 — MQL5, temps réel, uniquement des calculs rapides** : l'EA ne recalcule que ce qui est intrinsèquement adapté au calcul continu tick par tick (DC natif, ATR/ADX, filtre horaire). **Jamais de recalcul live du Hurst exponent**.

### 5.2 Deux échelles de "fenêtre glissante" — ne pas confondre

- **Usage 1 (interne à l'analyse Python)** : le Hurst calculé sur fenêtres glissantes sur les 10 ans d'historique est un outil de **diagnostic** répondant à "le régime est-il stable dans le temps ?" — fait partie du rapport de caractérisation lui-même.
- **Usage 2 (boucle de recalibrage)** : processus **séparé et à fréquence beaucoup plus basse** (ex : annuel, ou déclenché par dégradation de performance), qui rejoue l'analyse complète avec les données les plus récentes pour détecter une dérive vs la version précédente de la config. C'est un cycle de révision périodique et discret (type audit), pas un processus continu.

---

## 6. Architecture technique — stack et structure de projet

### 6.1 Stack proposée
- **Python** comme langage d'analyse offline
- **Polars** (plutôt que pandas) comme moteur principal : sur 10 ans de ticks (potentiellement 100-300 millions de lignes pour une paire liquide), nettement plus rapide et économe en mémoire, syntaxe lazy évitant de charger tout en RAM. Pandas reste utilisable ponctuellement sur des sous-ensembles.
- **Parquet** comme format de stockage intermédiaire : conversion du CSV brut dès l'ingestion (lu une seule fois), toutes les analyses suivantes tournent sur le Parquet
- **scipy.stats** (skew, kurtosis, Jarque-Bera), **statsmodels** (ADF, KPSS, autocorrélation)
- **Hurst/VRT et Directional Change** : implémentation maison (pas de librairie Python mature et suffisamment flexible pour nos besoins de fenêtres glissantes configurables et de seuils DC multiples)
- **matplotlib/plotly** pour les visualisations du rapport

### 6.2 Structure de projet proposée

```
audcad-analysis/
├── config/
│   └── caracterisation.yaml       # seuils DC, tailles de fenêtres, etc. — centralisé et versionné
├── data/
│   ├── raw/                       # CSV bruts (lecture seule)
│   └── processed/                 # Parquet nettoyé, partitionné par année
├── src/
│   ├── ingestion.py                # parsing CSV -> Parquet
│   ├── cleaning.py                 # dédoublonnage, outliers, validation
│   ├── resampling/
│   │   ├── physical_time.py        # M30, H1 OHLC + stats de spread
│   │   └── intrinsic_time.py       # Directional Change multi-seuils
│   ├── metrics/
│   │   ├── distribution.py         # bloc A
│   │   ├── regime.py               # bloc B — Hurst, VRT, scaling DC
│   │   ├── volatility.py           # bloc C
│   │   └── seasonality.py          # bloc D
│   └── report.py                   # assemblage de la carte d'identité
├── output/
│   └── {symbole}/{date_analyse}/
│       ├── carte_identite.json
│       ├── carte_identite.html
│       └── figures/
└── notebooks/                      # exploration ponctuelle, pas de logique définitive ici
```

**Principes de conception à respecter** :
- Config centralisée en YAML (tous les paramètres ajustables dans un seul fichier, pas dispersés dans le code) — facilite la reproductibilité et la boucle de recalibrage
- Traçabilité des runs : chaque exécution horodatée et versionnée
- Le pipeline (`src/`) ne contient **aucune** logique spécifique à un symbole — le symbole est un paramètre d'entrée
- Ordre de test recommandé : USDJPY d'abord (disponible, plus volumineux) comme test de charge du pipeline (notamment ingestion Polars et DC), avant le run officiel sur AUDCAD dès disponibilité

### 6.3 Convention de nommage des fichiers

Cohérente avec la convention `ACTIF_TIMEFRAME_LOGIQUE` déjà en usage dans CORE.md, adaptée à ce projet pour permettre la comparaison inter-symboles et inter-runs.

**Données traitées (Parquet)** :
```
data/processed/{SYMBOLE}/{SYMBOLE}__{DATA_DEBUT}_{DATA_FIN}.parquet
```
Exemple : `data/processed/AUDCAD/AUDCAD__2016-01-03_2026-06-23.parquet`

**Sorties de la carte d'identité** :
```
output/{SYMBOLE}/{SYMBOLE}__{DATA_DEBUT}_{DATA_FIN}__run-{HORODATAGE_RUN}__carte_identite.json
```
Exemple : `AUDCAD__2016-01-03_2026-06-23__run-20260702_1500__carte_identite.json`

Trois informations distinctes nécessaires à la comparaison :
- le **symbole** (comparaison inter-actifs)
- la **période de données couverte** (utile si analyse sur sous-période un jour)
- l'**horodatage du run** (comparaison inter-temporelle, détection de dérive)

**Règle complémentaire** : archiver systématiquement le fichier de config YAML utilisé, à côté de chaque sortie, avec le même nom de base (extension `.yaml`) — sans ça, impossible de distinguer un écart entre deux runs dû à une vraie dérive de marché d'un écart dû à un changement de paramètre.

---

## 7. Environnement de développement

- Développement via **Claude Code dans VS Code**, sur PC local (Windows 10 Pro)
- Accès direct aux fichiers locaux sans upload — les fichiers tick volumineux (plusieurs Go) restent sur le poste, lus directement depuis `C:\Users\ericf\Documents\Trading algo\Tick Data Suite\`
- Raison du choix Claude Code plutôt que l'interface chat classique : (1) taille des fichiers tick (plusieurs Go, incompatible avec un upload chat), (2) nature multi-fichiers du projet (modules Python, config YAML, outputs versionnés) évoluant sur plusieurs sessions — cas d'usage adapté à un environnement de développement local avec suivi de version

---

## 8. Décisions validées (récapitulatif)

- [x] Démarche générique multi-actifs, aucune logique spécifique à un symbole dans le pipeline
- [x] Pipeline en 5 étapes : ingestion → nettoyage → rééchantillonnage multi-échelle → 4 blocs de métriques → carte d'identité
- [x] Rééchantillonnage en parallèle : temps physique (M30/H1) ET temps intrinsèque (Directional Change)
- [x] Approche "recalcul direct des vraies métriques" retenue plutôt que "proxy MT5 + test de fidélité"
- [x] Directional Change : recalcul natif en MQL5 pertinent et simple
- [x] Hurst/VRT : jamais recalculé en live dans l'EA — rôle strictement diagnostic/architectural en amont (Python offline)
- [x] Saisonnalité : filtre horaire figé, calibré offline, pas de recalcul
- [x] Architecture à 3 étages : Python offline périodique → config calibrée → MQL5 temps réel, avec boucle de recalibrage à basse fréquence (pas de recalcul continu)
- [x] Stack : Polars + Parquet + scipy/statsmodels + implémentations maison (Hurst, VRT, DC)
- [x] Convention de nommage : SYMBOLE + période de données + horodatage de run, pour données traitées et sorties
- [x] Config YAML archivée systématiquement avec chaque sortie
- [x] Développement via Claude Code / VS Code, accès direct aux fichiers locaux

## 9. Prochaines étapes

1. Attendre disponibilité du fichier AUDCAD (en cours de chargement) — en parallèle, possibilité de démarrer le test de charge sur USDJPY (déjà disponible)
2. Développer le module d'ingestion (`src/ingestion.py`) : parsing CSV → Parquet, avec vérification première/dernière ligne (règle CORE.md)
3. Développer le module de nettoyage (`src/cleaning.py`)
4. Valider le fichier de config YAML initial (seuils DC à tester, tailles de fenêtres Hurst/VRT, découpage horaire)
5. Développer le rééchantillonnage multi-échelle
6. Développer les 4 modules de métriques, en commençant probablement par le bloc D (saisonnalité) et le bloc A (distribution), les plus simples, avant le bloc B (régime, incluant DC et Hurst/VRT)
7. Premier run complet sur USDJPY (test de charge) puis AUDCAD (cas d'application cible)
8. Une fois les résultats obtenus sur AUDCAD : décision sur l'architecture de l'EA (mean-reversion, trend-following, ou bascule adaptative) et sélection/calibration finale des indicateurs MT5

---

## 10. Références bibliographiques complètes

- Cont, R. (2001). "Empirical properties of asset returns: stylized facts and statistical issues." *Quantitative Finance*, 1, 223-236.
- Guillaume, D.M., Dacorogna, M.M., Davé, R.R., Müller, U.A., Olsen, R.B., Pictet, O.V. (1997). "From the bird's eye to the microscope: A survey of new stylized facts of the intra-daily foreign exchange markets." *Finance and Stochastics*, 1, 95-129.
- Müller, U.A., Dacorogna, M.M., Olsen, R.B., Pictet, O.V., Schwarz, M., Morgenegg, C. (1990). "Statistical study of foreign exchange rates, empirical evidence of a price change scaling law, and intraday analysis." *Journal of Banking & Finance*, 14(6), 1189-1208.
- Glattfelder, J.B., Dupuis, A., Olsen, R.B. (2011). Travaux sur les lois d'échelle du Directional Change et des overshoots.
- Golub, A., Glattfelder, J.B., Olsen, R.B. (2016-2022). Extensions du cadre Directional Change / temps intrinsèque (liquidité multi-échelle, stratégies systématiques).
- Tsang, E.P.K. et al. (2010-2024). Travaux sur le Directional Change appliqué au trading et au nowcasting en FX.
- Sewell, M. (2011). "Characterization of financial time series." Research Note RN/11/01, University College London.

*Sources complémentaires consultées (brokers, blogs spécialisés) pour le contexte de marché AUDCAD (sessions, corrélations commodities) — non académiques, à utiliser avec prudence méthodologique, voir échanges antérieurs du projet pour le détail.*

