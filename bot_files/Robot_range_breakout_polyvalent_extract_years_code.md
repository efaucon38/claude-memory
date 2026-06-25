# -*- coding: utf-8 -*-
"""
================================================================================
EXTRACT YEARS — Extraction d'une plage d'années depuis un fichier tick CSV
================================================================================

DESCRIPTION
-----------
Extrait les lignes d'un fichier CSV Tick Data Suite correspondant à une
plage d'années définie, et les sauvegarde dans un nouveau fichier plus petit.

UTILISATION
-----------
1. Configurer FILE_PATH, YEAR_START, YEAR_END et OUTPUT_PATH ci-dessous
2. Lancer : python extract_years.py

POURQUOI CE SCRIPT
------------------
Les fichiers Tick Data Suite peuvent couvrir 20+ ans et peser 20+ Go.
Pour les analyses exploratoires ou les backtests sur une période récente,
il est plus efficace de travailler sur un fichier réduit plutôt que de
lire l'intégralité du fichier à chaque fois.

FORMAT
------
CSV Tick Data Suite, GMT+0, NO-DST, sans header.
Format timestamp : DD.MM.YYYY HH:MM:SS.mmm (23 caractères fixes)
Extraction basée sur les 4 caractères de l'année (positions 6-9).

PERFORMANCE
-----------
Lecture et écriture en streaming par chunks — mémoire constante quelle
que soit la taille du fichier source.
================================================================================
"""

import os
import sys
import time

# ==============================================================================
# ► CONFIG — À MODIFIER
# ==============================================================================

FILE_PATH   = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite\2003-05-05 - 2026-05-31 - XAUUSD_GMT+0_NO-DST ticks.csv"

YEAR_START  = 2015   # Première année à extraire (incluse)
YEAR_END    = 2026   # Dernière année à extraire (incluse)

# Nom du fichier de sortie — généré automatiquement si laissé vide
OUTPUT_PATH = ""     # Laisser vide pour générer automatiquement

# Taille des chunks de lecture (en nombre de lignes)
CHUNK_SIZE  = 500_000

# ==============================================================================
# GÉNÉRATION DU NOM DE SORTIE
# ==============================================================================

if not OUTPUT_PATH:
    src_dir  = os.path.dirname(FILE_PATH)
    src_name = os.path.basename(FILE_PATH)
    # Remplacer la plage d'années dans le nom du fichier source
    # Ex : "2003-05-05 - 2026-05-31 - XAUUSD..." → "2015-01-01 - 2026-12-31 - XAUUSD..."
    # On cherche le premier " - " pour isoler le préfixe de dates
    parts = src_name.split(' - ', 2)
    if len(parts) >= 3:
        new_name = f"{YEAR_START}-01-01 - {YEAR_END}-12-31 - {parts[2]}"
    else:
        new_name = f"{YEAR_START}_{YEAR_END}_{src_name}"
    OUTPUT_PATH = os.path.join(src_dir, new_name)

# ==============================================================================
# EXTRACTION
# ==============================================================================

def pause():
    print("\n" + "="*60)
    print(" Appuyez sur Entrée pour fermer...")
    print("="*60)
    try: input()
    except Exception: pass

print("="*65)
print(f" EXTRACTION ANNÉES {YEAR_START}–{YEAR_END}")
print(f" Source  : {FILE_PATH}")
print(f" Sortie  : {OUTPUT_PATH}")
print("="*65)

if not os.path.exists(FILE_PATH):
    print(f"\n[ERREUR] Fichier source introuvable :\n  {FILE_PATH}")
    pause(); sys.exit(1)

if os.path.exists(OUTPUT_PATH):
    print(f"\n[ATTENTION] Le fichier de sortie existe déjà :")
    print(f"  {OUTPUT_PATH}")
    rep = input("Écraser ? (o/N) : ").strip().lower()
    if rep != 'o':
        print("Annulé.")
        pause(); sys.exit(0)

file_size   = os.path.getsize(FILE_PATH)
t0          = time.time()
last_print  = 0.
bytes_read  = 0
lines_in    = 0
lines_out   = 0
cur_year    = None

# Années valides sous forme de strings pour comparaison rapide
# Le timestamp a le format DD.MM.YYYY → l'année est aux positions 6:10
years_valid = {str(y) for y in range(YEAR_START, YEAR_END + 1)}

print(f"\n Taille source : {file_size/1e9:.2f} Go")
print(f" Lecture en chunks de {CHUNK_SIZE:,} lignes")
print(f" Progression toutes les 3 secondes\n")

try:
    with open(FILE_PATH, 'r', encoding='utf-8', buffering=8*1024*1024) as fin, \
         open(OUTPUT_PATH, 'w', encoding='utf-8', buffering=8*1024*1024) as fout:

        chunk = []
        for line in fin:
            lines_in += 1

            # Extraction rapide de l'année depuis le timestamp
            # Format fixe : DD.MM.YYYY → positions 6:10
            if len(line) >= 10:
                year_str = line[6:10]
                if year_str in years_valid:
                    chunk.append(line)
                    lines_out += 1
                elif year_str > str(YEAR_END):
                    # On a dépassé la plage — on peut arrêter
                    break

            # Écriture par chunks pour limiter les I/O
            if len(chunk) >= CHUNK_SIZE:
                fout.writelines(chunk)
                chunk = []

            # Progression
            bytes_read += len(line.encode('utf-8'))
            now = time.time()
            if now - last_print >= 3.:
                last_print = now
                pct  = min(bytes_read / file_size * 100, 100.)
                el   = now - t0
                sp   = bytes_read / el / 1e6 if el > 0 else 0
                eta  = int((file_size - bytes_read) / (bytes_read / el)) if bytes_read > 0 else 0
                em, es = divmod(eta, 60)
                bl = 28; fi = int(bl * pct / 100)
                bar = '█'*fi + '░'*(bl-fi)
                yr  = line[6:10] if len(line) >= 10 else '????'
                print(f"\r  [{bar}] {pct:5.1f}%  {sp:4.1f} Mo/s  "
                      f"ETA {em}m{es:02d}s  {yr}  →{lines_out:,} lignes",
                      end='', flush=True)

        # Écrire le reste du chunk
        if chunk:
            fout.writelines(chunk)

    print(f"\r  {'─'*65}", flush=True)
    el = time.time() - t0
    out_size = os.path.getsize(OUTPUT_PATH)
    ratio    = out_size / file_size * 100

    print(f"\n ✓ Extraction terminée en {el/60:.1f} min")
    print(f"\n RÉSULTATS :")
    print(f"   Lignes lues    : {lines_in:>12,}")
    print(f"   Lignes écrites : {lines_out:>12,}  ({lines_out/lines_in*100:.1f}% du total)")
    print(f"   Taille source  : {file_size/1e9:>8.2f} Go")
    print(f"   Taille sortie  : {out_size/1e9:>8.2f} Go  ({ratio:.1f}% de la source)")
    print(f"\n Fichier créé :")
    print(f"   {OUTPUT_PATH}")

except Exception as e:
    import traceback
    print(f"\n\n[ERREUR] {e}")
    print(traceback.format_exc())

finally:
    pause()

