
# PROJET : FOURIER
_Période : 2026-06 | Statut : CLOS — verdict négatif_
_Dernière mise à jour : 2026-06-08_

---

## Objectif

Tester si une décomposition spectrale du prix (FFT et ondelettes, en fenêtre
glissante, sans biais de look-ahead) permet de prédire la direction de la
prochaine bougie H1, en vue d'un futur EA "suivi de tendance" (entrée sur
direction prédite, SL = ATR × multiple, trailing stop, pas de TP fixe).

Recherche purement exploratoire : pas de code d'EA, juste valider ou invalider
l'idée de départ avant d'investir du temps dans un développement.

---

## Méthode

- Construction de barres H1 à partir des ticks bruts (mid-price), sur une
  tranche récente (~16 derniers mois, jusqu'à 2026-02-12), via recherche
  dichotomique de l'offset dans les fichiers de ticks (évite de charger des
  fichiers de plusieurs Go en mémoire)
- Pour chaque instant t : décomposition de la fenêtre [t-W, t) (jamais t),
  reconstruction du signal lissé (FFT : conservation des K plus grandes
  amplitudes ; ondelettes : conservation de l'approximation, mise à zéro des
  coefficients de détail), lecture de la pente de fin de fenêtre comme
  prédiction directionnelle, comparaison à la direction réelle observée
- Mesure du taux de réussite directionnelle vs. référence aléatoire (50%)
- Grille testée : tailles de fenêtre W = 50/100/200, K = 3/5/10 (FFT),
  familles d'ondelettes db4/sym5/coif3 avec niveau de décomposition adaptatif
  (FFT : 9 configs ; ondelettes : 9 configs par instrument)

---

## Résultats

**EURUSD H1** (FFT + ondelettes, 18 configs) :
- FFT : 50.31% à 50.86% de réussite (écarts de +0.31 à +0.86 pts)
- Ondelettes : 48.53% à 50.74% (écarts de -1.47 à +0.74 pts)
- Bande de bruit statistique à 95% (n≈8000) : ±1.1 pts autour de 50%
- → tous les résultats dans la bande de bruit, aucun signal exploitable

**USDJPY H1** (ondelettes, 9 configs) : 48.86% à 51.07% — léger biais
systématique **positif**, croissant avec la taille de fenêtre (jusqu'à
+1.07 pts en W=200)

**NAS100 H1** (ondelettes, 9 configs) : 48.82% à 49.82% — léger biais
systématique **négatif** (-0.18 à -1.18 pts), donc de signe opposé à EURUSD/USDJPY

**Élément clé de la synthèse** : le signe et l'amplitude du léger biais
observé varient de façon incohérente d'un instrument à l'autre (positif sur
les deux paires forex, négatif sur l'indice). Si la décomposition spectrale
révélait un vrai phénomène de prévisibilité, on attendrait un effet de même
signe et d'amplitude comparable sur des marchés similaires. Cette hétérogénéité
est la signature d'un artefact statistique propre à chaque série (légères
autocorrélations idiosyncratiques de la période échantillonnée), pas d'un
edge réel et universel.

---

## Verdict final

**L'idée de départ — extraire un signal directionnel court terme à partir de
la forme spectrale récente du prix (Fourier ou ondelettes) — n'est validée sur
aucun des trois marchés testés (EURUSD, USDJPY, NAS100), ni en H1.**

36 configurations testées au total, méthodologie rigoureuse (pas de
look-ahead, comparaison statistique à la référence aléatoire, bande de bruit
calculée). Conclusion nette : ne pas poursuivre cette piste pour un futur EA.

**Piste envisagée puis écartée** : tester sur un timeframe plus court (M15).
Écartée par anticipation — le rapport signal/bruit se dégrade aux échelles
courtes (microstructure dominante), et même un signal hypothétique serait
probablement inexploitable une fois les coûts de transaction pris en compte.
Le constat négatif, déjà cohérent sur 3 marchés et 2 méthodes, n'avait pas
besoin d'être re-testé sur une 4e dimension pour être considéré comme solide.

---

## Fichiers produits

| Fichier | Rôle |
|---|---|
| `01_build_h1_bars.py`, `04_build_h1_bars_multi.py` | Construction des barres H1 depuis les ticks bruts (extraction par tranche récente, sans charger le fichier complet) |
| `02_test_fft_direction.py` | Test directionnel par décomposition FFT (grille W × K) |
| `03_test_wavelet_direction.py`, `05_test_wavelet_multi.py` | Test directionnel par décomposition en ondelettes (grille W × famille, niveau adaptatif) |
| `EURUSD_H1_recent.csv`, `USDJPY_H1_recent.csv`, `NAS100_H1_recent.csv` | Barres H1 réutilisables pour d'autres projets (~16 mois, jusqu'au 2026-02-12) |

Tous dans `C:\TradingBots\Fourier\`.

---

## Enseignements méthodologiques (réutilisables pour d'autres recherches)

- Recherche dichotomique d'offset dans un gros fichier de ticks = extraction
  rapide d'une tranche récente sans charger des Go en mémoire
- Toujours calculer la bande de bruit statistique (écart-type d'une proportion
  aléatoire) avant de juger un résultat "intéressant"
- Tester la même méthode sur plusieurs marchés est un bon levier pour
  distinguer un vrai effet (cohérent) d'un artefact (hétérogène)
- Heartbeats avec `flush=True` indispensables pour suivre un traitement long
  sans avoir à inspecter le process depuis l'extérieur
