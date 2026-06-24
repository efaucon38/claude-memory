"""
================================================================================
RÉCUPÉRATION DES PARAMÈTRES SYMBOLE DEPUIS MT5
================================================================================
Ce script se connecte à MT5 et récupère toutes les valeurs nécessaires
au paramétrage du backtest et de l'EA pour un symbole donné.

UTILISATION
-----------
1. Assurez-vous que MT5 est installé sur ce PC
2. Installez la librairie : pip install MetaTrader5
3. Renseignez les identifiants ci-dessous
4. Lancez le script : python get_symbol_params.py
   OU double-cliquez sur le fichier .py depuis l'explorateur Windows

SORTIE
------
- Affichage console (ne se ferme pas automatiquement)
- Fichier JSON exporté dans le même dossier que ce script
- Fichier LOG en cas d'erreur, dans le même dossier
================================================================================
"""

import json
import os
import sys
import traceback
import logging
from datetime import datetime

# ==============================================================================
# DOSSIER DE SORTIE — affiché immédiatement pour diagnostic
# ==============================================================================

OUTPUT_DIR = os.path.dirname(os.path.abspath(__file__))
LOG_PATH   = os.path.join(OUTPUT_DIR, "get_symbol_params.log")

# Configuration du logger — écrit dans la console ET dans un fichier .log
logging.basicConfig(
    level    = logging.DEBUG,
    format   = "%(asctime)s  %(levelname)-8s  %(message)s",
    datefmt  = "%Y-%m-%d %H:%M:%S",
    handlers = [
        logging.FileHandler(LOG_PATH, encoding='utf-8', mode='w'),
        logging.StreamHandler(sys.stdout)
    ]
)
log = logging.getLogger()

def pause():
    """Empêche la fermeture de la console — attend une touche."""
    print("\n" + "="*60)
    print(" Appuyez sur Entrée pour fermer...")
    print("="*60)
    input()

# ==============================================================================
# ► CONFIG — À RENSEIGNER
# ==============================================================================

LOGIN    = 5007258
PASSWORD = "zL3!gvG8Ol"
SERVER   = "RaiseGlobal-Live"
SYMBOL   = "NAS100"

# ==============================================================================
# POINT D'ENTRÉE PRINCIPAL — tout dans un try/except global
# ==============================================================================

try:
    log.info("=" * 58)
    log.info(f" RÉCUPÉRATION PARAMÈTRES — {SYMBOL}")
    log.info(f" Serveur : {SERVER}  |  Login : {LOGIN}")
    log.info("=" * 58)
    log.info(f" Dossier de sortie : {OUTPUT_DIR}")
    log.info(f" Fichier log       : {LOG_PATH}")

    # ── Import MetaTrader5 ───────────────────────────────────────────
    log.info("\nImport de la librairie MetaTrader5...")
    try:
        import MetaTrader5 as mt5
        log.info("✓ MetaTrader5 importé avec succès")
    except ImportError:
        log.error("La librairie MetaTrader5 n'est pas installée.")
        log.error("  Solution : ouvrez un terminal et lancez : pip install MetaTrader5")
        pause(); sys.exit(1)

    # ── Connexion MT5 ────────────────────────────────────────────────
    log.info("\nConnexion à MT5...")
    if not mt5.initialize(login=LOGIN, password=PASSWORD, server=SERVER):
        err = mt5.last_error()
        log.error(f"Connexion MT5 échouée : {err}")
        log.error("Vérifications à faire :")
        log.error("  1. MT5 est-il bien lancé sur ce PC ?")
        log.error("  2. Le serveur est-il correct ? (RaiseGlobal-Live)")
        log.error("  3. Les identifiants sont-ils corrects ?")
        pause(); sys.exit(1)
    log.info("✓ Connexion MT5 établie")

    # ── Activation du symbole ────────────────────────────────────────
    log.info(f"\nActivation du symbole {SYMBOL}...")
    if not mt5.symbol_select(SYMBOL, True):
        log.error(f"Symbole '{SYMBOL}' introuvable sur ce compte.")
        log.error("Vérifiez le nom exact du symbole dans MT5 (Market Watch).")
        mt5.shutdown(); pause(); sys.exit(1)
    log.info(f"✓ Symbole {SYMBOL} activé")

    # ── Récupération des infos ───────────────────────────────────────
    log.info("\nRécupération des informations du symbole...")
    info = mt5.symbol_info(SYMBOL)
    tick = mt5.symbol_info_tick(SYMBOL)

    if info is None:
        log.error(f"Impossible de récupérer les infos du symbole {SYMBOL}")
        log.error(f"  Dernière erreur MT5 : {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)

    if tick is None:
        log.warning("Impossible de récupérer le tick actuel — marché fermé ?")
        tick_ask = tick_bid = tick_spread = None
    else:
        tick_ask    = tick.ask
        tick_bid    = tick.bid
        tick_spread = round(tick.ask - tick.bid, info.digits)

    log.info("✓ Informations récupérées")

    # ── Calcul CONTRACT_VALUE ────────────────────────────────────────
    contract_value = None
    contract_value_method = "non calculable (tick_size = 0)"
    if info.trade_tick_size > 0 and info.volume_min > 0:
        # tick_value est fourni pour le volume_min, pas pour 1 lot
        # On normalise : tick_value / volume_min = valeur du tick pour 1 lot
        # Puis : * point / tick_size = valeur d un point pour 1 lot
        contract_value = (info.trade_tick_value / info.volume_min) / info.trade_tick_size * info.point
        contract_value_method = "(tick_value / volume_min) / tick_size * point"
        # Affichage de contrôle pour validation manuelle
        cv_raw = info.trade_tick_value / info.trade_tick_size * info.point
        cv_corrected = contract_value

    # ── Affichage ────────────────────────────────────────────────────
    log.info(f"\n{'─'*58}")
    log.info(f" SYMBOLE : {info.name} — {info.description}")
    log.info(f"{'─'*58}")

    log.info("\n PRIX ACTUELS")
    if tick_ask is not None:
        log.info(f"   Ask            : {tick_ask:.{info.digits}f}")
        log.info(f"   Bid            : {tick_bid:.{info.digits}f}")
        log.info(f"   Spread actuel  : {tick_spread} points")
    else:
        log.info("   [Marché fermé — prix non disponibles]")

    log.info("\n SPÉCIFICATIONS CONTRAT")
    calc_mode_str = "CFD" if info.trade_calc_mode == 0 else f"Mode {info.trade_calc_mode}"
    log.info(f"   Type           : {calc_mode_str}")
    log.info(f"   Taille contrat : {info.trade_contract_size}")
    log.info(f"   Tick size      : {info.trade_tick_size}")
    log.info(f"   Tick value     : {info.trade_tick_value:.6f} {info.currency_profit}")
    log.info(f"   Point          : {info.point}")
    log.info(f"   Chiffres       : {info.digits}")
    log.info(f"   Devise profit  : {info.currency_profit}")
    log.info(f"   Devise marge   : {info.currency_margin}")

    log.info(f"\n ► CONTRACT_VALUE (valeur 1 point / 1 lot)")
    if contract_value is not None:
        log.info(f"   {contract_value:.6f} {info.currency_profit}/point/lot")
        log.info(f"   Méthode : {contract_value_method}")
    else:
        log.warning("   NON CALCULABLE — tick_size = 0")
        log.warning("   → Vérification manuelle nécessaire via trade test")

    log.info("\n VOLUMES")
    log.info(f"   Volume min     : {info.volume_min}")
    log.info(f"   Volume max     : {info.volume_max}")
    log.info(f"   Pas de volume  : {info.volume_step}")

    log.info("\n SESSIONS DE TRADING (GMT)")
    sessions_data = {}
    # symbol_info_session_trade absent de certaines versions de la lib MT5
    # On utilise un fallback avec les valeurs connues pour NAS100 RaiseFX
    _has_session_func = hasattr(mt5, 'symbol_info_session_trade')
    if _has_session_func:
        for day_idx, day_name in enumerate(
                ['Dimanche','Lundi','Mardi','Mercredi','Jeudi','Vendredi','Samedi']):
            sessions = []
            for session_idx in range(5):
                res = mt5.symbol_info_session_trade(SYMBOL, day_idx, session_idx)
                if res is None:
                    break
                ok, from_t, to_t = res
                if ok and to_t > 0:
                    hf, mf = from_t // 3600, (from_t % 3600) // 60
                    ht, mt_ = to_t  // 3600, (to_t  % 3600) // 60
                    sessions.append(f"{hf:02d}:{mf:02d}-{ht:02d}:{mt_:02d}")
            if sessions:
                log.info(f"   {day_name:<12} : {', '.join(sessions)}")
                sessions_data[day_name] = sessions
    else:
        log.warning("   symbol_info_session_trade absent de cette version MT5.")
        log.warning("   Sessions renseignees depuis capture d ecran RaiseFX :")
        sessions_data = {
            "Lundi"    : ["01:05-23:55"],
            "Mardi"    : ["01:05-23:55"],
            "Mercredi" : ["01:05-23:55"],
            "Jeudi"    : ["01:05-23:55"],
            "Vendredi" : ["01:05-23:55"],
        }
        for day_name, sess in sessions_data.items():
            log.info(f"   {day_name:<12} : {', '.join(sess)}")

    log.info("\n SWAP (coût overnight par lot)")
    log.info(f"   Swap long      : {info.swap_long} {info.currency_profit}")
    log.info(f"   Swap short     : {info.swap_short} {info.currency_profit}")

    log.info("\n MARGES (pour 1 lot)")
    log.info(f"   Marge initiale : {info.margin_initial} {info.currency_margin}")
    log.info(f"   Marge maintien : {info.margin_maintenance} {info.currency_margin}")

    # ── Export JSON ──────────────────────────────────────────────────
    params = {
        "meta": {
            "generated_at"      : datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "broker"            : SERVER,
            "login"             : LOGIN,
            "symbol"            : SYMBOL,
            "description"       : info.description,
        },
        "auto": {
            "contract_value_per_point_per_lot" : round(contract_value, 6) if contract_value else None,
            "contract_value_currency"          : info.currency_profit,
            "contract_value_method"            : contract_value_method,
            "tick_size"                        : info.trade_tick_size,
            "tick_value"                       : info.trade_tick_value,
            "point"                            : info.point,
            "digits"                           : info.digits,
            "volume_min"                       : info.volume_min,
            "volume_max"                       : info.volume_max,
            "volume_step"                      : info.volume_step,
            "currency_profit"                  : info.currency_profit,
            "currency_margin"                  : info.currency_margin,
            "swap_long_per_lot"                : info.swap_long,
            "swap_short_per_lot"               : info.swap_short,
            "spread_at_retrieval"              : tick_spread,
            "calc_mode"                        : calc_mode_str,
            "sessions_gmt"                     : sessions_data,
            "margin_initial_per_lot"           : info.margin_initial,
            "margin_maintenance_per_lot"       : info.margin_maintenance,
        },
        "manual": {
            "_NOTE"                           : "Renseigner ces valeurs après vérification — ne pas laisser à null",
            "commission_per_lot_roundtrip"    : None,
            "commission_currency"             : info.currency_profit,
            "contract_value_manual_override"  : None,
            "session_end_gmt"                 : "23:55",
            "eod_gap_minutes"                 : 30,
            "reference_capital"               : 10000,
            "risk_percent"                    : 1.0,
            "risk_reward"                     : 2.0,
            "lot_tolerance_ratio"             : 1.5,
            "symbol_mt5_name"                 : SYMBOL,
            "asset_name"                      : "NASDAQ (US100)",
        }
    }

    out_path = os.path.join(
        OUTPUT_DIR,
        f"params_{SYMBOL}_{SERVER.split('-')[0]}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    )
    with open(out_path, 'w', encoding='utf-8') as f:
        json.dump(params, f, indent=2, ensure_ascii=False)

    log.info(f"\n{'─'*58}")
    log.info(f" ✓ Fichier JSON exporté :")
    log.info(f"   {out_path}")
    log.info(f" ✓ Fichier log :")
    log.info(f"   {LOG_PATH}")
    log.info(f"{'─'*58}")

    log.info("\n PROCHAINES ÉTAPES")
    if contract_value is None:
        log.warning("   ⚠ CONTRACT_VALUE non calculable — trade test nécessaire")
    log.info("   1. Transmettre le fichier JSON pour validation des paramètres")
    log.info("   2. Renseigner commission_per_lot_roundtrip dans le JSON")
    log.info("   3. Lancer le script d'analyse exploratoire (script 1)")

    mt5.shutdown()
    log.info("\n✓ Connexion MT5 fermée proprement")

except Exception:
    # Capture toute erreur inattendue et l'écrit dans le log
    log.error("\n" + "="*58)
    log.error(" ERREUR INATTENDUE — détail ci-dessous")
    log.error("="*58)
    log.error(traceback.format_exc())
    log.error(f"\n Le fichier log complet est disponible ici :")
    log.error(f"   {LOG_PATH}")
    try:
        mt5.shutdown()
    except Exception:
        pass

finally:
    pause()

