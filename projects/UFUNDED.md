## Projet : UFUNDED_OVERNIGHT_PORTFOLIO
_Démarré : 2026-06-17_

### Contexte
- Plateforme : UFUNDED (ufunded.com) — environnement simulé, invite-only
- Compte : Commander — $90,000 de capital fictif
- Capital overnight disponible : $18,000 (20% × $90,000)
- Maximum Drawdown absolu : $4,000 (static, ne remonte jamais)
- Profit Split : 75%
- Payout minimum : $500 / Payout maximum : $9,000/mois
- Levier : 1:1 retenu (pas de levier sur actions/ETF)
- Interface : TradingView intégré — pas d'API externe possible
- Exécution : manuelle dans l'interface UFUNDED

### Conditions de trading
- Commission : $0.007/action, minimum $1.25/exécution
- Frais min par aller-retour : $2.50
- Seuil de rentabilité commissions : ≥ 179 actions/trade
- Swap overnight : variable selon actif (à vérifier dans l'interface)
- News trading : autorisé (sauf événements restreints ponctuels)
- Heures de marché : 9h30 – 16h00 ET

### Workflow
1. Analyse et allocation calculées en Python (hors plateforme)
2. Exécution manuelle des trades dans l'interface UFUNDED
3. Pas de connexion API possible entre les deux

### Avancement
- ✅ Étape 1 : Mémoire lue
- ✅ Étape 2 : Fonctionnement UFUNDED compris (conditions, User Agreement)
- ✅ Étape 3 : Univers = S&P 500 + ETF majeurs (hypothèse large, vérification manuelle des tickers à faire dans l'interface avant exécution)
- ✅ Étape 4 : Source de données retenue = `yfinance` (daily, Python)
- ⏳ Étape 5 : Méthodologie d'allocation — À faire

### Structure Python prévue (4 modules — pas encore codés)
- MODULE 1 : Univers & données (download yfinance, gaps overnight, volatilité, volume)
- MODULE 2 : Filtrage & sélection (liquidité, volatilité, momentum)
- MODULE 3 : Allocation & sizing ($18,000, nb shares, intégration frais)
- MODULE 4 : Simulation & validation (backtest, vérif drawdown < $4,000, rapport)

### ⚠️ Points en suspens
- Liste exhaustive des actifs disponibles sur UFUNDED non confirmée (à demander à l'account manager ou vérifier via le Scanner de la plateforme)
- Taux de swap overnight à vérifier actif par actif dans l'interface avant sélection finale
