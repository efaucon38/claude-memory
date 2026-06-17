# Déploiement Momentum Scanner sur VPS — Guide complet

## 1. Vue d'ensemble Architecture

Le système de trading **Momentum Scanner** comprend deux composants parallèles :

### Composant 1: robot.py (Trading live)
- Boucle principale 24/7 (slot unique, LONG uniquement)
- Scan D1 quotidien à 23h05 Paris : update caches H1 + calcul scores → watchlist.json
- Gestion des positions : entrée sur H1, exit (EMA50), trailing SL
- Circuit-breaker A2+20j : détecte 2 SL consécutifs → blacklist 20 jours
- Connexion MT5 write : ouvre/ferme trades réels

### Composant 2: 17_paper_trading_simulator.py (Paper trading)
- Boucle horaire infinie (simulation temps réel prospectif)
- Update H1 caches depuis MT5 chaque heure (read-only)
- Lit watchlist.json générée par robot.py (pas de scan propre)
- Simule exactement la même logique que robot.py
- Journalise toutes décisions dans journal_paper_trading.csv
- Détecte discrepances robot vs simulation (débugging)

**Avantage :** Les deux systèmes tournent en parallèle sur le même MT5, permettant de valider la stabilité avant live.

---

## 2. Infrastructure VPS requise

### Specs minimales serveur Windows
- **OS :** Windows Server 2019/2022 ou Windows 10/11 Pro
- **RAM :** 8 GB minimum (16 GB recommandé pour MT5 stable)
- **Disque :** 50 GB (MT5 + historique H1 pour 407 actifs = ~3-5 GB)
- **CPU :** 2+ cores (MT5 est mono-thread, mais OS + Python ont besoin de ressources)
- **Connexion :** 10 Mbps stable

### Services à installer
1. **MetaTrader 5 Terminal** (version 64-bit)
   - Télécharger depuis https://www.metatrader5.com/en/download
   - Installer sans interface graphique possible (mode background avec python-mt5)

2. **Python 3.11+**
   - Installer depuis https://www.python.org/downloads/
   - **Important :** Cocher "Add Python to PATH" lors installation

3. **Git** (optionnel, pour cloner le repo)
   - Installer depuis https://git-scm.com/download/win

### Firewall & Sécurité
- Ouvrir port 23/TCP sortant vers serveurs MT5 (RaiseGlobal)
- Optionnel : ouvrir 22/TCP pour SSH si accès distant via WSL2
- Ne **jamais** ouvrir port inbound pour MT5
- Stocker credentials dans fichier sécurisé (pas en version control)

---

## 3. Structure répertoires VPS

```
C:\TradingBots\
├── Momentum/
│   ├── robot.py                          (main trade executor)
│   ├── 17_paper_trading_simulator.py    (paper trading)
│   ├── scanner.py                        (D1 scan + watchlist.json)
│   ├── signal_h1.py                      (entry/exit logic)
│   ├── risk_manager.py                   (time checks, lot size)
│   ├── trade_executor.py                 (MT5 API wrapper)
│   ├── logger.py                         (logging + Telegram alerts)
│   ├── config.py                         (parameters)
│   ├── symbol_specs.csv                  (407 actifs faisables)
│   ├── h1_cache/                         (caches H1 par symbole)
│   │   ├── LINK_H1.csv
│   │   ├── NVDIA_H1.csv
│   │   └── ...
│   ├── journal_paper_trading.csv         (output simulator)
│   ├── paper_trading_simulator.log       (logs debug)
│   ├── watchlist.json                    (input partagé robot↔simulator)
│   ├── requirements.txt                  (dépendances Python)
│   └── CLAUDE.md                         (instructions de dev)
│
├── Momentum_logs/                        (logs robot.py)
│   ├── robot_YYYY-MM-DD.log
│   └── ...
│
└── backtest_data/                        (résultats historiques)
    ├── multiperiod_results.csv
    └── full_universe_results.csv
```

---

## 4. Installation & Setup

### Étape 1 : Configurer le VPS
```bash
# En PowerShell admin
# Installer Python 3.11+
# Télécharger MT5 installer et exécuter

# Créer la structure répertoires
mkdir C:\TradingBots\Momentum
mkdir C:\TradingBots\Momentum\h1_cache
mkdir C:\TradingBots\Momentum_logs

# Changer de répertoire
cd C:\TradingBots\Momentum
```

### Étape 2 : Installer dépendances Python
```bash
# Copier requirements.txt
# Contenu:
# MetaTrader5==5.0.45
# pandas>=2.0
# numpy>=1.24
# pytz>=2023.3
# python-telegram-bot>=20.0  (optionnel, pour alertes)

pip install -r requirements.txt
```

### Étape 3 : Configurer les credentials
Éditer `config.py` avec les vrais paramètres du compte live :

```python
# config.py - Section MT5
LOGIN = 5007258              # Remplacer par vrai login
PASSWORD = "zL3!gvG8Ol"      # Remplacer par vrai password
SERVER = "RaiseGlobal-Live"  # Vérifier le serveur exact
MT5_PATH = r"C:\Program Files\Raise Global MT5 Terminal\terminal64.exe"

# Paramètres robot
ENABLE_TRADING = True  # ATTENTION: false = read-only pour tests
RISK_EUR = 250         # Risk par trade (validé en backtest)
CAPITAL = 50_000       # Capital initial (pour calcul lot)

# CB A2+20j (NE PAS MODIFIER)
CB_A_N = 2             # 2 SL consécutifs
CB_A_DAYS = 20         # Blacklist 20 jours

# Telegram alerts (optionnel)
TELEGRAM_ENABLED = False  # True si vous avez bot Telegram
TELEGRAM_CHAT_ID = ...
TELEGRAM_TOKEN = ...
```

### Étape 4 : Télécharger les caches H1 initiaux
```bash
# Lancer le backtest historique qui download les données
python 15_full_universe_backtest.py

# Cela remplit h1_cache/ avec 50+ symboles
# Pour les 407 actifs, utiliser download_missing.py (script séparé)
```

### Étape 5 : Préparer MT5 en background (optionnel mais recommandé)

Pour éviter que le terminal MT5 "freeze" :

1. **Lancer MT5 normalement** une fois pour configurer le compte
2. **Créer un batch file** `start_mt5.bat`:
```batch
@echo off
cd "C:\Program Files\Raise Global MT5 Terminal"
start /min terminal64.exe
timeout /t 10
exit
```

3. **Planifier le démarrage via Task Scheduler Windows**:
   - Ouvrir `taskschd.msc`
   - "Créer une tâche simple"
   - Trigger: "Au démarrage"
   - Action: Exécuter `start_mt5.bat`
   - Cocher "Exécuter avec privilèges élevés"

---

## 5. Lancer les deux systèmes en parallèle

### Option A : Deux PowerShell terminals (simple, local testing)
```powershell
# Terminal 1 : Robot trading live
cd C:\TradingBots\Momentum
python robot.py

# Terminal 2 : Paper trading simulator (dans un autre terminal)
cd C:\TradingBots\Momentum
python 17_paper_trading_simulator.py
```

### Option B : Via Task Scheduler (production VPS)

**Tâche 1 : robot.py**
- Nouvelle tâche planifiée
- Trigger: "Au démarrage du système"
- Action: `python C:\TradingBots\Momentum\robot.py`
- Cocher "Exécuter avec privilèges élevés"
- Cocher "Ne pas arrêter si déjà en cours d'exécution"

**Tâche 2 : 17_paper_trading_simulator.py**
- Même configuration que Tâche 1

### Option C : Via wrapper Python (recommandé)

Créer `launcher.py` à la racine :
```python
import subprocess
import sys
import time

processes = []

try:
    print("[LAUNCHER] Démarrage robot.py...")
    p1 = subprocess.Popen([sys.executable, r"C:\TradingBots\Momentum\robot.py"])
    processes.append(("robot.py", p1))
    
    time.sleep(5)  # Laisser robot démarrer
    
    print("[LAUNCHER] Démarrage 17_paper_trading_simulator.py...")
    p2 = subprocess.Popen([sys.executable, r"C:\TradingBots\Momentum\17_paper_trading_simulator.py"])
    processes.append(("simulator.py", p2))
    
    # Monitor processes
    while True:
        for name, proc in processes:
            if proc.poll() is not None:
                print(f"[LAUNCHER] {name} s'est arrêté (code {proc.returncode})")
        time.sleep(60)
        
except KeyboardInterrupt:
    print("[LAUNCHER] Arrêt demandé...")
    for name, proc in processes:
        proc.terminate()
    for name, proc in processes:
        proc.wait()
    print("[LAUNCHER] Tous les processus arrêtés.")
```

Ensuite créer tâche planifiée qui exécute `launcher.py`.

---

## 6. Monitoring & Alertes

### Logs à surveiller

#### robot.py
- **Fichier :** `Momentum_logs/robot_YYYY-MM-DD.log`
- **À vérifier :** Scan D1 réussi, entrées/exits sans erreurs, CB triggers
- **Alerte :** Erreurs connexion MT5, balance breakdown

Exemple log sain :
```
2026-06-18 23:05:12 | INFO | === Scan D1 quotidien ===
2026-06-18 23:05:45 | INFO | Watchlist : LINK(0.891) Softbank(0.847) NVDIA(0.823)
2026-06-18 23:45:00 | INFO | LINK ENTRY | price=28.45 | SL=26.12
2026-06-18 22:15:00 | INFO | LINK EXIT signal (close < EMA50)
2026-06-18 22:15:00 | INFO | PnL: +0.567R (+141 EUR)
```

#### 17_paper_trading_simulator.py
- **Fichier :** `paper_trading_simulator.log` + `journal_paper_trading.csv`
- **À vérifier :** Watchlist lue chaque heure, entrées/exits simulées, PnL calculés
- **Alerte :** Discrepance robot vs simulator (différence PnL >5%)

Exemple discrepance à investiguer :
```
Simulator: NVDIA ENTRY à 120.45, EXIT à 121.10 = +0.25R
Robot real: NVDIA ENTRY à 120.45, EXIT à 120.90 = +0.15R
Raison possible: Spread différent, ou slippage MT5 broker
```

### Dashboard CSV pour analyse (Excel)

Le fichier `journal_paper_trading.csv` contient :
- `timestamp` : UTC
- `event_type` : h1_bar, scan, entry, exit, trailing_sl, cb_state
- `symbole` : LINK, NVDIA, etc.
- `close, ema50, atr14, adx14` : Prix + indicateurs
- `pnl_r, pnl_eur` : Résultat trade (en risque units + EUR)
- `raison_exit` : signal_ema50, sl_hit, max_bars
- `cb_a_consec_sl`, `cb_a_blacklist` : État circuit-breaker

Pour analyser :
```python
import pandas as pd

j = pd.read_csv("journal_paper_trading.csv")

# Résumé par event
print(j.value_counts("event_type"))

# Entrées réussies par jour
entries = j[j["event_type"] == "entry"].groupby(j["timestamp"].str[:10]).size()
print(entries)

# PnL quotidien
j["date"] = j["timestamp"].str[:10]
pnl_daily = j[j["event_type"] == "exit"].groupby("date")[["pnl_r", "pnl_eur"]].sum()
print(pnl_daily)

# Comparaison robot vs simulator
# (à faire manuellement en comparant logs)
```

### Alertes recommandées (Telegram optionnel)

Si `TELEGRAM_ENABLED = True` dans config.py :

```
✅ Entrée LINK à 28.45
❌ SL HIT LINK → CB activé (2 SL consécutifs)
⚠️ Reconnexion MT5 tentative 2/3
📊 Journée: 5 trades, WR=60%, PnL=+287 EUR
```

Tokens pour setup Telegram : voir `logger.py` ligne 45-55.

---

## 7. Gestion des caches H1

### Cycle de mise à jour des données

1. **Initialisation (day 1)**
   - Lancer `15_full_universe_backtest.py`
   - Télécharge 500 barres H1 pour 50 symboles (~10 min)
   - Crée `h1_cache/*.csv`

2. **Maintenance quotidienne**
   - `robot.py` et `simulator.py` update leurs caches chaque heure
   - Update delta : récupère dernières ~500 barres, fusionne avec existant (déduplique sur `time`)
   - Aucun action manuelle requise

3. **Expiration données (optionnel)**
   - Après 6 mois, caches peuvent avoir 10k+ rows (ralentissement)
   - Créer script `cleanup_caches.py` pour archiver les vieilles barres:
   ```python
   import pandas as pd
   from pathlib import Path
   from datetime import datetime, timedelta
   
   H1_DIR = Path("h1_cache")
   KEEP_DAYS = 180  # Garder 6 derniers mois
   cutoff = datetime.now() - timedelta(days=KEEP_DAYS)
   
   for csv_file in H1_DIR.glob("*_H1.csv"):
       df = pd.read_csv(csv_file, parse_dates=["datetime_utc"])
       df_clean = df[df["datetime_utc"] >= cutoff]
       df_clean.to_csv(csv_file, index=False)
       print(f"{csv_file.name}: {len(df)} → {len(df_clean)} rows")
   ```

### Debugging data issues

**Problème :** Cache corrompu ou données manquantes
```bash
# Supprimer le cache et re-télécharger
rm h1_cache/LINK_H1.csv
python -c "
import MetaTrader5 as mt5
mt5.initialize(path=r'C:\Program Files\Raise Global MT5 Terminal\terminal64.exe', ...)
mt5.symbol_select('LINK', True)
rates = mt5.copy_rates_from_pos('LINK', mt5.TIMEFRAME_H1, 0, 500)
# Sauvegarder manuellement en CSV
"
```

---

## 8. Troubleshooting

| Symptôme | Cause | Solution |
|----------|-------|----------|
| MT5 connexion timeout | Serveur RaiseGlobal down | Vérifier status.raiseglobal.com, retry |
| Password rejected | Mauvais credentials | Vérifier config.py LOGIN/PASSWORD/SERVER |
| H1 cache vide pour symbole | Symbol non disponible sur broker | Retirer de symbol_specs.csv si faisable=False |
| Paper trading lag (>2h de retard) | Python trop lent ou MT5 hang | Augmenter RAM, vérifier tâches background |
| Watchlist.json not found | robot.py pas encore lancé | Attendre scan D1 à 23h05, ou lancer manuel |
| PnL simulator >> robot PnL | Spread/slippage différent | Normal, documenté dans journal, <5% acceptable |
| Circuit-breaker spam | Mauvais marché ou SL trop tight | Revalider SL_MULT=2.5 sur les 6 derniers mois |

---

## 9. Checklist pre-deployment

- [ ] VPS a 8GB+ RAM, Windows Server 2019+
- [ ] MT5 Terminal installé et configuré
- [ ] Python 3.11+ installé, pip fonctionne
- [ ] `requirements.txt` installé : `pip install -r requirements.txt`
- [ ] `config.py` a les vrais credentials du compte live
- [ ] `h1_cache/` peuplé avec au moins 30 symboles (lancer backtest)
- [ ] `robot.py` lance sans erreur (Ctrl+C après test)
- [ ] `17_paper_trading_simulator.py` lance sans erreur
- [ ] Watchlist.json se crée automatiquement après 23h05 Paris
- [ ] Logs arrivent dans `robot_*.log` et `paper_trading_simulator.log`
- [ ] Journaux CSV se remplissent : `journal_paper_trading.csv`
- [ ] Alertes Telegram testées (si activées)
- [ ] Tâches planifiées Task Scheduler créées pour auto-start
- [ ] Backup strategy en place (caches, logs, journal CSV)

---

## 10. Rollback & Sécurité

### Avant d'aller live
1. **Tester 1 semaine en paper trading** : observer logs, comparaison simulator vs réel
2. **Valider méthode B EMA50** : PF > 1.0 sur derniers 6 mois (vérifier dans logs)
3. **Vérifier CB A2+20j** : doit trigger ~1 fois par mois max sur 407 actifs
4. **Tester reconnect auto** : débrancher VPS réseau 30s, vérifier recover

### Rollback rapide
Si erreur détectée :
```bash
# Arrêter robot immédiatement
taskkill /F /IM terminal64.exe  # Ferme MT5
taskkill /F /FI "WINDOWTITLE eq python*"  # Tue tous scripts Python

# Réanalyser logs
tail -50 Momentum_logs/robot_$(date +%Y-%m-%d).log

# Rétablir config.py à version stable
git checkout -- config.py

# Relancer prudemment
python robot.py  # En console, pas en background
```

### Sécurité

- **Ne JAMAIS mettre de credentials en Git** : utiliser variables environnement ou fichier .env (gitignored)
- **Restreindre permissions** : `h1_cache/` et logs en read-only pour autres utilisateurs
- **Auditer accès** : une fois live, revoir Access Logs pour patterns anormaux
- **Backups** : copier `Momentum/` et logs quotidiennement sur NAS/cloud sécurisé

---

## 11. Contacts & Support

- **Issue trading :** Vérifier d'abord `robot_*.log` et `paper_trading_simulator.log`
- **Question architecture :** Consulter `CLAUDE.md` et commentaires dans robot.py
- **Problème MT5 broker :** Contacter support RaiseGlobal (vérifier status page)

---

**Version :** 2026-06-18  
**Système :** Momentum Scanner v1.0 (Method B EMA50, CB A2+20j)  
**Auteur :** Generated by Claude Code



