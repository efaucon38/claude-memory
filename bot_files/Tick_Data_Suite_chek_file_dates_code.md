# -*- coding: utf-8 -*-
import os

DATA_DIR  = r"C:\Users\ericf\Documents\Trading algo\Tick Data Suite"
FILE_NAME = "2003-05-04 - 2026-05-31 - EURUSD_GMT+0_NO-DST H1_H1.csv"
FILE_PATH = os.path.join(DATA_DIR, FILE_NAME)

print("=" * 65)
print(f" Fichier : {FILE_NAME}")
print(f" Taille  : {os.path.getsize(FILE_PATH)/1e9:.2f} Go")
print("=" * 65)

# Première ligne
with open(FILE_PATH, 'r', encoding='utf-8') as f:
    first = f.readline().strip()
print(f"\n Premiere ligne : {first}")

# Dernière ligne — on lit 5 Mo depuis la fin, une seule fois
print(" Recherche derniere ligne en cours...")
with open(FILE_PATH, 'rb') as f:
    f.seek(max(0, os.path.getsize(FILE_PATH) - 5 * 1024 * 1024))
    data = f.read().decode('utf-8', errors='ignore')

last = ''
for line in reversed(data.splitlines()):
    if line.strip():
        last = line.strip()
        break

print(f" Derniere ligne : {last}")
print("\n" + "=" * 65)
input(" Appuyez sur Entree pour fermer...")

