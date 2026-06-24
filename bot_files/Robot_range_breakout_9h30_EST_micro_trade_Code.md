# -*- coding: utf-8 -*-
"""
================================================================================
MICRO-TRADE TEST — VALEUR D'UN POINT NAS100
================================================================================
Ce script ouvre automatiquement un micro-lot sur NAS100, attend un mouvement
de prix suffisant, ferme le trade, et calcule la valeur exacte d'un point.

UTILISATION
-----------
1. MT5 doit etre lance et connecte au compte Demo
2. Lancez : python micro_trade_test.py
3. Le script s occupe de tout
4. Resultats affiches en console ET dans micro_trade_test.log

SECURITE
--------
- Verifie que c est un compte DEMO avant tout trade
- Volume minimal (0.01 lot)
- Cloture automatique dans tous les cas
- Cloture d urgence en cas d erreur
================================================================================
"""

# ==============================================================================
# ETAPE 0 : pause() definie EN PREMIER — avant tout import
# Garantit que la console reste ouverte quoi qu il arrive
# ==============================================================================
import sys
import os

def pause():
    print("\n" + "="*60)
    print(" Appuyez sur Entree pour fermer...")
    print("="*60)
    try:
        input()
    except Exception:
        pass

# Redefinir sys.excepthook pour capturer TOUTE erreur non geree
def global_exception_handler(exc_type, exc_value, exc_tb):
    import traceback
    msg = "".join(traceback.format_exception(exc_type, exc_value, exc_tb))
    print("\n[ERREUR FATALE NON CAPTUREE]")
    print(msg)
    # Tenter d ecrire dans un fichier de secours
    try:
        script_dir = os.path.dirname(os.path.abspath(__file__))
        with open(os.path.join(script_dir, "micro_trade_test_FATAL.log"), "w") as f:
            f.write(msg)
        print(f"Log d urgence ecrit dans : {script_dir}\\micro_trade_test_FATAL.log")
    except Exception as e2:
        print(f"Impossible d ecrire le log d urgence : {e2}")
    pause()

sys.excepthook = global_exception_handler

# ==============================================================================
# ETAPE 1 : initialisation du dossier et du logger
# ==============================================================================

try:
    script_dir = os.path.dirname(os.path.abspath(__file__))
except Exception:
    script_dir = os.getcwd()

LOG_PATH = os.path.join(script_dir, "micro_trade_test.log")

print(f"Dossier de travail : {script_dir}")
print(f"Fichier log        : {LOG_PATH}")
print("Initialisation du logger...")

try:
    import logging

    logging.basicConfig(
        level    = logging.DEBUG,
        format   = "%(asctime)s  %(levelname)-8s  %(message)s",
        datefmt  = "%Y-%m-%d %H:%M:%S",
        handlers = [
            logging.FileHandler(LOG_PATH, encoding="utf-8", mode="w"),
            logging.StreamHandler(sys.stdout),
        ]
    )
    log = logging.getLogger()
    log.info("Logger initialise avec succes")
    log.info(f"Dossier : {script_dir}")

except Exception as e:
    print(f"[ERREUR] Impossible d initialiser le logger : {e}")
    pause()
    sys.exit(1)

# ==============================================================================
# ETAPE 2 : imports standards
# ==============================================================================

try:
    import json
    import time
    import traceback
    from datetime import datetime
    log.info("Imports standards OK")
except Exception as e:
    log.error(f"Erreur import standards : {e}")
    pause(); sys.exit(1)

# ==============================================================================
# ETAPE 3 : import MetaTrader5
# ==============================================================================

try:
    import MetaTrader5 as mt5
    log.info("MetaTrader5 importe avec succes")
except ImportError as e:
    log.error(f"MetaTrader5 non installe : {e}")
    log.error("Solution : pip install MetaTrader5")
    pause(); sys.exit(1)
except Exception as e:
    log.error(f"Erreur inattendue lors de l import MT5 : {e}")
    log.error(traceback.format_exc())
    pause(); sys.exit(1)

# ==============================================================================
# CONFIG
# ==============================================================================

LOGIN             = 5007258
PASSWORD          = "zL3!gvG8Ol"
SERVER            = "RaiseGlobal-Live"
SYMBOL            = "NAS100"
LOT_SIZE          = 0.01     # Volume minimal
MIN_MOVE_POINTS   = 2.0      # Mouvement minimal avant mesure
MAX_WAIT_SECONDS  = 120      # Timeout securite
POLL_INTERVAL     = 0.5      # Intervalle verification prix (secondes)

# ==============================================================================
# CORPS PRINCIPAL
# ==============================================================================

trade_ticket = None

try:
    log.info("="*60)
    log.info(f" MICRO-TRADE TEST — {SYMBOL}")
    log.info(f" Serveur : {SERVER}  |  Login : {LOGIN}")
    log.info("="*60)

    # ── Connexion MT5 ────────────────────────────────────────────────
    log.info("\nConnexion a MT5...")
    init_ok = mt5.initialize(login=LOGIN, password=PASSWORD, server=SERVER)
    log.info(f"mt5.initialize retourne : {init_ok}")

    if not init_ok:
        err = mt5.last_error()
        log.error(f"Connexion MT5 echouee : {err}")
        log.error("Verifications :")
        log.error("  1. MT5 est-il lance ?")
        log.error("  2. Identifiants corrects ?")
        log.error("  3. Connexion internet active ?")
        pause(); sys.exit(1)
    log.info("Connexion MT5 etablie")

    # ── Verification compte DEMO ─────────────────────────────────────
    log.info("\nVerification du type de compte...")
    account = mt5.account_info()
    if account is None:
        log.error(f"Impossible de recuperer les infos compte : {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)

    log.info(f"  Compte  : {account.login} — {account.name}")
    log.info(f"  Type    : {account.trade_mode} (0=DEMO, 1=REEL)")
    log.info(f"  Balance : {account.balance:.2f} {account.currency}")
    log.info(f"  Serveur : {account.server}")

    if account.trade_mode != 0:
        log.error("COMPTE REEL DETECTE — arret par securite.")
        log.error("Ce script ne tourne que sur compte DEMO.")
        mt5.shutdown(); pause(); sys.exit(1)
    log.info("Compte DEMO confirme")

    # ── Activation symbole ───────────────────────────────────────────
    log.info(f"\nActivation du symbole {SYMBOL}...")
    if not mt5.symbol_select(SYMBOL, True):
        log.error(f"Symbole '{SYMBOL}' introuvable : {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)

    info = mt5.symbol_info(SYMBOL)
    if info is None:
        log.error(f"symbol_info None : {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)
    log.info(f"Symbole active : {info.name} — {info.description}")
    log.info(f"  digits={info.digits} point={info.point} "
             f"tick_size={info.trade_tick_size} "
             f"tick_value={info.trade_tick_value:.6f}")

    # ── Prix a l ouverture ───────────────────────────────────────────
    tick_open = mt5.symbol_info_tick(SYMBOL)
    if tick_open is None:
        log.error(f"Tick introuvable — marche ferme ? {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)

    ask_open    = tick_open.ask
    bid_open    = tick_open.bid
    spread_open = round(ask_open - bid_open, info.digits)
    log.info(f"\nPrix a l ouverture : Ask={ask_open:.2f} Bid={bid_open:.2f} "
             f"Spread={spread_open:.2f}")

    # ── Verification trading autorise ────────────────────────────────
    term_info = mt5.terminal_info()
    if term_info and not term_info.trade_allowed:
        log.error("Trading non autorise dans MT5.")
        log.error("  Dans MT5 : Outils > Options > Expert Advisors")
        log.error("  Cochez 'Autoriser le trading automatise'")
        mt5.shutdown(); pause(); sys.exit(1)
    log.info(f"Trading autorise : {term_info.trade_allowed if term_info else 'inconnu'}")

    # ── Ouverture du trade ───────────────────────────────────────────
    log.info(f"\nOuverture BUY {LOT_SIZE} lot {SYMBOL}...")

    request = {
        "action"      : mt5.TRADE_ACTION_DEAL,
        "symbol"      : SYMBOL,
        "volume"      : LOT_SIZE,
        "type"        : mt5.ORDER_TYPE_BUY,
        "price"       : ask_open,
        "deviation"   : 20,
        "magic"       : 999999,
        "comment"     : "micro_test",
        "type_time"   : mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    log.info(f"Requete ordre : {request}")

    result = mt5.order_send(request)
    log.info(f"Reponse ordre : {result}")

    if result is None:
        log.error(f"order_send retourne None : {mt5.last_error()}")
        mt5.shutdown(); pause(); sys.exit(1)

    if result.retcode != mt5.TRADE_RETCODE_DONE:
        log.error(f"Ordre refuse — retcode={result.retcode} : {result.comment}")
        log.error("Codes courants :")
        log.error("  10018 = marche ferme")
        log.error("  10019 = pas assez de fonds")
        log.error("  10014 = volume invalide")
        mt5.shutdown(); pause(); sys.exit(1)

    trade_ticket = result.order
    price_open   = result.price
    log.info(f"Trade ouvert — ticket=#{trade_ticket} prix={price_open:.2f}")

    # ── Attente mouvement prix ───────────────────────────────────────
    log.info(f"\nAttente mouvement >= {MIN_MOVE_POINTS} pts "
             f"(timeout {MAX_WAIT_SECONDS}s)...")

    t_start       = time.time()
    price_measure = None
    pnl_measure   = None
    move_detected = False
    last_log_t    = 0

    while time.time() - t_start < MAX_WAIT_SECONDS:
        time.sleep(POLL_INTERVAL)
        tick_now = mt5.symbol_info_tick(SYMBOL)
        if tick_now is None:
            continue

        bid_now = tick_now.bid
        move    = abs(bid_now - bid_open)
        elapsed = time.time() - t_start

        # Log toutes les 10 secondes
        if elapsed - last_log_t >= 10:
            positions = mt5.positions_get(ticket=trade_ticket)
            pnl_now   = positions[0].profit if positions else 0
            log.info(f"  t={elapsed:4.0f}s Bid={bid_now:.2f} "
                     f"move={move:.2f}pts P&L={pnl_now:+.4f}$")
            last_log_t = elapsed

        if move >= MIN_MOVE_POINTS:
            positions = mt5.positions_get(ticket=trade_ticket)
            if positions:
                pnl_measure   = positions[0].profit
                price_measure = bid_now
                move_detected = True
                log.info(f"\nMouvement detecte !")
                log.info(f"  Bid ouverture : {bid_open:.2f}")
                log.info(f"  Bid mesure    : {price_measure:.2f}")
                log.info(f"  Mouvement     : {price_measure - bid_open:+.2f} pts")
                log.info(f"  P&L mesure    : {pnl_measure:+.4f} $")
                break

    if not move_detected:
        log.warning(f"Timeout {MAX_WAIT_SECONDS}s — mesure au dernier prix disponible")
        tick_now   = mt5.symbol_info_tick(SYMBOL)
        positions  = mt5.positions_get(ticket=trade_ticket)
        if tick_now and positions:
            price_measure = tick_now.bid
            pnl_measure   = positions[0].profit
            log.warning(f"  Bid={price_measure:.2f} "
                        f"move={price_measure-bid_open:+.2f} "
                        f"P&L={pnl_measure:+.4f}$")

    # ── Cloture ──────────────────────────────────────────────────────
    log.info(f"\nCloture du trade #{trade_ticket}...")
    tick_close = mt5.symbol_info_tick(SYMBOL)
    close_req  = {
        "action"      : mt5.TRADE_ACTION_DEAL,
        "symbol"      : SYMBOL,
        "volume"      : LOT_SIZE,
        "type"        : mt5.ORDER_TYPE_SELL,
        "position"    : trade_ticket,
        "price"       : tick_close.bid,
        "deviation"   : 20,
        "magic"       : 999999,
        "comment"     : "micro_test_close",
        "type_time"   : mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }
    close_result = mt5.order_send(close_req)
    log.info(f"Reponse cloture : {close_result}")

    if close_result and close_result.retcode == mt5.TRADE_RETCODE_DONE:
        price_close  = close_result.price
        trade_ticket = None
        log.info(f"Trade cloture au prix {price_close:.2f}")
    else:
        retcode = close_result.retcode if close_result else "None"
        log.error(f"Cloture echouee retcode={retcode}")
        log.error("FERMEZ MANUELLEMENT LA POSITION DANS MT5 !")

    # ── P&L final depuis historique ──────────────────────────────────
    time.sleep(1)
    history      = mt5.history_deals_get(datetime(2000, 1, 1), datetime.now())
    pnl_final    = None
    commissions  = 0.0
    if history:
        our_deals = [d for d in history if d.magic == 999999]
        if our_deals:
            pnl_final   = sum(d.profit     for d in our_deals)
            commissions = sum(d.commission for d in our_deals)
            log.info(f"\nHistorique deals (magic=999999) :")
            for d in our_deals:
                log.info(f"  #{d.ticket} {d.symbol} vol={d.volume} "
                         f"price={d.price:.2f} profit={d.profit:+.4f}$ "
                         f"commission={d.commission:+.4f}$")
            log.info(f"  Total P&L    : {pnl_final:+.4f}$")
            log.info(f"  Commissions  : {commissions:+.4f}$")

    # ── Calcul CONTRACT_VALUE ─────────────────────────────────────────
    log.info(f"\n{'='*60}")
    log.info(" RESULTATS")
    log.info(f"{'='*60}")

    cv_measured = None
    if price_measure is not None and pnl_measure is not None:
        move_pts = price_measure - bid_open
        if abs(move_pts) > 0.001:
            cv_measured = pnl_measure / (move_pts * LOT_SIZE)
            log.info(f"CONTRACT_VALUE mesuree  : {cv_measured:.4f} $/point/lot")

    cv_theoretical = (info.trade_tick_value / info.volume_min /
                      info.trade_tick_size * info.point)
    log.info(f"CONTRACT_VALUE theorique: {cv_theoretical:.4f} $/point/lot")

    if cv_measured and cv_theoretical > 0:
        ratio = cv_measured / cv_theoretical
        log.info(f"Ratio mesure/theorique  : {ratio:.4f}")
        if 0.9 <= ratio <= 1.1:
            log.info("Coherence confirmee (ecart < 10%)")
        else:
            log.warning("Ecart > 10% — verifier la formule")

    if commissions != 0:
        cv_per_lot = abs(commissions) / LOT_SIZE
        log.info(f"Commissions aller+retour: {cv_per_lot:.4f} $/lot")
    else:
        log.info("Commissions : 0 (integrees dans le spread)")

    # ── Export JSON ───────────────────────────────────────────────────
    out_path = os.path.join(
        script_dir,
        f"micro_trade_test_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
    )
    results = {
        "meta": {
            "generated_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "broker": SERVER, "symbol": SYMBOL, "lot_size": LOT_SIZE,
        },
        "trade": {
            "bid_open": bid_open, "ask_open": ask_open,
            "spread_open_points": spread_open,
            "price_executed": price_open,
            "bid_at_measure": price_measure,
            "move_points": round(price_measure - bid_open, 2) if price_measure else None,
            "pnl_at_measure_usd": pnl_measure,
            "pnl_final_usd": pnl_final,
            "commission_total_usd": commissions,
        },
        "results": {
            "contract_value_measured": round(cv_measured, 4) if cv_measured else None,
            "contract_value_theoretical": round(cv_theoretical, 4),
            "commission_per_lot_roundtrip": round(abs(commissions) / LOT_SIZE, 4),
            "currency": info.currency_profit,
        }
    }
    with open(out_path, "w", encoding="utf-8") as f:
        json.dump(results, f, indent=2, ensure_ascii=False)

    log.info(f"\nFichier JSON : {out_path}")
    log.info(f"Fichier log  : {LOG_PATH}")

    mt5.shutdown()
    log.info("Connexion MT5 fermee")

except Exception:
    log.error("\n" + "="*60)
    log.error(" ERREUR INATTENDUE")
    log.error("="*60)
    log.error(traceback.format_exc())

    # Cloture d urgence
    if trade_ticket is not None:
        log.error(f"\nTENTATIVE CLOTURE D URGENCE ticket #{trade_ticket}...")
        try:
            tick_urg = mt5.symbol_info_tick(SYMBOL)
            if tick_urg:
                urg = {
                    "action": mt5.TRADE_ACTION_DEAL, "symbol": SYMBOL,
                    "volume": LOT_SIZE, "type": mt5.ORDER_TYPE_SELL,
                    "position": trade_ticket, "price": tick_urg.bid,
                    "deviation": 50, "magic": 999999,
                    "comment": "emergency", "type_time": mt5.ORDER_TIME_GTC,
                    "type_filling": mt5.ORDER_FILLING_IOC,
                }
                r = mt5.order_send(urg)
                if r and r.retcode == mt5.TRADE_RETCODE_DONE:
                    log.info("Cloture d urgence reussie")
                else:
                    log.error("CLOTURE D URGENCE ECHOUEE — FERMEZ MANUELLEMENT !")
        except Exception as e2:
            log.error(f"Erreur cloture urgence : {e2}")
    try:
        mt5.shutdown()
    except Exception:
        pass

finally:
    pause()

