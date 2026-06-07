
//+------------------------------------------------------------------+
//| EA_FX_Universal.mq5  EA Desk Quant v2.9                         |
//| v2.5 : restauration BESet, fermeture vendredi, filtre magic,    |
//|        plafond lot CSV, persistance InitialBalance               |
//| v2.6 : solidite signal, gap lundi, ATR breaker, liquidite,       |
//|        cooldown echec ordre, slippage dans journal               |
//| v2.7 : MAGIC_NUMBER en premier parametre, lecture InitBalance    |
//| v2.8 : lecture RiskEUR depuis CSV, filtre cout reel par trade,   |
//|        correction BE = open_price+spread (PnL neutre garanti)   |
//| v2.8.1 : Nikkei exclu, Gold auto-veille propfirm, EU50 session  |
//|          etendue, seuils historique assouplis, MAGIC=20260506    |
//| v2.9 : [1] Sessions DAX40/EU50 corrigees (07h-15h UTC)          |
//|            + fermeture vendredi EU a 14h30 UTC                  |
//|        [2] Flag g_FridayCloseStarted — bloque toute rouverture  |
//|            apres fermeture vendredi (corrige boucle 16h)         |
//|        [3] CloseAllPositions resilient — log [Market closed]     |
//|            sans boucle infinie si marche ferme                   |
//|        [4] Plafond SL en % capital (MAX_RISK_PCT_PER_TRADE=1%)  |
//|            corrige risque Silver/Gold disproportionne            |
//|        [5] Trailing : seuil min TRAIL_MIN_MOVE avant PositionModify |
//|            reduit les 30+ modify Silver en 2h                   |
//|        [6] EMA_D1 reduite 50->20 sur S&P500, NAS100, DAX40      |
//|            filtre tendance journalier plus reactif sur indices   |
//|        [7] Monitoring Python : filtre alertes Gold VEILLE        |
//+------------------------------------------------------------------+
#property strict
#property version   "2.9"
#property description "EA Desk Quant Universel v2.9"

#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>

CTrade         trade;
CPositionInfo  posInfo;

//--- Identification
input int      MAGIC_NUMBER        = 20260529; // Numero magique unique — identifier visuellement le robot dans MT5

//--- Fichiers
input string   CSV_FILENAME        = "portfolio_weights_RaiseFX_demo_5007258.csv";
input string   BROKER_NAME         = "RaiseFX";
input string   ACCOUNT_TYPE        = "demo";

//--- Mode de trading
// "propfirm"      : regles strictes prop firms
// "compte_propre" : regles assouplies
input string   TRADING_MODE        = "propfirm";
input bool     COMPAT_PROPFIRM     = true;
input bool     COMPAT_COMPTE_PROPRE= true;

//--- Risque
input double   MAX_DAILY_LOSS_PCT  = 0.045;
input double   MAX_TOTAL_DD_PCT    = 0.090;
input double   MAX_DAILY_LOSS_CP   = 0.080;
input double   MAX_TOTAL_DD_CP     = 0.200;
input int      MAX_SL_PER_DAY      = 3;
input int      COOLDOWN_AFTER_SL   = 60;

//--- Timing
input int      FRIDAY_CLOSE_HOUR   = 20;
input int      FRIDAY_CLOSE_MIN    = 30;  // Arret nouvelles entrees vendredi (Forex/US)
input int      FRIDAY_CLOSE_POS_MIN= 15;  // Fermeture positions vendredi (avant CLOSE_MIN)
// [v2.9] Fermeture anticipee vendredi pour indices europeens (DAX40, EU50)
// Ces marches ferment ~15h30 UTC — on ferme positions a 14h30 et entrees a 14h00
input int      FRIDAY_CLOSE_EU_HOUR    = 14; // Heure fermeture positions EU vendredi
input int      FRIDAY_CLOSE_EU_MIN     = 30; // Minute fermeture positions EU vendredi
input int      FRIDAY_STOP_EU_HOUR     = 14; // Heure arret entrees EU vendredi
input int      FRIDAY_STOP_EU_MIN      =  0; // Minute arret entrees EU vendredi
input int      MONDAY_OPEN_HOUR    = 8;
input int      NEWS_FILTER_BEFORE  = 30;
input int      NEWS_FILTER_AFTER   = 30;

//--- [v2.9] Protection risque par trade
// Plafond SL : si risque reel > MAX_RISK_PCT_PER_TRADE x capital -> SL reduit
// Evite les pertes disproportionnees sur Silver/Gold (ex: -76 EUR sur 0.01 lot)
input double   MAX_RISK_PCT_PER_TRADE = 0.01; // 1% du capital max par trade

//--- [v2.9] Trailing minimum
// PositionModify n'est envoye que si le nouveau SL se deplace d'au moins
// TRAIL_MIN_MOVE x ATR — evite les 30+ modify Silver en 2h
input double   TRAIL_MIN_MOVE = 0.10; // fraction d'ATR minimum avant modify trailing

//--- Protection
input int      MAX_RETRY_ORDERS    = 3;
input int      MAX_SLIPPAGE        = 10;
input int      MAX_CSV_AGE_NORMAL  = 300;
input double   MAX_LOT_RISK        = 0.02;     // [v2.5] Plafond lot : max 2% capital
input int      COOLDOWN_FAILED_ORDER = 5;      // [v2.6] Cooldown apres echec ordre (min)
input int      MIN_TICK_VOLUME     = 10;       // [v2.6] Volume tick mini sur 3 bougies
input double   GAP_MAX_ATR         = 2.0;      // [v2.6] Gap max ouverture lundi (x ATR)
input bool     ATR_PERIOD_20_BARS  = true;     // [v2.6] Periode ATR moyen = 20 bougies
input int      ATR_PERIOD          = 14;

//--- USDJPY — Backtest v2 : score=67 propfirm | SL=2.5 TP=2.5 Fast=20 Slow=50
input double   USDJPY_SL      = 2.5;
input double   USDJPY_BE      = 1.0;
input double   USDJPY_TP      = 2.5;
input double   USDJPY_TRAIL   = 1.0;
input int      USDJPY_FAST    = 20;
input int      USDJPY_SLOW    = 50;
input int      USDJPY_H4      = 50;
input int      USDJPY_D1      = 50;
input double   USDJPY_SPREAD  = 20;
input double   USDJPY_LOT     = 0.01;
input bool     USDJPY_SESSION = false;
input int      USDJPY_START   = 0;
input int      USDJPY_END     = 23;
input bool     USDJPY_SIGNAL_ACTIF  = false; // [v2.6] Filtre solidite signal (desactive)
input double   USDJPY_SIGNAL_MAX    = 2.5;   // Distance prix/SMA_lente max (x ATR) — provisoire
input bool     USDJPY_ATR_ACTIF     = false; // [v2.6] Circuit breaker ATR (desactive)
input double   USDJPY_ATR_SPIKE     = 2.5;   // Seuil spike ATR (x ATR moyen 20p) — provisoire

//--- SP500 — Backtest v2 : score=71 propfirm | SL=2.5 TP=3.0 Fast=30 Slow=100
input double   SP500_SL      = 2.5;
input double   SP500_BE      = 1.0;
input double   SP500_TP      = 3.0;
input double   SP500_TRAIL   = 1.0;
input int      SP500_FAST    = 30;
input int      SP500_SLOW    = 100;
input int      SP500_H4      = 50;
input int      SP500_D1      = 20;  // [v2.9] Reduit 50->20 : filtre tendance J plus reactif
input double   SP500_SPREAD  = 50;
input double   SP500_LOT     = 0.01;
input bool     SP500_SESSION = true;
input int      SP500_START   = 13;
input int      SP500_END     = 21;
input bool     SP500_SIGNAL_ACTIF   = false;
input double   SP500_SIGNAL_MAX     = 3.0;   // Indices US : tendances longues frequentes
input bool     SP500_ATR_ACTIF      = false;
input double   SP500_ATR_SPIKE      = 2.0;   // Indices US : volatilite contenue

//--- NAS100 — Backtest v2 : score=71 propfirm | SL=2.5 TP=2.0 Fast=10 Slow=30
input double   NAS100_SL      = 2.5;
input double   NAS100_BE      = 1.0;
input double   NAS100_TP      = 2.0;
input double   NAS100_TRAIL   = 1.5;
input int      NAS100_FAST    = 10;
input int      NAS100_SLOW    = 30;
input int      NAS100_H4      = 50;
input int      NAS100_D1      = 20;  // [v2.9] Reduit 50->20 : filtre tendance J plus reactif
input double   NAS100_SPREAD  = 50;
input double   NAS100_LOT     = 0.01;
input bool     NAS100_SESSION = true;
input int      NAS100_START   = 13;
input int      NAS100_END     = 21;
input bool     NAS100_SIGNAL_ACTIF  = false;
input double   NAS100_SIGNAL_MAX    = 3.0;
input bool     NAS100_ATR_ACTIF     = false;
input double   NAS100_ATR_SPIKE     = 2.0;

//--- Gold — Backtest v2 : score=50 COMPTE PROPRE uniquement | SL=2.5 TP=3.0 Fast=10 Slow=30
input double   Gold_SL      = 2.5;
input double   Gold_BE      = 1.0;
input double   Gold_TP      = 3.0;
input double   Gold_TRAIL   = 1.5;
input int      Gold_FAST    = 10;
input int      Gold_SLOW    = 30;
input int      Gold_H4      = 50;
input int      Gold_D1      = 50;
input double   Gold_SPREAD  = 35;
input double   Gold_LOT     = 0.01;
input bool     Gold_SESSION = true;
input int      Gold_START   = 12;
input int      Gold_END     = 20;
input bool     Gold_SIGNAL_ACTIF    = false;
input double   Gold_SIGNAL_MAX      = 3.5;   // Or : tendances tres longues possibles
input bool     Gold_ATR_ACTIF       = false;
input double   Gold_ATR_SPIKE       = 3.0;   // Safe haven : spikes geopolitiques

//--- DJ30 — Backtest v2 : score=64 propfirm | SL=2.5 TP=2.5 Fast=10 Slow=100
//    NOTE: correle S&P500 — deployer sur un seul des deux en propfirm
input double   DJ30_SL      = 2.5;
input double   DJ30_BE      = 1.0;
input double   DJ30_TP      = 2.5;
input double   DJ30_TRAIL   = 1.0;
input int      DJ30_FAST    = 10;
input int      DJ30_SLOW    = 100;
input int      DJ30_H4      = 50;
input int      DJ30_D1      = 50;
input double   DJ30_SPREAD  = 50;
input double   DJ30_LOT     = 0.01;
input bool     DJ30_SESSION = true;
input int      DJ30_START   = 13;
input int      DJ30_END     = 21;
input bool     DJ30_SIGNAL_ACTIF    = false;
input double   DJ30_SIGNAL_MAX      = 3.0;
input bool     DJ30_ATR_ACTIF       = false;
input double   DJ30_ATR_SPIKE       = 2.0;

//--- DAX40 — Backtest v2 : score=68 propfirm | SL=2.5 TP=2.0 Fast=10 Slow=30
input double   DAX40_SL      = 2.5;
input double   DAX40_BE      = 1.0;
input double   DAX40_TP      = 2.0;
input double   DAX40_TRAIL   = 1.0;
input int      DAX40_FAST    = 10;
input int      DAX40_SLOW    = 30;
input int      DAX40_H4      = 50;
input int      DAX40_D1      = 20;  // [v2.9] Reduit 50->20 : filtre tendance J plus reactif
input double   DAX40_SPREAD  = 60;
input double   DAX40_LOT     = 0.01;
input bool     DAX40_SESSION = true;
input int      DAX40_START   = 7;
input int      DAX40_END     = 15;  // [v2.9] Fermeture a 15h UTC (CFD RaiseFX ferme ~15h30)
input bool     DAX40_SIGNAL_ACTIF   = false;
input double   DAX40_SIGNAL_MAX     = 2.5;   // Indices EU : tendances plus courtes
input bool     DAX40_ATR_ACTIF      = false;
input double   DAX40_ATR_SPIKE      = 2.5;   // Gaps ouverture frequents

//--- FTSE100 — Backtest v2 : score=33 EXCLU (aucune combo propfirm ou CP)
//    Conserver en VEILLE : mettre COMPAT_PROPFIRM=false COMPAT_COMPTE_PROPRE=false
input double   FTSE100_SL      = 1.5;
input double   FTSE100_BE      = 1.0;
input double   FTSE100_TP      = 2.5;
input double   FTSE100_TRAIL   = 1.0;
input int      FTSE100_FAST    = 20;
input int      FTSE100_SLOW    = 50;
input int      FTSE100_H4      = 50;
input int      FTSE100_D1      = 50;
input double   FTSE100_SPREAD  = 60;
input double   FTSE100_LOT     = 0.01;
input bool     FTSE100_SESSION = true;
input int      FTSE100_START   = 8;
input int      FTSE100_END     = 16;
input bool     FTSE100_SIGNAL_ACTIF = false;
input double   FTSE100_SIGNAL_MAX   = 2.5;
input bool     FTSE100_ATR_ACTIF    = false;
input double   FTSE100_ATR_SPIKE    = 2.5;

//--- EU50 — Backtest v2 : score=57 propfirm | SL=2.0 TP=3.0 Fast=10 Slow=50
input double   EU50_SL      = 2.0;
input double   EU50_BE      = 1.0;
input double   EU50_TP      = 3.0;
input double   EU50_TRAIL   = 1.0;
input int      EU50_FAST    = 10;
input int      EU50_SLOW    = 50;
input int      EU50_H4      = 50;
input int      EU50_D1      = 50;
input double   EU50_SPREAD  = 60;
input double   EU50_LOT     = 0.01;
input bool     EU50_SESSION = true;
input int      EU50_START   = 0;
input int      EU50_END     = 15;  // [v2.9] Etendu debut + fermeture a 15h UTC (CFD ~15h30)
input bool     EU50_SIGNAL_ACTIF    = false;
input double   EU50_SIGNAL_MAX      = 2.5;
input bool     EU50_ATR_ACTIF       = false;
input double   EU50_ATR_SPIKE       = 2.5;

//--- Nikkei — Backtest v2 : score=59 propfirm | SL=2.5 TP=1.5 Fast=30 Slow=100
input double   Nikkei_SL      = 2.5;
input double   Nikkei_BE      = 1.0;
input double   Nikkei_TP      = 1.5;
input double   Nikkei_TRAIL   = 1.0;
input int      Nikkei_FAST    = 30;
input int      Nikkei_SLOW    = 100;
input int      Nikkei_H4      = 50;
input int      Nikkei_D1      = 50;
input double   Nikkei_SPREAD  = 80;
input double   Nikkei_LOT     = 0.01;
input bool     Nikkei_SESSION = true;
input int      Nikkei_START   = 23;
input int      Nikkei_END     = 7;
input bool     Nikkei_SIGNAL_ACTIF  = false;
input double   Nikkei_SIGNAL_MAX    = 3.5;   // BoJ : grands mouvements structurels
input bool     Nikkei_ATR_ACTIF     = false;
input double   Nikkei_ATR_SPIKE     = 3.0;   // Interventions BoJ violentes

//--- GBPJPY — Backtest v2 : score=55 propfirm | SL=2.5 TP=3.0 Fast=10 Slow=50
input double   GBPJPY_SL      = 2.5;
input double   GBPJPY_BE      = 1.0;
input double   GBPJPY_TP      = 3.0;
input double   GBPJPY_TRAIL   = 1.5;
input int      GBPJPY_FAST    = 10;
input int      GBPJPY_SLOW    = 50;
input int      GBPJPY_H4      = 50;
input int      GBPJPY_D1      = 50;
input double   GBPJPY_SPREAD  = 40;
input double   GBPJPY_LOT     = 0.01;
input bool     GBPJPY_SESSION = true;
input int      GBPJPY_START   = 7;
input int      GBPJPY_END     = 17;
input bool     GBPJPY_SIGNAL_ACTIF  = false;
input double   GBPJPY_SIGNAL_MAX    = 3.0;
input bool     GBPJPY_ATR_ACTIF     = false;
input double   GBPJPY_ATR_SPIKE     = 3.0;   // The Beast : spikes frequents

//--- EURJPY — Backtest v2 : score=36 EXCLU
input double   EURJPY_SL      = 2.0;
input double   EURJPY_BE      = 1.0;
input double   EURJPY_TP      = 2.5;
input double   EURJPY_TRAIL   = 1.0;
input int      EURJPY_FAST    = 20;
input int      EURJPY_SLOW    = 50;
input int      EURJPY_H4      = 50;
input int      EURJPY_D1      = 50;
input double   EURJPY_SPREAD  = 30;
input double   EURJPY_LOT     = 0.01;
input bool     EURJPY_SESSION = false;
input int      EURJPY_START   = 0;
input int      EURJPY_END     = 23;
input bool     EURJPY_SIGNAL_ACTIF  = false;
input double   EURJPY_SIGNAL_MAX    = 2.5;
input bool     EURJPY_ATR_ACTIF     = false;
input double   EURJPY_ATR_SPIKE     = 2.5;

//--- AUDJPY — Backtest v2 : score=39 EXCLU
input double   AUDJPY_SL      = 2.0;
input double   AUDJPY_BE      = 1.0;
input double   AUDJPY_TP      = 2.5;
input double   AUDJPY_TRAIL   = 1.0;
input int      AUDJPY_FAST    = 20;
input int      AUDJPY_SLOW    = 50;
input int      AUDJPY_H4      = 50;
input int      AUDJPY_D1      = 50;
input double   AUDJPY_SPREAD  = 30;
input double   AUDJPY_LOT     = 0.01;
input bool     AUDJPY_SESSION = false;
input int      AUDJPY_START   = 0;
input int      AUDJPY_END     = 23;
input bool     AUDJPY_SIGNAL_ACTIF  = false;
input double   AUDJPY_SIGNAL_MAX    = 2.5;
input bool     AUDJPY_ATR_ACTIF     = false;
input double   AUDJPY_ATR_SPIKE     = 2.5;

//--- GBPUSD — Backtest v2 : score=33 EXCLU
input double   GBPUSD_SL      = 1.5;
input double   GBPUSD_BE      = 1.0;
input double   GBPUSD_TP      = 2.5;
input double   GBPUSD_TRAIL   = 1.0;
input int      GBPUSD_FAST    = 20;
input int      GBPUSD_SLOW    = 50;
input int      GBPUSD_H4      = 50;
input int      GBPUSD_D1      = 50;
input double   GBPUSD_SPREAD  = 20;
input double   GBPUSD_LOT     = 0.01;
input bool     GBPUSD_SESSION = false;
input int      GBPUSD_START   = 0;
input int      GBPUSD_END     = 23;
input bool     GBPUSD_SIGNAL_ACTIF  = false;
input double   GBPUSD_SIGNAL_MAX    = 2.5;
input bool     GBPUSD_ATR_ACTIF     = false;
input double   GBPUSD_ATR_SPIKE     = 2.5;

//--- AUDUSD — Backtest v2 : score=29 EXCLU
input double   AUDUSD_SL      = 1.5;
input double   AUDUSD_BE      = 1.0;
input double   AUDUSD_TP      = 2.0;
input double   AUDUSD_TRAIL   = 1.0;
input int      AUDUSD_FAST    = 20;
input int      AUDUSD_SLOW    = 50;
input int      AUDUSD_H4      = 50;
input int      AUDUSD_D1      = 50;
input double   AUDUSD_SPREAD  = 20;
input double   AUDUSD_LOT     = 0.01;
input bool     AUDUSD_SESSION = false;
input int      AUDUSD_START   = 0;
input int      AUDUSD_END     = 23;
input bool     AUDUSD_SIGNAL_ACTIF  = false;
input double   AUDUSD_SIGNAL_MAX    = 2.5;
input bool     AUDUSD_ATR_ACTIF     = false;
input double   AUDUSD_ATR_SPIKE     = 2.5;

//--- Silver — Backtest v2 : score=61 propfirm+CP | SL=2.5 TP=2.5 Fast=30 Slow=50
input double   Silver_SL      = 2.5;
input double   Silver_BE      = 1.0;
input double   Silver_TP      = 2.5;
input double   Silver_TRAIL   = 1.5;
input int      Silver_FAST    = 30;
input int      Silver_SLOW    = 50;
input int      Silver_H4      = 50;
input int      Silver_D1      = 50;
input double   Silver_SPREAD  = 80;
input double   Silver_LOT     = 0.01;
input bool     Silver_SESSION = true;
input int      Silver_START   = 12;
input int      Silver_END     = 20;
input bool     Silver_SIGNAL_ACTIF  = false;
input double   Silver_SIGNAL_MAX    = 4.0;   // Beta Gold + marche peu liquide
input bool     Silver_ATR_ACTIF     = false;
input double   Silver_ATR_SPIKE     = 3.5;   // Spikes extremes possibles

//--- BRENT — Backtest v2 : score=32 EXCLU (meme comportement que oilWTI)
input double   BRENT_SL      = 2.0;
input double   BRENT_BE      = 1.0;
input double   BRENT_TP      = 2.5;
input double   BRENT_TRAIL   = 1.0;
input int      BRENT_FAST    = 20;
input int      BRENT_SLOW    = 50;
input int      BRENT_H4      = 50;
input int      BRENT_D1      = 50;
input double   BRENT_SPREAD  = 70;
input double   BRENT_LOT     = 0.01;
input bool     BRENT_SESSION = true;
input int      BRENT_START   = 12;
input int      BRENT_END     = 20;
input bool     BRENT_SIGNAL_ACTIF   = false;
input double   BRENT_SIGNAL_MAX     = 3.0;
input bool     BRENT_ATR_ACTIF      = false;
input double   BRENT_ATR_SPIKE      = 3.0;   // OPEC decisions

//--- Bitcoin — Backtest v2 : score=32 EXCLU (SMA non adaptee aux cryptos)
input double   Bitcoin_SL      = 2.5;
input double   Bitcoin_BE      = 1.0;
input double   Bitcoin_TP      = 3.0;
input double   Bitcoin_TRAIL   = 2.0;
input int      Bitcoin_FAST    = 10;
input int      Bitcoin_SLOW    = 30;
input int      Bitcoin_H4      = 50;
input int      Bitcoin_D1      = 50;
input double   Bitcoin_SPREAD  = 150;
input double   Bitcoin_LOT     = 0.01;
input bool     Bitcoin_SESSION = false;
input int      Bitcoin_START   = 0;
input int      Bitcoin_END     = 23;
input bool     Bitcoin_SIGNAL_ACTIF = false;
input double   Bitcoin_SIGNAL_MAX   = 4.0;
input bool     Bitcoin_ATR_ACTIF    = false;
input double   Bitcoin_ATR_SPIKE    = 4.0;   // Volatilite structurellement elevee

//--- Ethereum — Backtest v2 : score=32 EXCLU
input double   Ethereum_SL      = 2.5;
input double   Ethereum_BE      = 1.0;
input double   Ethereum_TP      = 3.0;
input double   Ethereum_TRAIL   = 2.0;
input int      Ethereum_FAST    = 10;
input int      Ethereum_SLOW    = 30;
input int      Ethereum_H4      = 50;
input int      Ethereum_D1      = 50;
input double   Ethereum_SPREAD  = 200;
input double   Ethereum_LOT     = 0.01;
input bool     Ethereum_SESSION = false;
input int      Ethereum_START   = 0;
input int      Ethereum_END     = 23;
input bool     Ethereum_SIGNAL_ACTIF= false;
input double   Ethereum_SIGNAL_MAX  = 4.0;
input bool     Ethereum_ATR_ACTIF   = false;
input double   Ethereum_ATR_SPIKE   = 4.0;
//+------------------------------------------------------------------+
//| VARIABLES GLOBALES                                               |
//+------------------------------------------------------------------+
string   g_AccountID           = "";
string   g_Symbol              = "";
string   g_InitBalFile         = ""; // [v2.7] Chemin fichier InitBalance
bool     g_ModeActive          = true;
double   g_Lot                 = 0.01;
double   g_Weight              = 0.0;
double   g_RiskFactor          = 1.0;
double   g_RiskMult            = 1.0;
int      g_TradeAllowed_Global = 1;
int      g_TradeAllowed_Symbol = 1;
int      g_MarketOpen          = 1;
int      g_EnableTrading       = 1;
int      g_Restart             = 0;
double   g_DailyDD             = 0.0;
double   g_TotalDD             = 0.0;
datetime g_CSVTimestamp        = 0;
int      g_SL_today            = 0;
int      g_TP_today            = 0;
double   g_RiskEUR             = 0.0;   // [v2.8] Risque max EUR/trade (lu depuis CSV)
double   g_DayStartBalance     = 0.0;
double   g_InitialBalance      = 0.0;
string   g_TodayDate           = "";
int      g_SL_count_today      = 0;
int      g_TP_count_today      = 0;
datetime g_CooldownUntil       = 0;
bool     g_BESet               = false;
datetime g_FailedOrderCooldown = 0;   // [v2.6] Cooldown apres echec MAX_RETRY_ORDERS
datetime g_MondayGapChecked    = 0;   // [v2.6] Flag gap lundi deja verifie aujourd'hui
bool     g_FridayCloseStarted  = false; // [v2.9] Vendredi : fermeture declenchee -> bloquer rouvertures
int      g_Handle_ATR          = INVALID_HANDLE;
int      g_Handle_SMA_Fast     = INVALID_HANDLE;
int      g_Handle_SMA_Slow     = INVALID_HANDLE;
int      g_Handle_SMA_H4       = INVALID_HANDLE;
int      g_Handle_EMA_D1       = INVALID_HANDLE;

struct SymbolParams {
   double sl, be, tp, trail;
   int    smaFast, smaSlow, smaH4, emaD1;
   double maxSpread, defaultLot;
   bool   sessionFilter;
   int    sessionStart, sessionEnd;
   // v2.6 — filtres avancés
   bool   signalActif;   // Filtre solidite signal actif
   double signalMax;     // Distance max prix/SMA_lente (x ATR)
   bool   atrActif;      // Circuit breaker ATR actif
   double atrSpike;      // Seuil spike ATR (x ATR moyen 20p)
};

datetime ParseISO8601(string s) {
   if(StringLen(s)<19) return 0;
   string r=StringSubstr(s,0,19);
   StringReplace(r,"-","."); StringReplace(r,"T"," ");
   return StringToTime(r);
}

// [v2.9] Identifies les indices europeens soumis a fermeture anticipee vendredi
bool IsEUIndex() {
   return (g_Symbol=="DAX40" || g_Symbol=="EU50");
}

bool IsModeCompatible() {
   // [v2.8.1] Gold reserve au mode compte_propre (DD trop eleve pour propfirm)
   if(g_Symbol=="Gold" && TRADING_MODE=="propfirm") return false;
   if(TRADING_MODE=="propfirm")      return COMPAT_PROPFIRM;
   if(TRADING_MODE=="compte_propre") return COMPAT_COMPTE_PROPRE;
   return true;
}

double GetMaxDailyLoss() { return (TRADING_MODE=="propfirm")?MAX_DAILY_LOSS_PCT:MAX_DAILY_LOSS_CP; }
double GetMaxTotalDD()   { return (TRADING_MODE=="propfirm")?MAX_TOTAL_DD_PCT  :MAX_TOTAL_DD_CP;   }

SymbolParams GetSymbolParams() {
   SymbolParams p;
   if(g_Symbol=="USDJPY") {
      p.sl=USDJPY_SL; p.be=USDJPY_BE; p.tp=USDJPY_TP; p.trail=USDJPY_TRAIL;
      p.smaFast=USDJPY_FAST; p.smaSlow=USDJPY_SLOW; p.smaH4=USDJPY_H4; p.emaD1=USDJPY_D1;
      p.maxSpread=USDJPY_SPREAD; p.defaultLot=USDJPY_LOT;
      p.sessionFilter=USDJPY_SESSION; p.sessionStart=USDJPY_START; p.sessionEnd=USDJPY_END;
      p.signalActif=USDJPY_SIGNAL_ACTIF; p.signalMax=USDJPY_SIGNAL_MAX;
      p.atrActif=USDJPY_ATR_ACTIF; p.atrSpike=USDJPY_ATR_SPIKE;
   }
   else if(g_Symbol=="S&P500") {
      p.sl=SP500_SL; p.be=SP500_BE; p.tp=SP500_TP; p.trail=SP500_TRAIL;
      p.smaFast=SP500_FAST; p.smaSlow=SP500_SLOW; p.smaH4=SP500_H4; p.emaD1=SP500_D1;
      p.maxSpread=SP500_SPREAD; p.defaultLot=SP500_LOT;
      p.sessionFilter=SP500_SESSION; p.sessionStart=SP500_START; p.sessionEnd=SP500_END;
      p.signalActif=SP500_SIGNAL_ACTIF; p.signalMax=SP500_SIGNAL_MAX;
      p.atrActif=SP500_ATR_ACTIF; p.atrSpike=SP500_ATR_SPIKE;
   }
   else if(g_Symbol=="NAS100") {
      p.sl=NAS100_SL; p.be=NAS100_BE; p.tp=NAS100_TP; p.trail=NAS100_TRAIL;
      p.smaFast=NAS100_FAST; p.smaSlow=NAS100_SLOW; p.smaH4=NAS100_H4; p.emaD1=NAS100_D1;
      p.maxSpread=NAS100_SPREAD; p.defaultLot=NAS100_LOT;
      p.sessionFilter=NAS100_SESSION; p.sessionStart=NAS100_START; p.sessionEnd=NAS100_END;
      p.signalActif=NAS100_SIGNAL_ACTIF; p.signalMax=NAS100_SIGNAL_MAX;
      p.atrActif=NAS100_ATR_ACTIF; p.atrSpike=NAS100_ATR_SPIKE;
   }
   else if(g_Symbol=="Gold") {
      p.sl=Gold_SL; p.be=Gold_BE; p.tp=Gold_TP; p.trail=Gold_TRAIL;
      p.smaFast=Gold_FAST; p.smaSlow=Gold_SLOW; p.smaH4=Gold_H4; p.emaD1=Gold_D1;
      p.maxSpread=Gold_SPREAD; p.defaultLot=Gold_LOT;
      p.sessionFilter=Gold_SESSION; p.sessionStart=Gold_START; p.sessionEnd=Gold_END;
      p.signalActif=Gold_SIGNAL_ACTIF; p.signalMax=Gold_SIGNAL_MAX;
      p.atrActif=Gold_ATR_ACTIF; p.atrSpike=Gold_ATR_SPIKE;
   }
   else if(g_Symbol=="DJ30") {
      p.sl=DJ30_SL; p.be=DJ30_BE; p.tp=DJ30_TP; p.trail=DJ30_TRAIL;
      p.smaFast=DJ30_FAST; p.smaSlow=DJ30_SLOW; p.smaH4=DJ30_H4; p.emaD1=DJ30_D1;
      p.maxSpread=DJ30_SPREAD; p.defaultLot=DJ30_LOT;
      p.sessionFilter=DJ30_SESSION; p.sessionStart=DJ30_START; p.sessionEnd=DJ30_END;
      p.signalActif=DJ30_SIGNAL_ACTIF; p.signalMax=DJ30_SIGNAL_MAX;
      p.atrActif=DJ30_ATR_ACTIF; p.atrSpike=DJ30_ATR_SPIKE;
   }
   else if(g_Symbol=="DAX40") {
      p.sl=DAX40_SL; p.be=DAX40_BE; p.tp=DAX40_TP; p.trail=DAX40_TRAIL;
      p.smaFast=DAX40_FAST; p.smaSlow=DAX40_SLOW; p.smaH4=DAX40_H4; p.emaD1=DAX40_D1;
      p.maxSpread=DAX40_SPREAD; p.defaultLot=DAX40_LOT;
      p.sessionFilter=DAX40_SESSION; p.sessionStart=DAX40_START; p.sessionEnd=DAX40_END;
      p.signalActif=DAX40_SIGNAL_ACTIF; p.signalMax=DAX40_SIGNAL_MAX;
      p.atrActif=DAX40_ATR_ACTIF; p.atrSpike=DAX40_ATR_SPIKE;
   }
   else if(g_Symbol=="FTSE100") {
      p.sl=FTSE100_SL; p.be=FTSE100_BE; p.tp=FTSE100_TP; p.trail=FTSE100_TRAIL;
      p.smaFast=FTSE100_FAST; p.smaSlow=FTSE100_SLOW; p.smaH4=FTSE100_H4; p.emaD1=FTSE100_D1;
      p.maxSpread=FTSE100_SPREAD; p.defaultLot=FTSE100_LOT;
      p.sessionFilter=FTSE100_SESSION; p.sessionStart=FTSE100_START; p.sessionEnd=FTSE100_END;
      p.signalActif=FTSE100_SIGNAL_ACTIF; p.signalMax=FTSE100_SIGNAL_MAX;
      p.atrActif=FTSE100_ATR_ACTIF; p.atrSpike=FTSE100_ATR_SPIKE;
   }
   else if(g_Symbol=="EU50") {
      p.sl=EU50_SL; p.be=EU50_BE; p.tp=EU50_TP; p.trail=EU50_TRAIL;
      p.smaFast=EU50_FAST; p.smaSlow=EU50_SLOW; p.smaH4=EU50_H4; p.emaD1=EU50_D1;
      p.maxSpread=EU50_SPREAD; p.defaultLot=EU50_LOT;
      p.sessionFilter=EU50_SESSION; p.sessionStart=EU50_START; p.sessionEnd=EU50_END;
      p.signalActif=EU50_SIGNAL_ACTIF; p.signalMax=EU50_SIGNAL_MAX;
      p.atrActif=EU50_ATR_ACTIF; p.atrSpike=EU50_ATR_SPIKE;
   }
   else if(g_Symbol=="Nikkei") {
      p.sl=Nikkei_SL; p.be=Nikkei_BE; p.tp=Nikkei_TP; p.trail=Nikkei_TRAIL;
      p.smaFast=Nikkei_FAST; p.smaSlow=Nikkei_SLOW; p.smaH4=Nikkei_H4; p.emaD1=Nikkei_D1;
      p.maxSpread=Nikkei_SPREAD; p.defaultLot=Nikkei_LOT;
      p.sessionFilter=Nikkei_SESSION; p.sessionStart=Nikkei_START; p.sessionEnd=Nikkei_END;
      p.signalActif=Nikkei_SIGNAL_ACTIF; p.signalMax=Nikkei_SIGNAL_MAX;
      p.atrActif=Nikkei_ATR_ACTIF; p.atrSpike=Nikkei_ATR_SPIKE;
   }
   else if(g_Symbol=="GBPJPY") {
      p.sl=GBPJPY_SL; p.be=GBPJPY_BE; p.tp=GBPJPY_TP; p.trail=GBPJPY_TRAIL;
      p.smaFast=GBPJPY_FAST; p.smaSlow=GBPJPY_SLOW; p.smaH4=GBPJPY_H4; p.emaD1=GBPJPY_D1;
      p.maxSpread=GBPJPY_SPREAD; p.defaultLot=GBPJPY_LOT;
      p.sessionFilter=GBPJPY_SESSION; p.sessionStart=GBPJPY_START; p.sessionEnd=GBPJPY_END;
      p.signalActif=GBPJPY_SIGNAL_ACTIF; p.signalMax=GBPJPY_SIGNAL_MAX;
      p.atrActif=GBPJPY_ATR_ACTIF; p.atrSpike=GBPJPY_ATR_SPIKE;
   }
   else if(g_Symbol=="EURJPY") {
      p.sl=EURJPY_SL; p.be=EURJPY_BE; p.tp=EURJPY_TP; p.trail=EURJPY_TRAIL;
      p.smaFast=EURJPY_FAST; p.smaSlow=EURJPY_SLOW; p.smaH4=EURJPY_H4; p.emaD1=EURJPY_D1;
      p.maxSpread=EURJPY_SPREAD; p.defaultLot=EURJPY_LOT;
      p.sessionFilter=EURJPY_SESSION; p.sessionStart=EURJPY_START; p.sessionEnd=EURJPY_END;
      p.signalActif=EURJPY_SIGNAL_ACTIF; p.signalMax=EURJPY_SIGNAL_MAX;
      p.atrActif=EURJPY_ATR_ACTIF; p.atrSpike=EURJPY_ATR_SPIKE;
   }
   else if(g_Symbol=="AUDJPY") {
      p.sl=AUDJPY_SL; p.be=AUDJPY_BE; p.tp=AUDJPY_TP; p.trail=AUDJPY_TRAIL;
      p.smaFast=AUDJPY_FAST; p.smaSlow=AUDJPY_SLOW; p.smaH4=AUDJPY_H4; p.emaD1=AUDJPY_D1;
      p.maxSpread=AUDJPY_SPREAD; p.defaultLot=AUDJPY_LOT;
      p.sessionFilter=AUDJPY_SESSION; p.sessionStart=AUDJPY_START; p.sessionEnd=AUDJPY_END;
      p.signalActif=AUDJPY_SIGNAL_ACTIF; p.signalMax=AUDJPY_SIGNAL_MAX;
      p.atrActif=AUDJPY_ATR_ACTIF; p.atrSpike=AUDJPY_ATR_SPIKE;
   }
   else if(g_Symbol=="GBPUSD") {
      p.sl=GBPUSD_SL; p.be=GBPUSD_BE; p.tp=GBPUSD_TP; p.trail=GBPUSD_TRAIL;
      p.smaFast=GBPUSD_FAST; p.smaSlow=GBPUSD_SLOW; p.smaH4=GBPUSD_H4; p.emaD1=GBPUSD_D1;
      p.maxSpread=GBPUSD_SPREAD; p.defaultLot=GBPUSD_LOT;
      p.sessionFilter=GBPUSD_SESSION; p.sessionStart=GBPUSD_START; p.sessionEnd=GBPUSD_END;
      p.signalActif=GBPUSD_SIGNAL_ACTIF; p.signalMax=GBPUSD_SIGNAL_MAX;
      p.atrActif=GBPUSD_ATR_ACTIF; p.atrSpike=GBPUSD_ATR_SPIKE;
   }
   else if(g_Symbol=="AUDUSD") {
      p.sl=AUDUSD_SL; p.be=AUDUSD_BE; p.tp=AUDUSD_TP; p.trail=AUDUSD_TRAIL;
      p.smaFast=AUDUSD_FAST; p.smaSlow=AUDUSD_SLOW; p.smaH4=AUDUSD_H4; p.emaD1=AUDUSD_D1;
      p.maxSpread=AUDUSD_SPREAD; p.defaultLot=AUDUSD_LOT;
      p.sessionFilter=AUDUSD_SESSION; p.sessionStart=AUDUSD_START; p.sessionEnd=AUDUSD_END;
      p.signalActif=AUDUSD_SIGNAL_ACTIF; p.signalMax=AUDUSD_SIGNAL_MAX;
      p.atrActif=AUDUSD_ATR_ACTIF; p.atrSpike=AUDUSD_ATR_SPIKE;
   }
   else if(g_Symbol=="Silver") {
      p.sl=Silver_SL; p.be=Silver_BE; p.tp=Silver_TP; p.trail=Silver_TRAIL;
      p.smaFast=Silver_FAST; p.smaSlow=Silver_SLOW; p.smaH4=Silver_H4; p.emaD1=Silver_D1;
      p.maxSpread=Silver_SPREAD; p.defaultLot=Silver_LOT;
      p.sessionFilter=Silver_SESSION; p.sessionStart=Silver_START; p.sessionEnd=Silver_END;
      p.signalActif=Silver_SIGNAL_ACTIF; p.signalMax=Silver_SIGNAL_MAX;
      p.atrActif=Silver_ATR_ACTIF; p.atrSpike=Silver_ATR_SPIKE;
   }
   else if(g_Symbol=="BRENT") {
      p.sl=BRENT_SL; p.be=BRENT_BE; p.tp=BRENT_TP; p.trail=BRENT_TRAIL;
      p.smaFast=BRENT_FAST; p.smaSlow=BRENT_SLOW; p.smaH4=BRENT_H4; p.emaD1=BRENT_D1;
      p.maxSpread=BRENT_SPREAD; p.defaultLot=BRENT_LOT;
      p.sessionFilter=BRENT_SESSION; p.sessionStart=BRENT_START; p.sessionEnd=BRENT_END;
      p.signalActif=BRENT_SIGNAL_ACTIF; p.signalMax=BRENT_SIGNAL_MAX;
      p.atrActif=BRENT_ATR_ACTIF; p.atrSpike=BRENT_ATR_SPIKE;
   }
   else if(g_Symbol=="Bitcoin") {
      p.sl=Bitcoin_SL; p.be=Bitcoin_BE; p.tp=Bitcoin_TP; p.trail=Bitcoin_TRAIL;
      p.smaFast=Bitcoin_FAST; p.smaSlow=Bitcoin_SLOW; p.smaH4=Bitcoin_H4; p.emaD1=Bitcoin_D1;
      p.maxSpread=Bitcoin_SPREAD; p.defaultLot=Bitcoin_LOT;
      p.sessionFilter=Bitcoin_SESSION; p.sessionStart=Bitcoin_START; p.sessionEnd=Bitcoin_END;
      p.signalActif=Bitcoin_SIGNAL_ACTIF; p.signalMax=Bitcoin_SIGNAL_MAX;
      p.atrActif=Bitcoin_ATR_ACTIF; p.atrSpike=Bitcoin_ATR_SPIKE;
   }
   else if(g_Symbol=="Ethereum") {
      p.sl=Ethereum_SL; p.be=Ethereum_BE; p.tp=Ethereum_TP; p.trail=Ethereum_TRAIL;
      p.smaFast=Ethereum_FAST; p.smaSlow=Ethereum_SLOW; p.smaH4=Ethereum_H4; p.emaD1=Ethereum_D1;
      p.maxSpread=Ethereum_SPREAD; p.defaultLot=Ethereum_LOT;
      p.sessionFilter=Ethereum_SESSION; p.sessionStart=Ethereum_START; p.sessionEnd=Ethereum_END;
      p.signalActif=Ethereum_SIGNAL_ACTIF; p.signalMax=Ethereum_SIGNAL_MAX;
      p.atrActif=Ethereum_ATR_ACTIF; p.atrSpike=Ethereum_ATR_SPIKE;
   }
   else {
      p.sl=2.0; p.be=1.0; p.tp=2.0; p.trail=1.0;
      p.smaFast=20; p.smaSlow=50; p.smaH4=50; p.emaD1=50;
      p.maxSpread=50; p.defaultLot=0.01;
      p.sessionFilter=false; p.sessionStart=0; p.sessionEnd=23;
      p.signalActif=false; p.signalMax=3.0;
      p.atrActif=false; p.atrSpike=2.5;
      Print("Symbole non configure: ",g_Symbol," — params defaut");
   }
   return p;
}

bool IsInSession(const SymbolParams &p) {
   if(!p.sessionFilter) return true;
   MqlDateTime dt; TimeToStruct(TimeGMT(),dt);
   int h=dt.hour;
   if(p.sessionStart<=p.sessionEnd) return(h>=p.sessionStart && h<p.sessionEnd);
   return(h>=p.sessionStart || h<p.sessionEnd); // chevauchant minuit (ex Nikkei)
}

void LogEvent(string action,string detail,double lot=0,double price=0,double sl=0,double tp=0,double pnl=0) {
   string fname="EA_Journal_"+g_AccountID+".csv";
   string ts=TimeToString(TimeGMT(),TIME_DATE|TIME_SECONDS);
   int digits=(int)SymbolInfoInteger(g_Symbol,SYMBOL_DIGITS);
   int h=FileOpen(fname,FILE_READ|FILE_WRITE|FILE_CSV|FILE_ANSI|FILE_COMMON);
   if(h!=INVALID_HANDLE) { FileSeek(h,0,SEEK_END); }
   else {
      h=FileOpen(fname,FILE_WRITE|FILE_CSV|FILE_ANSI|FILE_COMMON);
      if(h==INVALID_HANDLE){Print("Journal: ",GetLastError());return;}
      FileWrite(h,"TimestampUTC","Symbol","Mode","Action","Lot","Price","SL","TP","PnL","DailyDD","TotalDD","Detail");
   }
   FileWrite(h,ts,g_Symbol,TRADING_MODE,action,
             DoubleToString(lot,2),DoubleToString(price,digits),
             DoubleToString(sl,digits),DoubleToString(tp,digits),
             DoubleToString(pnl,2),DoubleToString(g_DailyDD,3),
             DoubleToString(g_TotalDD,3),detail);
   FileClose(h);
}

void WriteHeartbeat() {
   string fname="Heartbeat_"+g_Symbol+"_"+g_AccountID+".csv";
   string ts=TimeToString(TimeGMT(),TIME_DATE|TIME_SECONDS);
   int h=FileOpen(fname,FILE_WRITE|FILE_CSV|FILE_ANSI|FILE_COMMON);
   if(h==INVALID_HANDLE){Print("Heartbeat: ",GetLastError());return;}
   FileWrite(h,"TimestampUTC","Symbol","Mode","ModeActive","Status","Position",
             "DailyDD","SL_today","TP_today","Spread","CSVage");
   string pos="NONE";
   if(PositionSelect(g_Symbol)) pos=(PositionGetInteger(POSITION_TYPE)==POSITION_TYPE_BUY)?"BUY":"SELL";
   long spread=SymbolInfoInteger(g_Symbol,SYMBOL_SPREAD);
   long age=(g_CSVTimestamp>0)?(long)(TimeGMT()-g_CSVTimestamp):-1;
   FileWrite(h,ts,g_Symbol,TRADING_MODE,(g_ModeActive?"ACTIF":"VEILLE"),"OK",pos,
             DoubleToString(g_DailyDD,3),IntegerToString(g_SL_count_today),
             IntegerToString(g_TP_count_today),IntegerToString((int)spread),IntegerToString((int)age));
   FileClose(h);
}

bool ReadCSV() {
   int h=FileOpen(CSV_FILENAME,FILE_READ|FILE_CSV|FILE_ANSI|FILE_COMMON);
   if(h==INVALID_HANDLE){Print("CSV absent: ",CSV_FILENAME," err:",GetLastError());return false;}
   FileReadString(h); // header
   bool found=false;
   while(!FileIsEnding(h)) {
      string line=FileReadString(h);
      if(StringLen(line)<5) continue;
      string arr[]; int n=StringSplit(line,',',arr);
      if(n<18) continue;
      if(arr[1]!=g_Symbol) continue;
      if(arr[0]!=g_AccountID){Print("AccountID CSV!=EA");FileClose(h);return false;}
      g_Weight=StringToDouble(arr[2]); g_Lot=StringToDouble(arr[3]);
      g_CSVTimestamp=ParseISO8601(arr[4]);
      g_MarketOpen=(int)StringToInteger(arr[5]);
      g_EnableTrading=(int)StringToInteger(arr[6]);
      g_TradeAllowed_Global=(int)StringToInteger(arr[7]);
      g_TradeAllowed_Symbol=(int)StringToInteger(arr[8]);
      g_Restart=(int)StringToInteger(arr[9]);
      g_RiskFactor=StringToDouble(arr[10]); g_RiskMult=StringToDouble(arr[11]);
      g_DailyDD=StringToDouble(arr[12]); g_TotalDD=StringToDouble(arr[13]);
      g_SL_today=(int)StringToInteger(arr[14]); g_TP_today=(int)StringToInteger(arr[15]);
      // [v2.8] RiskEUR — colonne 16 (optionnelle, 0 si absente = pas de filtre)
      if(n>=17) g_RiskEUR=StringToDouble(arr[16]); else g_RiskEUR=0.0;
      found=true; break;
   }
   FileClose(h);
   if(!found) Print("Symbole ",g_Symbol," absent du CSV");
   return found;
}

enum CSV_STATUS{CSV_FRESH,CSV_STALE,CSV_WEEKEND};
CSV_STATUS GetCSVStatus() {
   if(g_CSVTimestamp<=0) return CSV_STALE;
   long age=(long)(TimeGMT()-g_CSVTimestamp);
   MqlDateTime dt; TimeToStruct(TimeGMT(),dt);
   if(dt.day_of_week==0||dt.day_of_week==6) return CSV_WEEKEND;
   if(dt.day_of_week==1&&dt.hour<MONDAY_OPEN_HOUR) return CSV_WEEKEND;
   return (age<=MAX_CSV_AGE_NORMAL)?CSV_FRESH:CSV_STALE;
}

double GetSafeLot() {
   SymbolParams p=GetSymbolParams();
   CSV_STATUS st=GetCSVStatus();
   double vMin=SymbolInfoDouble(g_Symbol,SYMBOL_VOLUME_MIN);
   double vMax=SymbolInfoDouble(g_Symbol,SYMBOL_VOLUME_MAX);
   double vStep=SymbolInfoDouble(g_Symbol,SYMBOL_VOLUME_STEP);
   if(vStep<=0) vStep=0.01;
   double lot=(st==CSV_STALE)?p.defaultLot:g_Lot;
   if(st==CSV_STALE) LogEvent("WARN_STALE","CSV trop vieux lot defaut",lot);
   // [v2.5] Plafond securite : lot max = MAX_LOT_RISK% du capital
   double capital=AccountInfoDouble(ACCOUNT_BALANCE);
   double lotCap=MathFloor((capital*MAX_LOT_RISK)/vStep)*vStep;
   if(lotCap<vMin) lotCap=vMin;
   if(lot>lotCap){
      LogEvent("WARN_LOT",StringFormat("Lot %.2f>plafond %.2f->plafond",lot,lotCap));
      lot=lotCap;
   }
   lot=MathFloor(lot/vStep)*vStep;
   return MathMax(vMin,MathMin(vMax,lot));
}

bool HasEnoughHistory() {
   SymbolParams p=GetSymbolParams();
   // [v2.8.1] Seuils assouplis : emaD1*24 exclu du calcul H1 (bloquait terminal neuf)
   // H1 : smaSlow * 1.5 suffisant pour initialiser indicateurs
   // H4/D1 : 20 bougies minimum (convergence EMA/SMA rapide)
   int warmup  = (int)(MathMax((double)p.smaSlow,(double)ATR_PERIOD)*1.5);
   int need_h4 = MathMax(20,(int)(p.smaH4*0.4));
   int need_d1 = MathMax(20,(int)(p.emaD1*0.4));
   if(Bars(g_Symbol,PERIOD_H1)<warmup)
   {Print("H1 insuffisant:",Bars(g_Symbol,PERIOD_H1),"<",warmup);return false;}
   if(Bars(g_Symbol,PERIOD_H4)<need_h4)
   {Print("H4 insuffisant:",Bars(g_Symbol,PERIOD_H4),"<",need_h4);return false;}
   if(Bars(g_Symbol,PERIOD_D1)<need_d1)
   {Print("D1 insuffisant:",Bars(g_Symbol,PERIOD_D1),"<",need_d1);return false;}
   return true;
}

bool InitIndicators() {
   SymbolParams p=GetSymbolParams();
   g_Handle_ATR     =iATR(g_Symbol,PERIOD_H1,ATR_PERIOD);
   g_Handle_SMA_Fast=iMA(g_Symbol,PERIOD_H1,p.smaFast,0,MODE_SMA,PRICE_CLOSE);
   g_Handle_SMA_Slow=iMA(g_Symbol,PERIOD_H1,p.smaSlow,0,MODE_SMA,PRICE_CLOSE);
   g_Handle_SMA_H4  =iMA(g_Symbol,PERIOD_H4,p.smaH4,0,MODE_SMA,PRICE_CLOSE);
   g_Handle_EMA_D1  =iMA(g_Symbol,PERIOD_D1,p.emaD1,0,MODE_EMA,PRICE_CLOSE);
   if(g_Handle_ATR==INVALID_HANDLE||g_Handle_SMA_Fast==INVALID_HANDLE||
      g_Handle_SMA_Slow==INVALID_HANDLE||g_Handle_SMA_H4==INVALID_HANDLE||
      g_Handle_EMA_D1==INVALID_HANDLE)
   {Print("Erreur handles: ",GetLastError());return false;}
   return true;
}

void ReleaseIndicators() {
   if(g_Handle_ATR     !=INVALID_HANDLE) IndicatorRelease(g_Handle_ATR);
   if(g_Handle_SMA_Fast!=INVALID_HANDLE) IndicatorRelease(g_Handle_SMA_Fast);
   if(g_Handle_SMA_Slow!=INVALID_HANDLE) IndicatorRelease(g_Handle_SMA_Slow);
   if(g_Handle_SMA_H4  !=INVALID_HANDLE) IndicatorRelease(g_Handle_SMA_H4);
   if(g_Handle_EMA_D1  !=INVALID_HANDLE) IndicatorRelease(g_Handle_EMA_D1);
   g_Handle_ATR=g_Handle_SMA_Fast=g_Handle_SMA_Slow=g_Handle_SMA_H4=g_Handle_EMA_D1=INVALID_HANDLE;
}


//+------------------------------------------------------------------+
//| [v2.6] ATR MOYEN 20 PERIODES — pour circuit breaker             |
//+------------------------------------------------------------------+
double GetATRMean20()
{
   double buf[];
   ArraySetAsSeries(buf, true);
   if(CopyBuffer(g_Handle_ATR, 0, 0, 22, buf) < 21) return 0.0;
   double sum = 0.0;
   for(int i = 1; i <= 20; i++) sum += buf[i];
   return sum / 20.0;
}

//+------------------------------------------------------------------+
//| [v2.6] FILTRE LIQUIDITE — tick volume mini sur 3 bougies        |
//+------------------------------------------------------------------+
bool IsLiquidityOK()
{
   long vol[];
   ArraySetAsSeries(vol, true);
   if(CopyTickVolume(g_Symbol, PERIOD_H1, 1, 3, vol) < 3) return true; // doute -> OK
   long avg = (vol[0] + vol[1] + vol[2]) / 3;
   if(avg < MIN_TICK_VOLUME) {
      LogEvent("REFUS_LIQUIDITE",
               StringFormat("TickVol moy %d < min %d", (int)avg, MIN_TICK_VOLUME));
      return false;
   }
   return true;
}

//+------------------------------------------------------------------+
//| [v2.6] PROTECTION GAP LUNDI                                     |
//| Si le gap ouverture > GAP_MAX_ATR x ATR -> pas de trade         |
//+------------------------------------------------------------------+
bool IsMondayGapOK()
{
   MqlDateTime dt; TimeToStruct(TimeGMT(), dt);
   if(dt.day_of_week != 1) return true; // pas lundi -> OK

   double atr = GetATR();
   if(atr <= 0.0) return true;

   // Close du vendredi = close[1] si on est encore sur la 1ere bougie lundi
   // Close de la bougie precedente vs open actuelle
   double closes[], opens[];
   ArraySetAsSeries(closes, true); ArraySetAsSeries(opens, true);
   if(CopyClose(g_Symbol, PERIOD_H1, 1, 2, closes) < 2) return true;
   if(CopyOpen(g_Symbol,  PERIOD_H1, 0, 1, opens)  < 1) return true;

   double gap = MathAbs(opens[0] - closes[0]);
   if(gap > atr * GAP_MAX_ATR) {
      LogEvent("REFUS_GAP",
               StringFormat("Gap %.5f > %.1fx ATR %.5f", gap, GAP_MAX_ATR, atr));
      return false;
   }
   return true;
}

//+------------------------------------------------------------------+
//| [v2.6] CIRCUIT BREAKER ATR                                      |
//| ATR courant > atrSpike x ATR moyen 20p -> signal suspect        |
//+------------------------------------------------------------------+
bool IsATROK(const SymbolParams &p)
{
   if(!p.atrActif) return true; // desactive par defaut
   double atr     = GetATR();
   double atrMean = GetATRMean20();
   if(atr <= 0.0 || atrMean <= 0.0) return true;
   if(atr > atrMean * p.atrSpike) {
      LogEvent("REFUS_ATR_SPIKE",
               StringFormat("ATR %.5f > %.1fx ATRmoy %.5f",
                            atr, p.atrSpike, atrMean));
      return false;
   }
   return true;
}

//+------------------------------------------------------------------+
//| [v2.6] FILTRE SOLIDITE SIGNAL                                   |
//| Distance prix/SMA_lente > signalMax x ATR -> signal essoufle    |
//+------------------------------------------------------------------+
bool IsSignalStrengthOK(const SymbolParams &p, int direction)
{
   if(!p.signalActif) return true; // desactive par defaut
   double atr = GetATR();
   if(atr <= 0.0) return true;

   double buf[]; ArraySetAsSeries(buf, true);
   if(CopyBuffer(g_Handle_SMA_Slow, 0, 0, 3, buf) < 2) return true;
   double sma_slow = buf[1];
   double price    = SymbolInfoDouble(g_Symbol, SYMBOL_BID);
   double dist     = MathAbs(price - sma_slow);

   if(dist > atr * p.signalMax) {
      LogEvent("REFUS_SIGNAL_FORT",
               StringFormat("Dist %.5f > %.1fx ATR %.5f (signal essoufle)",
                            dist, p.signalMax, atr));
      return false;
   }
   return true;
}

double GetATR() {
   double buf[]; ArraySetAsSeries(buf,true);
   if(CopyBuffer(g_Handle_ATR,0,0,3,buf)<2) return 0.0;
   return buf[1];
}

bool GetTrendValues(double &sf,double &ss,double &h4,double &d1) {
   double buf[]; ArraySetAsSeries(buf,true);
   if(CopyBuffer(g_Handle_SMA_Fast,0,0,3,buf)<2) return false; sf=buf[1];
   if(CopyBuffer(g_Handle_SMA_Slow,0,0,3,buf)<2) return false; ss=buf[1];
   if(CopyBuffer(g_Handle_SMA_H4,  0,0,3,buf)<2) return false; h4=buf[1];
   if(CopyBuffer(g_Handle_EMA_D1,  0,0,3,buf)<2) return false; d1=buf[1];
   return true;
}

int GetTrend() {
   double sf,ss,h4,d1;
   if(!GetTrendValues(sf,ss,h4,d1)) return 0;
   double price=SymbolInfoDouble(g_Symbol,SYMBOL_BID);
   if(price>sf && sf>ss && price>h4 && price>d1) return 1;
   if(price<sf && sf<ss && price<h4 && price<d1) return -1;
   return 0;
}

bool IsSpreadOK() {
   SymbolParams p=GetSymbolParams();
   double spread=(double)SymbolInfoInteger(g_Symbol,SYMBOL_SPREAD);
   if(spread>p.maxSpread){LogEvent("REFUS_SPREAD",StringFormat("%.0f>%.0f",spread,p.maxSpread));return false;}
   return true;
}

bool IsNewsWindow() {
   datetime now=TimeGMT();
   MqlCalendarValue vals[];
   if(CalendarValueHistory(vals,now-NEWS_FILTER_BEFORE*60,now+NEWS_FILTER_AFTER*60)<=0) return false;
   string cur[]={"USD","EUR","JPY","GBP","AUD","CAD","XAU","CNY","CHF"};
   for(int i=0;i<ArraySize(vals);i++) {
      MqlCalendarEvent ev; if(!CalendarEventById(vals[i].event_id,ev)) continue;
      if(ev.importance!=CALENDAR_IMPORTANCE_HIGH) continue;
      MqlCalendarCountry co; if(!CalendarCountryById(ev.country_id,co)) continue;
      for(int j=0;j<ArraySize(cur);j++) {
         if(co.currency==cur[j]) {
            LogEvent("FILTRE_NEWS",StringFormat("News %s %s",co.currency,
                     TimeToString(vals[i].time,TIME_DATE|TIME_SECONDS)));
            return true;
         }
      }
   }
   return false;
}

void CheckDailyReset() {
   MqlDateTime dt; TimeToStruct(TimeGMT(),dt);
   string today=StringFormat("%04d-%02d-%02d",dt.year,dt.mon,dt.day);
   if(today==g_TodayDate) return;
   g_TodayDate=today; g_SL_count_today=0; g_TP_count_today=0;
   g_CooldownUntil=0; g_DayStartBalance=AccountInfoDouble(ACCOUNT_BALANCE); g_BESet=false;
   Print("Nouveau jour ",today," | Balance:",DoubleToString(g_DayStartBalance,2));
   LogEvent("NEW_DAY",StringFormat("Balance:%.2f",g_DayStartBalance));
}

double GetLocalDailyDD() {
   if(g_DayStartBalance<=0) return 0.0;
   return MathMax(0.0,(g_DayStartBalance-AccountInfoDouble(ACCOUNT_BALANCE))/g_DayStartBalance);
}
double GetLocalTotalDD() {
   if(g_InitialBalance<=0) return 0.0;
   return MathMax(0.0,(g_InitialBalance-AccountInfoDouble(ACCOUNT_BALANCE))/g_InitialBalance);
}

bool CanTrade(string &reason) {
   if(!g_ModeActive){reason=StringFormat("VEILLE mode=%s",TRADING_MODE);return false;}
   if(g_EnableTrading==0){reason="ENABLE_TRADING=0";return false;}
   if(g_TradeAllowed_Global==0){reason="TradeAllowed_Global=0";return false;}
   if(g_TradeAllowed_Symbol==0){reason="TradeAllowed_Symbol=0";return false;}
   double dd=GetLocalDailyDD(),ddt=GetLocalTotalDD(),md=GetMaxDailyLoss(),mt=GetMaxTotalDD();
   if(dd>=md){reason=StringFormat("DD jour %.2f%%>=%.2f%%",dd*100,md*100);return false;}
   if(ddt>=mt){reason=StringFormat("DD total %.2f%%>=%.2f%%",ddt*100,mt*100);return false;}
   if(g_SL_count_today>=MAX_SL_PER_DAY&&TimeGMT()<g_CooldownUntil)
   {reason=StringFormat("Cooldown SL %s",TimeToString(g_CooldownUntil,TIME_SECONDS));return false;}
   // [v2.9] Bloquer toute ouverture si fermeture vendredi deja declenchee sur ce symbole
   if(g_FridayCloseStarted){reason="Vendredi cloture deja declenchee";return false;}
   MqlDateTime dt; TimeToStruct(TimeGMT(),dt);
   // [v2.9] Fermeture vendredi anticipee pour indices europeens (DAX40, EU50)
   if(dt.day_of_week==5 && IsEUIndex()) {
      bool eu_stop=(dt.hour>FRIDAY_STOP_EU_HOUR||(dt.hour==FRIDAY_STOP_EU_HOUR&&dt.min>=FRIDAY_STOP_EU_MIN));
      if(eu_stop){reason="Fermeture vendredi EU";return false;}
   }
   if(dt.day_of_week==5&&(dt.hour>FRIDAY_CLOSE_HOUR||(dt.hour==FRIDAY_CLOSE_HOUR&&dt.min>=FRIDAY_CLOSE_MIN)))
   {reason="Fermeture vendredi";return false;}
   if(dt.day_of_week==1&&dt.hour<MONDAY_OPEN_HOUR){reason="Lundi pas ouvert";return false;}
   if(dt.day_of_week==0||dt.day_of_week==6){reason="Weekend";return false;}
   SymbolParams p=GetSymbolParams();
   if(!IsInSession(p)){reason=StringFormat("Hors session %dh-%dh",p.sessionStart,p.sessionEnd);return false;}
   if(IsNewsWindow()){reason="News";return false;}
   if(!IsSpreadOK()){reason="Spread";return false;}
   // Filtres v2.6
   if(!IsMondayGapOK()){reason="Gap lundi";return false;}
   if(!IsLiquidityOK()){reason="Liquidite insuffisante";return false;}
   SymbolParams _p=GetSymbolParams();
   if(!IsATROK(_p)){reason="ATR spike";return false;}
   if(!TerminalInfoInteger(TERMINAL_CONNECTED)){reason="Terminal deconnecte";return false;}
   if(!TerminalInfoInteger(TERMINAL_TRADE_ALLOWED)){reason="Trade non autorise";return false;}
   return true;
}

bool HasOpenPosition() { return PositionSelect(g_Symbol); }

void CloseAllPositions(string reason) {
   if(!PositionSelect(g_Symbol)) return;
   double pnl  =PositionGetDouble(POSITION_PROFIT);
   double price =PositionGetDouble(POSITION_PRICE_CURRENT);
   double lot   =PositionGetDouble(POSITION_VOLUME);
   // [v2.9] Flag vendredi : marquer la fermeture pour bloquer les rouvertures
   if(StringFind(reason,"vendredi")>=0 || StringFind(reason,"Vendredi")>=0)
      g_FridayCloseStarted = true;
   for(int i=1;i<=MAX_RETRY_ORDERS;i++) {
      if(trade.PositionClose(g_Symbol,MAX_SLIPPAGE)) {
         LogEvent("CLOSE",reason,lot,price,0,0,pnl);
         Print("Ferme ",reason," PnL:",pnl);
         return;
      }
      // [v2.9] Detecter marche ferme — ne pas boucler inutilement
      int rc=(int)trade.ResultRetcode();
      if(rc==TRADE_RETCODE_MARKET_CLOSED || rc==TRADE_RETCODE_TRADE_DISABLED) {
         LogEvent("CLOSE_FAIL",StringFormat("%s — Marche ferme (code %d), abandon",reason,rc),lot,price);
         Print("Impossible fermer ",g_Symbol," : marche ferme rc=",rc);
         return;
      }
      Sleep(1000);
   }
   Print("Impossible fermer ",g_Symbol," apres ",MAX_RETRY_ORDERS," tentatives");
   LogEvent("CLOSE_FAIL",StringFormat("%s — echec %d tentatives",reason,MAX_RETRY_ORDERS),lot,price);
}

double GetMinStop() {
   double pt=SymbolInfoDouble(g_Symbol,SYMBOL_POINT);
   return MathMax((double)SymbolInfoInteger(g_Symbol,SYMBOL_TRADE_STOPS_LEVEL),
                  (double)SymbolInfoInteger(g_Symbol,SYMBOL_TRADE_FREEZE_LEVEL))*pt;
}

void OpenTrade(int dir) {
   // Cooldown apres echec MAX_RETRY_ORDERS [v2.6]
   if(TimeGMT() < g_FailedOrderCooldown) {
      LogEvent("REFUS_COOLDOWN_ORDRE","Cooldown echec ordre actif");
      return;
   }
   double atr=GetATR(); if(atr<=0.0){LogEvent("REFUS_ATR","ATR nul");return;}
   SymbolParams p=GetSymbolParams();
   // Filtre solidite signal [v2.6 — desactive par defaut]
   if(!IsSignalStrengthOK(p, dir)) return;
   // [v2.8] Filtre cout reel en EUR par trade
   // cout_reel = SL_pts x (TICK_VALUE / TICK_SIZE) x lot_min
   // TICK_VALUE est deja en devise du compte (EUR) — pas de conversion necessaire
   // SAFETY_FACTOR dans RISK_EUR absorbe les fluctuations EURUSD residuelles
   if(g_RiskEUR > 0.0) {
      double sl_pts    = atr * p.sl;
      double tick_val  = SymbolInfoDouble(g_Symbol, SYMBOL_TRADE_TICK_VALUE);
      double tick_size = SymbolInfoDouble(g_Symbol, SYMBOL_TRADE_TICK_SIZE);
      double lot_min   = SymbolInfoDouble(g_Symbol, SYMBOL_VOLUME_MIN);
      if(tick_val > 0.0 && tick_size > 0.0 && lot_min > 0.0) {
         double cout_reel = sl_pts * (tick_val / tick_size) * lot_min;
         if(cout_reel > g_RiskEUR) {
            LogEvent("REFUS_RISQUE_EUR",
                     StringFormat("Cout %.2fEUR > RiskEUR %.2fEUR "
                                  "(SL_pts=%.5f TickVal=%.5f LotMin=%.2f)",
                                  cout_reel, g_RiskEUR,
                                  sl_pts, tick_val, lot_min));
            return;
         }
      }
   }
   double sld=atr*p.sl, tpd=atr*p.tp, mn=GetMinStop();
   if(sld<mn) sld=mn*1.1; if(tpd<mn) tpd=mn*1.1;
   // [v2.9] Plafond SL en % capital — evite risque disproportionne sur Silver/Gold
   if(MAX_RISK_PCT_PER_TRADE > 0.0) {
      double capital     = AccountInfoDouble(ACCOUNT_BALANCE);
      double tick_val    = SymbolInfoDouble(g_Symbol, SYMBOL_TRADE_TICK_VALUE);
      double tick_size   = SymbolInfoDouble(g_Symbol, SYMBOL_TRADE_TICK_SIZE);
      double lot_used    = GetSafeLot();
      if(tick_val>0.0 && tick_size>0.0 && lot_used>0.0) {
         double max_loss_eur = capital * MAX_RISK_PCT_PER_TRADE;
         double val_per_pt   = tick_val / tick_size * lot_used;
         double sld_max      = max_loss_eur / val_per_pt;
         if(sld > sld_max) {
            LogEvent("SL_PLAFOND",StringFormat(
               "SL %.5f reduit a %.5f (risque %.2fEUR > plafond %.2fEUR)",
               sld, sld_max, sld*val_per_pt, max_loss_eur));
            sld = sld_max;
            // Ajuster TP proportionnellement pour maintenir le ratio R:R
            double ratio = tpd / (atr * p.sl);
            tpd = sld_max * ratio;
         }
      }
   }
   double ask=SymbolInfoDouble(g_Symbol,SYMBOL_ASK);
   double bid=SymbolInfoDouble(g_Symbol,SYMBOL_BID);
   int digits=(int)SymbolInfoInteger(g_Symbol,SYMBOL_DIGITS);
   double lot=GetSafeLot();
   double slp,tpp,ep;
   if(dir==1){ep=ask;slp=NormalizeDouble(ask-sld,digits);tpp=NormalizeDouble(ask+tpd,digits);}
   else      {ep=bid;slp=NormalizeDouble(bid+sld,digits);tpp=NormalizeDouble(bid-tpd,digits);}
   if(MathAbs(slp-bid)<mn||MathAbs(tpp-bid)<mn)
   {LogEvent("REFUS_STOPS",StringFormat("mn=%.5f",mn));return;}
   trade.SetDeviationInPoints(MAX_SLIPPAGE);
   for(int i=1;i<=MAX_RETRY_ORDERS;i++) {
      bool ok=(dir==1)?trade.Buy(lot,g_Symbol,ask,slp,tpp):trade.Sell(lot,g_Symbol,bid,slp,tpp);
      if(ok){g_BESet=false;string d=(dir==1)?"BUY":"SELL";
             // Calcul slippage reel [v2.6]
             double exec_price=trade.ResultPrice();
             double slippage_pts=MathAbs(exec_price-ep)
                                /SymbolInfoDouble(g_Symbol,SYMBOL_POINT);
             LogEvent(d,StringFormat("Ouvert slip=%.1fpts",slippage_pts),
                      lot,exec_price,slp,tpp);
             Print(d," ",lot," ",g_Symbol,
                   " SL=",DoubleToString(slp,digits),
                   " TP=",DoubleToString(tpp,digits),
                   " Slip=",DoubleToString(slippage_pts,1),"pts");
             return;}
      int rc=(int)trade.ResultRetcode();
      Sleep((rc==TRADE_RETCODE_REQUOTE||rc==TRADE_RETCODE_PRICE_CHANGED)?500:1000);
   }
   // Cooldown apres echec MAX_RETRY_ORDERS [v2.6]
   g_FailedOrderCooldown = TimeGMT() + COOLDOWN_FAILED_ORDER * 60;
   LogEvent("REFUS_ORDRE",
            StringFormat("Echec %d tentatives — cooldown %dmin",
                         MAX_RETRY_ORDERS, COOLDOWN_FAILED_ORDER));
   Print("Ordre echoue — cooldown jusqu'a ",
         TimeToString(g_FailedOrderCooldown, TIME_SECONDS));
}

void ManageOpenPosition() {
   if(!PositionSelect(g_Symbol)) return;
   double atr=GetATR(); if(atr<=0.0) return;
   SymbolParams p=GetSymbolParams();
   double op=PositionGetDouble(POSITION_PRICE_OPEN);
   double csl=PositionGetDouble(POSITION_SL);
   double ctp=PositionGetDouble(POSITION_TP);
   double bid=SymbolInfoDouble(g_Symbol,SYMBOL_BID);
   double ask=SymbolInfoDouble(g_Symbol,SYMBOL_ASK);
   int    pt=(int)PositionGetInteger(POSITION_TYPE);
   int    dg=(int)SymbolInfoInteger(g_Symbol,SYMBOL_DIGITS);
   double mn=GetMinStop();
   if(pt==POSITION_TYPE_BUY) {
      double pd=bid-op;
      if(!g_BESet&&pd>=atr*p.be) {
         // [v2.8] BE = open_price + spread_pts
         // BUY entre sur ASK. Clôture BE sur BID.
         // Sans correction : BID touche ASK → PnL = BID - ASK = -spread (perte)
         // Avec correction : BID touche ASK+spread → PnL = ASK+spread-ASK = 0 (neutre)
         double spread_pts = (double)SymbolInfoInteger(g_Symbol,SYMBOL_SPREAD)
                             * SymbolInfoDouble(g_Symbol,SYMBOL_POINT);
         double ns=NormalizeDouble(op+spread_pts,dg);
         if(ns>csl&&(bid-ns)>=mn&&trade.PositionModify(g_Symbol,ns,ctp))
         {g_BESet=true;
          LogEvent("BE_SET",StringFormat("BE+spread pose spread=%.5f",spread_pts),0,op,ns,ctp);
          Print("BE ",g_Symbol,"->",ns," (spread=",DoubleToString(spread_pts,5),"pt)");}
      }
      if(g_BESet&&pd>=atr*p.tp) {
         double ns=NormalizeDouble(bid-atr*p.trail,dg);
         // [v2.9] N'envoyer PositionModify que si deplacement >= TRAIL_MIN_MOVE x ATR
         double min_move = atr * TRAIL_MIN_MOVE;
         if(ns>csl&&(bid-ns)>=mn&&(ns-csl)>=min_move) trade.PositionModify(g_Symbol,ns,ctp);
      }
   } else if(pt==POSITION_TYPE_SELL) {
      double pd=op-ask;
      if(!g_BESet&&pd>=atr*p.be) {
         // [v2.8] BE = open_price - spread_pts
         // SELL entre sur BID. Clôture BE sur ASK.
         // Sans correction : ASK touche BID → PnL = BID - ASK = -spread (perte)
         // Avec correction : ASK touche BID-spread → PnL = BID-(BID-spread) = 0 (neutre)
         double spread_pts = (double)SymbolInfoInteger(g_Symbol,SYMBOL_SPREAD)
                             * SymbolInfoDouble(g_Symbol,SYMBOL_POINT);
         double ns=NormalizeDouble(op-spread_pts,dg);
         if(ns<csl&&(ns-ask)>=mn&&trade.PositionModify(g_Symbol,ns,ctp))
         {g_BESet=true;
          LogEvent("BE_SET",StringFormat("BE-spread pose spread=%.5f",spread_pts),0,op,ns,ctp);
          Print("BE ",g_Symbol,"->",ns," (spread=",DoubleToString(spread_pts,5),"pt)");}
      }
      if(g_BESet&&pd>=atr*p.tp) {
         double ns=NormalizeDouble(ask+atr*p.trail,dg);
         // [v2.9] N'envoyer PositionModify que si deplacement >= TRAIL_MIN_MOVE x ATR
         double min_move = atr * TRAIL_MIN_MOVE;
         if(ns<csl&&(ns-ask)>=mn&&(csl-ns)>=min_move) trade.PositionModify(g_Symbol,ns,ctp);
      }
   }
}

void UpdateComment() {
   SymbolParams p=GetSymbolParams();
   string pos="Aucune";
   if(PositionSelect(g_Symbol)) pos=(PositionGetInteger(POSITION_TYPE)==POSITION_TYPE_BUY)?"BUY":"SELL";
   double dd=GetLocalDailyDD(),ddt=GetLocalTotalDD();
   long age=(g_CSVTimestamp>0)?(long)(TimeGMT()-g_CSVTimestamp):-1;
   long sp=SymbolInfoInteger(g_Symbol,SYMBOL_SPREAD);
   Comment(StringFormat(
      "EA Desk Quant v2.9 -- %s\n"
      "Mode: %s | %s\n"
      "Session: %s | %s\n"
      "DD Jour  [P]%.2f%% [E]%.2f%% max%.1f%%\n"
      "DD Total [P]%.2f%% [E]%.2f%% max%.1f%%\n"
      "SL=%d TP=%d | Pos: %s\n"
      "Params: SL%.1f TP%.1f BE%.1f Trail%.1f\n"
      "SMA F%d S%d H4=%d D1=%d\n"
      "Spread:%d%s | CSV:%ds | Lot:%.2f",
      g_Symbol,TRADING_MODE,(g_ModeActive?"ACTIF":"VEILLE"),
      (p.sessionFilter?StringFormat("%dh-%dh",p.sessionStart,p.sessionEnd):"24/7"),
      (IsInSession(p)?"EN SESSION":"HORS SESSION"),
      g_DailyDD,dd*100,GetMaxDailyLoss()*100,
      g_TotalDD,ddt*100,GetMaxTotalDD()*100,
      g_SL_count_today,g_TP_count_today,pos,
      p.sl,p.tp,p.be,p.trail,
      p.smaFast,p.smaSlow,p.smaH4,p.emaD1,
      (int)sp,(IsSpreadOK()?" OK":" ALERTE"),(int)age,g_Lot));
}

void OnTradeTransaction(const MqlTradeTransaction &t,const MqlTradeRequest &rq,const MqlTradeResult &rs) {
   if(t.type!=TRADE_TRANSACTION_DEAL_ADD||t.deal==0) return;
   if(!HistoryDealSelect(t.deal)) return;
   if(HistoryDealGetString(t.deal,DEAL_SYMBOL)!=g_Symbol) return;
   if(HistoryDealGetInteger(t.deal,DEAL_ENTRY)!=DEAL_ENTRY_OUT) return;
   // [v2.5] Ignorer les deals non emis par cet EA (trades manuels etc.)
   if(HistoryDealGetInteger(t.deal,DEAL_MAGIC)!=MAGIC_NUMBER) return;
   double pnl=HistoryDealGetDouble(t.deal,DEAL_PROFIT);
   if(pnl<0.0) {
      g_SL_count_today++;
      if(g_SL_count_today>=MAX_SL_PER_DAY) {
         g_CooldownUntil=TimeGMT()+COOLDOWN_AFTER_SL*60;
         LogEvent("COOLDOWN_SL",StringFormat("%d SL %dmin",g_SL_count_today,COOLDOWN_AFTER_SL));
         Print("Cooldown SL -> ",TimeToString(g_CooldownUntil,TIME_SECONDS));
      }
      g_BESet=false; LogEvent("SL_HIT","SL touche",0,0,0,0,pnl);
   } else {
      g_TP_count_today++; g_BESet=false; LogEvent("TP_HIT","TP touche",0,0,0,0,pnl);
   }
}


//+------------------------------------------------------------------+
//| [v2.7] LECTURE BALANCE INITIALE depuis Common\Files             |
//| Fichier ecrit par Ponderation.py a chaque cycle (60s)           |
//| Fallback : balance courante si fichier absent                    |
//+------------------------------------------------------------------+
double ReadInitialBalance()
{
   string fname = "InitBalance_" + g_AccountID + ".csv";
   int h = FileOpen(fname, FILE_READ|FILE_CSV|FILE_ANSI|FILE_COMMON);
   if(h == INVALID_HANDLE)
   {
      double bal = AccountInfoDouble(ACCOUNT_BALANCE);
      Print("WARN_INIT_BALANCE : fichier absent — fallback balance courante ",
            DoubleToString(bal, 2));
      LogEvent("WARN_INIT_BALANCE",
               StringFormat("Fichier absent — fallback %.2f", bal));
      return bal;
   }
   string header = FileReadString(h); // sauter en-tete
   double val = 0.0;
   if(!FileIsEnding(h))
   {
      string line = FileReadString(h);
      string arr[]; int n = StringSplit(line, ',', arr);
      if(n >= 2) val = StringToDouble(arr[1]);
   }
   FileClose(h);
   if(val <= 0.0)
   {
      double bal = AccountInfoDouble(ACCOUNT_BALANCE);
      Print("WARN_INIT_BALANCE : valeur nulle — fallback balance courante ",
            DoubleToString(bal, 2));
      LogEvent("WARN_INIT_BALANCE",
               StringFormat("Valeur nulle — fallback %.2f", bal));
      return bal;
   }
   Print("InitialBalance lu depuis fichier : ", DoubleToString(val, 2));
   return val;
}

int OnInit() {
   Print("=== Init EA Desk Quant v2.9 ===");
   long login=AccountInfoInteger(ACCOUNT_LOGIN);
   g_AccountID=BROKER_NAME+"_"+ACCOUNT_TYPE+"_"+IntegerToString((int)login);
   g_Symbol=_Symbol;
   Print("AccountID:",g_AccountID," Symbole:",g_Symbol," Mode:",TRADING_MODE);
   g_ModeActive=IsModeCompatible();
   if(!g_ModeActive) {
      Print("VEILLE — ",g_Symbol," non compatible avec mode ",TRADING_MODE);
      LogEvent("VEILLE",StringFormat("Mode %s incompatible",TRADING_MODE));
      EventSetTimer(300);
      return INIT_SUCCEEDED;
   }
   if(!HasEnoughHistory()) {Print("Historique insuffisant");return INIT_FAILED;}
   if(!InitIndicators()) return INIT_FAILED;
   // [v2.7] Lecture balance initiale depuis fichier Ponderation.py
   g_InitialBalance  = ReadInitialBalance();
   g_DayStartBalance = AccountInfoDouble(ACCOUNT_BALANCE);
   Print("InitialBalance=",DoubleToString(g_InitialBalance,2),
         " | DayStartBalance=",DoubleToString(g_DayStartBalance,2));
   MqlDateTime dt; TimeToStruct(TimeGMT(),dt);
   g_TodayDate=StringFormat("%04d-%02d-%02d",dt.year,dt.mon,dt.day);
   // [v2.5] Restauration g_BESet si position ouverte au redemarrage
   if(PositionSelect(g_Symbol)) {
      double pos_open=(double)PositionGetDouble(POSITION_PRICE_OPEN);
      double pos_sl  =(double)PositionGetDouble(POSITION_SL);
      int    pos_type=(int)PositionGetInteger(POSITION_TYPE);
      long   pos_magic=(long)PositionGetInteger(POSITION_MAGIC);
      if(pos_magic==MAGIC_NUMBER) {
         if(pos_type==POSITION_TYPE_BUY  && pos_sl>=pos_open) g_BESet=true;
         if(pos_type==POSITION_TYPE_SELL && pos_sl<=pos_open && pos_sl>0) g_BESet=true;
         LogEvent("REPRISE_POSITION",
                  StringFormat("BESet=%s Open=%.5f SL=%.5f",
                  (g_BESet?"true":"false"),pos_open,pos_sl));
      }
   }
   bool csvOK=false;
   for(int i=1;i<=5;i++){if(ReadCSV()){csvOK=true;break;}Print("CSV tentative ",i,"/5...");Sleep(10000);}
   if(!csvOK) Print("CSV absent au demarrage — trading suspendu");
   EventSetTimer(60);
   trade.SetExpertMagicNumber(MAGIC_NUMBER);
   trade.SetDeviationInPoints(MAX_SLIPPAGE);
   Print("EA init OK ",g_Symbol," Mode=",TRADING_MODE," Actif=",g_ModeActive);
   LogEvent("INIT",StringFormat("v2.9 Mode=%s InitBal=%.2f DayBal=%.2f",
         TRADING_MODE,g_InitialBalance,g_DayStartBalance));
   return INIT_SUCCEEDED;
}

void OnDeinit(const int reason) {
   EventKillTimer(); ReleaseIndicators(); Comment("");
   Print("EA arrete raison:",reason);
   LogEvent("DEINIT",StringFormat("Raison=%d",reason));
}

void OnTimer() {
   if(!g_ModeActive){WriteHeartbeat();UpdateComment();return;}
   CheckDailyReset();
   WriteHeartbeat();
   bool csvOK=ReadCSV();
   if(csvOK&&g_Restart==1) {
      Print("Flag Restart - reinit indicateurs");
      LogEvent("RESTART","Demande Python");
      ReleaseIndicators(); InitIndicators(); g_BESet=false;
   }
   if(csvOK&&(g_TradeAllowed_Global==0||g_EnableTrading==0)) {
      if(HasOpenPosition()) CloseAllPositions(g_EnableTrading==0?"ENABLE=0":"TradeAllowed=0");
      UpdateComment(); return;
   }
   double dd=GetLocalDailyDD(),ddt=GetLocalTotalDD();
   if(dd>=GetMaxDailyLoss()||ddt>=GetMaxTotalDD()) {
      if(HasOpenPosition()) CloseAllPositions("DD local depasse");
      UpdateComment(); return;
   }
   // [v2.9] Reset flag vendredi au lundi matin
   {
      MqlDateTime _dt; TimeToStruct(TimeGMT(),_dt);
      if(_dt.day_of_week==1 && _dt.hour>=MONDAY_OPEN_HOUR) g_FridayCloseStarted=false;
   }
   // [v2.9] Fermeture anticipee vendredi pour indices europeens (DAX40, EU50)
   {
      MqlDateTime _dt; TimeToStruct(TimeGMT(),_dt);
      if(_dt.day_of_week==5 && IsEUIndex()) {
         bool _eu_fclose=(_dt.hour>FRIDAY_CLOSE_EU_HOUR||
                         (_dt.hour==FRIDAY_CLOSE_EU_HOUR&&_dt.min>=FRIDAY_CLOSE_EU_MIN));
         if(_eu_fclose&&HasOpenPosition())
         {CloseAllPositions("Fermeture vendredi EU 14h30");UpdateComment();return;}
      }
   }
   // [v2.5] Fermeture position vendredi a FRIDAY_CLOSE_POS_MIN (20h15) — Forex/US
   {
      MqlDateTime _dt; TimeToStruct(TimeGMT(),_dt);
      bool _fclose=(_dt.day_of_week==5&&
         (_dt.hour>FRIDAY_CLOSE_HOUR||
         (_dt.hour==FRIDAY_CLOSE_HOUR&&_dt.min>=FRIDAY_CLOSE_POS_MIN)));
      if(_fclose&&HasOpenPosition())
      {CloseAllPositions("Fermeture vendredi 20h15");UpdateComment();return;}
   }
   if(HasOpenPosition()){ManageOpenPosition();UpdateComment();return;}
   string reason="";
   if(!CanTrade(reason)) {
      bool silent=(reason=="Weekend"||reason=="Lundi pas ouvert"||
                   reason=="Fermeture vendredi"||StringFind(reason,"Hors session")>=0||
                   StringFind(reason,"Hors session")>=0);
      if(!silent) LogEvent("REFUS",reason);
      UpdateComment(); return;
   }
   int trend=GetTrend();
   if(trend!=0) OpenTrade(trend);
   UpdateComment();
}

void OnTick() {
   if(!g_ModeActive) return;
   if(HasOpenPosition()) ManageOpenPosition();
}
//+------------------------------------------------------------------+
