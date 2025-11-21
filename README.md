//+------------------------------------------------------------------+
//|                                                    EMITER_V1.mq5 |
//|                                          Version: 01_SCALP       |
//|                                      Advanced Scalping System    |
//+------------------------------------------------------------------+
#property copyright "EMITER_V1"
#property link      "SCALPING SYSTEM"
#property version   "1.00"
#property description "🎯 10 Perfect Trades Daily - 1$ Target Per Trade"
#property description "⚡ Multi-Layer Voting System with 25 Strategies"
#property description "🛡️ Advanced Filters - 15 Scalping Indicators"

//+------------------------------------------------------------------+
//| شامل کتابخانه‌های ضروری                                           |
//+------------------------------------------------------------------+
#include <Trade\Trade.mqh>
#include <Trade\PositionInfo.mqh>
#include <Trade\OrderInfo.mqh>
#include <Trade\SymbolInfo.mqh>
#include <Trade\AccountInfo.mqh>

//+------------------------------------------------------------------+
//| ایجاد اشیاء کلاس‌ها                                              |
//+------------------------------------------------------------------+
CTrade         trade;
CPositionInfo  position;
COrderInfo     order;
CSymbolInfo    symbolInfo;
CAccountInfo   accountInfo;

//+------------------------------------------------------------------+
//| 📌 بخش 1: تنظیمات اصلی ربات                                     |
//+------------------------------------------------------------------+
input group "========== ⚙️ MAIN SETTINGS =========="
input string   InpEA_Name          = "EMITER_V1";           // 🤖 نام ربات
input string   InpEA_Version       = "01_SCALP";            // 📦 نسخه
input int      InpMagicNumber      = 100125;                // 🔢 Magic Number
input string   InpTradeSymbol      = "XAUUSD";              // 💰 نماد معاملاتی
input ENUM_TIMEFRAMES InpMainTF    = PERIOD_M1;             // ⏰ تایم فریم اصلی
input ENUM_TIMEFRAMES InpHelperTF  = PERIOD_M5;             // ⏱️ تایم فریم کمکی

//+------------------------------------------------------------------+
//| 📌 بخش 2: تنظیمات حجم و سود/ضرر                                 |
//+------------------------------------------------------------------+
input group "========== 💰 MONEY MANAGEMENT =========="
input double   InpLotSize          = 0.05;                  // 📊 حجم معاملات (Lot)
input double   InpTakeProfitPoint  = 10.0;                  // 🎯 حد سود (Points × 0.1)
input double   InpStopLossPoint    = 20.0;                  // 🛑 حد ضرر (Points × 0.1)
input bool     InpUseTrailingStop  = false;                 // 📈 استفاده از Trailing Stop
input double   InpTrailingStart    = 5.0;                   // 🏁 شروع Trailing (Points × 0.1)
input double   InpTrailingStep     = 2.0;                   // 👣 گام Trailing (Points × 0.1)

//+------------------------------------------------------------------+
//| 📌 بخش 3: مدیریت ریسک و محدودیت‌ها                              |
//+------------------------------------------------------------------+
input group "========== 🛡️ RISK MANAGEMENT =========="
input int      InpMaxTradesPerDay     = 10;                 // 📅 حداکثر معاملات در روز
input int      InpMaxOpenTrades       = 1;                  // 🔓 حداکثر معاملات باز همزمان
input int      InpMaxConsecutiveLoss  = 10;                  // ❌ حداکثر ضرر متوالی
input int      InpMinutesBetweenTrade = 10;                 // ⏳ فاصله زمانی بین معاملات (دقیقه)
input double   InpMaxDailyLossPercent = 5.0;                // 📉 حداکثر ضرر روزانه (%)
input double   InpMaxDailyProfitPercent = 10.0;             // 📈 حداکثر سود روزانه (%)

//+------------------------------------------------------------------+
//| 📌 بخش 4: محدودیت‌های زمانی - 11:30 تا 16:30 تهران            |
//+------------------------------------------------------------------+
input group "========== ⏰ TIME FILTERS (Tehran Time) =========="
input bool     InpUseTimeFilter       = true;               // 🕐 فعال‌سازی فیلتر زمانی
input int      InpStartHour           = 11;                 // 🌅 ساعت شروع (11)
input int      InpStartMinute         = 30;                 // 🌅 دقیقه شروع (30) = 11:30
input int      InpEndHour             = 22;                 // 🌆 ساعت پایان (16)
input int      InpEndMinute           = 30;                 // 🌆 دقیقه پایان (30) = 16:30
input bool     InpTradeMonday         = true;               // 📅 معامله در دوشنبه
input bool     InpTradeTuesday        = true;               // 📅 معامله در سه‌شنبه
input bool     InpTradeWednesday      = true;               // 📅 معامله در چهارشنبه
input bool     InpTradeThursday       = true;               // 📅 معامله در پنج‌شنبه
input bool     InpTradeFriday         = false;              // 📅 معامله در جمعه (غیرفعال)
input bool     InpAvoidNews           = true;               // 📰 اجتناب از زمان اخبار
input int      InpNewsBufferMinutes   = 30;                 // ⏱️ فاصله از اخبار (دقیقه) (دقیقه)


//+------------------------------------------------------------------+
//| 📌 بخش 5: سیستم SL پویا و هوشمند                               |
//+------------------------------------------------------------------+
input group "========== 🧠 DYNAMIC SMART STOP LOSS =========="
input bool     InpUseDynamicSL        = true;               // 🧠 فعال‌سازی SL پویا
input int      InpMinutesReanalysis   = 1;                  // ⏱️ بازه تحلیل مجدد (دقیقه)
input int      InpMinConfirmToHold    = 6;                  // 🔢 حداقل تایید برای نگه داشتن (از 25 استراتژی)
input double   InpMaxLossToHold       = 15.0;               // 💸 حداکثر ضرر برای نگه داشتن (پوینت)
input double   InpEmergencySL         = 30.0;               // 🚨 SL اضطراری (پوینت) - حد نهایی
input bool     InpCloseOnSignalChange = true;               // 🔄 بستن در صورت تغییر سیگنال
input int      InpMinScoreToHold      = 50;                 // 📊 حداقل امتیاز برای نگه داشتن


//+------------------------------------------------------------------+
//| 📌 متغیرهای سراسری - مدیریت وضعیت ربات                          |
//+------------------------------------------------------------------+

// 📊 متغیرهای آماری
int      g_TodayTrades          = 0;        // تعداد معاملات امروز
int      g_ConsecutiveLosses    = 0;        // تعداد ضررهای متوالی
int      g_ConsecutiveWins      = 0;        // تعداد سودهای متوالی
double   g_TodayProfit          = 0.0;      // سود/ضرر امروز
double   g_StartDayBalance      = 0.0;      // موجودی اول روز
datetime g_LastTradeTime        = 0;        // زمان آخرین معامله
datetime g_CurrentDay           = 0;        // روز جاری
int      g_TotalSignals         = 0;        // کل سیگنال‌های دریافتی

// 🎯 متغیرهای پوزیشن
double   g_CurrentTP            = 0.0;      // حد سود فعلی
double   g_CurrentSL            = 0.0;      // حد ضرر فعلی
ulong    g_CurrentTicket        = 0;        // تیکت پوزیشن فعلی


//--- متغیرهای SL پویا
datetime g_LastReanalysisTime   = 0;        // آخرین زمان تحلیل مجدد
int      g_ReanalysisCount      = 0;        // تعداد تحلیل‌های مجدد
bool     g_PositionUnderReview  = false;    // آیا پوزیشن در حال بررسی است؟


// 📏 متغیرهای محاسباتی نماد
double   g_Point;                           // مقدار Point نماد
double   g_TickSize;                        // اندازه Tick
double   g_TickValue;                       // ارزش Tick
int      g_Digits;                          // تعداد ارقام اعشار
double   g_MinLot;                          // حداقل Lot
double   g_MaxLot;                          // حداکثر Lot
double   g_LotStep;                         // گام Lot
double   g_StopLevel;                       // Stop Level

// 🎨 متغیرهای اندیکاتورها (15 اندیکاتور)
int      handle_EMA_Fast;                   // 1. EMA سریع
int      handle_EMA_Slow;                   // 2. EMA کند
int      handle_RSI;                        // 3. RSI
int      handle_Stoch;                      // 4. Stochastic
int      handle_MACD;                       // 5. MACD
int      handle_ATR;                        // 6. ATR
int      handle_BB;                         // 7. Bollinger Bands
int      handle_CCI;                        // 8. CCI
int      handle_ADX;                        // 9. ADX
int      handle_WPR;                        // 10. Williams %R
int      handle_MOM;                        // 11. Momentum
int      handle_SAR;                        // 12. Parabolic SAR
int      handle_OBV;                        // 13. OBV
int      handle_AO;                         // 14. Awesome Oscillator
int      handle_DeMarker;                   // 15. DeMarker

// 📊 بافرهای اندیکاتورها
double   buffer_EMA_Fast[];
double   buffer_EMA_Slow[];
double   buffer_RSI[];
double   buffer_Stoch_Main[];
double   buffer_Stoch_Signal[];
double   buffer_MACD_Main[];
double   buffer_MACD_Signal[];
double   buffer_ATR[];
double   buffer_BB_Upper[];
double   buffer_BB_Middle[];
double   buffer_BB_Lower[];
double   buffer_CCI[];
double   buffer_ADX_Main[];
double   buffer_ADX_Plus[];
double   buffer_ADX_Minus[];
double   buffer_WPR[];
double   buffer_MOM[];
double   buffer_SAR[];
double   buffer_OBV[];
double   buffer_AO[];
double   buffer_DeMarker[];

// 🎲 متغیرهای سیستم رای‌گیری
int      g_VoteBuy              = 0;        // رای خرید
int      g_VoteSell             = 0;        // رای فروش
int      g_VoteNeutral          = 0;        // رای خنثی
double   g_SignalStrength       = 0.0;      // قدرت سیگنال (0-100)



//+------------------------------------------------------------------+
//| 📊 اولیه‌سازی پارامترهای نماد                                    |
//+------------------------------------------------------------------+
bool InitSymbolParameters()
{
   //--- انتخاب نماد
   if(!symbolInfo.Name(InpTradeSymbol))
   {
      Print("❌ Symbol ", InpTradeSymbol, " not found!");
      return false;
   }
   
   //--- بررسی دسترسی به نماد
   if(!symbolInfo.Select())
   {
      Print("❌ Failed to select symbol: ", InpTradeSymbol);
      return false;
   }
   
   //--- رفرش کردن اطلاعات
   if(!symbolInfo.RefreshRates())
   {
      Print("❌ Failed to refresh rates for: ", InpTradeSymbol);
      return false;
   }
   
   //--- دریافت مشخصات نماد
   g_Point     = symbolInfo.Point();
   g_Digits    = (int)symbolInfo.Digits();
   g_TickSize  = symbolInfo.TickSize();
   g_TickValue = symbolInfo.TickValue();
   g_MinLot    = symbolInfo.LotsMin();
   g_MaxLot    = symbolInfo.LotsMax();
   g_LotStep   = symbolInfo.LotsStep();
   g_StopLevel = symbolInfo.StopsLevel() * g_Point;
   
   //--- نمایش اطلاعات نماد
   Print("📊 Symbol Information:");
   Print("   └─ Symbol: ", InpTradeSymbol);
   Print("   └─ Digits: ", g_Digits);
   Print("   └─ Point: ", g_Point);
   Print("   └─ Tick Size: ", g_TickSize);
   Print("   └─ Tick Value: ", g_TickValue);
   Print("   └─ Min Lot: ", g_MinLot);
   Print("   └─ Max Lot: ", g_MaxLot);
   Print("   └─ Lot Step: ", g_LotStep);
   Print("   └─ Stop Level: ", g_StopLevel);
   
   //--- بررسی حجم معاملات
   if(InpLotSize < g_MinLot || InpLotSize > g_MaxLot)
   {
      Print("❌ Invalid lot size! Min: ", g_MinLot, " Max: ", g_MaxLot);
      return false;
   }
   
   return true;
}

//+------------------------------------------------------------------+
//| 📈 اولیه‌سازی 15 اندیکاتور اسکلپینگ                             |
//+------------------------------------------------------------------+
bool InitIndicators()
{
   Print("📈 Initializing 15 Scalping Indicators...");
   
   //--- 1️⃣ EMA Fast (9)
   handle_EMA_Fast = iMA(InpTradeSymbol, InpMainTF, 9, 0, MODE_EMA, PRICE_CLOSE);
   if(handle_EMA_Fast == INVALID_HANDLE)
   {
      Print("❌ Failed to create EMA Fast indicator");
      return false;
   }
   
   //--- 2️⃣ EMA Slow (21)
   handle_EMA_Slow = iMA(InpTradeSymbol, InpMainTF, 21, 0, MODE_EMA, PRICE_CLOSE);
   if(handle_EMA_Slow == INVALID_HANDLE)
   {
      Print("❌ Failed to create EMA Slow indicator");
      return false;
   }
   
   //--- 3️⃣ RSI (14)
   handle_RSI = iRSI(InpTradeSymbol, InpMainTF, 14, PRICE_CLOSE);
   if(handle_RSI == INVALID_HANDLE)
   {
      Print("❌ Failed to create RSI indicator");
      return false;
   }
   
   //--- 4️⃣ Stochastic (5,3,3)
   handle_Stoch = iStochastic(InpTradeSymbol, InpMainTF, 5, 3, 3, MODE_SMA, STO_LOWHIGH);
   if(handle_Stoch == INVALID_HANDLE)
   {
      Print("❌ Failed to create Stochastic indicator");
      return false;
   }
   
   //--- 5️⃣ MACD (12,26,9)
   handle_MACD = iMACD(InpTradeSymbol, InpMainTF, 12, 26, 9, PRICE_CLOSE);
   if(handle_MACD == INVALID_HANDLE)
   {
      Print("❌ Failed to create MACD indicator");
      return false;
   }
   
   //--- 6️⃣ ATR (14)
   handle_ATR = iATR(InpTradeSymbol, InpMainTF, 14);
   if(handle_ATR == INVALID_HANDLE)
   {
      Print("❌ Failed to create ATR indicator");
      return false;
   }
   
   //--- 7️⃣ Bollinger Bands (20,2)
   handle_BB = iBands(InpTradeSymbol, InpMainTF, 20, 0, 2.0, PRICE_CLOSE);
   if(handle_BB == INVALID_HANDLE)
   {
      Print("❌ Failed to create Bollinger Bands indicator");
      return false;
   }
   
   //--- 8️⃣ CCI (14)
   handle_CCI = iCCI(InpTradeSymbol, InpMainTF, 14, PRICE_TYPICAL);
   if(handle_CCI == INVALID_HANDLE)
   {
      Print("❌ Failed to create CCI indicator");
      return false;
   }
   
   //--- 9️⃣ ADX (14)
   handle_ADX = iADX(InpTradeSymbol, InpMainTF, 14);
   if(handle_ADX == INVALID_HANDLE)
   {
      Print("❌ Failed to create ADX indicator");
      return false;
   }
   
   //--- 🔟 Williams %R (14)
   handle_WPR = iWPR(InpTradeSymbol, InpMainTF, 14);
   if(handle_WPR == INVALID_HANDLE)
   {
      Print("❌ Failed to create Williams %R indicator");
      return false;
   }
   
   //--- 1️⃣1️⃣ Momentum (14)
   handle_MOM = iMomentum(InpTradeSymbol, InpMainTF, 14, PRICE_CLOSE);
   if(handle_MOM == INVALID_HANDLE)
   {
      Print("❌ Failed to create Momentum indicator");
      return false;
   }
   
   //--- 1️⃣2️⃣ Parabolic SAR (0.02, 0.2)
   handle_SAR = iSAR(InpTradeSymbol, InpMainTF, 0.02, 0.2);
   if(handle_SAR == INVALID_HANDLE)
   {
      Print("❌ Failed to create Parabolic SAR indicator");
      return false;
   }
   
   //--- 1️⃣3️⃣ OBV
   handle_OBV = iOBV(InpTradeSymbol, InpMainTF, VOLUME_TICK);
   if(handle_OBV == INVALID_HANDLE)
   {
      Print("❌ Failed to create OBV indicator");
      return false;
   }
   
   //--- 1️⃣4️⃣ Awesome Oscillator
   handle_AO = iAO(InpTradeSymbol, InpMainTF);
   if(handle_AO == INVALID_HANDLE)
   {
      Print("❌ Failed to create Awesome Oscillator indicator");
      return false;
   }
   
   //--- 1️⃣5️⃣ DeMarker (14)
   handle_DeMarker = iDeMarker(InpTradeSymbol, InpMainTF, 14);
   if(handle_DeMarker == INVALID_HANDLE)
   {
      Print("❌ Failed to create DeMarker indicator");
      return false;
   }
   
   Print("✅ All 15 indicators initialized successfully!");
   return true;
}

//+------------------------------------------------------------------+
//| 🎨 تنظیم بافرهای اندیکاتورها به عنوان سری زمانی                 |
//+------------------------------------------------------------------+
bool SetBufferArrays()
{
   //--- تنظیم به عنوان سری زمانی (جدیدترین داده در ایندکس 0)
   ArraySetAsSeries(buffer_EMA_Fast, true);
   ArraySetAsSeries(buffer_EMA_Slow, true);
   ArraySetAsSeries(buffer_RSI, true);
   ArraySetAsSeries(buffer_Stoch_Main, true);
   ArraySetAsSeries(buffer_Stoch_Signal, true);
   ArraySetAsSeries(buffer_MACD_Main, true);
   ArraySetAsSeries(buffer_MACD_Signal, true);
   ArraySetAsSeries(buffer_ATR, true);
   ArraySetAsSeries(buffer_BB_Upper, true);
   ArraySetAsSeries(buffer_BB_Middle, true);
   ArraySetAsSeries(buffer_BB_Lower, true);
   ArraySetAsSeries(buffer_CCI, true);
   ArraySetAsSeries(buffer_ADX_Main, true);
   ArraySetAsSeries(buffer_ADX_Plus, true);
   ArraySetAsSeries(buffer_ADX_Minus, true);
   ArraySetAsSeries(buffer_WPR, true);
   ArraySetAsSeries(buffer_MOM, true);
   ArraySetAsSeries(buffer_SAR, true);
   ArraySetAsSeries(buffer_OBV, true);
   ArraySetAsSeries(buffer_AO, true);
   ArraySetAsSeries(buffer_DeMarker, true);
   
   Print("✅ All indicator buffers set as time series");
   return true;
}

//+------------------------------------------------------------------+
//| ✔️ اعتبارسنجی پارامترهای ورودی                                  |
//+------------------------------------------------------------------+
bool ValidateInputParameters()
{
   bool isValid = true;
   
   //--- بررسی حجم معاملات
   if(InpLotSize <= 0)
   {
      Print("❌ Lot size must be greater than 0");
      isValid = false;
   }
   
   //--- بررسی TP و SL
   if(InpTakeProfitPoint <= 0 || InpStopLossPoint <= 0)
   {
      Print("❌ TP and SL must be greater than 0");
      isValid = false;
   }
   
   //--- بررسی محدودیت‌های معاملات
   if(InpMaxTradesPerDay <= 0 || InpMaxTradesPerDay > 100)
   {
      Print("❌ Max trades per day must be between 1-100");
      isValid = false;
   }
   
   if(InpMaxOpenTrades <= 0 || InpMaxOpenTrades > 10)
   {
      Print("❌ Max open trades must be between 1-10");
      isValid = false;
   }
   
   if(InpMaxConsecutiveLoss <= 0)
   {
      Print("❌ Max consecutive losses must be greater than 0");
      isValid = false;
   }
   
   //--- بررسی فاصله زمانی
   if(InpMinutesBetweenTrade < 0)
   {
      Print("❌ Minutes between trades cannot be negative");
      isValid = false;
   }
   
   return isValid;
}

//+------------------------------------------------------------------+
//| 🔄 ریست آمار روزانه                                              |
//+------------------------------------------------------------------+
void ResetDailyStats()
{
   g_TodayTrades = 0;
   g_ConsecutiveLosses = 0;
   g_ConsecutiveWins = 0;
   g_TodayProfit = 0.0;
   g_LastTradeTime = 0;
   g_TotalSignals = 0;
   
   Print("🔄 Daily statistics reset");
}

//+------------------------------------------------------------------+
//| 📋 نمایش اطلاعات اولیه‌سازی                                      |
//+------------------------------------------------------------------+
void PrintInitInfo()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("📊 TRADING CONFIGURATION:");
   Print("   ├─ Symbol: ", InpTradeSymbol);
   Print("   ├─ Main Timeframe: ", EnumToString(InpMainTF));
   Print("   ├─ Helper Timeframe: ", EnumToString(InpHelperTF));
   Print("   ├─ Lot Size: ", InpLotSize);
   Print("   ├─ Take Profit: ", InpTakeProfitPoint, " points");
   Print("   ├─ Stop Loss: ", InpStopLossPoint, " points");
   Print("   ├─ Max Trades/Day: ", InpMaxTradesPerDay);
   Print("   ├─ Max Open Trades: ", InpMaxOpenTrades);
   Print("   ├─ Max Consecutive Loss: ", InpMaxConsecutiveLoss);
   Print("   └─ Minutes Between Trades: ", InpMinutesBetweenTrade);
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
}



//+------------------------------------------------------------------+
//| 🛡️ بخش مدیریت ریسک - RISK MANAGEMENT SYSTEM                    |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| ✅ بررسی امکان باز کردن معامله جدید                             |
//+------------------------------------------------------------------+
bool CanOpenNewTrade()
{
   //--- بررسی 1: محدودیت تعداد معاملات روزانه
   if(g_TodayTrades >= InpMaxTradesPerDay)
   {
      Print("⛔ Daily trade limit reached: ", g_TodayTrades, "/", InpMaxTradesPerDay);
      return false;
   }
   
   //--- بررسی 2: محدودیت معاملات باز همزمان
   if(CountOpenPositions() >= InpMaxOpenTrades)
   {
      Print("⛔ Max open positions reached: ", CountOpenPositions(), "/", InpMaxOpenTrades);
      return false;
   }
   
   //--- بررسی 3: محدودیت ضررهای متوالی
   if(g_ConsecutiveLosses >= InpMaxConsecutiveLoss)
   {
      Print("⛔ Max consecutive losses reached: ", g_ConsecutiveLosses, "/", InpMaxConsecutiveLoss);
      return false;
   }
   
   //--- بررسی 4: فاصله زمانی بین معاملات
   if(!CheckTimeBetweenTrades())
   {
      int minutesPassed = (int)((TimeCurrent() - g_LastTradeTime) / 60);
      Print("⛔ Time filter: Need ", InpMinutesBetweenTrade, " minutes, passed: ", minutesPassed);
      return false;
   }
   
   //--- بررسی 5: محدودیت سود/ضرر روزانه
   if(!CheckDailyProfitLoss())
   {
      return false;
   }
   
   //--- بررسی 6: فیلتر زمانی
   if(!CheckTimeFilter())
   {
      Print("⛔ Time filter: Trading not allowed at this time");
      return false;
   }
   
   //--- بررسی 7: روز جدید - ریست آمار
   CheckAndResetDailyStats();
   
   //--- بررسی 8: وضعیت حساب
   if(!CheckAccountStatus())
   {
      return false;
   }
   
   return true;
}

//+------------------------------------------------------------------+
//| 🔢 شمارش پوزیشن‌های باز                                          |
//+------------------------------------------------------------------+
int CountOpenPositions()
{
   int count = 0;
   
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(position.SelectByIndex(i))
      {
         if(position.Symbol() == InpTradeSymbol && 
            position.Magic() == InpMagicNumber)
         {
            count++;
         }
      }
   }
   
   return count;
}

//+------------------------------------------------------------------+
//| ⏰ بررسی فاصله زمانی بین معاملات                                |
//+------------------------------------------------------------------+
bool CheckTimeBetweenTrades()
{
   if(g_LastTradeTime == 0)
      return true;
   
   datetime currentTime = TimeCurrent();
   int minutesPassed = (int)((currentTime - g_LastTradeTime) / 60);
   
   return (minutesPassed >= InpMinutesBetweenTrade);
}

//+------------------------------------------------------------------+
//| 💰 بررسی محدودیت سود/ضرر روزانه                                 |
//+------------------------------------------------------------------+
bool CheckDailyProfitLoss()
{
   double startBalance = g_StartDayBalance;
   if(startBalance <= 0)
      startBalance = accountInfo.Balance();
   
   double currentProfit = g_TodayProfit;
   double profitPercent = (currentProfit / startBalance) * 100.0;
   
   //--- بررسی حداکثر سود روزانه
   if(InpMaxDailyProfitPercent > 0 && profitPercent >= InpMaxDailyProfitPercent)
   {
      Print("🎯 Daily profit target reached: ", DoubleToString(profitPercent, 2), "%");
      Print("💰 Profit: $", DoubleToString(currentProfit, 2));
      return false;
   }
   
   //--- بررسی حداکثر ضرر روزانه
   if(InpMaxDailyLossPercent > 0 && profitPercent <= -InpMaxDailyLossPercent)
   {
      Print("🛑 Daily loss limit reached: ", DoubleToString(profitPercent, 2), "%");
      Print("💸 Loss: $", DoubleToString(currentProfit, 2));
      return false;
   }
   
   return true;
}

//+------------------------------------------------------------------+
//| 🏦 بررسی وضعیت حساب                                              |
//+------------------------------------------------------------------+
bool CheckAccountStatus()
{
   //--- بررسی اتصال به سرور
   if(!TerminalInfoInteger(TERMINAL_CONNECTED))
   {
      Print("❌ No connection to trade server");
      return false;
   }
   
   //--- بررسی امکان معامله
   if(!TerminalInfoInteger(TERMINAL_TRADE_ALLOWED))
   {
      Print("❌ AutoTrading is disabled");
      return false;
   }
   
   //--- بررسی مجوز معامله برای حساب
   if(!accountInfo.TradeAllowed())
   {
      Print("❌ Trading is not allowed for this account");
      return false;
   }
   
   //--- بررسی امکان معامله برای EA
   if(!MQLInfoInteger(MQL_TRADE_ALLOWED))
   {
      Print("❌ EA trading is not allowed");
      return false;
   }
   
   //--- بررسی موجودی
   double balance = accountInfo.Balance();
   double equity = accountInfo.Equity();
   double margin_free = accountInfo.FreeMargin();
   
   if(balance <= 0)
   {
      Print("❌ Invalid account balance");
      return false;
   }
   
   //--- بررسی مارجین آزاد
   double requiredMargin = 0;
   if(!OrderCalcMargin(ORDER_TYPE_BUY, InpTradeSymbol, InpLotSize, 
                       symbolInfo.Ask(), requiredMargin))
   {
      Print("❌ Failed to calculate required margin");
      return false;
   }
   
   if(margin_free < requiredMargin * 2) // حداقل 2 برابر مارجین مورد نیاز
   {
      Print("❌ Insufficient free margin. Required: ", requiredMargin * 2, 
            " Available: ", margin_free);
      return false;
   }
   
   return true;
}


//+------------------------------------------------------------------+
//| 🔄 تحلیل مجدد سیگنال برای پوزیشن باز                            |
//+------------------------------------------------------------------+
bool ReanalyzeSignalForOpenPosition(ENUM_POSITION_TYPE positionType)
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🔄 RE-ANALYZING SIGNAL FOR OPEN POSITION...");
   Print("   └─ Position Type: ", EnumToString(positionType));
   
   //--- به‌روزرسانی اندیکاتورها
   if(!UpdateAllIndicators())
   {
      Print("⚠️ Failed to update indicators for reanalysis");
      return false;
   }
   
   //--- اجرای استراتژی‌ها
   if(!ExecuteAllStrategies())
   {
      Print("⚠️ Failed to execute strategies for reanalysis");
      return false;
   }
   
   //--- سیستم رای‌گیری (بدون فیلترهای امنیتی - فقط برای بررسی)
   VoteLayer1_Indicators();
   VoteLayer2_Strategies();
   
   //--- محاسبه نتیجه
   CalculateStrategyResults();
   FinalDecisionAlgorithm();
   
   //--- تعیین سیگنال مورد نیاز بر اساس نوع پوزیشن
   ENUM_SIGNAL_TYPE requiredSignal;
   
   if(positionType == POSITION_TYPE_BUY)
      requiredSignal = SIGNAL_BUY;
   else if(positionType == POSITION_TYPE_SELL)
      requiredSignal = SIGNAL_SELL;
   else
      return false;
   
   //--- شمارش تایید استراتژی‌ها
   int confirmations = 0;
   
   if(requiredSignal == SIGNAL_BUY)
      confirmations = g_VotingResult.strategyBuyVotes;
   else
      confirmations = g_VotingResult.strategySellVotes;
   
   //--- نمایش نتایج
   Print("📊 REANALYSIS RESULTS:");
   Print("   ├─ Required Signal: ", EnumToString(requiredSignal));
   Print("   ├─ Final Decision: ", EnumToString(g_VotingResult.finalDecision));
   Print("   ├─ Confirmations: ", confirmations, "/25 strategies");
   Print("   ├─ Final Score: ", DoubleToString(g_VotingResult.finalScore, 2));
   Print("   └─ Consensus: ", DoubleToString(g_VotingResult.consensusLevel, 2), "%");
   
   //--- بررسی شرایط نگه داشتن
   bool shouldHold = false;
   string reason = "";
   
   // شرط 1: سیگنال نهایی همچنان همان جهت است
   if(g_VotingResult.finalDecision == requiredSignal)
   {
      shouldHold = true;
      reason = "Final decision matches position";
   }
   // شرط 2: حداقل تعداد مشخصی استراتژی تایید می‌کنند
   else if(confirmations >= InpMinConfirmToHold)
   {
      shouldHold = true;
      reason = StringFormat("%d strategies still confirm", confirmations);
   }
   // شرط 3: امتیاز بالا است
   else if(g_VotingResult.finalScore >= InpMinScoreToHold && 
           g_VotingResult.finalDecision != SIGNAL_NEUTRAL)
   {
      shouldHold = true;
      reason = StringFormat("High score (%.1f) - giving more time", g_VotingResult.finalScore);
   }
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   if(shouldHold)
   {
      Print("✅ DECISION: HOLD POSITION");
      Print("   └─ Reason: ", reason);
   }
   else
   {
      Print("❌ DECISION: SIGNAL CHANGED - CLOSE RECOMMENDED");
      Print("   └─ Reason: Insufficient confirmations or signal reversed");
   }
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   return shouldHold;
}


//+------------------------------------------------------------------+
//| ⏰ بخش فیلترهای زمانی - TIME FILTERS                            |
//+------------------------------------------------------------------+


//+------------------------------------------------------------------+
//| 📅 دریافت نام روز - تابع کمکی                                  |
//+------------------------------------------------------------------+
string GetDayName(int dayOfWeek)
{
   switch(dayOfWeek)
   {
      case 0: return "Sunday";
      case 1: return "Monday";
      case 2: return "Tuesday";
      case 3: return "Wednesday";
      case 4: return "Thursday";
      case 5: return "Friday";
      case 6: return "Saturday";
      default: return "Unknown";
   }
}


//+------------------------------------------------------------------+
//| 🕐 بررسی فیلتر زمانی اصلی - 11:30 تا 16:30 تهران              |
//+------------------------------------------------------------------+
bool CheckTimeFilter()
{
   if(!InpUseTimeFilter)
      return true;
   
   MqlDateTime dt;
   TimeCurrent(dt);
   
   //--- بررسی ساعت معاملاتی (با دقیقه)
   if(!CheckTradingHours(dt.hour, dt.min))
   {
      // فقط یک بار در هر ساعت پیام بده (تا لاگ شلوغ نشه)
      static int lastPrintHour = -1;
      if(dt.hour != lastPrintHour)
      {
         Print("⏰ Outside trading hours!");
         Print("   ├─ Current Time: ", dt.hour, ":", 
               (dt.min < 10 ? "0" : ""), dt.min, " Tehran");
         Print("   └─ Allowed: ", InpStartHour, ":", 
               (InpStartMinute < 10 ? "0" : ""), InpStartMinute, 
               " - ", InpEndHour, ":", 
               (InpEndMinute < 10 ? "0" : ""), InpEndMinute);
         lastPrintHour = dt.hour;
      }
      return false;
   }
   
   //--- بررسی روز معاملاتی
   if(!CheckTradingDay(dt.day_of_week))
   {
      static int lastPrintDay = -1;
      if(dt.day_of_week != lastPrintDay)
      {
         Print("⏰ Trading not allowed on this day!");
         Print("   └─ Current Day: ", GetDayName(dt.day_of_week));
         lastPrintDay = dt.day_of_week;
      }
      return false;
   }
   
   //--- بررسی اخبار
   if(InpAvoidNews && !CheckNewsFilter())
   {
      return false;
   }
   
   return true;
}

//+------------------------------------------------------------------+
//| 🌅 بررسی ساعات معاملاتی - با دقیقه (11:30 - 16:30 تهران)     |
//+------------------------------------------------------------------+
bool CheckTradingHours(int current_hour, int current_minute)
{
   //--- تبدیل به دقیقه برای مقایسه راحت‌تر
   int currentTimeInMinutes = current_hour * 60 + current_minute;
   int startTimeInMinutes = InpStartHour * 60 + InpStartMinute;
   int endTimeInMinutes = InpEndHour * 60 + InpEndMinute;
   
   //--- بررسی بازه زمانی
   if(startTimeInMinutes <= endTimeInMinutes)
   {
      // بازه عادی (مثل 11:30 - 16:30)
      if(currentTimeInMinutes >= startTimeInMinutes && 
         currentTimeInMinutes <= endTimeInMinutes)
      {
         return true;
      }
      else
      {
         return false;
      }
   }
   else
   {
      // بازه که از نیمه‌شب عبور می‌کند (مثل 22:00 - 02:00)
      if(currentTimeInMinutes >= startTimeInMinutes || 
         currentTimeInMinutes <= endTimeInMinutes)
      {
         return true;
      }
      else
      {
         return false;
      }
   }
}
//+------------------------------------------------------------------+
//| 📅 بررسی روزهای معاملاتی                                        |
//+------------------------------------------------------------------+
bool CheckTradingDay(int day_of_week)
{
   switch(day_of_week)
   {
      case 1: return InpTradeMonday;      // دوشنبه
      case 2: return InpTradeTuesday;     // سه‌شنبه
      case 3: return InpTradeWednesday;   // چهارشنبه
      case 4: return InpTradeThursday;    // پنج‌شنبه
      case 5: return InpTradeFriday;      // جمعه
      default: return false;              // شنبه و یکشنبه
   }
}

//+------------------------------------------------------------------+
//| 📰 فیلتر اخبار                                                   |
//+------------------------------------------------------------------+
bool CheckNewsFilter()
{
   //--- این تابع می‌تواند با تقویم اقتصادی ادغام شود
   //--- برای سادگی، فعلاً یک پیاده‌سازی ساده انجام می‌دهیم
   
   MqlDateTime dt;
   TimeCurrent(dt);
   
   //--- اجتناب از معامله در 30 دقیقه اول و آخر ساعات کاری مهم
   int criticalHours[] = {8, 12, 14, 16, 20}; // ساعات مهم اخبار
   
   for(int i = 0; i < ArraySize(criticalHours); i++)
   {
      if(dt.hour == criticalHours[i])
      {
         if(dt.min < InpNewsBufferMinutes || dt.min > (60 - InpNewsBufferMinutes))
         {
            Print("📰 News filter active - avoiding trading near critical hours");
            return false;
         }
      }
   }
   
   return true;
}

//+------------------------------------------------------------------+
//| 🔄 بررسی و ریست آمار روزانه                                     |
//+------------------------------------------------------------------+
void CheckAndResetDailyStats()
{
   MqlDateTime dt_current, dt_last;
   TimeCurrent(dt_current);
   TimeToStruct(g_CurrentDay, dt_last);
   
   //--- اگر روز عوض شده
   if(dt_current.day != dt_last.day || 
      dt_current.mon != dt_last.mon || 
      dt_current.year != dt_last.year)
   {
      Print("═══════════════════════════════════════════════════════");
      Print("📅 New Trading Day: ", dt_current.year, ".", dt_current.mon, ".", dt_current.day);
      Print("📊 Previous Day Statistics:");
      Print("   ├─ Total Trades: ", g_TodayTrades);
      Print("   ├─ Profit/Loss: $", DoubleToString(g_TodayProfit, 2));
      Print("   ├─ Win Streak: ", g_ConsecutiveWins);
      Print("   └─ Loss Streak: ", g_ConsecutiveLosses);
      Print("═══════════════════════════════════════════════════════");
      
      //--- ریست آمار
      ResetDailyStats();
      g_StartDayBalance = accountInfo.Balance();
      g_CurrentDay = TimeCurrent();
   }
}

//+------------------------------------------------------------------+
//| 📊 بخش محاسبات آماری - STATISTICS SYSTEM                        |
//+------------------------------------------------------------------+


//+------------------------------------------------------------------+
//| 📈 به‌روزرسانی آمار - فقط برای معاملات واقعی                   |
//+------------------------------------------------------------------+
void UpdateTradeStatistics(double profit)
{
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║         📊 UPDATING TRADE STATISTICS                  ║");
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Previous Trades: %d                                 ║", g_TodayTrades));
   
   //--- افزایش شمارنده
   g_TodayTrades++;
   g_TodayProfit += profit;
   
   Print(StringFormat("║ Current Trades: %d                                  ║", g_TodayTrades));
   Print(StringFormat("║ This Trade P/L: $%.2f                              ║", profit));
   Print(StringFormat("║ Total Daily P/L: $%.2f                             ║", g_TodayProfit));
   
   //--- به‌روزرسانی برد/باخت
   if(profit > 0.01) // بیشتر از 1 سنت = سود
   {
      g_ConsecutiveWins++;
      g_ConsecutiveLosses = 0;
      
      Print("╠════════════════════════════════════════════════════════╣");
      Print("║ ✅ ✅ ✅ PROFITABLE TRADE! ✅ ✅ ✅                    ║");
      Print(StringFormat("║ Consecutive Wins: %d                                ║", g_ConsecutiveWins));
      Print("╚════════════════════════════════════════════════════════╝");
   }
   else if(profit < -0.01) // کمتر از -1 سنت = ضرر
   {
      g_ConsecutiveLosses++;
      g_ConsecutiveWins = 0;
      
      Print("╠════════════════════════════════════════════════════════╣");
      Print("║ ❌ ❌ ❌ LOSS TRADE! ❌ ❌ ❌                          ║");
      Print(StringFormat("║ Consecutive Losses: %d                              ║", g_ConsecutiveLosses));
      
      if(g_ConsecutiveLosses >= InpMaxConsecutiveLoss - 1)
      {
         Print("║ ⚠️⚠️⚠️ WARNING: ONE MORE LOSS = HALT! ⚠️⚠️⚠️        ║");
      }
      
      Print("╚════════════════════════════════════════════════════════╝");
   }
   else
   {
      Print("╠════════════════════════════════════════════════════════╣");
      Print("║ ⚪ BREAK EVEN TRADE                                   ║");
      Print("╚════════════════════════════════════════════════════════╝");
   }
   
   //--- نمایش پیشرفت
   double progress = ((double)g_TodayTrades / InpMaxTradesPerDay) * 100;
   Print("");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print(StringFormat("📊 Daily Progress: %.1f%% (%d/%d trades)", 
                     progress, g_TodayTrades, InpMaxTradesPerDay));
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("");
}


//+------------------------------------------------------------------+
//| 📋 نمایش آمار فعلی                                               |
//+------------------------------------------------------------------+
void PrintCurrentStatistics()
{
   double profitPercent = 0;
   if(g_StartDayBalance > 0)
      profitPercent = (g_TodayProfit / g_StartDayBalance) * 100.0;
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("📊 Current Statistics:");
   Print("   ├─ Trades Today: ", g_TodayTrades, "/", InpMaxTradesPerDay);
   Print("   ├─ Daily P/L: $", DoubleToString(g_TodayProfit, 2), 
         " (", DoubleToString(profitPercent, 2), "%)");
   Print("   ├─ Win Streak: ", g_ConsecutiveWins);
   Print("   ├─ Loss Streak: ", g_ConsecutiveLosses);
   Print("   ├─ Account Balance: $", DoubleToString(accountInfo.Balance(), 2));
   Print("   └─ Account Equity: $", DoubleToString(accountInfo.Equity(), 2));
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
}

//+------------------------------------------------------------------+
//| 💹 محاسبه سود/ضرر احتمالی                                       |
//+------------------------------------------------------------------+
double CalculateExpectedProfit(double lotSize, double points)
{
   double tickValue = symbolInfo.TickValue();
   double tickSize = symbolInfo.TickSize();
   double point = symbolInfo.Point();
   
   // محاسبه سود بر اساس پوینت
   double profit = (points * point / tickSize) * tickValue * lotSize;
   
   return profit;
}

//+------------------------------------------------------------------+
//| 🎯 محاسبه نسبت ریسک به ریوارد                                   |
//+------------------------------------------------------------------+
double CalculateRiskRewardRatio()
{
   if(InpStopLossPoint == 0)
      return 0;
   
   return (InpTakeProfitPoint / InpStopLossPoint);
}

//+------------------------------------------------------------------+
//| 📊 دریافت آمار کامل روز                                         |
//+------------------------------------------------------------------+
string GetDailyStatsString()
{
   string stats = "";
   
   stats += "╔════════════════════════════════════╗\n";
   stats += "║     📊 DAILY STATISTICS            ║\n";
   stats += "╠════════════════════════════════════╣\n";
   stats += StringFormat("║ Trades: %d/%d                     ║\n", 
                         g_TodayTrades, InpMaxTradesPerDay);
   stats += StringFormat("║ Profit: $%.2f                    ║\n", 
                         g_TodayProfit);
   stats += StringFormat("║ Win Streak: %d                    ║\n", 
                         g_ConsecutiveWins);
   stats += StringFormat("║ Loss Streak: %d                   ║\n", 
                         g_ConsecutiveLosses);
   stats += StringFormat("║ Balance: $%.2f                   ║\n", 
                         accountInfo.Balance());
   stats += "╚════════════════════════════════════╝";
   
   return stats;
}


//+------------------------------------------------------------------+
//| 🧮 بخش توابع کمکی محاسباتی - CALCULATION HELPERS               |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 📏 نرمال‌سازی حجم معامله                                         |
//+------------------------------------------------------------------+
double NormalizeLot(double lot)
{
   double minLot = symbolInfo.LotsMin();
   double maxLot = symbolInfo.LotsMax();
   double lotStep = symbolInfo.LotsStep();
   
   //--- محدود کردن به رنج مجاز
   if(lot < minLot)
      lot = minLot;
   if(lot > maxLot)
      lot = maxLot;
   
   //--- گرد کردن به نزدیکترین LotStep
   lot = MathRound(lot / lotStep) * lotStep;
   
   //--- نرمال‌سازی اعشار
   int lotDigits = 2;
   if(lotStep >= 0.1)
      lotDigits = 1;
   if(lotStep >= 1.0)
      lotDigits = 0;
   
   lot = NormalizeDouble(lot, lotDigits);
   
   return lot;
}

//+------------------------------------------------------------------+
//| 💲 نرمال‌سازی قیمت                                              |
//+------------------------------------------------------------------+
double NormalizePrice(double price)
{
   return NormalizeDouble(price, g_Digits);
}

//+------------------------------------------------------------------+
//| 📊 محاسبه قیمت Take Profit                                      |
//+------------------------------------------------------------------+
double CalculateTakeProfit(ENUM_ORDER_TYPE orderType, double openPrice)
{
   double tp = 0;
   double points = InpTakeProfitPoint * g_Point * 10; // ضرب در 10 برای تبدیل به واحد واقعی
   
   if(orderType == ORDER_TYPE_BUY)
      tp = openPrice + points;
   else if(orderType == ORDER_TYPE_SELL)
      tp = openPrice - points;
   
   return NormalizePrice(tp);
}

//+------------------------------------------------------------------+
//| 🛑 محاسبه قیمت Stop Loss                                        |
//+------------------------------------------------------------------+
double CalculateStopLoss(ENUM_ORDER_TYPE orderType, double openPrice)
{
   double sl = 0;
   double points = InpStopLossPoint * g_Point * 10; // ضرب در 10 برای تبدیل به واحد واقعی
   
   if(orderType == ORDER_TYPE_BUY)
      sl = openPrice - points;
   else if(orderType == ORDER_TYPE_SELL)
      sl = openPrice + points;
   
   return NormalizePrice(sl);
}

//+------------------------------------------------------------------+
//| ✔️ بررسی اعتبار سطوح SL/TP                                      |
//+------------------------------------------------------------------+
bool ValidateStopLevels(ENUM_ORDER_TYPE orderType, double price, double sl, double tp)
{
   double stopLevel = g_StopLevel;
   double currentPrice = (orderType == ORDER_TYPE_BUY) ? symbolInfo.Ask() : symbolInfo.Bid();
   
   //--- بررسی Stop Loss
   if(sl > 0)
   {
      double slDistance = MathAbs(currentPrice - sl);
      if(slDistance < stopLevel)
      {
         Print("❌ Stop Loss too close. Required: ", stopLevel, " Current: ", slDistance);
         return false;
      }
   }
   
   //--- بررسی Take Profit
   if(tp > 0)
   {
      double tpDistance = MathAbs(currentPrice - tp);
      if(tpDistance < stopLevel)
      {
         Print("❌ Take Profit too close. Required: ", stopLevel, " Current: ", tpDistance);
         return false;
      }
   }
   
   return true;
}


//+------------------------------------------------------------------+
//| 🧠 مدیریت هوشمند پوزیشن با SL پویا                             |
//+------------------------------------------------------------------+
void ManageSmartDynamicSL()
{
   if(!InpUseDynamicSL)
      return;
   
   //--- بررسی زمان تحلیل مجدد
   datetime currentTime = TimeCurrent();
   int secondsSinceLastCheck = (int)(currentTime - g_LastReanalysisTime);
   int requiredSeconds = InpMinutesReanalysis * 60;
   
   if(secondsSinceLastCheck < requiredSeconds)
      return;
   
   g_LastReanalysisTime = currentTime;
   
   //--- بررسی هر پوزیشن باز
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!position.SelectByIndex(i))
         continue;
      
      if(position.Symbol() != InpTradeSymbol)
         continue;
      
      if(position.Magic() != InpMagicNumber)
         continue;
      
      //--- دریافت اطلاعات پوزیشن
      ulong ticket = position.Ticket();
      ENUM_POSITION_TYPE posType = position.PositionType();
      double openPrice = position.PriceOpen();
      double currentSL = position.StopLoss();
      double currentTP = position.TakeProfit();
      double currentProfit = position.Profit();
      
      double currentPrice = (posType == POSITION_TYPE_BUY) ? symbolInfo.Bid() : symbolInfo.Ask();
      
      //--- محاسبه ضرر به پوینت
      double lossInPoints = 0;
      
      if(posType == POSITION_TYPE_BUY)
         lossInPoints = (openPrice - currentPrice) / g_Point;
      else
         lossInPoints = (currentPrice - openPrice) / g_Point;
      
      //--- اگر در سود است، فقط Trailing Stop
      if(currentProfit > 0)
      {
         Print("💰 Position in profit - using trailing stop");
         ManageTrailingStop();
         continue;
      }
      
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║        🧠 SMART SL MANAGEMENT - POSITION #", ticket, "        ║");
      Print("╠════════════════════════════════════════════════════════╣");
      Print(StringFormat("║ Type: %-47s ║", EnumToString(posType)));
      Print(StringFormat("║ Open Price: %-40s ║", DoubleToString(openPrice, g_Digits)));
      Print(StringFormat("║ Current Price: %-37s ║", DoubleToString(currentPrice, g_Digits)));
      Print(StringFormat("║ Loss: %.2f points ($%.2f)                           ║", 
                        lossInPoints, currentProfit));
      Print("╚════════════════════════════════════════════════════════╝");
      
      //--- بررسی SL اضطراری
      if(lossInPoints > InpEmergencySL)
      {
         Print("🚨 EMERGENCY SL HIT - CLOSING POSITION!");
         Print("   └─ Loss exceeded emergency limit: ", lossInPoints, " > ", InpEmergencySL);
         
         if(trade.PositionClose(ticket))
         {
            Print("✅ Position closed by emergency SL");
         }
         else
         {
            Print("❌ Failed to close position - Error: ", trade.ResultRetcode());
         }
         continue;
      }
      
      //--- بررسی حد نگه داشتن
      if(lossInPoints > InpMaxLossToHold)
      {
         Print("⚠️ Loss exceeds hold limit (", lossInPoints, " > ", InpMaxLossToHold, ")");
         Print("   └─ Closing position for safety");
         
         if(trade.PositionClose(ticket))
         {
            Print("✅ Position closed - max loss to hold exceeded");
         }
         else
         {
            Print("❌ Failed to close position - Error: ", trade.ResultRetcode());
         }
         continue;
      }
      
      //--- تحلیل مجدد
      g_ReanalysisCount++;
      
      Print("🔄 Reanalysis #", g_ReanalysisCount, " - Checking if signal still valid...");
      
      bool shouldHold = ReanalyzeSignalForOpenPosition(posType);
      
      if(shouldHold)
      {
         Print("✅ HOLDING POSITION - Signal still supports this trade");
         Print("   └─ Will recheck in ", InpMinutesReanalysis, " minute(s)");
      }
      else
      {
         if(InpCloseOnSignalChange)
         {
            Print("❌ CLOSING POSITION - Signal changed or weakened");
            
            if(trade.PositionClose(ticket))
            {
               Print("✅ Position closed due to signal change");
               Print("   ├─ Loss: $", DoubleToString(currentProfit, 2));
               Print("   └─ Reason: Smart SL - Signal no longer valid");
            }
            else
            {
               Print("❌ Failed to close position - Error: ", trade.ResultRetcode());
            }
         }
         else
         {
            Print("⚠️ Signal changed but CloseOnSignalChange = false");
            Print("   └─ Keeping position open (relying on hard SL)");
         }
      }
   }
}


//+------------------------------------------------------------------+
//| 🎲 محاسبه اسپرد فعلی                                            |
//+------------------------------------------------------------------+
double GetCurrentSpread()
{
   return (symbolInfo.Ask() - symbolInfo.Bid()) / g_Point;
}

//+------------------------------------------------------------------+
//| ⚠️ بررسی اسپرد                                                  |
//+------------------------------------------------------------------+
bool CheckSpread(double maxSpreadPoints = 50)
{
   double currentSpread = GetCurrentSpread();
   
   if(currentSpread > maxSpreadPoints)
   {
      Print("⚠️ High spread detected: ", DoubleToString(currentSpread, 1), 
            " points (Max: ", maxSpreadPoints, ")");
      return false;
   }
   
   return true;
}


//+------------------------------------------------------------------+
//| 📊 بخش ساختار داده اندیکاتورها - INDICATOR DATA STRUCTURES      |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 🎯 انواع سیگنال                                                  |
//+------------------------------------------------------------------+
enum ENUM_SIGNAL_TYPE
{
   SIGNAL_NONE = 0,      // بدون سیگنال
   SIGNAL_BUY = 1,       // سیگنال خرید
   SIGNAL_SELL = -1,     // سیگنال فروش
   SIGNAL_NEUTRAL = 2    // خنثی
};

//+------------------------------------------------------------------+
//| 📋 ساختار داده سیگنال اندیکاتور                                  |
//+------------------------------------------------------------------+
struct IndicatorSignal
{
   string            name;           // نام اندیکاتور
   ENUM_SIGNAL_TYPE  signal;         // نوع سیگنال
   double            strength;       // قدرت سیگنال (0-100)
   double            value;          // مقدار فعلی
   string            description;    // توضیحات
   bool              isValid;        // معتبر بودن
};

//+------------------------------------------------------------------+
//| 🎨 ساختار کامل وضعیت اندیکاتورها                                |
//+------------------------------------------------------------------+
struct IndicatorState
{
   // اندیکاتورهای 15 گانه
   IndicatorSignal   ema_fast;
   IndicatorSignal   ema_slow;
   IndicatorSignal   rsi;
   IndicatorSignal   stochastic;
   IndicatorSignal   macd;
   IndicatorSignal   atr;
   IndicatorSignal   bollinger;
   IndicatorSignal   cci;
   IndicatorSignal   adx;
   IndicatorSignal   williams;
   IndicatorSignal   momentum;
   IndicatorSignal   sar;
   IndicatorSignal   obv;
   IndicatorSignal   awesome;
   IndicatorSignal   demarker;
   
   // آمار کلی
   int               totalBuySignals;
   int               totalSellSignals;
   int               totalNeutralSignals;
   double            averageStrength;
   datetime          lastUpdate;
};

//--- متغیر سراسری وضعیت اندیکاتورها
IndicatorState g_IndicatorState;

//+------------------------------------------------------------------+
//| 🔄 به‌روزرسانی همه اندیکاتورها                                  |
//+------------------------------------------------------------------+
bool UpdateAllIndicators()
{
   //--- ریست آمار
   g_IndicatorState.totalBuySignals = 0;
   g_IndicatorState.totalSellSignals = 0;
   g_IndicatorState.totalNeutralSignals = 0;
   g_IndicatorState.averageStrength = 0;
   
   double totalStrength = 0;
   int validSignals = 0;
   
   //--- به‌روزرسانی رفرش قیمت‌ها
   if(!symbolInfo.RefreshRates())
   {
      Print("❌ Failed to refresh rates");
      return false;
   }
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🔄 Updating 15 Indicators...");
   
   //--- 1️⃣ EMA Fast & Slow
   if(!UpdateEMA())
   {
      Print("⚠️ Failed to update EMA");
      return false;
   }
   
   //--- 2️⃣ RSI
   if(!UpdateRSI())
   {
      Print("⚠️ Failed to update RSI");
      return false;
   }
   
   //--- 3️⃣ Stochastic
   if(!UpdateStochastic())
   {
      Print("⚠️ Failed to update Stochastic");
      return false;
   }
   
   //--- 4️⃣ MACD
   if(!UpdateMACD())
   {
      Print("⚠️ Failed to update MACD");
      return false;
   }
   
   //--- 5️⃣ ATR
   if(!UpdateATR())
   {
      Print("⚠️ Failed to update ATR");
      return false;
   }
   
   //--- 6️⃣ Bollinger Bands
   if(!UpdateBollingerBands())
   {
      Print("⚠️ Failed to update Bollinger Bands");
      return false;
   }
   
   //--- 7️⃣ CCI
   if(!UpdateCCI())
   {
      Print("⚠️ Failed to update CCI");
      return false;
   }
   
   //--- 8️⃣ ADX
   if(!UpdateADX())
   {
      Print("⚠️ Failed to update ADX");
      return false;
   }
   
   //--- 9️⃣ Williams %R
   if(!UpdateWilliams())
   {
      Print("⚠️ Failed to update Williams %R");
      return false;
   }
   
   //--- 🔟 Momentum
   if(!UpdateMomentum())
   {
      Print("⚠️ Failed to update Momentum");
      return false;
   }
   
   //--- 1️⃣1️⃣ Parabolic SAR
   if(!UpdateSAR())
   {
      Print("⚠️ Failed to update SAR");
      return false;
   }
   
   //--- 1️⃣2️⃣ OBV
   if(!UpdateOBV())
   {
      Print("⚠️ Failed to update OBV");
      return false;
   }
   
   //--- 1️⃣3️⃣ Awesome Oscillator
   if(!UpdateAwesome())
   {
      Print("⚠️ Failed to update Awesome Oscillator");
      return false;
   }
   
   //--- 1️⃣4️⃣ DeMarker
   if(!UpdateDeMarker())
   {
      Print("⚠️ Failed to update DeMarker");
      return false;
   }
   
   //--- محاسبه آمار کلی
   CalculateIndicatorStats();
   
   //--- نمایش خلاصه
   PrintIndicatorSummary();
   
   g_IndicatorState.lastUpdate = TimeCurrent();
   
   Print("✅ All indicators updated successfully");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   return true;
}


//+------------------------------------------------------------------+
//| 📈 به‌روزرسانی EMA (Moving Averages)                            |
//+------------------------------------------------------------------+
bool UpdateEMA()
{
   //--- خواندن داده EMA Fast
   if(CopyBuffer(handle_EMA_Fast, 0, 0, 3, buffer_EMA_Fast) <= 0)
      return false;
   
   //--- خواندن داده EMA Slow
   if(CopyBuffer(handle_EMA_Slow, 0, 0, 3, buffer_EMA_Slow) <= 0)
      return false;
   
   //--- تحلیل
   double ema_fast_0 = buffer_EMA_Fast[0];
   double ema_fast_1 = buffer_EMA_Fast[1];
   double ema_slow_0 = buffer_EMA_Slow[0];
   double ema_slow_1 = buffer_EMA_Slow[1];
   
   double currentPrice = symbolInfo.Bid();
   
   //--- تعیین سیگنال
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // شرایط خرید: EMA Fast بالای EMA Slow و قیمت بالای هر دو
   if(ema_fast_0 > ema_slow_0 && ema_fast_1 <= ema_slow_1)
   {
      signal = SIGNAL_BUY;
      strength = 85;
      description = "Golden Cross - EMA Fast crossed above EMA Slow";
   }
   // شرایط فروش: EMA Fast پایین EMA Slow
   else if(ema_fast_0 < ema_slow_0 && ema_fast_1 >= ema_slow_1)
   {
      signal = SIGNAL_SELL;
      strength = 85;
      description = "Death Cross - EMA Fast crossed below EMA Slow";
   }
   // روند خرید قوی
   else if(ema_fast_0 > ema_slow_0 && currentPrice > ema_fast_0)
   {
      signal = SIGNAL_BUY;
      strength = 60;
      description = "Bullish trend - Price above EMAs";
   }
   // روند فروش قوی
   else if(ema_fast_0 < ema_slow_0 && currentPrice < ema_fast_0)
   {
      signal = SIGNAL_SELL;
      strength = 60;
      description = "Bearish trend - Price below EMAs";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 20;
      description = "No clear trend";
   }
   
   //--- ذخیره در ساختار
   g_IndicatorState.ema_fast.name = "EMA Cross";
   g_IndicatorState.ema_fast.signal = signal;
   g_IndicatorState.ema_fast.strength = strength;
   g_IndicatorState.ema_fast.value = ema_fast_0;
   g_IndicatorState.ema_fast.description = description;
   g_IndicatorState.ema_fast.isValid = true;
   
   Print("   ├─ EMA: ", EnumToString(signal), " (", DoubleToString(strength, 0), "%) - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📊 به‌روزرسانی RSI                                               |
//+------------------------------------------------------------------+
bool UpdateRSI()
{
   //--- خواندن داده
   if(CopyBuffer(handle_RSI, 0, 0, 3, buffer_RSI) <= 0)
      return false;
   
   double rsi_0 = buffer_RSI[0];
   double rsi_1 = buffer_RSI[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // اشباع فروش شدید (خرید قوی)
   if(rsi_0 < 20)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Extremely Oversold";
   }
   // اشباع فروش
   else if(rsi_0 < 30)
   {
      signal = SIGNAL_BUY;
      strength = 75;
      description = "Oversold";
   }
   // خروج از اشباع فروش
   else if(rsi_0 > 30 && rsi_1 <= 30)
   {
      signal = SIGNAL_BUY;
      strength = 70;
      description = "Exiting Oversold Zone";
   }
   // اشباع خرید شدید (فروش قوی)
   else if(rsi_0 > 80)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Extremely Overbought";
   }
   // اشباع خرید
   else if(rsi_0 > 70)
   {
      signal = SIGNAL_SELL;
      strength = 75;
      description = "Overbought";
   }
   // خروج از اشباع خرید
   else if(rsi_0 < 70 && rsi_1 >= 70)
   {
      signal = SIGNAL_SELL;
      strength = 70;
      description = "Exiting Overbought Zone";
   }
   // روند صعودی
   else if(rsi_0 > 50 && rsi_0 > rsi_1)
   {
      signal = SIGNAL_BUY;
      strength = 50;
      description = "Bullish momentum";
   }
   // روند نزولی
   else if(rsi_0 < 50 && rsi_0 < rsi_1)
   {
      signal = SIGNAL_SELL;
      strength = 50;
      description = "Bearish momentum";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "Neutral zone";
   }
   
   //--- ذخیره
   g_IndicatorState.rsi.name = "RSI";
   g_IndicatorState.rsi.signal = signal;
   g_IndicatorState.rsi.strength = strength;
   g_IndicatorState.rsi.value = rsi_0;
   g_IndicatorState.rsi.description = description;
   g_IndicatorState.rsi.isValid = true;
   
   Print("   ├─ RSI: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(rsi_0, 2), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📉 به‌روزرسانی Stochastic                                        |
//+------------------------------------------------------------------+
bool UpdateStochastic()
{
   //--- خواندن داده
   if(CopyBuffer(handle_Stoch, MAIN_LINE, 0, 3, buffer_Stoch_Main) <= 0)
      return false;
   if(CopyBuffer(handle_Stoch, SIGNAL_LINE, 0, 3, buffer_Stoch_Signal) <= 0)
      return false;
   
   double stoch_main_0 = buffer_Stoch_Main[0];
   double stoch_main_1 = buffer_Stoch_Main[1];
   double stoch_signal_0 = buffer_Stoch_Signal[0];
   double stoch_signal_1 = buffer_Stoch_Signal[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // تقاطع صعودی در ناحیه اشباع فروش
   if(stoch_main_0 > stoch_signal_0 && stoch_main_1 <= stoch_signal_1 && stoch_main_0 < 20)
   {
      signal = SIGNAL_BUY;
      strength = 95;
      description = "Bullish crossover in oversold zone";
   }
   // تقاطع نزولی در ناحیه اشباع خرید
   else if(stoch_main_0 < stoch_signal_0 && stoch_main_1 >= stoch_signal_1 && stoch_main_0 > 80)
   {
      signal = SIGNAL_SELL;
      strength = 95;
      description = "Bearish crossover in overbought zone";
   }
   // اشباع فروش
   else if(stoch_main_0 < 20)
   {
      signal = SIGNAL_BUY;
      strength = 70;
      description = "Oversold condition";
   }
   // اشباع خرید
   else if(stoch_main_0 > 80)
   {
      signal = SIGNAL_SELL;
      strength = 70;
      description = "Overbought condition";
   }
   // تقاطع صعودی معمولی
   else if(stoch_main_0 > stoch_signal_0 && stoch_main_1 <= stoch_signal_1)
   {
      signal = SIGNAL_BUY;
      strength = 60;
      description = "Bullish crossover";
   }
   // تقاطع نزولی معمولی
   else if(stoch_main_0 < stoch_signal_0 && stoch_main_1 >= stoch_signal_1)
   {
      signal = SIGNAL_SELL;
      strength = 60;
      description = "Bearish crossover";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 25;
      description = "No clear signal";
   }
   
   //--- ذخیره
   g_IndicatorState.stochastic.name = "Stochastic";
   g_IndicatorState.stochastic.signal = signal;
   g_IndicatorState.stochastic.strength = strength;
   g_IndicatorState.stochastic.value = stoch_main_0;
   g_IndicatorState.stochastic.description = description;
   g_IndicatorState.stochastic.isValid = true;
   
   Print("   ├─ STOCH: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(stoch_main_0, 2), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📊 به‌روزرسانی MACD                                              |
//+------------------------------------------------------------------+
bool UpdateMACD()
{
   //--- خواندن داده
   if(CopyBuffer(handle_MACD, MAIN_LINE, 0, 3, buffer_MACD_Main) <= 0)
      return false;
   if(CopyBuffer(handle_MACD, SIGNAL_LINE, 0, 3, buffer_MACD_Signal) <= 0)
      return false;
   
   double macd_main_0 = buffer_MACD_Main[0];
   double macd_main_1 = buffer_MACD_Main[1];
   double macd_signal_0 = buffer_MACD_Signal[0];
   double macd_signal_1 = buffer_MACD_Signal[1];
   
   double histogram_0 = macd_main_0 - macd_signal_0;
   double histogram_1 = macd_main_1 - macd_signal_1;
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // تقاطع صعودی
   if(macd_main_0 > macd_signal_0 && macd_main_1 <= macd_signal_1)
   {
      if(macd_main_0 < 0) // زیر خط صفر
      {
         signal = SIGNAL_BUY;
         strength = 90;
         description = "Bullish crossover below zero";
      }
      else
      {
         signal = SIGNAL_BUY;
         strength = 75;
         description = "Bullish crossover above zero";
      }
   }
   // تقاطع نزولی
   else if(macd_main_0 < macd_signal_0 && macd_main_1 >= macd_signal_1)
   {
      if(macd_main_0 > 0) // بالای خط صفر
      {
         signal = SIGNAL_SELL;
         strength = 90;
         description = "Bearish crossover above zero";
      }
      else
      {
         signal = SIGNAL_SELL;
         strength = 75;
         description = "Bearish crossover below zero";
      }
   }
   // هیستوگرام در حال افزایش
   else if(histogram_0 > histogram_1 && histogram_0 > 0)
   {
      signal = SIGNAL_BUY;
      strength = 55;
      description = "Increasing bullish histogram";
   }
   // هیستوگرام در حال کاهش
   else if(histogram_0 < histogram_1 && histogram_0 < 0)
   {
      signal = SIGNAL_SELL;
      strength = 55;
      description = "Increasing bearish histogram";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "No clear momentum";
   }
   
   //--- ذخیره
   g_IndicatorState.macd.name = "MACD";
   g_IndicatorState.macd.signal = signal;
   g_IndicatorState.macd.strength = strength;
   g_IndicatorState.macd.value = macd_main_0;
   g_IndicatorState.macd.description = description;
   g_IndicatorState.macd.isValid = true;
   
   Print("   ├─ MACD: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Histogram: ", DoubleToString(histogram_0, 5), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📏 به‌روزرسانی ATR (Volatility)                                 |
//+------------------------------------------------------------------+
bool UpdateATR()
{
   //--- خواندن داده
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return false;
   
   double atr_0 = buffer_ATR[0];
   double atr_avg = 0;
   
   //--- محاسبه میانگین ATR
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   //--- تحلیل نوسان
   ENUM_SIGNAL_TYPE signal = SIGNAL_NEUTRAL;
   double strength = 0;
   string description = "";
   
   // نوسان بالا - مناسب برای اسکلپینگ
   if(atr_0 > atr_avg * 1.5)
   {
      signal = SIGNAL_NEUTRAL; // ATR جهت ندارد، فقط نوسان را نشان می‌دهد
      strength = 80;
      description = "High volatility - Good for scalping";
   }
   // نوسان متوسط
   else if(atr_0 > atr_avg * 0.8)
   {
      signal = SIGNAL_NEUTRAL;
      strength = 60;
      description = "Normal volatility";
   }
   // نوسان پایین - خطرناک برای اسکلپینگ
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "Low volatility - Risky for scalping";
   }
   
   //--- ذخیره
   g_IndicatorState.atr.name = "ATR";
   g_IndicatorState.atr.signal = signal;
   g_IndicatorState.atr.strength = strength;
   g_IndicatorState.atr.value = atr_0;
   g_IndicatorState.atr.description = description;
   g_IndicatorState.atr.isValid = true;
   
   Print("   ├─ ATR: Value: ", DoubleToString(atr_0, g_Digits), 
         " Avg: ", DoubleToString(atr_avg, g_Digits), " - ", description);
   
   return true;
}


//+------------------------------------------------------------------+
//| 📊 به‌روزرسانی Bollinger Bands                                  |
//+------------------------------------------------------------------+
bool UpdateBollingerBands()
{
   //--- خواندن داده
   if(CopyBuffer(handle_BB, BASE_LINE, 0, 3, buffer_BB_Middle) <= 0)
      return false;
   if(CopyBuffer(handle_BB, UPPER_BAND, 0, 3, buffer_BB_Upper) <= 0)
      return false;
   if(CopyBuffer(handle_BB, LOWER_BAND, 0, 3, buffer_BB_Lower) <= 0)
      return false;
   
   double bb_upper_0 = buffer_BB_Upper[0];
   double bb_middle_0 = buffer_BB_Middle[0];
   double bb_lower_0 = buffer_BB_Lower[0];
   
   double currentPrice = symbolInfo.Bid();
   double pricePosition = 0;
   
   //--- محاسبه موقعیت قیمت در باند (0-100%)
   if(bb_upper_0 != bb_lower_0)
      pricePosition = ((currentPrice - bb_lower_0) / (bb_upper_0 - bb_lower_0)) * 100;
   
   //--- محاسبه عرض باند (Bandwidth)
   double bandwidth = ((bb_upper_0 - bb_lower_0) / bb_middle_0) * 100;
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // قیمت از باند پایین به بالا عبور کرده (سیگنال خرید قوی)
   if(currentPrice > bb_lower_0 && pricePosition < 20)
   {
      signal = SIGNAL_BUY;
      strength = 85;
      description = "Price bouncing from lower band";
   }
   // قیمت از باند بالا به پایین عبور کرده (سیگنال فروش قوی)
   else if(currentPrice < bb_upper_0 && pricePosition > 80)
   {
      signal = SIGNAL_SELL;
      strength = 85;
      description = "Price bouncing from upper band";
   }
   // قیمت زیر باند پایین - اشباع فروش شدید
   else if(currentPrice < bb_lower_0)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Price below lower band - extreme oversold";
   }
   // قیمت بالای باند بالا - اشباع خرید شدید
   else if(currentPrice > bb_upper_0)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Price above upper band - extreme overbought";
   }
   // قیمت نزدیک میانگین و باند در حال باریک شدن (Squeeze)
   else if(bandwidth < 2 && MathAbs(currentPrice - bb_middle_0) < (bb_upper_0 - bb_middle_0) * 0.3)
   {
      signal = SIGNAL_NEUTRAL;
      strength = 40;
      description = "Band squeeze - low volatility";
   }
   // قیمت بالای میانگین
   else if(currentPrice > bb_middle_0)
   {
      signal = SIGNAL_BUY;
      strength = 55;
      description = "Price above middle band";
   }
   // قیمت پایین میانگین
   else if(currentPrice < bb_middle_0)
   {
      signal = SIGNAL_SELL;
      strength = 55;
      description = "Price below middle band";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "Price at middle band";
   }
   
   //--- ذخیره
   g_IndicatorState.bollinger.name = "Bollinger Bands";
   g_IndicatorState.bollinger.signal = signal;
   g_IndicatorState.bollinger.strength = strength;
   g_IndicatorState.bollinger.value = pricePosition;
   g_IndicatorState.bollinger.description = description;
   g_IndicatorState.bollinger.isValid = true;
   
   Print("   ├─ BB: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Position: ", DoubleToString(pricePosition, 1), "% - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📈 به‌روزرسانی CCI (Commodity Channel Index)                    |
//+------------------------------------------------------------------+
bool UpdateCCI()
{
   //--- خواندن داده
   if(CopyBuffer(handle_CCI, 0, 0, 3, buffer_CCI) <= 0)
      return false;
   
   double cci_0 = buffer_CCI[0];
   double cci_1 = buffer_CCI[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // اشباع فروش شدید (زیر -200)
   if(cci_0 < -200)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Extreme oversold (< -200)";
   }
   // خروج از اشباع فروش شدید
   else if(cci_0 > -200 && cci_1 <= -200)
   {
      signal = SIGNAL_BUY;
      strength = 85;
      description = "Exiting extreme oversold zone";
   }
   // اشباع فروش معمولی (زیر -100)
   else if(cci_0 < -100)
   {
      signal = SIGNAL_BUY;
      strength = 70;
      description = "Oversold (< -100)";
   }
   // عبور از صفر به سمت بالا
   else if(cci_0 > 0 && cci_1 <= 0)
   {
      signal = SIGNAL_BUY;
      strength = 75;
      description = "Crossing zero upward";
   }
   // اشباع خرید شدید (بالای +200)
   else if(cci_0 > 200)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Extreme overbought (> +200)";
   }
   // خروج از اشباع خرید شدید
   else if(cci_0 < 200 && cci_1 >= 200)
   {
      signal = SIGNAL_SELL;
      strength = 85;
      description = "Exiting extreme overbought zone";
   }
   // اشباع خرید معمولی (بالای +100)
   else if(cci_0 > 100)
   {
      signal = SIGNAL_SELL;
      strength = 70;
      description = "Overbought (> +100)";
   }
   // عبور از صفر به سمت پایین
   else if(cci_0 < 0 && cci_1 >= 0)
   {
      signal = SIGNAL_SELL;
      strength = 75;
      description = "Crossing zero downward";
   }
   // بالای صفر (صعودی)
   else if(cci_0 > 0)
   {
      signal = SIGNAL_BUY;
      strength = 50;
      description = "Positive zone";
   }
   // پایین صفر (نزولی)
   else if(cci_0 < 0)
   {
      signal = SIGNAL_SELL;
      strength = 50;
      description = "Negative zone";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 25;
      description = "At zero level";
   }
   
   //--- ذخیره
   g_IndicatorState.cci.name = "CCI";
   g_IndicatorState.cci.signal = signal;
   g_IndicatorState.cci.strength = strength;
   g_IndicatorState.cci.value = cci_0;
   g_IndicatorState.cci.description = description;
   g_IndicatorState.cci.isValid = true;
   
   Print("   ├─ CCI: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(cci_0, 2), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 💪 به‌روزرسانی ADX (Average Directional Index)                  |
//+------------------------------------------------------------------+
bool UpdateADX()
{
   //--- خواندن داده
   if(CopyBuffer(handle_ADX, MAIN_LINE, 0, 3, buffer_ADX_Main) <= 0)
      return false;
   if(CopyBuffer(handle_ADX, PLUSDI_LINE, 0, 3, buffer_ADX_Plus) <= 0)
      return false;
   if(CopyBuffer(handle_ADX, MINUSDI_LINE, 0, 3, buffer_ADX_Minus) <= 0)
      return false;
   
   double adx_0 = buffer_ADX_Main[0];
   double adx_1 = buffer_ADX_Main[1];
   double plus_di_0 = buffer_ADX_Plus[0];
   double plus_di_1 = buffer_ADX_Plus[1];
   double minus_di_0 = buffer_ADX_Minus[0];
   double minus_di_1 = buffer_ADX_Minus[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // روند قوی صعودی (ADX > 25 و +DI > -DI)
   if(adx_0 > 25 && plus_di_0 > minus_di_0)
   {
      if(adx_0 > adx_1) // روند در حال تقویت
      {
         signal = SIGNAL_BUY;
         strength = 90;
         description = "Strong uptrend strengthening";
      }
      else
      {
         signal = SIGNAL_BUY;
         strength = 75;
         description = "Strong uptrend";
      }
   }
   // روند قوی نزولی (ADX > 25 و -DI > +DI)
   else if(adx_0 > 25 && minus_di_0 > plus_di_0)
   {
      if(adx_0 > adx_1) // روند در حال تقویت
      {
         signal = SIGNAL_SELL;
         strength = 90;
         description = "Strong downtrend strengthening";
      }
      else
      {
         signal = SIGNAL_SELL;
         strength = 75;
         description = "Strong downtrend";
      }
   }
   // تقاطع +DI و -DI با ADX متوسط
   else if(plus_di_0 > minus_di_0 && plus_di_1 <= minus_di_1 && adx_0 > 20)
   {
      signal = SIGNAL_BUY;
      strength = 80;
      description = "+DI crossed above -DI";
   }
   else if(minus_di_0 > plus_di_0 && minus_di_1 <= plus_di_1 && adx_0 > 20)
   {
      signal = SIGNAL_SELL;
      strength = 80;
      description = "-DI crossed above +DI";
   }
   // روند ضعیف (ADX < 20)
   else if(adx_0 < 20)
   {
      signal = SIGNAL_NEUTRAL;
      strength = 20;
      description = "Weak trend - Not suitable for scalping";
   }
   // روند متوسط صعودی
   else if(plus_di_0 > minus_di_0)
   {
      signal = SIGNAL_BUY;
      strength = 55;
      description = "Moderate uptrend";
   }
   // روند متوسط نزولی
   else if(minus_di_0 > plus_di_0)
   {
      signal = SIGNAL_SELL;
      strength = 55;
      description = "Moderate downtrend";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "No clear trend";
   }
   
   //--- ذخیره
   g_IndicatorState.adx.name = "ADX";
   g_IndicatorState.adx.signal = signal;
   g_IndicatorState.adx.strength = strength;
   g_IndicatorState.adx.value = adx_0;
   g_IndicatorState.adx.description = description;
   g_IndicatorState.adx.isValid = true;
   
   Print("   ├─ ADX: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - ADX: ", DoubleToString(adx_0, 2), " +DI: ", DoubleToString(plus_di_0, 2), 
         " -DI: ", DoubleToString(minus_di_0, 2), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📉 به‌روزرسانی Williams %R                                      |
//+------------------------------------------------------------------+
bool UpdateWilliams()
{
   //--- خواندن داده
   if(CopyBuffer(handle_WPR, 0, 0, 3, buffer_WPR) <= 0)
      return false;
   
   double wpr_0 = buffer_WPR[0];
   double wpr_1 = buffer_WPR[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // اشباع فروش شدید (بالای -20 نزدیک به 0)
   if(wpr_0 > -20)
   {
      signal = SIGNAL_SELL; // توجه: Williams %R معکوس است
      strength = 90;
      description = "Extreme overbought";
   }
   // خروج از اشباع خرید
   else if(wpr_0 < -20 && wpr_1 >= -20)
   {
      signal = SIGNAL_SELL;
      strength = 80;
      description = "Exiting overbought zone";
   }
   // ناحیه اشباع خرید (-20 تا -40)
   else if(wpr_0 > -40)
   {
      signal = SIGNAL_SELL;
      strength = 65;
      description = "Overbought zone";
   }
   // اشباع فروش شدید (پایین -80)
   else if(wpr_0 < -80)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Extreme oversold";
   }
   // خروج از اشباع فروش
   else if(wpr_0 > -80 && wpr_1 <= -80)
   {
      signal = SIGNAL_BUY;
      strength = 80;
      description = "Exiting oversold zone";
   }
   // ناحیه اشباع فروش (-60 تا -80)
   else if(wpr_0 < -60)
   {
      signal = SIGNAL_BUY;
      strength = 65;
      description = "Oversold zone";
   }
   // ناحیه خنثی صعودی (-40 تا -60)
   else if(wpr_0 > -60 && wpr_0 > wpr_1)
   {
      signal = SIGNAL_BUY;
      strength = 45;
      description = "Neutral zone - upward movement";
   }
   // ناحیه خنثی نزولی
   else if(wpr_0 < wpr_1)
   {
      signal = SIGNAL_SELL;
      strength = 45;
      description = "Neutral zone - downward movement";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 25;
      description = "No clear signal";
   }
   
   //--- ذخیره
   g_IndicatorState.williams.name = "Williams %R";
   g_IndicatorState.williams.signal = signal;
   g_IndicatorState.williams.strength = strength;
   g_IndicatorState.williams.value = wpr_0;
   g_IndicatorState.williams.description = description;
   g_IndicatorState.williams.isValid = true;
   
   Print("   ├─ WPR: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(wpr_0, 2), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 🚀 به‌روزرسانی Momentum                                         |
//+------------------------------------------------------------------+
bool UpdateMomentum()
{
   //--- خواندن داده
   if(CopyBuffer(handle_MOM, 0, 0, 5, buffer_MOM) <= 0)
      return false;
   
   double mom_0 = buffer_MOM[0];
   double mom_1 = buffer_MOM[1];
   double mom_2 = buffer_MOM[2];
   
   //--- محاسبه میانگین Momentum
   double mom_avg = (buffer_MOM[0] + buffer_MOM[1] + buffer_MOM[2] + 
                     buffer_MOM[3] + buffer_MOM[4]) / 5.0;
   
   //--- محاسبه تغییرات
   double mom_change = mom_0 - mom_1;
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // Momentum قوی صعودی و در حال افزایش
   if(mom_0 > 100000 && mom_0 > mom_1 && mom_1 > mom_2)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Strong bullish momentum accelerating";
   }
   // Momentum قوی نزولی و در حال کاهش
   else if(mom_0 < 100000 && mom_0 < mom_1 && mom_1 < mom_2)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Strong bearish momentum accelerating";
   }
   // عبور از 100000 به سمت بالا
   else if(mom_0 > 100000 && mom_1 <= 100000)
   {
      signal = SIGNAL_BUY;
      strength = 85;
      description = "Momentum crossing above 100000";
   }
   // عبور از 100000 به سمت پایین
   else if(mom_0 < 100000 && mom_1 >= 100000)
   {
      signal = SIGNAL_SELL;
      strength = 85;
      description = "Momentum crossing below 100000";
   }
   // Momentum بالای میانگین و صعودی
   else if(mom_0 > mom_avg && mom_change > 0)
   {
      signal = SIGNAL_BUY;
      strength = 70;
      description = "Above average - bullish";
   }
   // Momentum پایین میانگین و نزولی
   else if(mom_0 < mom_avg && mom_change < 0)
   {
      signal = SIGNAL_SELL;
      strength = 70;
      description = "Below average - bearish";
   }
   // واگرایی صعودی (قیمت پایین می‌رود اما Momentum بالا می‌رود)
   else if(mom_0 > mom_1 && mom_1 < mom_2)
   {
      signal = SIGNAL_BUY;
      strength = 60;
      description = "Potential bullish divergence";
   }
   // واگرایی نزولی (قیمت بالا می‌رود اما Momentum پایین می‌رود)
   else if(mom_0 < mom_1 && mom_1 > mom_2)
   {
      signal = SIGNAL_SELL;
      strength = 60;
      description = "Potential bearish divergence";
   }
   // Momentum صعودی ضعیف
   else if(mom_0 > 100000)
   {
      signal = SIGNAL_BUY;
      strength = 40;
      description = "Weak bullish momentum";
   }
   // Momentum نزولی ضعیف
   else if(mom_0 < 100000)
   {
      signal = SIGNAL_SELL;
      strength = 40;
      description = "Weak bearish momentum";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 20;
      description = "No momentum";
   }
   
   //--- ذخیره
   g_IndicatorState.momentum.name = "Momentum";
   g_IndicatorState.momentum.signal = signal;
   g_IndicatorState.momentum.strength = strength;
   g_IndicatorState.momentum.value = mom_0;
   g_IndicatorState.momentum.description = description;
   g_IndicatorState.momentum.isValid = true;
   
   Print("   ├─ MOM: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(mom_0, 2), " - ", description);
   
   return true;
}


//+------------------------------------------------------------------+
//| 🎯 به‌روزرسانی Parabolic SAR                                    |
//+------------------------------------------------------------------+
bool UpdateSAR()
{
   //--- خواندن داده
   if(CopyBuffer(handle_SAR, 0, 0, 3, buffer_SAR) <= 0)
      return false;
   
   double sar_0 = buffer_SAR[0];
   double sar_1 = buffer_SAR[1];
   
   double currentPrice = symbolInfo.Bid();
   double previousPrice = 0;
   
   //--- دریافت قیمت قبلی
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 3, rates) < 3)
      return false;
   
   previousPrice = rates[1].close;
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // SAR زیر قیمت است - روند صعودی
   bool sarBelow = (sar_0 < currentPrice);
   bool sarBelowPrev = (sar_1 < previousPrice);
   
   // تغییر جهت از نزولی به صعودی (سیگنال خرید قوی)
   if(sarBelow && !sarBelowPrev)
   {
      signal = SIGNAL_BUY;
      strength = 95;
      description = "SAR reversal - Buy signal";
   }
   // تغییر جهت از صعودی به نزولی (سیگنال فروش قوی)
   else if(!sarBelow && sarBelowPrev)
   {
      signal = SIGNAL_SELL;
      strength = 95;
      description = "SAR reversal - Sell signal";
   }
   // روند صعودی مستمر
   else if(sarBelow)
   {
      // محاسبه فاصله SAR از قیمت
      double distance = ((currentPrice - sar_0) / currentPrice) * 100;
      
      if(distance > 0.5)
      {
         signal = SIGNAL_BUY;
         strength = 75;
         description = "Strong uptrend - SAR far below price";
      }
      else
      {
         signal = SIGNAL_BUY;
         strength = 60;
         description = "Uptrend confirmed";
      }
   }
   // روند نزولی مستمر
   else if(!sarBelow)
   {
      // محاسبه فاصله SAR از قیمت
      double distance = ((sar_0 - currentPrice) / currentPrice) * 100;
      
      if(distance > 0.5)
      {
         signal = SIGNAL_SELL;
         strength = 75;
         description = "Strong downtrend - SAR far above price";
      }
      else
      {
         signal = SIGNAL_SELL;
         strength = 60;
         description = "Downtrend confirmed";
      }
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 25;
      description = "No clear trend";
   }
   
   //--- ذخیره
   g_IndicatorState.sar.name = "Parabolic SAR";
   g_IndicatorState.sar.signal = signal;
   g_IndicatorState.sar.strength = strength;
   g_IndicatorState.sar.value = sar_0;
   g_IndicatorState.sar.description = description;
   g_IndicatorState.sar.isValid = true;
   
   Print("   ├─ SAR: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - SAR: ", DoubleToString(sar_0, g_Digits), 
         " Price: ", DoubleToString(currentPrice, g_Digits), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 📊 به‌روزرسانی OBV (On Balance Volume)                          |
//+------------------------------------------------------------------+
bool UpdateOBV()
{
   //--- خواندن داده
   if(CopyBuffer(handle_OBV, 0, 0, 10, buffer_OBV) <= 0)
      return false;
   
   double obv_0 = buffer_OBV[0];
   double obv_1 = buffer_OBV[1];
   double obv_2 = buffer_OBV[2];
   
   //--- محاسبه میانگین OBV
   double obv_avg = 0;
   for(int i = 0; i < 10; i++)
      obv_avg += buffer_OBV[i];
   obv_avg /= 10;
   
   //--- محاسبه روند OBV
   double obv_trend = obv_0 - obv_2;
   
   //--- دریافت قیمت
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 3, rates) < 3)
      return false;
   
   double price_change = rates[0].close - rates[2].close;
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // واگرایی صعودی: قیمت پایین می‌رود اما OBV بالا می‌رود
   if(price_change < 0 && obv_trend > 0)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Bullish divergence - Accumulation";
   }
   // واگرایی نزولی: قیمت بالا می‌رود اما OBV پایین می‌رود
   else if(price_change > 0 && obv_trend < 0)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Bearish divergence - Distribution";
   }
   // OBV و قیمت هر دو صعودی
   else if(obv_trend > 0 && price_change > 0)
   {
      if(obv_0 > obv_avg)
      {
         signal = SIGNAL_BUY;
         strength = 80;
         description = "Strong buying pressure";
      }
      else
      {
         signal = SIGNAL_BUY;
         strength = 65;
         description = "Buying pressure confirmed";
      }
   }
   // OBV و قیمت هر دو نزولی
   else if(obv_trend < 0 && price_change < 0)
   {
      if(obv_0 < obv_avg)
      {
         signal = SIGNAL_SELL;
         strength = 80;
         description = "Strong selling pressure";
      }
      else
      {
         signal = SIGNAL_SELL;
         strength = 65;
         description = "Selling pressure confirmed";
      }
   }
   // OBV در حال افزایش
   else if(obv_0 > obv_1 && obv_1 > obv_2)
   {
      signal = SIGNAL_BUY;
      strength = 55;
      description = "Volume accumulation";
   }
   // OBV در حال کاهش
   else if(obv_0 < obv_1 && obv_1 < obv_2)
   {
      signal = SIGNAL_SELL;
      strength = 55;
      description = "Volume distribution";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "No clear volume trend";
   }
   
   //--- ذخیره
   g_IndicatorState.obv.name = "OBV";
   g_IndicatorState.obv.signal = signal;
   g_IndicatorState.obv.strength = strength;
   g_IndicatorState.obv.value = obv_0;
   g_IndicatorState.obv.description = description;
   g_IndicatorState.obv.isValid = true;
   
   Print("   ├─ OBV: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Trend: ", DoubleToString(obv_trend, 0), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 🌊 به‌روزرسانی Awesome Oscillator                               |
//+------------------------------------------------------------------+
bool UpdateAwesome()
{
   //--- خواندن داده
   if(CopyBuffer(handle_AO, 0, 0, 5, buffer_AO) <= 0)
      return false;
   
   double ao_0 = buffer_AO[0];
   double ao_1 = buffer_AO[1];
   double ao_2 = buffer_AO[2];
   double ao_3 = buffer_AO[3];
   double ao_4 = buffer_AO[4];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // الگوی Twin Peaks (دو قله) - سیگنال خرید
   // قله دوم بالاتر از صفر اما پایین‌تر از قله اول و سپس صعود
   if(ao_2 < 0 && ao_1 < ao_2 && ao_0 > ao_1 && ao_0 < 0)
   {
      signal = SIGNAL_BUY;
      strength = 85;
      description = "Twin Peaks buy pattern";
   }
   // الگوی Twin Peaks - سیگنال فروش
   else if(ao_2 > 0 && ao_1 > ao_2 && ao_0 < ao_1 && ao_0 > 0)
   {
      signal = SIGNAL_SELL;
      strength = 85;
      description = "Twin Peaks sell pattern";
   }
   // الگوی Saucer (نعلبکی) - خرید
   // سه میله متوالی زیر صفر که اولی نزولی، دومی نزولی کمتر، سومی صعودی
   else if(ao_2 < ao_3 && ao_1 < ao_2 && ao_0 > ao_1 && ao_0 < 0)
   {
      signal = SIGNAL_BUY;
      strength = 80;
      description = "Saucer buy pattern";
   }
   // الگوی Saucer - فروش
   else if(ao_2 > ao_3 && ao_1 > ao_2 && ao_0 < ao_1 && ao_0 > 0)
   {
      signal = SIGNAL_SELL;
      strength = 80;
      description = "Saucer sell pattern";
   }
   // عبور از خط صفر به سمت بالا
   else if(ao_0 > 0 && ao_1 <= 0)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Zero line crossover - Buy";
   }
   // عبور از خط صفر به سمت پایین
   else if(ao_0 < 0 && ao_1 >= 0)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Zero line crossover - Sell";
   }
   // بالای صفر و در حال افزایش
   else if(ao_0 > 0 && ao_0 > ao_1 && ao_1 > ao_2)
   {
      signal = SIGNAL_BUY;
      strength = 70;
      description = "Strong bullish momentum";
   }
   // پایین صفر و در حال کاهش
   else if(ao_0 < 0 && ao_0 < ao_1 && ao_1 < ao_2)
   {
      signal = SIGNAL_SELL;
      strength = 70;
      description = "Strong bearish momentum";
   }
   // بالای صفر (صعودی)
   else if(ao_0 > 0)
   {
      signal = SIGNAL_BUY;
      strength = 50;
      description = "Above zero line";
   }
   // پایین صفر (نزولی)
   else if(ao_0 < 0)
   {
      signal = SIGNAL_SELL;
      strength = 50;
      description = "Below zero line";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 25;
      description = "At zero line";
   }
   
   //--- ذخیره
   g_IndicatorState.awesome.name = "Awesome Oscillator";
   g_IndicatorState.awesome.signal = signal;
   g_IndicatorState.awesome.strength = strength;
   g_IndicatorState.awesome.value = ao_0;
   g_IndicatorState.awesome.description = description;
   g_IndicatorState.awesome.isValid = true;
   
   Print("   ├─ AO: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(ao_0, 5), " - ", description);
   
   return true;
}

//+------------------------------------------------------------------+
//| 💎 به‌روزرسانی DeMarker                                         |
//+------------------------------------------------------------------+
bool UpdateDeMarker()
{
   //--- خواندن داده
   if(CopyBuffer(handle_DeMarker, 0, 0, 3, buffer_DeMarker) <= 0)
      return false;
   
   double dem_0 = buffer_DeMarker[0];
   double dem_1 = buffer_DeMarker[1];
   
   //--- تحلیل
   ENUM_SIGNAL_TYPE signal = SIGNAL_NONE;
   double strength = 0;
   string description = "";
   
   // اشباع فروش شدید (زیر 0.3)
   if(dem_0 < 0.2)
   {
      signal = SIGNAL_BUY;
      strength = 95;
      description = "Extreme oversold (< 0.2)";
   }
   // خروج از اشباع فروش شدید
   else if(dem_0 > 0.2 && dem_1 <= 0.2)
   {
      signal = SIGNAL_BUY;
      strength = 90;
      description = "Exiting extreme oversold";
   }
   // اشباع فروش (زیر 0.3)
   else if(dem_0 < 0.3)
   {
      signal = SIGNAL_BUY;
      strength = 75;
      description = "Oversold (< 0.3)";
   }
   // خروج از اشباع فروش
   else if(dem_0 > 0.3 && dem_1 <= 0.3)
   {
      signal = SIGNAL_BUY;
      strength = 80;
      description = "Exiting oversold zone";
   }
   // اشباع خرید شدید (بالای 0.8)
   else if(dem_0 > 0.8)
   {
      signal = SIGNAL_SELL;
      strength = 95;
      description = "Extreme overbought (> 0.8)";
   }
   // خروج از اشباع خرید شدید
   else if(dem_0 < 0.8 && dem_1 >= 0.8)
   {
      signal = SIGNAL_SELL;
      strength = 90;
      description = "Exiting extreme overbought";
   }
   // اشباع خرید (بالای 0.7)
   else if(dem_0 > 0.7)
   {
      signal = SIGNAL_SELL;
      strength = 75;
      description = "Overbought (> 0.7)";
   }
   // خروج از اشباع خرید
   else if(dem_0 < 0.7 && dem_1 >= 0.7)
   {
      signal = SIGNAL_SELL;
      strength = 80;
      description = "Exiting overbought zone";
   }
   // عبور از 0.5 به سمت بالا
   else if(dem_0 > 0.5 && dem_1 <= 0.5)
   {
      signal = SIGNAL_BUY;
      strength = 65;
      description = "Crossing 0.5 upward";
   }
   // عبور از 0.5 به سمت پایین
   else if(dem_0 < 0.5 && dem_1 >= 0.5)
   {
      signal = SIGNAL_SELL;
      strength = 65;
      description = "Crossing 0.5 downward";
   }
   // بالای 0.5 و صعودی
   else if(dem_0 > 0.5 && dem_0 > dem_1)
   {
      signal = SIGNAL_BUY;
      strength = 50;
      description = "Above midpoint - rising";
   }
   // پایین 0.5 و نزولی
   else if(dem_0 < 0.5 && dem_0 < dem_1)
   {
      signal = SIGNAL_SELL;
      strength = 50;
      description = "Below midpoint - falling";
   }
   else
   {
      signal = SIGNAL_NEUTRAL;
      strength = 30;
      description = "Neutral zone";
   }
   
   //--- ذخیره
   g_IndicatorState.demarker.name = "DeMarker";
   g_IndicatorState.demarker.signal = signal;
   g_IndicatorState.demarker.strength = strength;
   g_IndicatorState.demarker.value = dem_0;
   g_IndicatorState.demarker.description = description;
   g_IndicatorState.demarker.isValid = true;
   
   Print("   ├─ DeMarker: ", EnumToString(signal), " (", DoubleToString(strength, 0), 
         "%) - Value: ", DoubleToString(dem_0, 3), " - ", description);
   
   return true;
}


//+------------------------------------------------------------------+
//| 📊 محاسبه آمار کلی اندیکاتورها                                  |
//+------------------------------------------------------------------+
void CalculateIndicatorStats()
{
   //--- ریست آمار
   g_IndicatorState.totalBuySignals = 0;
   g_IndicatorState.totalSellSignals = 0;
   g_IndicatorState.totalNeutralSignals = 0;
   
   double totalStrength = 0;
   int validIndicators = 0;
   
   //--- آرایه تمام سیگنال‌ها
   IndicatorSignal signals[];
   ArrayResize(signals, 15);
   
   signals[0] = g_IndicatorState.ema_fast;
   signals[1] = g_IndicatorState.rsi;
   signals[2] = g_IndicatorState.stochastic;
   signals[3] = g_IndicatorState.macd;
   signals[4] = g_IndicatorState.atr;
   signals[5] = g_IndicatorState.bollinger;
   signals[6] = g_IndicatorState.cci;
   signals[7] = g_IndicatorState.adx;
   signals[8] = g_IndicatorState.williams;
   signals[9] = g_IndicatorState.momentum;
   signals[10] = g_IndicatorState.sar;
   signals[11] = g_IndicatorState.obv;
   signals[12] = g_IndicatorState.awesome;
   signals[13] = g_IndicatorState.demarker;
   
   //--- شمارش سیگنال‌ها
   for(int i = 0; i < 14; i++) // 14 اندیکاتور با سیگنال (ATR سیگنال ندارد)
   {
      if(signals[i].isValid)
      {
         validIndicators++;
         totalStrength += signals[i].strength;
         
         if(signals[i].signal == SIGNAL_BUY)
            g_IndicatorState.totalBuySignals++;
         else if(signals[i].signal == SIGNAL_SELL)
            g_IndicatorState.totalSellSignals++;
         else if(signals[i].signal == SIGNAL_NEUTRAL)
            g_IndicatorState.totalNeutralSignals++;
      }
   }
   
   //--- محاسبه میانگین قدرت
   if(validIndicators > 0)
      g_IndicatorState.averageStrength = totalStrength / validIndicators;
   else
      g_IndicatorState.averageStrength = 0;
}

//+------------------------------------------------------------------+
//| 📋 نمایش خلاصه اندیکاتورها                                      |
//+------------------------------------------------------------------+
void PrintIndicatorSummary()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("📊 INDICATOR SUMMARY:");
   Print("   ├─ Buy Signals: ", g_IndicatorState.totalBuySignals);
   Print("   ├─ Sell Signals: ", g_IndicatorState.totalSellSignals);
   Print("   ├─ Neutral Signals: ", g_IndicatorState.totalNeutralSignals);
   Print("   ├─ Average Strength: ", DoubleToString(g_IndicatorState.averageStrength, 2), "%");
   
   //--- تعیین سیگنال غالب
   string dominantSignal = "NONE";
   if(g_IndicatorState.totalBuySignals > g_IndicatorState.totalSellSignals && 
      g_IndicatorState.totalBuySignals > g_IndicatorState.totalNeutralSignals)
      dominantSignal = "BUY";
   else if(g_IndicatorState.totalSellSignals > g_IndicatorState.totalBuySignals && 
           g_IndicatorState.totalSellSignals > g_IndicatorState.totalNeutralSignals)
      dominantSignal = "SELL";
   else if(g_IndicatorState.totalNeutralSignals >= g_IndicatorState.totalBuySignals && 
           g_IndicatorState.totalNeutralSignals >= g_IndicatorState.totalSellSignals)
      dominantSignal = "NEUTRAL";
   
   Print("   └─ Dominant Signal: ", dominantSignal);
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
}

//+------------------------------------------------------------------+
//| 🎯 دریافت سیگنال اندیکاتور خاص                                  |
//+------------------------------------------------------------------+
IndicatorSignal GetIndicatorSignal(string indicatorName)
{
   IndicatorSignal empty;
   empty.isValid = false;
   
   if(indicatorName == "EMA") return g_IndicatorState.ema_fast;
   else if(indicatorName == "RSI") return g_IndicatorState.rsi;
   else if(indicatorName == "STOCH") return g_IndicatorState.stochastic;
   else if(indicatorName == "MACD") return g_IndicatorState.macd;
   else if(indicatorName == "ATR") return g_IndicatorState.atr;
   else if(indicatorName == "BB") return g_IndicatorState.bollinger;
   else if(indicatorName == "CCI") return g_IndicatorState.cci;
   else if(indicatorName == "ADX") return g_IndicatorState.adx;
   else if(indicatorName == "WPR") return g_IndicatorState.williams;
   else if(indicatorName == "MOM") return g_IndicatorState.momentum;
   else if(indicatorName == "SAR") return g_IndicatorState.sar;
   else if(indicatorName == "OBV") return g_IndicatorState.obv;
   else if(indicatorName == "AO") return g_IndicatorState.awesome;
   else if(indicatorName == "DEM") return g_IndicatorState.demarker;
   
   return empty;
}

//+------------------------------------------------------------------+
//| 🔍 بررسی سیگنال قوی (چند اندیکاتور هم‌راستا)                   |
//+------------------------------------------------------------------+
bool IsStrongSignalConfirmation(ENUM_SIGNAL_TYPE signalType, int minConfirmations = 5)
{
   int confirmations = 0;
   
   if(g_IndicatorState.ema_fast.signal == signalType && g_IndicatorState.ema_fast.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.rsi.signal == signalType && g_IndicatorState.rsi.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.stochastic.signal == signalType && g_IndicatorState.stochastic.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.macd.signal == signalType && g_IndicatorState.macd.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.bollinger.signal == signalType && g_IndicatorState.bollinger.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.cci.signal == signalType && g_IndicatorState.cci.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.adx.signal == signalType && g_IndicatorState.adx.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.williams.signal == signalType && g_IndicatorState.williams.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.momentum.signal == signalType && g_IndicatorState.momentum.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.sar.signal == signalType && g_IndicatorState.sar.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.obv.signal == signalType && g_IndicatorState.obv.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.awesome.signal == signalType && g_IndicatorState.awesome.strength > 60)
      confirmations++;
   
   if(g_IndicatorState.demarker.signal == signalType && g_IndicatorState.demarker.strength > 60)
      confirmations++;
   
   return (confirmations >= minConfirmations);
}

//+------------------------------------------------------------------+
//| 💯 محاسبه امتیاز کلی سیگنال (0-100)                             |
//+------------------------------------------------------------------+
double CalculateOverallSignalScore(ENUM_SIGNAL_TYPE signalType)
{
   double totalScore = 0;
   int count = 0;
   
   IndicatorSignal signals[];
   ArrayResize(signals, 14);
   
   signals[0] = g_IndicatorState.ema_fast;
   signals[1] = g_IndicatorState.rsi;
   signals[2] = g_IndicatorState.stochastic;
   signals[3] = g_IndicatorState.macd;
   signals[4] = g_IndicatorState.bollinger;
   signals[5] = g_IndicatorState.cci;
   signals[6] = g_IndicatorState.adx;
   signals[7] = g_IndicatorState.williams;
   signals[8] = g_IndicatorState.momentum;
   signals[9] = g_IndicatorState.sar;
   signals[10] = g_IndicatorState.obv;
   signals[11] = g_IndicatorState.awesome;
   signals[12] = g_IndicatorState.demarker;
   
   for(int i = 0; i < 13; i++)
   {
      if(signals[i].isValid)
      {
         if(signals[i].signal == signalType)
         {
            totalScore += signals[i].strength;
            count++;
         }
         else if(signals[i].signal != SIGNAL_NEUTRAL)
         {
            // اگر سیگنال مخالف است، از امتیاز کم می‌کنیم
            totalScore -= (signals[i].strength * 0.5);
         }
      }
   }
   
   if(count == 0)
      return 0;
   
   // نرمال‌سازی به بازه 0-100
   double score = (totalScore / count);
   if(score < 0) score = 0;
   if(score > 100) score = 100;
   
   return score;
}

//+------------------------------------------------------------------+
//| 📊 دریافت گزارش کامل اندیکاتورها                                |
//+------------------------------------------------------------------+
string GetIndicatorReport()
{
   string report = "";
   
   report += "╔════════════════════════════════════════════════════════╗\n";
   report += "║           📊 INDICATOR ANALYSIS REPORT                ║\n";
   report += "╠════════════════════════════════════════════════════════╣\n";
   
   // خلاصه آماری
   report += StringFormat("║ Buy Signals:     %2d/14                               ║\n", 
                          g_IndicatorState.totalBuySignals);
   report += StringFormat("║ Sell Signals:    %2d/14                               ║\n", 
                          g_IndicatorState.totalSellSignals);
   report += StringFormat("║ Neutral Signals: %2d/14                               ║\n", 
                          g_IndicatorState.totalNeutralSignals);
   report += StringFormat("║ Avg Strength:    %.2f%%                              ║\n", 
                          g_IndicatorState.averageStrength);
   report += "╠════════════════════════════════════════════════════════╣\n";
   
   // محاسبه امتیازها
   double buyScore = CalculateOverallSignalScore(SIGNAL_BUY);
   double sellScore = CalculateOverallSignalScore(SIGNAL_SELL);
   
   report += StringFormat("║ Buy Score:       %.2f/100                           ║\n", buyScore);
   report += StringFormat("║ Sell Score:      %.2f/100                           ║\n", sellScore);
   report += "╠════════════════════════════════════════════════════════╣\n";
   
   // توصیه نهایی
   string recommendation = "WAIT";
   if(buyScore > 70 && buyScore > sellScore)
      recommendation = "STRONG BUY";
   else if(buyScore > 60 && buyScore > sellScore)
      recommendation = "BUY";
   else if(sellScore > 70 && sellScore > buyScore)
      recommendation = "STRONG SELL";
   else if(sellScore > 60 && sellScore > buyScore)
      recommendation = "SELL";
   
   report += StringFormat("║ Recommendation:  %-30s       ║\n", recommendation);
   report += "╚════════════════════════════════════════════════════════╝\n";
   
   return report;
}


//+------------------------------------------------------------------+
//| 🎯 بخش استراتژی‌های اسکلپینگ - SCALPING STRATEGIES             |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 📋 ساختار داده استراتژی                                         |
//+------------------------------------------------------------------+
struct Strategy
{
   int               id;              // شناسه استراتژی
   string            name;            // نام استراتژی
   ENUM_SIGNAL_TYPE  signal;          // سیگنال (خرید/فروش/خنثی)
   double            score;           // امتیاز (0-100)
   double            confidence;      // اطمینان (0-100)
   bool              isActive;        // فعال بودن
   string            description;     // توضیحات
   datetime          lastSignalTime;  // زمان آخرین سیگنال
};

//+------------------------------------------------------------------+
//| 🎲 ساختار نتیجه کلی استراتژی‌ها                                 |
//+------------------------------------------------------------------+
struct StrategyResult
{
   int               totalStrategies;        // تعداد کل استراتژی‌ها
   int               activeStrategies;       // تعداد استراتژی‌های فعال
   int               buySignals;             // تعداد سیگنال خرید
   int               sellSignals;            // تعداد سیگنال فروش
   int               neutralSignals;         // تعداد سیگنال خنثی
   double            buyScore;               // امتیاز کل خرید
   double            sellScore;              // امتیاز کل فروش
   double            averageConfidence;      // میانگین اطمینان
   ENUM_SIGNAL_TYPE  finalSignal;            // سیگنال نهایی
   double            finalScore;             // امتیاز نهایی
   string            recommendation;         // توصیه نهایی
};

//--- آرایه استراتژی‌ها
Strategy g_Strategies[25];
StrategyResult g_StrategyResult;


//+------------------------------------------------------------------+
//| 🎯 استراتژی 1: RSI Extreme Scalping                            |
//| توضیح: ورود در نواحی اشباع شدید RSI                             |
//+------------------------------------------------------------------+
Strategy Strategy_01_RSI_Extreme()
{
   Strategy s;
   s.id = 1;
   s.name = "RSI Extreme Scalping";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal rsi = g_IndicatorState.rsi;
   
   if(!rsi.isValid)
      return s;
   
   //--- شرایط ورود
   // خرید: RSI < 25 (اشباع فروش شدید)
   if(rsi.value < 25)
   {
      s.signal = SIGNAL_BUY;
      s.score = 85 + (25 - rsi.value); // هر چه پایین‌تر، امتیاز بیشتر
      s.confidence = 90;
      s.isActive = true;
      s.description = "RSI Extreme Oversold: " + DoubleToString(rsi.value, 2);
   }
   // فروش: RSI > 75 (اشباع خرید شدید)
   else if(rsi.value > 75)
   {
      s.signal = SIGNAL_SELL;
      s.score = 85 + (rsi.value - 75); // هر چه بالاتر، امتیاز بیشتر
      s.confidence = 90;
      s.isActive = true;
      s.description = "RSI Extreme Overbought: " + DoubleToString(rsi.value, 2);
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 2: Bollinger Band Bounce                           |
//| توضیح: برگشت قیمت از باندهای بولینگر                            |
//+------------------------------------------------------------------+
Strategy Strategy_02_BB_Bounce()
{
   Strategy s;
   s.id = 2;
   s.name = "Bollinger Band Bounce";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal bb = g_IndicatorState.bollinger;
   
   if(!bb.isValid)
      return s;
   
   double currentPrice = symbolInfo.Bid();
   double bbUpper = buffer_BB_Upper[0];
   double bbLower = buffer_BB_Lower[0];
   double bbMiddle = buffer_BB_Middle[0];
   
   //--- شرایط ورود
   // خرید: قیمت به باند پایین رسیده یا از آن عبور کرده
   if(currentPrice <= bbLower || bb.value < 15)
   {
      s.signal = SIGNAL_BUY;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Price at/below lower BB";
   }
   // فروش: قیمت به باند بالا رسیده یا از آن عبور کرده
   else if(currentPrice >= bbUpper || bb.value > 85)
   {
      s.signal = SIGNAL_SELL;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Price at/above upper BB";
   }
   // خرید: قیمت نزدیک باند پایین
   else if(bb.value < 25)
   {
      s.signal = SIGNAL_BUY;
      s.score = 70;
      s.confidence = 70;
      s.isActive = true;
      s.description = "Price near lower BB";
   }
   // فروش: قیمت نزدیک باند بالا
   else if(bb.value > 75)
   {
      s.signal = SIGNAL_SELL;
      s.score = 70;
      s.confidence = 70;
      s.isActive = true;
      s.description = "Price near upper BB";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 3: Stochastic Crossover                            |
//| توضیح: تقاطع استوکاستیک در نواحی اشباع                          |
//+------------------------------------------------------------------+
Strategy Strategy_03_Stoch_Cross()
{
   Strategy s;
   s.id = 3;
   s.name = "Stochastic Crossover";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   if(CopyBuffer(handle_Stoch, MAIN_LINE, 0, 3, buffer_Stoch_Main) <= 0)
      return s;
   if(CopyBuffer(handle_Stoch, SIGNAL_LINE, 0, 3, buffer_Stoch_Signal) <= 0)
      return s;
   
   double stoch_main_0 = buffer_Stoch_Main[0];
   double stoch_main_1 = buffer_Stoch_Main[1];
   double stoch_signal_0 = buffer_Stoch_Signal[0];
   double stoch_signal_1 = buffer_Stoch_Signal[1];
   
   //--- شرایط ورود
   // خرید: تقاطع صعودی در ناحیه اشباع فروش
   if(stoch_main_0 > stoch_signal_0 && stoch_main_1 <= stoch_signal_1 && stoch_main_0 < 25)
   {
      s.signal = SIGNAL_BUY;
      s.score = 92;
      s.confidence = 88;
      s.isActive = true;
      s.description = "Bullish cross in oversold (" + DoubleToString(stoch_main_0, 1) + ")";
   }
   // فروش: تقاطع نزولی در ناحیه اشباع خرید
   else if(stoch_main_0 < stoch_signal_0 && stoch_main_1 >= stoch_signal_1 && stoch_main_0 > 75)
   {
      s.signal = SIGNAL_SELL;
      s.score = 92;
      s.confidence = 88;
      s.isActive = true;
      s.description = "Bearish cross in overbought (" + DoubleToString(stoch_main_0, 1) + ")";
   }
   // خرید: تقاطع صعودی معمولی
   else if(stoch_main_0 > stoch_signal_0 && stoch_main_1 <= stoch_signal_1 && stoch_main_0 < 50)
   {
      s.signal = SIGNAL_BUY;
      s.score = 75;
      s.confidence = 70;
      s.isActive = true;
      s.description = "Bullish crossover";
   }
   // فروش: تقاطع نزولی معمولی
   else if(stoch_main_0 < stoch_signal_0 && stoch_main_1 >= stoch_signal_1 && stoch_main_0 > 50)
   {
      s.signal = SIGNAL_SELL;
      s.score = 75;
      s.confidence = 70;
      s.isActive = true;
      s.description = "Bearish crossover";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 4: MACD Zero Cross                                 |
//| توضیح: عبور MACD از خط صفر                                      |
//+------------------------------------------------------------------+
Strategy Strategy_04_MACD_Zero()
{
   Strategy s;
   s.id = 4;
   s.name = "MACD Zero Cross";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   if(CopyBuffer(handle_MACD, MAIN_LINE, 0, 3, buffer_MACD_Main) <= 0)
      return s;
   if(CopyBuffer(handle_MACD, SIGNAL_LINE, 0, 3, buffer_MACD_Signal) <= 0)
      return s;
   
   double macd_main_0 = buffer_MACD_Main[0];
   double macd_main_1 = buffer_MACD_Main[1];
   double macd_signal_0 = buffer_MACD_Signal[0];
   double macd_signal_1 = buffer_MACD_Signal[1];
   
   //--- شرایط ورود
   // خرید: تقاطع صعودی MACD با Signal زیر صفر
   if(macd_main_0 > macd_signal_0 && macd_main_1 <= macd_signal_1 && macd_main_0 < 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 90;
      s.confidence = 85;
      s.isActive = true;
      s.description = "MACD bullish cross below zero";
   }
   // فروش: تقاطع نزولی MACD با Signal بالای صفر
   else if(macd_main_0 < macd_signal_0 && macd_main_1 >= macd_signal_1 && macd_main_0 > 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 90;
      s.confidence = 85;
      s.isActive = true;
      s.description = "MACD bearish cross above zero";
   }
   // خرید: عبور MACD از صفر به سمت بالا
   else if(macd_main_0 > 0 && macd_main_1 <= 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 82;
      s.confidence = 78;
      s.isActive = true;
      s.description = "MACD crossing zero upward";
   }
   // فروش: عبور MACD از صفر به سمت پایین
   else if(macd_main_0 < 0 && macd_main_1 >= 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 82;
      s.confidence = 78;
      s.isActive = true;
      s.description = "MACD crossing zero downward";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 5: SAR Reversal                                    |
//| توضیح: تغییر جهت Parabolic SAR                                  |
//+------------------------------------------------------------------+
Strategy Strategy_05_SAR_Reversal()
{
   Strategy s;
   s.id = 5;
   s.name = "SAR Reversal";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal sar = g_IndicatorState.sar;
   
   if(!sar.isValid)
      return s;
   
   if(CopyBuffer(handle_SAR, 0, 0, 3, buffer_SAR) <= 0)
      return s;
   
   double sar_0 = buffer_SAR[0];
   double sar_1 = buffer_SAR[1];
   
   double currentPrice = symbolInfo.Bid();
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 3, rates) < 3)
      return s;
   
   double previousPrice = rates[1].close;
   
   //--- شرایط ورود
   bool sarBelow = (sar_0 < currentPrice);
   bool sarBelowPrev = (sar_1 < previousPrice);
   
   // خرید: SAR از بالا به پایین آمده (تغییر به روند صعودی)
   if(sarBelow && !sarBelowPrev)
   {
      s.signal = SIGNAL_BUY;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      s.description = "SAR reversal to bullish";
   }
   // فروش: SAR از پایین به بالا رفته (تغییر به روند نزولی)
   else if(!sarBelow && sarBelowPrev)
   {
      s.signal = SIGNAL_SELL;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      s.description = "SAR reversal to bearish";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}


//+------------------------------------------------------------------+
//| 🎯 استراتژی 6: RSI + Stochastic Combo                          |
//| توضیح: تایید RSI با Stochastic                                 |
//+------------------------------------------------------------------+
Strategy Strategy_06_RSI_Stoch()
{
   Strategy s;
   s.id = 6;
   s.name = "RSI + Stochastic Combo";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal rsi = g_IndicatorState.rsi;
   IndicatorSignal stoch = g_IndicatorState.stochastic;
   
   if(!rsi.isValid || !stoch.isValid)
      return s;
   
   //--- شرایط ورود
   // خرید: هر دو در ناحیه اشباع فروش
   if(rsi.value < 30 && buffer_Stoch_Main[0] < 20)
   {
      s.signal = SIGNAL_BUY;
      s.score = 94;
      s.confidence = 92;
      s.isActive = true;
      s.description = "Both oversold - RSI:" + DoubleToString(rsi.value, 1) + 
                      " Stoch:" + DoubleToString(buffer_Stoch_Main[0], 1);
   }
   // فروش: هر دو در ناحیه اشباع خرید
   else if(rsi.value > 70 && buffer_Stoch_Main[0] > 80)
   {
      s.signal = SIGNAL_SELL;
      s.score = 94;
      s.confidence = 92;
      s.isActive = true;
      s.description = "Both overbought - RSI:" + DoubleToString(rsi.value, 1) + 
                      " Stoch:" + DoubleToString(buffer_Stoch_Main[0], 1);
   }
   // خرید: یکی در اشباع فروش شدید و دیگری تایید می‌کند
   else if((rsi.value < 25 && buffer_Stoch_Main[0] < 30) || 
           (rsi.value < 30 && buffer_Stoch_Main[0] < 25))
   {
      s.signal = SIGNAL_BUY;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Strong oversold confirmation";
   }
   // فروش: یکی در اشباع خرید شدید و دیگری تایید می‌کند
   else if((rsi.value > 75 && buffer_Stoch_Main[0] > 70) || 
           (rsi.value > 70 && buffer_Stoch_Main[0] > 75))
   {
      s.signal = SIGNAL_SELL;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Strong overbought confirmation";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 7: EMA + MACD Trend                                |
//| توضیح: تایید روند EMA با MACD                                  |
//+------------------------------------------------------------------+
Strategy Strategy_07_EMA_MACD()
{
   Strategy s;
   s.id = 7;
   s.name = "EMA + MACD Trend";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   if(CopyBuffer(handle_EMA_Fast, 0, 0, 2, buffer_EMA_Fast) <= 0)
      return s;
   if(CopyBuffer(handle_EMA_Slow, 0, 0, 2, buffer_EMA_Slow) <= 0)
      return s;
   if(CopyBuffer(handle_MACD, MAIN_LINE, 0, 2, buffer_MACD_Main) <= 0)
      return s;
   if(CopyBuffer(handle_MACD, SIGNAL_LINE, 0, 2, buffer_MACD_Signal) <= 0)
      return s;
   
   double ema_fast = buffer_EMA_Fast[0];
   double ema_slow = buffer_EMA_Slow[0];
   double macd_main = buffer_MACD_Main[0];
   double macd_signal = buffer_MACD_Signal[0];
   
   double currentPrice = symbolInfo.Bid();
   
   //--- شرایط ورود
   // خرید قوی: EMA صعودی + MACD صعودی + قیمت بالای EMA Fast
   if(ema_fast > ema_slow && macd_main > macd_signal && 
      currentPrice > ema_fast && macd_main > 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 91;
      s.confidence = 88;
      s.isActive = true;
      s.description = "Strong uptrend confirmed by both";
   }
   // فروش قوی: EMA نزولی + MACD نزولی + قیمت پایین EMA Fast
   else if(ema_fast < ema_slow && macd_main < macd_signal && 
           currentPrice < ema_fast && macd_main < 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 91;
      s.confidence = 88;
      s.isActive = true;
      s.description = "Strong downtrend confirmed by both";
   }
   // خرید متوسط: فقط EMA صعودی و MACD مثبت
   else if(ema_fast > ema_slow && macd_main > 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 75;
      s.confidence = 72;
      s.isActive = true;
      s.description = "Uptrend signals";
   }
   // فروش متوسط: فقط EMA نزولی و MACD منفی
   else if(ema_fast < ema_slow && macd_main < 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 75;
      s.confidence = 72;
      s.isActive = true;
      s.description = "Downtrend signals";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 8: BB + CCI Extreme                                |
//| توضیح: ترکیب Bollinger و CCI برای نقاط برگشتی                  |
//+------------------------------------------------------------------+
Strategy Strategy_08_BB_CCI()
{
   Strategy s;
   s.id = 8;
   s.name = "BB + CCI Extreme";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal bb = g_IndicatorState.bollinger;
   IndicatorSignal cci = g_IndicatorState.cci;
   
   if(!bb.isValid || !cci.isValid)
      return s;
   
   double currentPrice = symbolInfo.Bid();
   double bbLower = buffer_BB_Lower[0];
   double bbUpper = buffer_BB_Upper[0];
   
   //--- شرایط ورود
   // خرید: قیمت زیر BB پایین + CCI زیر -200
   if(currentPrice < bbLower && cci.value < -200)
   {
      s.signal = SIGNAL_BUY;
      s.score = 96;
      s.confidence = 94;
      s.isActive = true;
      s.description = "Extreme oversold - Price below BB + CCI<-200";
   }
   // فروش: قیمت بالای BB بالا + CCI بالای +200
   else if(currentPrice > bbUpper && cci.value > 200)
   {
      s.signal = SIGNAL_SELL;
      s.score = 96;
      s.confidence = 94;
      s.isActive = true;
      s.description = "Extreme overbought - Price above BB + CCI>+200";
   }
   // خرید: قیمت نزدیک BB پایین + CCI زیر -100
   else if(bb.value < 20 && cci.value < -100)
   {
      s.signal = SIGNAL_BUY;
      s.score = 86;
      s.confidence = 83;
      s.isActive = true;
      s.description = "Oversold confirmed";
   }
   // فروش: قیمت نزدیک BB بالا + CCI بالای +100
   else if(bb.value > 80 && cci.value > 100)
   {
      s.signal = SIGNAL_SELL;
      s.score = 86;
      s.confidence = 83;
      s.isActive = true;
      s.description = "Overbought confirmed";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 9: ADX + SAR Trend Filter                          |
//| توضیح: تایید قدرت روند با ADX و جهت با SAR                     |
//+------------------------------------------------------------------+
Strategy Strategy_09_ADX_SAR()
{
   Strategy s;
   s.id = 9;
   s.name = "ADX + SAR Trend Filter";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal adx = g_IndicatorState.adx;
   IndicatorSignal sar = g_IndicatorState.sar;
   
   if(!adx.isValid || !sar.isValid)
      return s;
   
   if(CopyBuffer(handle_ADX, MAIN_LINE, 0, 2, buffer_ADX_Main) <= 0)
      return s;
   if(CopyBuffer(handle_ADX, PLUSDI_LINE, 0, 2, buffer_ADX_Plus) <= 0)
      return s;
   if(CopyBuffer(handle_ADX, MINUSDI_LINE, 0, 2, buffer_ADX_Minus) <= 0)
      return s;
   
   double adx_value = buffer_ADX_Main[0];
   double plus_di = buffer_ADX_Plus[0];
   double minus_di = buffer_ADX_Minus[0];
   
   //--- شرایط ورود
   // خرید: ADX > 25 (روند قوی) + SAR صعودی + +DI > -DI
   if(adx_value > 25 && sar.signal == SIGNAL_BUY && plus_di > minus_di)
   {
      s.signal = SIGNAL_BUY;
      s.score = 89;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Strong uptrend - ADX:" + DoubleToString(adx_value, 1);
   }
   // فروش: ADX > 25 (روند قوی) + SAR نزولی + -DI > +DI
   else if(adx_value > 25 && sar.signal == SIGNAL_SELL && minus_di > plus_di)
   {
      s.signal = SIGNAL_SELL;
      s.score = 89;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Strong downtrend - ADX:" + DoubleToString(adx_value, 1);
   }
   // خرید: ADX > 20 و SAR صعودی
   else if(adx_value > 20 && sar.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 78;
      s.confidence = 75;
      s.isActive = true;
      s.description = "Moderate uptrend";
   }
   // فروش: ADX > 20 و SAR نزولی
   else if(adx_value > 20 && sar.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 78;
      s.confidence = 75;
      s.isActive = true;
      s.description = "Moderate downtrend";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 10: Williams + DeMarker Double Check               |
//| توضیح: تایید دوگانه اشباع با Williams و DeMarker               |
//+------------------------------------------------------------------+
Strategy Strategy_10_WPR_DEM()
{
   Strategy s;
   s.id = 10;
   s.name = "Williams + DeMarker";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal wpr = g_IndicatorState.williams;
   IndicatorSignal dem = g_IndicatorState.demarker;
   
   if(!wpr.isValid || !dem.isValid)
      return s;
   
   //--- شرایط ورود
   // خرید: Williams اشباع فروش + DeMarker اشباع فروش
   if(wpr.value < -80 && dem.value < 0.3)
   {
      s.signal = SIGNAL_BUY;
      s.score = 92;
      s.confidence = 89;
      s.isActive = true;
      s.description = "Double oversold - WPR:" + DoubleToString(wpr.value, 1) + 
                      " DEM:" + DoubleToString(dem.value, 2);
   }
   // فروش: Williams اشباع خرید + DeMarker اشباع خرید
   else if(wpr.value > -20 && dem.value > 0.7)
   {
      s.signal = SIGNAL_SELL;
      s.score = 92;
      s.confidence = 89;
      s.isActive = true;
      s.description = "Double overbought - WPR:" + DoubleToString(wpr.value, 1) + 
                      " DEM:" + DoubleToString(dem.value, 2);
   }
   // خرید: یکی اشباع فروش شدید
   else if(wpr.value < -85 || dem.value < 0.25)
   {
      s.signal = SIGNAL_BUY;
      s.score = 82;
      s.confidence = 79;
      s.isActive = true;
      s.description = "Extreme oversold detected";
   }
   // فروش: یکی اشباع خرید شدید
   else if(wpr.value > -15 || dem.value > 0.75)
   {
      s.signal = SIGNAL_SELL;
      s.score = 82;
      s.confidence = 79;
      s.isActive = true;
      s.description = "Extreme overbought detected";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}


//+------------------------------------------------------------------+
//| 🎯 استراتژی 11: Triple Confirmation (RSI+MACD+Stoch)           |
//| توضیح: تایید سه‌گانه برای اطمینان بالا                          |
//+------------------------------------------------------------------+
Strategy Strategy_11_Triple_Confirm()
{
   Strategy s;
   s.id = 11;
   s.name = "Triple Confirmation";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal rsi = g_IndicatorState.rsi;
   IndicatorSignal macd = g_IndicatorState.macd;
   IndicatorSignal stoch = g_IndicatorState.stochastic;
   
   if(!rsi.isValid || !macd.isValid || !stoch.isValid)
      return s;
   
   //--- شرایط ورود
   // خرید: هر سه اندیکاتور سیگنال خرید می‌دهند
   if(rsi.signal == SIGNAL_BUY && macd.signal == SIGNAL_BUY && stoch.signal == SIGNAL_BUY)
   {
      // محاسبه میانگین قدرت
      double avgStrength = (rsi.strength + macd.strength + stoch.strength) / 3.0;
      
      s.signal = SIGNAL_BUY;
      s.score = 97;
      s.confidence = 95;
      s.isActive = true;
      s.description = "Triple BUY confirmation - Avg strength:" + DoubleToString(avgStrength, 1);
   }
   // فروش: هر سه اندیکاتور سیگنال فروش می‌دهند
   else if(rsi.signal == SIGNAL_SELL && macd.signal == SIGNAL_SELL && stoch.signal == SIGNAL_SELL)
   {
      double avgStrength = (rsi.strength + macd.strength + stoch.strength) / 3.0;
      
      s.signal = SIGNAL_SELL;
      s.score = 97;
      s.confidence = 95;
      s.isActive = true;
      s.description = "Triple SELL confirmation - Avg strength:" + DoubleToString(avgStrength, 1);
   }
   // خرید: دو از سه تایید می‌کنند
   else if((rsi.signal == SIGNAL_BUY && macd.signal == SIGNAL_BUY) ||
           (rsi.signal == SIGNAL_BUY && stoch.signal == SIGNAL_BUY) ||
           (macd.signal == SIGNAL_BUY && stoch.signal == SIGNAL_BUY))
   {
      s.signal = SIGNAL_BUY;
      s.score = 85;
      s.confidence = 82;
      s.isActive = true;
      s.description = "Double BUY confirmation (2/3)";
   }
   // فروش: دو از سه تایید می‌کنند
   else if((rsi.signal == SIGNAL_SELL && macd.signal == SIGNAL_SELL) ||
           (rsi.signal == SIGNAL_SELL && stoch.signal == SIGNAL_SELL) ||
           (macd.signal == SIGNAL_SELL && stoch.signal == SIGNAL_SELL))
   {
      s.signal = SIGNAL_SELL;
      s.score = 85;
      s.confidence = 82;
      s.isActive = true;
      s.description = "Double SELL confirmation (2/3)";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 12: Volume Price Trend (OBV+AO+Momentum)           |
//| توضیح: تحلیل حجم و مومنتوم برای شناسایی جریان پول              |
//+------------------------------------------------------------------+
Strategy Strategy_12_Volume_Price()
{
   Strategy s;
   s.id = 12;
   s.name = "Volume Price Trend";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal obv = g_IndicatorState.obv;
   IndicatorSignal ao = g_IndicatorState.awesome;
   IndicatorSignal mom = g_IndicatorState.momentum;
   
   if(!obv.isValid || !ao.isValid || !mom.isValid)
      return s;
   
   //--- بررسی روند قیمت
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 5, rates) < 5)
      return s;
   
   double priceChange = rates[0].close - rates[3].close;
   
   //--- شرایط ورود
   // خرید: واگرایی مثبت یا همسویی قوی صعودی
   if(obv.signal == SIGNAL_BUY && ao.signal == SIGNAL_BUY && mom.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      
      if(priceChange < 0 && obv.value > 0) // واگرایی صعودی
         s.description = "Bullish divergence - Strong accumulation";
      else
         s.description = "Strong buying volume + momentum";
   }
   // فروش: واگرایی منفی یا همسویی قوی نزولی
   else if(obv.signal == SIGNAL_SELL && ao.signal == SIGNAL_SELL && mom.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      
      if(priceChange > 0 && obv.value < 0) // واگرایی نزولی
         s.description = "Bearish divergence - Strong distribution";
      else
         s.description = "Strong selling volume + momentum";
   }
   // خرید: دو از سه با حجم بالا
   else if((obv.signal == SIGNAL_BUY && ao.signal == SIGNAL_BUY) ||
           (obv.signal == SIGNAL_BUY && mom.signal == SIGNAL_BUY))
   {
      s.signal = SIGNAL_BUY;
      s.score = 84;
      s.confidence = 81;
      s.isActive = true;
      s.description = "Volume confirms buying pressure";
   }
   // فروش: دو از سه با حجم بالا
   else if((obv.signal == SIGNAL_SELL && ao.signal == SIGNAL_SELL) ||
           (obv.signal == SIGNAL_SELL && mom.signal == SIGNAL_SELL))
   {
      s.signal = SIGNAL_SELL;
      s.score = 84;
      s.confidence = 81;
      s.isActive = true;
      s.description = "Volume confirms selling pressure";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 13: Trend + Oscillator Scalp (EMA+ADX+CCI)        |
//| توضیح: ترکیب روند و نوسانگر برای اسکلپ دقیق                    |
//+------------------------------------------------------------------+
Strategy Strategy_13_Trend_Osc()
{
   Strategy s;
   s.id = 13;
   s.name = "Trend + Oscillator Scalp";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal ema = g_IndicatorState.ema_fast;
   IndicatorSignal adx = g_IndicatorState.adx;
   IndicatorSignal cci = g_IndicatorState.cci;
   
   if(!ema.isValid || !adx.isValid || !cci.isValid)
      return s;
   
   if(CopyBuffer(handle_ADX, MAIN_LINE, 0, 2, buffer_ADX_Main) <= 0)
      return s;
   
   double adx_value = buffer_ADX_Main[0];
   
   //--- شرایط ورود
   // خرید: روند صعودی قوی + CCI اشباع فروش (pullback در روند صعودی)
   if(ema.signal == SIGNAL_BUY && adx_value > 25 && cci.value < -100)
   {
      s.signal = SIGNAL_BUY;
      s.score = 95;
      s.confidence = 92;
      s.isActive = true;
      s.description = "Strong uptrend + CCI pullback - ADX:" + DoubleToString(adx_value, 1);
   }
   // فروش: روند نزولی قوی + CCI اشباع خرید (pullback در روند نزولی)
   else if(ema.signal == SIGNAL_SELL && adx_value > 25 && cci.value > 100)
   {
      s.signal = SIGNAL_SELL;
      s.score = 95;
      s.confidence = 92;
      s.isActive = true;
      s.description = "Strong downtrend + CCI pullback - ADX:" + DoubleToString(adx_value, 1);
   }
   // خرید: روند صعودی متوسط + CCI صعودی
   else if(ema.signal == SIGNAL_BUY && adx_value > 20 && cci.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 83;
      s.confidence = 80;
      s.isActive = true;
      s.description = "Uptrend confirmed by CCI";
   }
   // فروش: روند نزولی متوسط + CCI نزولی
   else if(ema.signal == SIGNAL_SELL && adx_value > 20 && cci.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 83;
      s.confidence = 80;
      s.isActive = true;
      s.description = "Downtrend confirmed by CCI";
   }
   // خرید: هر سه موافق
   else if(ema.signal == SIGNAL_BUY && adx.signal == SIGNAL_BUY && cci.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Triple trend confirmation";
   }
   // فروش: هر سه موافق
   else if(ema.signal == SIGNAL_SELL && adx.signal == SIGNAL_SELL && cci.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 88;
      s.confidence = 85;
      s.isActive = true;
      s.description = "Triple trend confirmation";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 14: Extreme Reversal (BB+WPR+DEM)                  |
//| توضیح: شناسایی نقاط بازگشتی شدید                                |
//+------------------------------------------------------------------+
Strategy Strategy_14_Extreme_Reversal()
{
   Strategy s;
   s.id = 14;
   s.name = "Extreme Reversal";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal bb = g_IndicatorState.bollinger;
   IndicatorSignal wpr = g_IndicatorState.williams;
   IndicatorSignal dem = g_IndicatorState.demarker;
   
   if(!bb.isValid || !wpr.isValid || !dem.isValid)
      return s;
   
   double currentPrice = symbolInfo.Bid();
   double bbLower = buffer_BB_Lower[0];
   double bbUpper = buffer_BB_Upper[0];
   
   //--- شرایط ورود
   // خرید شدید: قیمت زیر BB + WPR زیر -85 + DeMarker زیر 0.25
   if(currentPrice < bbLower && wpr.value < -85 && dem.value < 0.25)
   {
      s.signal = SIGNAL_BUY;
      s.score = 98;
      s.confidence = 96;
      s.isActive = true;
      s.description = "EXTREME oversold - Triple confirmation";
   }
   // فروش شدید: قیمت بالای BB + WPR بالای -15 + DeMarker بالای 0.75
   else if(currentPrice > bbUpper && wpr.value > -15 && dem.value > 0.75)
   {
      s.signal = SIGNAL_SELL;
      s.score = 98;
      s.confidence = 96;
      s.isActive = true;
      s.description = "EXTREME overbought - Triple confirmation";
   }
   // خرید قوی: قیمت نزدیک BB پایین + اشباع فروش
   else if(bb.value < 20 && wpr.value < -80 && dem.value < 0.3)
   {
      s.signal = SIGNAL_BUY;
      s.score = 90;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Strong oversold - High reversal probability";
   }
   // فروش قوی: قیمت نزدیک BB بالا + اشباع خرید
   else if(bb.value > 80 && wpr.value > -20 && dem.value > 0.7)
   {
      s.signal = SIGNAL_SELL;
      s.score = 90;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Strong overbought - High reversal probability";
   }
   // خرید متوسط: دو از سه در اشباع فروش
   else if((bb.value < 25 && wpr.value < -80) ||
           (bb.value < 25 && dem.value < 0.3) ||
           (wpr.value < -80 && dem.value < 0.3))
   {
      s.signal = SIGNAL_BUY;
      s.score = 81;
      s.confidence = 78;
      s.isActive = true;
      s.description = "Oversold - 2/3 confirmation";
   }
   // فروش متوسط: دو از سه در اشباع خرید
   else if((bb.value > 75 && wpr.value > -20) ||
           (bb.value > 75 && dem.value > 0.7) ||
           (wpr.value > -20 && dem.value > 0.7))
   {
      s.signal = SIGNAL_SELL;
      s.score = 81;
      s.confidence = 78;
      s.isActive = true;
      s.description = "Overbought - 2/3 confirmation";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 15: Breakout Momentum (SAR+AO+ATR)                 |
//| توضیح: شناسایی شکست با مومنتوم و نوسان بالا                     |
//+------------------------------------------------------------------+
Strategy Strategy_15_Breakout_Mom()
{
   Strategy s;
   s.id = 15;
   s.name = "Breakout Momentum";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal sar = g_IndicatorState.sar;
   IndicatorSignal ao = g_IndicatorState.awesome;
   IndicatorSignal atr = g_IndicatorState.atr;
   
   if(!sar.isValid || !ao.isValid || !atr.isValid)
      return s;
   
   if(CopyBuffer(handle_SAR, 0, 0, 3, buffer_SAR) <= 0)
      return s;
   if(CopyBuffer(handle_AO, 0, 0, 3, buffer_AO) <= 0)
      return s;
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return s;
   
   double sar_0 = buffer_SAR[0];
   double sar_1 = buffer_SAR[1];
   double ao_0 = buffer_AO[0];
   double ao_1 = buffer_AO[1];
   double atr_0 = buffer_ATR[0];
   
   // محاسبه میانگین ATR
   double atr_avg = 0;
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   double currentPrice = symbolInfo.Bid();
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 3, rates) < 3)
      return s;
   
   double previousPrice = rates[1].close;
   
   bool sarBelow = (sar_0 < currentPrice);
   bool sarBelowPrev = (sar_1 < previousPrice);
   
   //--- شرایط ورود
   // خرید: SAR reversal + AO صعودی + ATR بالا (نوسان خوب)
   if(sarBelow && !sarBelowPrev && ao_0 > 0 && ao_0 > ao_1 && atr_0 > atr_avg * 1.2)
   {
      s.signal = SIGNAL_BUY;
      s.score = 94;
      s.confidence = 91;
      s.isActive = true;
      s.description = "Bullish breakout with momentum - High volatility";
   }
   // فروش: SAR reversal + AO نزولی + ATR بالا
   else if(!sarBelow && sarBelowPrev && ao_0 < 0 && ao_0 < ao_1 && atr_0 > atr_avg * 1.2)
   {
      s.signal = SIGNAL_SELL;
      s.score = 94;
      s.confidence = 91;
      s.isActive = true;
      s.description = "Bearish breakout with momentum - High volatility";
   }
   // خرید: SAR صعودی + AO مثبت + نوسان متوسط
   else if(sarBelow && ao_0 > 0 && atr_0 > atr_avg * 0.8)
   {
      s.signal = SIGNAL_BUY;
      s.score = 82;
      s.confidence = 79;
      s.isActive = true;
      s.description = "Uptrend with positive momentum";
   }
   // فروش: SAR نزولی + AO منفی + نوسان متوسط
   else if(!sarBelow && ao_0 < 0 && atr_0 > atr_avg * 0.8)
   {
      s.signal = SIGNAL_SELL;
      s.score = 82;
      s.confidence = 79;
      s.isActive = true;
      s.description = "Downtrend with negative momentum";
   }
   // هشدار: نوسان پایین - خطرناک برای اسکلپ
   else if(atr_0 < atr_avg * 0.6)
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 0;
      s.isActive = true;
      s.description = "Low volatility - Avoid scalping";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}


//+------------------------------------------------------------------+
//| 🎯 استراتژی 16: Price Action Candle Pattern                    |
//| توضیح: تشخیص الگوهای کندلی قوی                                  |
//+------------------------------------------------------------------+
Strategy Strategy_16_Price_Action()
{
   Strategy s;
   s.id = 16;
   s.name = "Price Action Candle";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 5, rates) < 5)
      return s;
   
   double open_0 = rates[0].open;
   double close_0 = rates[0].close;
   double high_0 = rates[0].high;
   double low_0 = rates[0].low;
   
   double open_1 = rates[1].open;
   double close_1 = rates[1].close;
   double high_1 = rates[1].high;
   double low_1 = rates[1].low;
   
   double open_2 = rates[2].open;
   double close_2 = rates[2].close;
   
   double body_0 = MathAbs(close_0 - open_0);
   double body_1 = MathAbs(close_1 - open_1);
   double range_0 = high_0 - low_0;
   double upperShadow_0 = high_0 - MathMax(open_0, close_0);
   double lowerShadow_0 = MathMin(open_0, close_0) - low_0;
   
   //--- الگوی Hammer (چکش صعودی)
   if(close_0 > open_0 && // کندل صعودی
      body_0 < range_0 * 0.3 && // بدنه کوچک
      lowerShadow_0 > body_0 * 2 && // سایه پایینی بلند
      upperShadow_0 < body_0 * 0.3 && // سایه بالایی کوتاه
      close_1 < open_1) // کندل قبلی نزولی
   {
      s.signal = SIGNAL_BUY;
      s.score = 87;
      s.confidence = 84;
      s.isActive = true;
      s.description = "Hammer pattern - Bullish reversal";
   }
   //--- الگوی Shooting Star (ستاره ثاقب نزولی)
   else if(close_0 < open_0 && // کندل نزولی
           body_0 < range_0 * 0.3 && // بدنه کوچک
           upperShadow_0 > body_0 * 2 && // سایه بالایی بلند
           lowerShadow_0 < body_0 * 0.3 && // سایه پایینی کوتاه
           close_1 > open_1) // کندل قبلی صعودی
   {
      s.signal = SIGNAL_SELL;
      s.score = 87;
      s.confidence = 84;
      s.isActive = true;
      s.description = "Shooting Star - Bearish reversal";
   }
   //--- الگوی Engulfing صعودی
   else if(close_0 > open_0 && close_1 < open_1 && // کندل فعلی صعودی، قبلی نزولی
           close_0 > open_1 && open_0 < close_1 && // در بر گرفتن کامل
           body_0 > body_1 * 1.5) // کندل فعلی بزرگتر
   {
      s.signal = SIGNAL_BUY;
      s.score = 90;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Bullish Engulfing pattern";
   }
   //--- الگوی Engulfing نزولی
   else if(close_0 < open_0 && close_1 > open_1 && // کندل فعلی نزولی، قبلی صعودی
           close_0 < open_1 && open_0 > close_1 && // در بر گرفتن کامل
           body_0 > body_1 * 1.5) // کندل فعلی بزرگتر
   {
      s.signal = SIGNAL_SELL;
      s.score = 90;
      s.confidence = 87;
      s.isActive = true;
      s.description = "Bearish Engulfing pattern";
   }
   //--- الگوی Pin Bar صعودی
   else if(lowerShadow_0 > body_0 * 2.5 && upperShadow_0 < body_0 * 0.5)
   {
      s.signal = SIGNAL_BUY;
      s.score = 83;
      s.confidence = 80;
      s.isActive = true;
      s.description = "Bullish Pin Bar";
   }
   //--- الگوی Pin Bar نزولی
   else if(upperShadow_0 > body_0 * 2.5 && lowerShadow_0 < body_0 * 0.5)
   {
      s.signal = SIGNAL_SELL;
      s.score = 83;
      s.confidence = 80;
      s.isActive = true;
      s.description = "Bearish Pin Bar";
   }
   //--- الگوی Doji (بی‌تصمیمی)
   else if(body_0 < range_0 * 0.1) // بدنه خیلی کوچک
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 70;
      s.isActive = true;
      s.description = "Doji - Indecision (potential reversal)";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 17: Support Resistance Bounce                      |
//| توضیح: برگشت از سطوح حمایت و مقاومت                             |
//+------------------------------------------------------------------+
Strategy Strategy_17_SR_Bounce()
{
   Strategy s;
   s.id = 17;
   s.name = "Support Resistance Bounce";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 50, rates) < 50)
      return s;
   
   double currentPrice = symbolInfo.Bid();
   
   //--- محاسبه سطوح حمایت و مقاومت (High/Low آخرین 20 کندل)
   double highest = rates[0].high;
   double lowest = rates[0].low;
   
   for(int i = 1; i < 20; i++)
   {
      if(rates[i].high > highest) highest = rates[i].high;
      if(rates[i].low < lowest) lowest = rates[i].low;
   }
   
   double range = highest - lowest;
   double support = lowest;
   double resistance = highest;
   double middle = (highest + lowest) / 2;
   
   //--- محاسبه فاصله قیمت از سطوح
   double distToSupport = currentPrice - support;
   double distToResistance = resistance - currentPrice;
   
   //--- شرایط ورود
   // خرید: نزدیک سطح حمایت (5% از رنج)
   if(distToSupport < range * 0.05 && distToSupport > 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 86;
      s.confidence = 83;
      s.isActive = true;
      s.description = "Near support level - Bounce expected";
   }
   // فروش: نزدیک سطح مقاومت (5% از رنج)
   else if(distToResistance < range * 0.05 && distToResistance > 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 86;
      s.confidence = 83;
      s.isActive = true;
      s.description = "Near resistance level - Rejection expected";
   }
   // خرید: برگشت از حمایت
   else if(currentPrice > support && currentPrice < support + range * 0.08 && 
           rates[0].close > rates[0].open) // کندل فعلی صعودی
   {
      s.signal = SIGNAL_BUY;
      s.score = 80;
      s.confidence = 77;
      s.isActive = true;
      s.description = "Bouncing from support";
   }
   // فروش: برگشت از مقاومت
   else if(currentPrice < resistance && currentPrice > resistance - range * 0.08 && 
           rates[0].close < rates[0].open) // کندل فعلی نزولی
   {
      s.signal = SIGNAL_SELL;
      s.score = 80;
      s.confidence = 77;
      s.isActive = true;
      s.description = "Rejecting from resistance";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 18: Moving Average Crossover Speed                 |
//| توضیح: سرعت تقاطع میانگین متحرک                                 |
//+------------------------------------------------------------------+
Strategy Strategy_18_MA_Speed()
{
   Strategy s;
   s.id = 18;
   s.name = "MA Crossover Speed";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   if(CopyBuffer(handle_EMA_Fast, 0, 0, 5, buffer_EMA_Fast) <= 0)
      return s;
   if(CopyBuffer(handle_EMA_Slow, 0, 0, 5, buffer_EMA_Slow) <= 0)
      return s;
   
   double ema_fast_0 = buffer_EMA_Fast[0];
   double ema_fast_1 = buffer_EMA_Fast[1];
   double ema_fast_2 = buffer_EMA_Fast[2];
   
   double ema_slow_0 = buffer_EMA_Slow[0];
   double ema_slow_1 = buffer_EMA_Slow[1];
   double ema_slow_2 = buffer_EMA_Slow[2];
   
   //--- محاسبه سرعت تغییرات
   double fastSpeed = ema_fast_0 - ema_fast_2;
   double slowSpeed = ema_slow_0 - ema_slow_2;
   double separation = MathAbs(ema_fast_0 - ema_slow_0);
   
   //--- شرایط ورود
   // تقاطع طلایی با سرعت بالا
   if(ema_fast_0 > ema_slow_0 && ema_fast_1 <= ema_slow_1 && fastSpeed > 0)
   {
      s.signal = SIGNAL_BUY;
      s.score = 89;
      s.confidence = 86;
      s.isActive = true;
      s.description = "Golden Cross with momentum";
   }
   // تقاطع مرگ با سرعت بالا
   else if(ema_fast_0 < ema_slow_0 && ema_fast_1 >= ema_slow_1 && fastSpeed < 0)
   {
      s.signal = SIGNAL_SELL;
      s.score = 89;
      s.confidence = 86;
      s.isActive = true;
      s.description = "Death Cross with momentum";
   }
   // جدایش در حال افزایش (روند قوی شونده)
   else if(ema_fast_0 > ema_slow_0 && separation > MathAbs(ema_fast_1 - ema_slow_1))
   {
      s.signal = SIGNAL_BUY;
      s.score = 77;
      s.confidence = 74;
      s.isActive = true;
      s.description = "Uptrend strengthening";
   }
   else if(ema_fast_0 < ema_slow_0 && separation > MathAbs(ema_fast_1 - ema_slow_1))
   {
      s.signal = SIGNAL_SELL;
      s.score = 77;
      s.confidence = 74;
      s.isActive = true;
      s.description = "Downtrend strengthening";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 19: Volatility Breakout (ATR+BB+Momentum)          |
//| توضیح: شکست با نوسان بالا                                       |
//+------------------------------------------------------------------+
Strategy Strategy_19_Volatility_Break()
{
   Strategy s;
   s.id = 19;
   s.name = "Volatility Breakout";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal atr = g_IndicatorState.atr;
   IndicatorSignal bb = g_IndicatorState.bollinger;
   IndicatorSignal mom = g_IndicatorState.momentum;
   
   if(!atr.isValid || !bb.isValid || !mom.isValid)
      return s;
   
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return s;
   
   double atr_0 = buffer_ATR[0];
   double atr_avg = 0;
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   double currentPrice = symbolInfo.Bid();
   double bbUpper = buffer_BB_Upper[0];
   double bbLower = buffer_BB_Lower[0];
   double bbMiddle = buffer_BB_Middle[0];
   
   //--- محاسبه عرض باند
   double bandwidth = ((bbUpper - bbLower) / bbMiddle) * 100;
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 3, rates) < 3)
      return s;
   
   double candleSize = MathAbs(rates[0].close - rates[0].open);
   
   //--- شرایط ورود
   // شکست صعودی: ATR بالا + شکست BB بالا + Momentum مثبت + کندل بزرگ
   if(atr_0 > atr_avg * 1.3 && currentPrice > bbUpper && 
      mom.signal == SIGNAL_BUY && candleSize > atr_0 * 0.5)
   {
      s.signal = SIGNAL_BUY;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      s.description = "Strong bullish breakout - High volatility";
   }
   // شکست نزولی: ATR بالا + شکست BB پایین + Momentum منفی + کندل بزرگ
   else if(atr_0 > atr_avg * 1.3 && currentPrice < bbLower && 
           mom.signal == SIGNAL_SELL && candleSize > atr_0 * 0.5)
   {
      s.signal = SIGNAL_SELL;
      s.score = 93;
      s.confidence = 90;
      s.isActive = true;
      s.description = "Strong bearish breakout - High volatility";
   }
   // Squeeze Breakout: باند باریک شده و حالا در حال گسترش
   else if(bandwidth < 2 && atr_0 > atr_avg * 1.2)
   {
      if(currentPrice > bbMiddle && mom.signal == SIGNAL_BUY)
      {
         s.signal = SIGNAL_BUY;
         s.score = 88;
         s.confidence = 85;
         s.isActive = true;
         s.description = "Squeeze breakout upward";
      }
      else if(currentPrice < bbMiddle && mom.signal == SIGNAL_SELL)
      {
         s.signal = SIGNAL_SELL;
         s.score = 88;
         s.confidence = 85;
         s.isActive = true;
         s.description = "Squeeze breakout downward";
      }
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 20: Multi-Timeframe Confirmation                   |
//| توضیح: تایید سیگنال با تایم فریم بالاتر                         |
//+------------------------------------------------------------------+
Strategy Strategy_20_MTF_Confirm()
{
   Strategy s;
   s.id = 20;
   s.name = "Multi-Timeframe Confirmation";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   //--- خواندن EMA از تایم فریم کمکی (M5)
   double ema_fast_m5[], ema_slow_m5[];
   ArraySetAsSeries(ema_fast_m5, true);
   ArraySetAsSeries(ema_slow_m5, true);
   
   int handle_ema_fast_m5 = iMA(InpTradeSymbol, InpHelperTF, 9, 0, MODE_EMA, PRICE_CLOSE);
   int handle_ema_slow_m5 = iMA(InpTradeSymbol, InpHelperTF, 21, 0, MODE_EMA, PRICE_CLOSE);
   
   if(handle_ema_fast_m5 == INVALID_HANDLE || handle_ema_slow_m5 == INVALID_HANDLE)
      return s;
   
   if(CopyBuffer(handle_ema_fast_m5, 0, 0, 3, ema_fast_m5) <= 0)
   {
      IndicatorRelease(handle_ema_fast_m5);
      IndicatorRelease(handle_ema_slow_m5);
      return s;
   }
   
   if(CopyBuffer(handle_ema_slow_m5, 0, 0, 3, ema_slow_m5) <= 0)
   {
      IndicatorRelease(handle_ema_fast_m5);
      IndicatorRelease(handle_ema_slow_m5);
      return s;
   }
   
   //--- روند M1
   bool m1_uptrend = (buffer_EMA_Fast[0] > buffer_EMA_Slow[0]);
   bool m1_downtrend = (buffer_EMA_Fast[0] < buffer_EMA_Slow[0]);
   
   //--- روند M5
   bool m5_uptrend = (ema_fast_m5[0] > ema_slow_m5[0]);
   bool m5_downtrend = (ema_fast_m5[0] < ema_slow_m5[0]);
   
   //--- شرایط ورود
   // خرید: هر دو تایم فریم صعودی
   if(m1_uptrend && m5_uptrend)
   {
      s.signal = SIGNAL_BUY;
      s.score = 91;
      s.confidence = 88;
      s.isActive = true;
      s.description = "MTF bullish - M1 & M5 aligned";
   }
   // فروش: هر دو تایم فریم نزولی
   else if(m1_downtrend && m5_downtrend)
   {
      s.signal = SIGNAL_SELL;
      s.score = 91;
      s.confidence = 88;
      s.isActive = true;
      s.description = "MTF bearish - M1 & M5 aligned";
   }
   // تضاد تایم فریم - اجتناب از ورود
   else if((m1_uptrend && m5_downtrend) || (m1_downtrend && m5_uptrend))
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 60;
      s.isActive = true;
      s.description = "MTF conflict - Avoid trading";
   }
   
   //--- پاکسازی
   IndicatorRelease(handle_ema_fast_m5);
   IndicatorRelease(handle_ema_slow_m5);
   
   s.lastSignalTime = TimeCurrent();
   return s;
}


//+------------------------------------------------------------------+
//| 🎯 استراتژی 21: Golden Scalp (ترکیب 5 اندیکاتور طلایی)        |
//| توضیح: بهترین ترکیب برای اسکلپ طلا                              |
//+------------------------------------------------------------------+
Strategy Strategy_21_Golden_Scalp()
{
   Strategy s;
   s.id = 21;
   s.name = "Golden Scalp XAUUSD";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   //--- اندیکاتورهای کلیدی برای طلا
   IndicatorSignal rsi = g_IndicatorState.rsi;
   IndicatorSignal bb = g_IndicatorState.bollinger;
   IndicatorSignal cci = g_IndicatorState.cci;
   IndicatorSignal atr = g_IndicatorState.atr;
   IndicatorSignal sar = g_IndicatorState.sar;
   
   if(!rsi.isValid || !bb.isValid || !cci.isValid || !atr.isValid || !sar.isValid)
      return s;
   
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return s;
   
   double atr_0 = buffer_ATR[0];
   double atr_avg = 0;
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   double currentPrice = symbolInfo.Bid();
   double bbLower = buffer_BB_Lower[0];
   double bbUpper = buffer_BB_Upper[0];
   
   int buyCount = 0;
   int sellCount = 0;
   double totalStrength = 0;
   
   //--- شمارش رای‌ها
   if(rsi.signal == SIGNAL_BUY) { buyCount++; totalStrength += rsi.strength; }
   else if(rsi.signal == SIGNAL_SELL) { sellCount++; totalStrength += rsi.strength; }
   
   if(bb.signal == SIGNAL_BUY) { buyCount++; totalStrength += bb.strength; }
   else if(bb.signal == SIGNAL_SELL) { sellCount++; totalStrength += bb.strength; }
   
   if(cci.signal == SIGNAL_BUY) { buyCount++; totalStrength += cci.strength; }
   else if(cci.signal == SIGNAL_SELL) { sellCount++; totalStrength += cci.strength; }
   
   if(sar.signal == SIGNAL_BUY) { buyCount++; totalStrength += sar.strength; }
   else if(sar.signal == SIGNAL_SELL) { sellCount++; totalStrength += sar.strength; }
   
   //--- شرایط ورود (حداقل 4 از 5 باید موافق باشند)
   // خرید EXTREME: همه 5 موافق + نوسان بالا + قیمت زیر/نزدیک BB پایین
   if(buyCount >= 4 && atr_0 > atr_avg * 1.2 && currentPrice <= bbLower * 1.001)
   {
      s.signal = SIGNAL_BUY;
      s.score = 99;
      s.confidence = 97;
      s.isActive = true;
      s.description = StringFormat("GOLDEN BUY - %d/5 confirmations - EXTREME", buyCount);
   }
   // فروش EXTREME: همه 5 موافق + نوسان بالا + قیمت بالا/نزدیک BB بالا
   else if(sellCount >= 4 && atr_0 > atr_avg * 1.2 && currentPrice >= bbUpper * 0.999)
   {
      s.signal = SIGNAL_SELL;
      s.score = 99;
      s.confidence = 97;
      s.isActive = true;
      s.description = StringFormat("GOLDEN SELL - %d/5 confirmations - EXTREME", sellCount);
   }
   // خرید قوی: 4 از 5 موافق + نوسان خوب
   else if(buyCount >= 4 && atr_0 > atr_avg * 0.9)
   {
      s.signal = SIGNAL_BUY;
      s.score = 94;
      s.confidence = 91;
      s.isActive = true;
      s.description = StringFormat("GOLDEN BUY - %d/5 confirmations", buyCount);
   }
   // فروش قوی: 4 از 5 موافق + نوسان خوب
   else if(sellCount >= 4 && atr_0 > atr_avg * 0.9)
   {
      s.signal = SIGNAL_SELL;
      s.score = 94;
      s.confidence = 91;
      s.isActive = true;
      s.description = StringFormat("GOLDEN SELL - %d/5 confirmations", sellCount);
   }
   // خرید متوسط: 3 از 5
   else if(buyCount >= 3 && atr_0 > atr_avg * 0.8)
   {
      s.signal = SIGNAL_BUY;
      s.score = 85;
      s.confidence = 82;
      s.isActive = true;
      s.description = StringFormat("GOLDEN BUY - %d/5 confirmations", buyCount);
   }
   // فروش متوسط: 3 از 5
   else if(sellCount >= 3 && atr_0 > atr_avg * 0.8)
   {
      s.signal = SIGNAL_SELL;
      s.score = 85;
      s.confidence = 82;
      s.isActive = true;
      s.description = StringFormat("GOLDEN SELL - %d/5 confirmations", sellCount);
   }
   // نوسان پایین - اجتناب
   else if(atr_0 < atr_avg * 0.6)
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 0;
      s.isActive = true;
      s.description = "Low volatility - Not suitable for gold scalping";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 22: Smart Money Concept                            |
//| توضیح: ردیابی پول هوشمند با حجم و قیمت                          |
//+------------------------------------------------------------------+
Strategy Strategy_22_Smart_Money()
{
   Strategy s;
   s.id = 22;
   s.name = "Smart Money Concept";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   IndicatorSignal obv = g_IndicatorState.obv;
   IndicatorSignal ao = g_IndicatorState.awesome;
   
   if(!obv.isValid || !ao.isValid)
      return s;
   
   MqlRates rates[];
   ArraySetAsSeries(rates, true);
   if(CopyRates(InpTradeSymbol, InpMainTF, 0, 10, rates) < 10)
      return s;
   
   if(CopyBuffer(handle_OBV, 0, 0, 10, buffer_OBV) <= 0)
      return s;
   
   //--- محاسبه تغییرات قیمت و حجم
   double priceChange = rates[0].close - rates[5].close;
   double obvChange = buffer_OBV[0] - buffer_OBV[5];
   
   //--- شناسایی کندل‌های با حجم بالا
   double avgVolume = 0;
   for(int i = 1; i < 10; i++)
      avgVolume += (double)rates[i].tick_volume;
   avgVolume /= 9;
   
   double currentVolume = (double)rates[0].tick_volume;
   bool highVolume = (currentVolume > avgVolume * 1.5);
   
   //--- محاسبه اندازه کندل‌ها
   double currentCandleSize = MathAbs(rates[0].close - rates[0].open);
   double avgCandleSize = 0;
   for(int i = 1; i < 6; i++)
      avgCandleSize += MathAbs(rates[i].close - rates[i].open);
   avgCandleSize /= 5;
   
   bool bigCandle = (currentCandleSize > avgCandleSize * 1.5);
   
   //--- شرایط ورود
   // واگرایی صعودی: قیمت پایین + OBV بالا + حجم بالا
   if(priceChange < 0 && obvChange > 0 && highVolume)
   {
      s.signal = SIGNAL_BUY;
      s.score = 96;
      s.confidence = 93;
      s.isActive = true;
      s.description = "Smart Money Accumulation - Bullish divergence";
   }
   // واگرایی نزولی: قیمت بالا + OBV پایین + حجم بالا
   else if(priceChange > 0 && obvChange < 0 && highVolume)
   {
      s.signal = SIGNAL_SELL;
      s.score = 96;
      s.confidence = 93;
      s.isActive = true;
      s.description = "Smart Money Distribution - Bearish divergence";
   }
   // خرید: کندل بزرگ صعودی + حجم بالا + OBV صعودی
   else if(rates[0].close > rates[0].open && bigCandle && highVolume && obv.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 89;
      s.confidence = 86;
      s.isActive = true;
      s.description = "Strong institutional buying detected";
   }
   // فروش: کندل بزرگ نزولی + حجم بالا + OBV نزولی
   else if(rates[0].close < rates[0].open && bigCandle && highVolume && obv.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 89;
      s.confidence = 86;
      s.isActive = true;
      s.description = "Strong institutional selling detected";
   }
   // جریان خرید: OBV و AO هر دو صعودی
   else if(obv.signal == SIGNAL_BUY && ao.signal == SIGNAL_BUY)
   {
      s.signal = SIGNAL_BUY;
      s.score = 81;
      s.confidence = 78;
      s.isActive = true;
      s.description = "Money flow is bullish";
   }
   // جریان فروش: OBV و AO هر دو نزولی
   else if(obv.signal == SIGNAL_SELL && ao.signal == SIGNAL_SELL)
   {
      s.signal = SIGNAL_SELL;
      s.score = 81;
      s.confidence = 78;
      s.isActive = true;
      s.description = "Money flow is bearish";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 23: Time-Based Momentum Scalp                      |
//| توضیح: اسکلپ بر اساس زمان و مومنتوم                             |
//+------------------------------------------------------------------+
Strategy Strategy_23_Time_Momentum()
{
   Strategy s;
   s.id = 23;
   s.name = "Time-Based Momentum";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   MqlDateTime dt;
   TimeCurrent(dt);
   
   IndicatorSignal mom = g_IndicatorState.momentum;
   IndicatorSignal macd = g_IndicatorState.macd;
   IndicatorSignal atr = g_IndicatorState.atr;
   
   if(!mom.isValid || !macd.isValid || !atr.isValid)
      return s;
   
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return s;
   
   double atr_0 = buffer_ATR[0];
   double atr_avg = 0;
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   //--- ساعات پرنوسان طلا (GMT)
   bool asianSession = (dt.hour >= 0 && dt.hour < 8);      // آسیا
   bool londonSession = (dt.hour >= 8 && dt.hour < 16);    // لندن
   bool nySession = (dt.hour >= 13 && dt.hour < 22);       // نیویورک
   bool overlap = (dt.hour >= 13 && dt.hour < 16);         // همپوشانی لندن-نیویورک
   
   //--- ضریب زمانی
   double timeMultiplier = 1.0;
   string sessionName = "Other";
   
   if(overlap)
   {
      timeMultiplier = 1.3;  // بهترین زمان
      sessionName = "London-NY Overlap";
   }
   else if(londonSession)
   {
      timeMultiplier = 1.2;
      sessionName = "London Session";
   }
   else if(nySession)
   {
      timeMultiplier = 1.15;
      sessionName = "NY Session";
   }
   else if(asianSession)
   {
      timeMultiplier = 0.9;
      sessionName = "Asian Session";
   }
   else
   {
      timeMultiplier = 0.8;
      sessionName = "Low Activity";
   }
   
   //--- شرایط ورود
   // خرید: مومنتوم قوی + زمان مناسب + نوسان بالا
   if(mom.signal == SIGNAL_BUY && macd.signal == SIGNAL_BUY && 
      atr_0 > atr_avg * 1.0 && timeMultiplier >= 1.15)
   {
      double baseScore = 87;
      s.signal = SIGNAL_BUY;
      s.score = baseScore * timeMultiplier;
      s.confidence = 84 * timeMultiplier;
      s.isActive = true;
      s.description = "Bullish momentum in " + sessionName;
   }
   // فروش: مومنتوم قوی + زمان مناسب + نوسان بالا
   else if(mom.signal == SIGNAL_SELL && macd.signal == SIGNAL_SELL && 
           atr_0 > atr_avg * 1.0 && timeMultiplier >= 1.15)
   {
      double baseScore = 87;
      s.signal = SIGNAL_SELL;
      s.score = baseScore * timeMultiplier;
      s.confidence = 84 * timeMultiplier;
      s.isActive = true;
      s.description = "Bearish momentum in " + sessionName;
   }
   // سیگنال در زمان ضعیف
   else if(timeMultiplier < 1.0)
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 50;
      s.isActive = true;
      s.description = sessionName + " - Low activity period";
   }
   // مومنتوم متوسط در زمان خوب
   else if((mom.signal == SIGNAL_BUY || macd.signal == SIGNAL_BUY) && timeMultiplier >= 1.2)
   {
      s.signal = SIGNAL_BUY;
      s.score = 75 * timeMultiplier;
      s.confidence = 72 * timeMultiplier;
      s.isActive = true;
      s.description = "Moderate buy in " + sessionName;
   }
   else if((mom.signal == SIGNAL_SELL || macd.signal == SIGNAL_SELL) && timeMultiplier >= 1.2)
   {
      s.signal = SIGNAL_SELL;
      s.score = 75 * timeMultiplier;
      s.confidence = 72 * timeMultiplier;
      s.isActive = true;
      s.description = "Moderate sell in " + sessionName;
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 24: Perfect Entry Filter                           |
//| توضیح: فیلتر نهایی برای ورود کامل                               |
//+------------------------------------------------------------------+
Strategy Strategy_24_Perfect_Entry()
{
   Strategy s;
   s.id = 24;
   s.name = "Perfect Entry Filter";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   //--- بررسی همه شرایط برای ورود کامل
   
   // 1. بررسی اسپرد
   double spread = GetCurrentSpread();
   if(spread > 30) // اسپرد بالا
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 0;
      s.isActive = true;
      s.description = "High spread - Entry rejected";
      return s;
   }
   
   // 2. بررسی نوسان
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) <= 0)
      return s;
   
   double atr_0 = buffer_ATR[0];
   double atr_avg = 0;
   for(int i = 0; i < 10; i++)
      atr_avg += buffer_ATR[i];
   atr_avg /= 10;
   
   if(atr_0 < atr_avg * 0.7)
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 0;
      s.isActive = true;
      s.description = "Low volatility - Entry rejected";
      return s;
   }
   
   // 3. بررسی تعداد اندیکاتورهای موافق
   int buyConfirm = g_IndicatorState.totalBuySignals;
   int sellConfirm = g_IndicatorState.totalSellSignals;
   
   // 4. بررسی قدرت میانگین
   double avgStrength = g_IndicatorState.averageStrength;
   
   // 5. بررسی تایم فریم بالاتر
   double ema_fast_m5[], ema_slow_m5[];
   ArraySetAsSeries(ema_fast_m5, true);
   ArraySetAsSeries(ema_slow_m5, true);
   
   int handle_ema_fast_m5 = iMA(InpTradeSymbol, InpHelperTF, 9, 0, MODE_EMA, PRICE_CLOSE);
   int handle_ema_slow_m5 = iMA(InpTradeSymbol, InpHelperTF, 21, 0, MODE_EMA, PRICE_CLOSE);
   
   bool mtf_aligned = false;
   
   if(handle_ema_fast_m5 != INVALID_HANDLE && handle_ema_slow_m5 != INVALID_HANDLE)
   {
      if(CopyBuffer(handle_ema_fast_m5, 0, 0, 2, ema_fast_m5) > 0 &&
         CopyBuffer(handle_ema_slow_m5, 0, 0, 2, ema_slow_m5) > 0)
      {
         bool m1_up = (buffer_EMA_Fast[0] > buffer_EMA_Slow[0]);
         bool m5_up = (ema_fast_m5[0] > ema_slow_m5[0]);
         
         mtf_aligned = (m1_up == m5_up);
      }
      
      IndicatorRelease(handle_ema_fast_m5);
      IndicatorRelease(handle_ema_slow_m5);
   }
   
   //--- شرایط PERFECT ENTRY
   // خرید کامل: 7+ اندیکاتور موافق + قدرت بالا + اسپرد پایین + نوسان خوب + MTF هم‌راستا
   if(buyConfirm >= 7 && avgStrength > 70 && spread < 20 && 
      atr_0 > atr_avg * 1.1 && mtf_aligned)
   {
      s.signal = SIGNAL_BUY;
      s.score = 100;
      s.confidence = 99;
      s.isActive = true;
      s.description = StringFormat("PERFECT BUY - %d/14 indicators - Spread:%.1f - ATR:%.2f", 
                                   buyConfirm, spread, atr_0);
   }
   // فروش کامل
   else if(sellConfirm >= 7 && avgStrength > 70 && spread < 20 && 
           atr_0 > atr_avg * 1.1 && mtf_aligned)
   {
      s.signal = SIGNAL_SELL;
      s.score = 100;
      s.confidence = 99;
      s.isActive = true;
      s.description = StringFormat("PERFECT SELL - %d/14 indicators - Spread:%.1f - ATR:%.2f", 
                                   sellConfirm, spread, atr_0);
   }
   // خرید خوب: 6+ موافق
   else if(buyConfirm >= 6 && avgStrength > 65 && spread < 25 && atr_0 > atr_avg * 0.9)
   {
      s.signal = SIGNAL_BUY;
      s.score = 92;
      s.confidence = 89;
      s.isActive = true;
      s.description = StringFormat("GOOD BUY - %d/14 indicators", buyConfirm);
   }
   // فروش خوب
   else if(sellConfirm >= 6 && avgStrength > 65 && spread < 25 && atr_0 > atr_avg * 0.9)
   {
      s.signal = SIGNAL_SELL;
      s.score = 92;
      s.confidence = 89;
      s.isActive = true;
      s.description = StringFormat("GOOD SELL - %d/14 indicators", sellConfirm);
   }
   // شرایط متوسط: 5 موافق
   else if(buyConfirm >= 5 && avgStrength > 60 && spread < 30)
   {
      s.signal = SIGNAL_BUY;
      s.score = 80;
      s.confidence = 77;
      s.isActive = true;
      s.description = StringFormat("MODERATE BUY - %d/14 indicators", buyConfirm);
   }
   else if(sellConfirm >= 5 && avgStrength > 60 && spread < 30)
   {
      s.signal = SIGNAL_SELL;
      s.score = 80;
      s.confidence = 77;
      s.isActive = true;
      s.description = StringFormat("MODERATE SELL - %d/14 indicators", sellConfirm);
   }
   else
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = 50;
      s.isActive = true;
      s.description = "Conditions not met for entry";
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}

//+------------------------------------------------------------------+
//| 🎯 استراتژی 25: MASTER STRATEGY (استراتژی نهایی)              |
//| توضیح: ترکیب وزن‌دار همه استراتژی‌ها                            |
//+------------------------------------------------------------------+
Strategy Strategy_25_Master()
{
   Strategy s;
   s.id = 25;
   s.name = "MASTER STRATEGY";
   s.signal = SIGNAL_NONE;
   s.score = 0;
   s.confidence = 0;
   s.isActive = false;
   s.description = "";
   s.lastSignalTime = 0;
   
   //--- جمع‌آوری نتایج 24 استراتژی قبلی
   double totalBuyScore = 0;
   double totalSellScore = 0;
   int buyCount = 0;
   int sellCount = 0;
   int neutralCount = 0;
   
   double totalConfidence = 0;
   int activeStrategies = 0;
   
   //--- وزن‌های استراتژی‌ها (مهم‌ترین‌ها وزن بیشتر)
   double weights[25];
   
   // استراتژی‌های معمولی - وزن 1.0
   for(int i = 0; i < 25; i++)
      weights[i] = 1.0;
   
   // استراتژی‌های مهم - وزن بیشتر
   weights[10] = 1.5;  // Triple Confirmation
   weights[13] = 1.4;  // Extreme Reversal
   weights[20] = 1.6;  // Golden Scalp
   weights[21] = 1.5;  // Smart Money
   weights[22] = 1.3;  // Time Momentum
   weights[23] = 1.7;  // Perfect Entry
   
   //--- پردازش هر استراتژی
   for(int i = 0; i < 24; i++) // 24 استراتژی اول (خود Master را حساب نمی‌کنیم)
   {
      if(g_Strategies[i].isActive)
      {
         activeStrategies++;
         totalConfidence += g_Strategies[i].confidence;
         
         if(g_Strategies[i].signal == SIGNAL_BUY)
         {
            buyCount++;
            totalBuyScore += g_Strategies[i].score * weights[i];
         }
         else if(g_Strategies[i].signal == SIGNAL_SELL)
         {
            sellCount++;
            totalSellScore += g_Strategies[i].score * weights[i];
         }
         else
         {
            neutralCount++;
         }
      }
   }
   
   //--- محاسبه میانگین‌ها
   double avgConfidence = (activeStrategies > 0) ? (totalConfidence / activeStrategies) : 0;
   
   //--- تصمیم‌گیری نهایی
   double buyPercentage = (activeStrategies > 0) ? ((double)buyCount / activeStrategies * 100) : 0;
   double sellPercentage = (activeStrategies > 0) ? ((double)sellCount / activeStrategies * 100) : 0;
   
   //--- شرایط MASTER
   // خرید قطعی: 70%+ استراتژی‌ها خرید + امتیاز بالا
   if(buyPercentage >= 70 && totalBuyScore > totalSellScore * 2)
   {
      s.signal = SIGNAL_BUY;
      s.score = MathMin(totalBuyScore / buyCount, 100);
      s.confidence = MathMin(avgConfidence * 1.1, 100);
      s.isActive = true;
      s.description = StringFormat("MASTER BUY - %.1f%% strategies agree (%d/%d)", 
                                   buyPercentage, buyCount, activeStrategies);
   }
   // فروش قطعی: 70%+ استراتژی‌ها فروش + امتیاز بالا
   else if(sellPercentage >= 70 && totalSellScore > totalBuyScore * 2)
   {
      s.signal = SIGNAL_SELL;
      s.score = MathMin(totalSellScore / sellCount, 100);
      s.confidence = MathMin(avgConfidence * 1.1, 100);
      s.isActive = true;
      s.description = StringFormat("MASTER SELL - %.1f%% strategies agree (%d/%d)", 
                                   sellPercentage, sellCount, activeStrategies);
   }
   // خرید قوی: 60%+ موافق
   else if(buyPercentage >= 60 && totalBuyScore > totalSellScore * 1.5)
   {
      s.signal = SIGNAL_BUY;
      s.score = MathMin(totalBuyScore / buyCount, 100);
      s.confidence = avgConfidence;
      s.isActive = true;
      s.description = StringFormat("MASTER BUY - %.1f%% strategies (%d/%d)", 
                                   buyPercentage, buyCount, activeStrategies);
   }
   // فروش قوی: 60%+ موافق
   else if(sellPercentage >= 60 && totalSellScore > totalBuyScore * 1.5)
   {
      s.signal = SIGNAL_SELL;
      s.score = MathMin(totalSellScore / sellCount, 100);
      s.confidence = avgConfidence;
      s.isActive = true;
      s.description = StringFormat("MASTER SELL - %.1f%% strategies (%d/%d)", 
                                   sellPercentage, sellCount, activeStrategies);
   }
   // خرید متوسط: 50%+ موافق
   else if(buyPercentage >= 50 && totalBuyScore > totalSellScore)
   {
      s.signal = SIGNAL_BUY;
      s.score = MathMin((totalBuyScore / buyCount) * 0.9, 100);
      s.confidence = avgConfidence * 0.9;
      s.isActive = true;
      s.description = StringFormat("MASTER BUY - %.1f%% strategies (%d/%d)", 
                                   buyPercentage, buyCount, activeStrategies);
   }
   // فروش متوسط: 50%+ موافق
   else if(sellPercentage >= 50 && totalSellScore > totalBuyScore)
   {
      s.signal = SIGNAL_SELL;
      s.score = MathMin((totalSellScore / sellCount) * 0.9, 100);
      s.confidence = avgConfidence * 0.9;
      s.isActive = true;
      s.description = StringFormat("MASTER SELL - %.1f%% strategies (%d/%d)", 
                                   sellPercentage, sellCount, activeStrategies);
   }
   // عدم اتفاق نظر
   else
   {
      s.signal = SIGNAL_NEUTRAL;
      s.score = 0;
      s.confidence = avgConfidence * 0.7;
      s.isActive = true;
      s.description = StringFormat("No consensus - Buy:%.1f%% Sell:%.1f%% Neutral:%.1f%%", 
                                   buyPercentage, sellPercentage, 
                                   (double)neutralCount/activeStrategies*100);
   }
   
   s.lastSignalTime = TimeCurrent();
   return s;
}


//+------------------------------------------------------------------+
//| 🎯 اجرای همه 25 استراتژی                                        |
//+------------------------------------------------------------------+
bool ExecuteAllStrategies()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🎯 Executing 25 Scalping Strategies...");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   //--- اجرای استراتژی‌ها
   g_Strategies[0]  = Strategy_01_RSI_Extreme();
   g_Strategies[1]  = Strategy_02_BB_Bounce();
   g_Strategies[2]  = Strategy_03_Stoch_Cross();
   g_Strategies[3]  = Strategy_04_MACD_Zero();
   g_Strategies[4]  = Strategy_05_SAR_Reversal();
   g_Strategies[5]  = Strategy_06_RSI_Stoch();
   g_Strategies[6]  = Strategy_07_EMA_MACD();
   g_Strategies[7]  = Strategy_08_BB_CCI();
   g_Strategies[8]  = Strategy_09_ADX_SAR();
   g_Strategies[9]  = Strategy_10_WPR_DEM();
   g_Strategies[10] = Strategy_11_Triple_Confirm();
   g_Strategies[11] = Strategy_12_Volume_Price();
   g_Strategies[12] = Strategy_13_Trend_Osc();
   g_Strategies[13] = Strategy_14_Extreme_Reversal();
   g_Strategies[14] = Strategy_15_Breakout_Mom();
   g_Strategies[15] = Strategy_16_Price_Action();
   g_Strategies[16] = Strategy_17_SR_Bounce();
   g_Strategies[17] = Strategy_18_MA_Speed();
   g_Strategies[18] = Strategy_19_Volatility_Break();
   g_Strategies[19] = Strategy_20_MTF_Confirm();
   g_Strategies[20] = Strategy_21_Golden_Scalp();
   g_Strategies[21] = Strategy_22_Smart_Money();
   g_Strategies[22] = Strategy_23_Time_Momentum();
   g_Strategies[23] = Strategy_24_Perfect_Entry();
   g_Strategies[24] = Strategy_25_Master();
   
   //--- محاسبه نتایج کلی
   CalculateStrategyResults();
   
   //--- نمایش خلاصه
   PrintStrategyResults();
   
   return true;
}

//+------------------------------------------------------------------+
//| 📊 محاسبه نتایج کلی استراتژی‌ها                                 |
//+------------------------------------------------------------------+
void CalculateStrategyResults()
{
   g_StrategyResult.totalStrategies = 25;
   g_StrategyResult.activeStrategies = 0;
   g_StrategyResult.buySignals = 0;
   g_StrategyResult.sellSignals = 0;
   g_StrategyResult.neutralSignals = 0;
   g_StrategyResult.buyScore = 0;
   g_StrategyResult.sellScore = 0;
   g_StrategyResult.averageConfidence = 0;
   
   double totalConfidence = 0;
   
   for(int i = 0; i < 25; i++)
   {
      if(g_Strategies[i].isActive)
      {
         g_StrategyResult.activeStrategies++;
         totalConfidence += g_Strategies[i].confidence;
         
         if(g_Strategies[i].signal == SIGNAL_BUY)
         {
            g_StrategyResult.buySignals++;
            g_StrategyResult.buyScore += g_Strategies[i].score;
         }
         else if(g_Strategies[i].signal == SIGNAL_SELL)
         {
            g_StrategyResult.sellSignals++;
            g_StrategyResult.sellScore += g_Strategies[i].score;
         }
         else
         {
            g_StrategyResult.neutralSignals++;
         }
      }
   }
   
   if(g_StrategyResult.activeStrategies > 0)
      g_StrategyResult.averageConfidence = totalConfidence / g_StrategyResult.activeStrategies;
   
   //--- تعیین سیگنال و امتیاز نهایی
   if(g_StrategyResult.buySignals > g_StrategyResult.sellSignals && 
      g_StrategyResult.buyScore > g_StrategyResult.sellScore)
   {
      g_StrategyResult.finalSignal = SIGNAL_BUY;
      g_StrategyResult.finalScore = g_StrategyResult.buyScore / MathMax(g_StrategyResult.buySignals, 1);
   }
   else if(g_StrategyResult.sellSignals > g_StrategyResult.buySignals && 
           g_StrategyResult.sellScore > g_StrategyResult.buyScore)
   {
      g_StrategyResult.finalSignal = SIGNAL_SELL;
      g_StrategyResult.finalScore = g_StrategyResult.sellScore / MathMax(g_StrategyResult.sellSignals, 1);
   }
   else
   {
      g_StrategyResult.finalSignal = SIGNAL_NEUTRAL;
      g_StrategyResult.finalScore = 0;
   }
   
   //--- تعیین توصیه
   double buyPercent = (double)g_StrategyResult.buySignals / g_StrategyResult.activeStrategies * 100;
   double sellPercent = (double)g_StrategyResult.sellSignals / g_StrategyResult.activeStrategies * 100;
   
   if(buyPercent >= 70 && g_StrategyResult.finalScore >= 90)
      g_StrategyResult.recommendation = "STRONG BUY ⭐⭐⭐";
   else if(buyPercent >= 60 && g_StrategyResult.finalScore >= 80)
      g_StrategyResult.recommendation = "BUY ⭐⭐";
   else if(buyPercent >= 50 && g_StrategyResult.finalScore >= 70)
      g_StrategyResult.recommendation = "MODERATE BUY ⭐";
   else if(sellPercent >= 70 && g_StrategyResult.finalScore >= 90)
      g_StrategyResult.recommendation = "STRONG SELL ⭐⭐⭐";
   else if(sellPercent >= 60 && g_StrategyResult.finalScore >= 80)
      g_StrategyResult.recommendation = "SELL ⭐⭐";
   else if(sellPercent >= 50 && g_StrategyResult.finalScore >= 70)
      g_StrategyResult.recommendation = "MODERATE SELL ⭐";
   else
      g_StrategyResult.recommendation = "WAIT / NO SIGNAL";
}

//+------------------------------------------------------------------+
//| 📋 نمایش نتایج استراتژی‌ها                                      |
//+------------------------------------------------------------------+
void PrintStrategyResults()
{
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║           🎯 STRATEGY ANALYSIS RESULTS                ║");
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Active Strategies:    %d/25                          ║", 
                      g_StrategyResult.activeStrategies));
   Print(StringFormat("║ Buy Signals:          %d (%.1f%%)                   ║", 
                      g_StrategyResult.buySignals,
                      (double)g_StrategyResult.buySignals/g_StrategyResult.activeStrategies*100));
   Print(StringFormat("║ Sell Signals:         %d (%.1f%%)                   ║", 
                      g_StrategyResult.sellSignals,
                      (double)g_StrategyResult.sellSignals/g_StrategyResult.activeStrategies*100));
   Print(StringFormat("║ Neutral Signals:      %d (%.1f%%)                   ║", 
                      g_StrategyResult.neutralSignals,
                      (double)g_StrategyResult.neutralSignals/g_StrategyResult.activeStrategies*100));
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Average Confidence:   %.2f%%                        ║", 
                      g_StrategyResult.averageConfidence));
   Print(StringFormat("║ Final Score:          %.2f/100                      ║", 
                      g_StrategyResult.finalScore));
   Print(StringFormat("║ Final Signal:         %-30s ║", 
                      EnumToString(g_StrategyResult.finalSignal)));
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ 🎯 RECOMMENDATION:    %-27s ║", 
                      g_StrategyResult.recommendation));
   Print("╚════════════════════════════════════════════════════════╝");
}


//+------------------------------------------------------------------+
//| 🗳️ بخش سیستم رای‌گیری چند لایه - MULTI-LAYER VOTING SYSTEM    |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 📋 ساختار رای اندیکاتور                                         |
//+------------------------------------------------------------------+
struct IndicatorVote
{
   string            name;              // نام اندیکاتور
   ENUM_SIGNAL_TYPE  vote;              // رای
   double            weight;            // وزن (1.0 - 2.0)
   double            strength;          // قدرت (0-100)
   double            confidence;        // اطمینان (0-100)
   int               priority;          // اولویت (1-5)
   bool              isValid;           // معتبر
};

//+------------------------------------------------------------------+
//| 📋 ساختار رای استراتژی                                          |
//+------------------------------------------------------------------+
struct StrategyVote
{
   string            name;              // نام استراتژی
   ENUM_SIGNAL_TYPE  vote;              // رای
   double            weight;            // وزن (1.0 - 2.0)
   double            score;             // امتیاز (0-100)
   double            confidence;        // اطمینان (0-100)
   int               priority;          // اولویت (1-5)
   bool              isValid;           // معتبر
};

//+------------------------------------------------------------------+
//| 🎲 ساختار نتیجه رای‌گیری                                        |
//+------------------------------------------------------------------+
struct VotingResult
{
   // لایه 1: رای‌گیری اندیکاتورها
   int               indicatorBuyVotes;
   int               indicatorSellVotes;
   int               indicatorNeutralVotes;
   double            indicatorBuyScore;
   double            indicatorSellScore;
   double            indicatorConfidence;
   
   // لایه 2: رای‌گیری استراتژی‌ها
   int               strategyBuyVotes;
   int               strategySellVotes;
   int               strategyNeutralVotes;
   double            strategyBuyScore;
   double            strategySellScore;
   double            strategyConfidence;
   
   // لایه 3: نتیجه نهایی
   ENUM_SIGNAL_TYPE  finalDecision;
   double            finalScore;
   double            finalConfidence;
   string            recommendation;
   
   // آمار
   int               totalVotes;
   double            consensusLevel;     // سطح اتفاق نظر (0-100%)
   bool              isHighQuality;      // کیفیت بالا
   string            decisionReason;     // دلیل تصمیم
};

//--- متغیرهای سراسری
VotingResult g_VotingResult;


//+------------------------------------------------------------------+
//| 🗳️ لایه 1: رای‌گیری از 15 اندیکاتور با وزن‌دهی                 |
//+------------------------------------------------------------------+
void VoteLayer1_Indicators()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🗳️ LAYER 1: Indicator Voting System");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   IndicatorVote votes[15];
   
   //--- 1. EMA (وزن: 1.3 - اولویت: 4)
   votes[0].name = "EMA Cross";
   votes[0].vote = g_IndicatorState.ema_fast.signal;
   votes[0].weight = 1.3;
   votes[0].strength = g_IndicatorState.ema_fast.strength;
   votes[0].confidence = (g_IndicatorState.ema_fast.strength > 70) ? 85 : 70;
   votes[0].priority = 4;
   votes[0].isValid = g_IndicatorState.ema_fast.isValid;
   
   //--- 2. RSI (وزن: 1.5 - اولویت: 5) - مهم برای اشباع
   votes[1].name = "RSI";
   votes[1].vote = g_IndicatorState.rsi.signal;
   votes[1].weight = 1.5;
   votes[1].strength = g_IndicatorState.rsi.strength;
   votes[1].confidence = (g_IndicatorState.rsi.strength > 80) ? 90 : 75;
   votes[1].priority = 5;
   votes[1].isValid = g_IndicatorState.rsi.isValid;
   
   //--- 3. Stochastic (وزن: 1.4 - اولویت: 5)
   votes[2].name = "Stochastic";
   votes[2].vote = g_IndicatorState.stochastic.signal;
   votes[2].weight = 1.4;
   votes[2].strength = g_IndicatorState.stochastic.strength;
   votes[2].confidence = (g_IndicatorState.stochastic.strength > 85) ? 88 : 72;
   votes[2].priority = 5;
   votes[2].isValid = g_IndicatorState.stochastic.isValid;
   
   //--- 4. MACD (وزن: 1.4 - اولویت: 4)
   votes[3].name = "MACD";
   votes[3].vote = g_IndicatorState.macd.signal;
   votes[3].weight = 1.4;
   votes[3].strength = g_IndicatorState.macd.strength;
   votes[3].confidence = (g_IndicatorState.macd.strength > 75) ? 86 : 73;
   votes[3].priority = 4;
   votes[3].isValid = g_IndicatorState.macd.isValid;
   
   //--- 5. ATR (وزن: 1.2 - اولویت: 3) - برای نوسان
   votes[4].name = "ATR";
   votes[4].vote = g_IndicatorState.atr.signal;
   votes[4].weight = 1.2;
   votes[4].strength = g_IndicatorState.atr.strength;
   votes[4].confidence = 70;
   votes[4].priority = 3;
   votes[4].isValid = g_IndicatorState.atr.isValid;
   
   //--- 6. Bollinger Bands (وزن: 1.6 - اولویت: 5) - خیلی مهم برای اسکلپ
   votes[5].name = "Bollinger Bands";
   votes[5].vote = g_IndicatorState.bollinger.signal;
   votes[5].weight = 1.6;
   votes[5].strength = g_IndicatorState.bollinger.strength;
   votes[5].confidence = (g_IndicatorState.bollinger.strength > 85) ? 92 : 78;
   votes[5].priority = 5;
   votes[5].isValid = g_IndicatorState.bollinger.isValid;
   
   //--- 7. CCI (وزن: 1.3 - اولویت: 4)
   votes[6].name = "CCI";
   votes[6].vote = g_IndicatorState.cci.signal;
   votes[6].weight = 1.3;
   votes[6].strength = g_IndicatorState.cci.strength;
   votes[6].confidence = (g_IndicatorState.cci.strength > 80) ? 87 : 74;
   votes[6].priority = 4;
   votes[6].isValid = g_IndicatorState.cci.isValid;
   
   //--- 8. ADX (وزن: 1.5 - اولویت: 5) - قدرت روند
   votes[7].name = "ADX";
   votes[7].vote = g_IndicatorState.adx.signal;
   votes[7].weight = 1.5;
   votes[7].strength = g_IndicatorState.adx.strength;
   votes[7].confidence = (g_IndicatorState.adx.strength > 85) ? 91 : 76;
   votes[7].priority = 5;
   votes[7].isValid = g_IndicatorState.adx.isValid;
   
   //--- 9. Williams %R (وزن: 1.3 - اولویت: 4)
   votes[8].name = "Williams %R";
   votes[8].vote = g_IndicatorState.williams.signal;
   votes[8].weight = 1.3;
   votes[8].strength = g_IndicatorState.williams.strength;
   votes[8].confidence = (g_IndicatorState.williams.strength > 80) ? 85 : 71;
   votes[8].priority = 4;
   votes[8].isValid = g_IndicatorState.williams.isValid;
   
   //--- 10. Momentum (وزن: 1.2 - اولویت: 3)
   votes[9].name = "Momentum";
   votes[9].vote = g_IndicatorState.momentum.signal;
   votes[9].weight = 1.2;
   votes[9].strength = g_IndicatorState.momentum.strength;
   votes[9].confidence = (g_IndicatorState.momentum.strength > 75) ? 82 : 68;
   votes[9].priority = 3;
   votes[9].isValid = g_IndicatorState.momentum.isValid;
   
   //--- 11. Parabolic SAR (وزن: 1.7 - اولویت: 5) - عالی برای اسکلپ
   votes[10].name = "Parabolic SAR";
   votes[10].vote = g_IndicatorState.sar.signal;
   votes[10].weight = 1.7;
   votes[10].strength = g_IndicatorState.sar.strength;
   votes[10].confidence = (g_IndicatorState.sar.strength > 90) ? 94 : 80;
   votes[10].priority = 5;
   votes[10].isValid = g_IndicatorState.sar.isValid;
   
   //--- 12. OBV (وزن: 1.4 - اولویت: 4)
   votes[11].name = "OBV";
   votes[11].vote = g_IndicatorState.obv.signal;
   votes[11].weight = 1.4;
   votes[11].strength = g_IndicatorState.obv.strength;
   votes[11].confidence = (g_IndicatorState.obv.strength > 85) ? 89 : 75;
   votes[11].priority = 4;
   votes[11].isValid = g_IndicatorState.obv.isValid;
   
   //--- 13. Awesome Oscillator (وزن: 1.3 - اولویت: 3)
   votes[12].name = "Awesome Oscillator";
   votes[12].vote = g_IndicatorState.awesome.signal;
   votes[12].weight = 1.3;
   votes[12].strength = g_IndicatorState.awesome.strength;
   votes[12].confidence = (g_IndicatorState.awesome.strength > 80) ? 86 : 72;
   votes[12].priority = 3;
   votes[12].isValid = g_IndicatorState.awesome.isValid;
   
   //--- 14. DeMarker (وزن: 1.4 - اولویت: 4)
   votes[13].name = "DeMarker";
   votes[13].vote = g_IndicatorState.demarker.signal;
   votes[13].weight = 1.4;
   votes[13].strength = g_IndicatorState.demarker.strength;
   votes[13].confidence = (g_IndicatorState.demarker.strength > 85) ? 88 : 74;
   votes[13].priority = 4;
   votes[13].isValid = g_IndicatorState.demarker.isValid;
   
   //--- محاسبه نتایج
   g_VotingResult.indicatorBuyVotes = 0;
   g_VotingResult.indicatorSellVotes = 0;
   g_VotingResult.indicatorNeutralVotes = 0;
   g_VotingResult.indicatorBuyScore = 0;
   g_VotingResult.indicatorSellScore = 0;
   
   double totalConfidence = 0;
   int validVotes = 0;
   
   for(int i = 0; i < 14; i++)
   {
      if(votes[i].isValid)
      {
         validVotes++;
         totalConfidence += votes[i].confidence;
         
         if(votes[i].vote == SIGNAL_BUY)
         {
            g_VotingResult.indicatorBuyVotes++;
            g_VotingResult.indicatorBuyScore += votes[i].strength * votes[i].weight;
            
            Print(StringFormat("   ✅ %s: BUY (Weight:%.1f Score:%.1f Conf:%.1f%%)", 
                              votes[i].name, votes[i].weight, votes[i].strength, votes[i].confidence));
         }
         else if(votes[i].vote == SIGNAL_SELL)
         {
            g_VotingResult.indicatorSellVotes++;
            g_VotingResult.indicatorSellScore += votes[i].strength * votes[i].weight;
            
            Print(StringFormat("   ❌ %s: SELL (Weight:%.1f Score:%.1f Conf:%.1f%%)", 
                              votes[i].name, votes[i].weight, votes[i].strength, votes[i].confidence));
         }
         else
         {
            g_VotingResult.indicatorNeutralVotes++;
            
            Print(StringFormat("   ⚪ %s: NEUTRAL (Weight:%.1f)", 
                              votes[i].name, votes[i].weight));
         }
      }
   }
   
   if(validVotes > 0)
      g_VotingResult.indicatorConfidence = totalConfidence / validVotes;
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print(StringFormat("📊 LAYER 1 RESULT: BUY:%d SELL:%d NEUTRAL:%d", 
                     g_VotingResult.indicatorBuyVotes, 
                     g_VotingResult.indicatorSellVotes, 
                     g_VotingResult.indicatorNeutralVotes));
   Print(StringFormat("💯 Scores: BUY:%.2f SELL:%.2f | Confidence:%.2f%%", 
                     g_VotingResult.indicatorBuyScore, 
                     g_VotingResult.indicatorSellScore,
                     g_VotingResult.indicatorConfidence));
}


//+------------------------------------------------------------------+
//| 🗳️ لایه 2: رای‌گیری از 25 استراتژی با وزن‌دهی                  |
//+------------------------------------------------------------------+
void VoteLayer2_Strategies()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🗳️ LAYER 2: Strategy Voting System");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   StrategyVote votes[25];
   
   //--- تنظیم وزن‌ها و اولویت‌های استراتژی‌ها
   for(int i = 0; i < 25; i++)
   {
      votes[i].name = g_Strategies[i].name;
      votes[i].vote = g_Strategies[i].signal;
      votes[i].score = g_Strategies[i].score;
      votes[i].confidence = g_Strategies[i].confidence;
      votes[i].isValid = g_Strategies[i].isActive;
      
      //--- تعیین وزن بر اساس شماره استراتژی
      if(i < 5) // استراتژی‌های تک اندیکاتوره
      {
         votes[i].weight = 1.0;
         votes[i].priority = 2;
      }
      else if(i < 10) // دو اندیکاتوره
      {
         votes[i].weight = 1.2;
         votes[i].priority = 3;
      }
      else if(i < 15) // سه اندیکاتوره
      {
         votes[i].weight = 1.4;
         votes[i].priority = 4;
      }
      else if(i < 20) // پرایس اکشن و پیشرفته
      {
         votes[i].weight = 1.3;
         votes[i].priority = 4;
      }
      else // استراتژی‌های امضایی (21-25)
      {
         votes[i].weight = 1.8;
         votes[i].priority = 5;
      }
   }
   
   //--- وزن‌های خاص برای استراتژی‌های مهم
   votes[10].weight = 1.6;  // Triple Confirmation
   votes[13].weight = 1.5;  // Extreme Reversal
   votes[20].weight = 2.0;  // Golden Scalp - مهم‌ترین
   votes[21].weight = 1.8;  // Smart Money
   votes[22].weight = 1.6;  // Time Momentum
   votes[23].weight = 1.9;  // Perfect Entry
   votes[24].weight = 2.0;  // Master Strategy - مهم‌ترین
   
   //--- محاسبه نتایج
   g_VotingResult.strategyBuyVotes = 0;
   g_VotingResult.strategySellVotes = 0;
   g_VotingResult.strategyNeutralVotes = 0;
   g_VotingResult.strategyBuyScore = 0;
   g_VotingResult.strategySellScore = 0;
   
   double totalConfidence = 0;
   int validVotes = 0;
   
   for(int i = 0; i < 25; i++)
   {
      if(votes[i].isValid)
      {
         validVotes++;
         totalConfidence += votes[i].confidence;
         
         if(votes[i].vote == SIGNAL_BUY)
         {
            g_VotingResult.strategyBuyVotes++;
            g_VotingResult.strategyBuyScore += votes[i].score * votes[i].weight;
            
            if(votes[i].priority >= 4) // فقط مهم‌ترین‌ها رو نمایش بده
            {
               Print(StringFormat("   ✅ [%d] %s: BUY (W:%.1f S:%.1f C:%.1f%%)", 
                                 i+1, votes[i].name, votes[i].weight, 
                                 votes[i].score, votes[i].confidence));
            }
         }
         else if(votes[i].vote == SIGNAL_SELL)
         {
            g_VotingResult.strategySellVotes++;
            g_VotingResult.strategySellScore += votes[i].score * votes[i].weight;
            
            if(votes[i].priority >= 4)
            {
               Print(StringFormat("   ❌ [%d] %s: SELL (W:%.1f S:%.1f C:%.1f%%)", 
                                 i+1, votes[i].name, votes[i].weight, 
                                 votes[i].score, votes[i].confidence));
            }
         }
         else
         {
            g_VotingResult.strategyNeutralVotes++;
            
            if(votes[i].priority >= 5) // فقط خیلی مهم‌ترین‌ها
            {
               Print(StringFormat("   ⚪ [%d] %s: NEUTRAL", i+1, votes[i].name));
            }
         }
      }
   }
   
   if(validVotes > 0)
      g_VotingResult.strategyConfidence = totalConfidence / validVotes;
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print(StringFormat("📊 LAYER 2 RESULT: BUY:%d SELL:%d NEUTRAL:%d", 
                     g_VotingResult.strategyBuyVotes, 
                     g_VotingResult.strategySellVotes, 
                     g_VotingResult.strategyNeutralVotes));
   Print(StringFormat("💯 Scores: BUY:%.2f SELL:%.2f | Confidence:%.2f%%", 
                     g_VotingResult.strategyBuyScore, 
                     g_VotingResult.strategySellScore,
                     g_VotingResult.strategyConfidence));
}


//+------------------------------------------------------------------+
//| 🛡️ لایه 3: فیلترهای امنیتی چند لایه                            |
//+------------------------------------------------------------------+
bool SecurityFilters_Layer3()
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("🛡️ LAYER 3: Security Filters");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   bool allFiltersPassed = true;
   
   //--- فیلتر 1: اسپرد
   double spread = GetCurrentSpread();
   if(spread > 35)
   {
      Print("   ❌ FILTER 1 FAILED: Spread too high (", DoubleToString(spread, 1), " > 35)");
      allFiltersPassed = false;
   }
   else
   {
      Print("   ✅ FILTER 1 PASSED: Spread OK (", DoubleToString(spread, 1), ")");
   }
   
   //--- فیلتر 2: نوسان (ATR)
   if(CopyBuffer(handle_ATR, 0, 0, 10, buffer_ATR) > 0)
   {
      double atr_0 = buffer_ATR[0];
      double atr_avg = 0;
      for(int i = 0; i < 10; i++)
         atr_avg += buffer_ATR[i];
      atr_avg /= 10;
      
      if(atr_0 < atr_avg * 0.65)
      {
         Print("   ❌ FILTER 2 FAILED: Volatility too low (ATR:", DoubleToString(atr_0, 2), 
               " < Avg:", DoubleToString(atr_avg * 0.65, 2), ")");
         allFiltersPassed = false;
      }
      else
      {
         Print("   ✅ FILTER 2 PASSED: Volatility OK (ATR:", DoubleToString(atr_0, 2), ")");
      }
   }
   
   //--- فیلتر 3: حداقل اتفاق نظر اندیکاتورها
   int totalIndicatorVotes = g_VotingResult.indicatorBuyVotes + 
                             g_VotingResult.indicatorSellVotes + 
                             g_VotingResult.indicatorNeutralVotes;
   
   int minIndicatorConsensus = 5; // حداقل 5 اندیکاتور باید موافق باشند
   
   if(g_VotingResult.indicatorBuyVotes < minIndicatorConsensus && 
      g_VotingResult.indicatorSellVotes < minIndicatorConsensus)
   {
      Print("   ❌ FILTER 3 FAILED: Insufficient indicator consensus (Min:", 
            minIndicatorConsensus, " Buy:", g_VotingResult.indicatorBuyVotes, 
            " Sell:", g_VotingResult.indicatorSellVotes, ")");
      allFiltersPassed = false;
   }
   else
   {
      Print("   ✅ FILTER 3 PASSED: Indicator consensus OK");
   }
   
   //--- فیلتر 4: حداقل اتفاق نظر استراتژی‌ها
   int minStrategyConsensus = 8; // حداقل 8 استراتژی باید موافق باشند
   
   if(g_VotingResult.strategyBuyVotes < minStrategyConsensus && 
      g_VotingResult.strategySellVotes < minStrategyConsensus)
   {
      Print("   ❌ FILTER 4 FAILED: Insufficient strategy consensus (Min:", 
            minStrategyConsensus, " Buy:", g_VotingResult.strategyBuyVotes, 
            " Sell:", g_VotingResult.strategySellVotes, ")");
      allFiltersPassed = false;
   }
   else
   {
      Print("   ✅ FILTER 4 PASSED: Strategy consensus OK");
   }
   
   //--- فیلتر 5: اطمینان کلی
   double avgConfidence = (g_VotingResult.indicatorConfidence + 
                          g_VotingResult.strategyConfidence) / 2.0;
   
   if(avgConfidence < 65)
   {
      Print("   ❌ FILTER 5 FAILED: Average confidence too low (", 
            DoubleToString(avgConfidence, 2), "% < 65%)");
      allFiltersPassed = false;
   }
   else
   {
      Print("   ✅ FILTER 5 PASSED: Confidence OK (", DoubleToString(avgConfidence, 2), "%)");
   }
   
   //--- فیلتر 6: تضاد بین لایه‌ها
   bool layer1Buy = (g_VotingResult.indicatorBuyVotes > g_VotingResult.indicatorSellVotes);
   bool layer2Buy = (g_VotingResult.strategyBuyVotes > g_VotingResult.strategySellVotes);
   
   if((layer1Buy && !layer2Buy) || (!layer1Buy && layer2Buy))
   {
      // تضاد وجود دارد - بررسی شدت تضاد
      double layer1Ratio = (double)MathAbs(g_VotingResult.indicatorBuyVotes - 
                                          g_VotingResult.indicatorSellVotes) / 
                          (g_VotingResult.indicatorBuyVotes + g_VotingResult.indicatorSellVotes);
      
      double layer2Ratio = (double)MathAbs(g_VotingResult.strategyBuyVotes - 
                                          g_VotingResult.strategySellVotes) / 
                          (g_VotingResult.strategyBuyVotes + g_VotingResult.strategySellVotes);
      
      if(MathAbs(layer1Ratio - layer2Ratio) > 0.3)
      {
         Print("   ⚠️ FILTER 6 WARNING: Layer conflict detected (Ratio diff:", 
               DoubleToString(MathAbs(layer1Ratio - layer2Ratio), 2), ")");
         // فقط هشدار - رد نمی‌کنیم
      }
      else
      {
         Print("   ✅ FILTER 6 PASSED: Minor layer conflict (acceptable)");
      }
   }
   else
   {
      Print("   ✅ FILTER 6 PASSED: Layers aligned");
   }
   
   //--- فیلتر 7: امتیاز نهایی
   double maxScore = MathMax(g_VotingResult.indicatorBuyScore + g_VotingResult.strategyBuyScore,
                             g_VotingResult.indicatorSellScore + g_VotingResult.strategySellScore);
   
   if(maxScore < 800) // حداقل امتیاز
   {
      Print("   ❌ FILTER 7 FAILED: Score too low (", DoubleToString(maxScore, 2), " < 800)");
      allFiltersPassed = false;
   }
   else
   {
      Print("   ✅ FILTER 7 PASSED: Score sufficient (", DoubleToString(maxScore, 2), ")");
   }
   
   //--- فیلتر 8: بررسی Master Strategy
   if(g_Strategies[24].isActive && g_Strategies[24].signal == SIGNAL_NEUTRAL)
   {
      Print("   ⚠️ FILTER 8 WARNING: Master Strategy says NEUTRAL");
      // اگر Master می‌گه neutral، باید خیلی محتاط بود
      if(!allFiltersPassed)
      {
         Print("   ❌ FILTER 8 FAILED: Master neutral + other filters failed");
         return false;
      }
   }
   else
   {
      Print("   ✅ FILTER 8 PASSED: Master Strategy active");
   }
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   if(allFiltersPassed)
   {
      Print("✅ ALL SECURITY FILTERS PASSED!");
      return true;
   }
   else
   {
      Print("❌ SECURITY FILTERS FAILED - TRADING NOT ALLOWED");
      return false;
   }
}


//+------------------------------------------------------------------+
//| 🎲 الگوریتم تصمیم‌گیری نهایی                                    |
//+------------------------------------------------------------------+
void FinalDecisionAlgorithm()
{
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║           🎲 FINAL DECISION ALGORITHM                 ║");
   Print("╠════════════════════════════════════════════════════════╣");
   
   //--- محاسبه امتیازات نهایی
   double totalBuyScore = g_VotingResult.indicatorBuyScore + g_VotingResult.strategyBuyScore;
   double totalSellScore = g_VotingResult.indicatorSellScore + g_VotingResult.strategySellScore;
   
   int totalBuyVotes = g_VotingResult.indicatorBuyVotes + g_VotingResult.strategyBuyVotes;
   int totalSellVotes = g_VotingResult.indicatorSellVotes + g_VotingResult.strategySellVotes;
   int totalNeutralVotes = g_VotingResult.indicatorNeutralVotes + g_VotingResult.strategyNeutralVotes;
   
   g_VotingResult.totalVotes = totalBuyVotes + totalSellVotes + totalNeutralVotes;
   
   //--- محاسبه نسبت‌ها
   double buyPercentage = (double)totalBuyVotes / g_VotingResult.totalVotes * 100;
   double sellPercentage = (double)totalSellVotes / g_VotingResult.totalVotes * 100;
   
   //--- محاسبه سطح اتفاق نظر
   double maxVotes = MathMax(totalBuyVotes, totalSellVotes);
   g_VotingResult.consensusLevel = (maxVotes / g_VotingResult.totalVotes) * 100;
   
   //--- محاسبه اطمینان نهایی
   g_VotingResult.finalConfidence = (g_VotingResult.indicatorConfidence * 0.4 + 
                                     g_VotingResult.strategyConfidence * 0.6);
   
   //--- تصمیم‌گیری بر اساس شرایط
   
   // شرط 1: اتفاق نظر بسیار بالا (85%+)
   if(buyPercentage >= 85 && totalBuyScore > totalSellScore * 3)
   {
      g_VotingResult.finalDecision = SIGNAL_BUY;
      g_VotingResult.finalScore = (totalBuyScore / totalBuyVotes) * 1.1; // بونوس 10%
      g_VotingResult.isHighQuality = true;
      g_VotingResult.recommendation = "⭐⭐⭐ STRONG BUY - EXCELLENT CONSENSUS";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   buyPercentage, totalBuyScore);
   }
   else if(sellPercentage >= 85 && totalSellScore > totalBuyScore * 3)
   {
      g_VotingResult.finalDecision = SIGNAL_SELL;
      g_VotingResult.finalScore = (totalSellScore / totalSellVotes) * 1.1;
      g_VotingResult.isHighQuality = true;
      g_VotingResult.recommendation = "⭐⭐⭐ STRONG SELL - EXCELLENT CONSENSUS";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   sellPercentage, totalSellScore);
   }
   // شرط 2: اتفاق نظر بالا (70-85%)
   else if(buyPercentage >= 70 && totalBuyScore > totalSellScore * 2)
   {
      g_VotingResult.finalDecision = SIGNAL_BUY;
      g_VotingResult.finalScore = totalBuyScore / totalBuyVotes;
      g_VotingResult.isHighQuality = true;
      g_VotingResult.recommendation = "⭐⭐ BUY - HIGH CONSENSUS";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   buyPercentage, totalBuyScore);
   }
   else if(sellPercentage >= 70 && totalSellScore > totalBuyScore * 2)
   {
      g_VotingResult.finalDecision = SIGNAL_SELL;
      g_VotingResult.finalScore = totalSellScore / totalSellVotes;
      g_VotingResult.isHighQuality = true;
      g_VotingResult.recommendation = "⭐⭐ SELL - HIGH CONSENSUS";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   sellPercentage, totalSellScore);
   }
   // شرط 3: اتفاق نظر متوسط (60-70%)
   else if(buyPercentage >= 60 && totalBuyScore > totalSellScore * 1.5)
   {
      g_VotingResult.finalDecision = SIGNAL_BUY;
      g_VotingResult.finalScore = totalBuyScore / totalBuyVotes;
      g_VotingResult.isHighQuality = (g_VotingResult.finalScore > 85);
      g_VotingResult.recommendation = "⭐ MODERATE BUY";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   buyPercentage, totalBuyScore);
   }
   else if(sellPercentage >= 60 && totalSellScore > totalBuyScore * 1.5)
   {
      g_VotingResult.finalDecision = SIGNAL_SELL;
      g_VotingResult.finalScore = totalSellScore / totalSellVotes;
      g_VotingResult.isHighQuality = (g_VotingResult.finalScore > 85);
      g_VotingResult.recommendation = "⭐ MODERATE SELL";
      g_VotingResult.decisionReason = StringFormat("%.1f%% agreement, Score:%.2f", 
                                                   sellPercentage, totalSellScore);
   }
   // شرط 4: اتفاق نظر ضعیف یا تضاد
   else
   {
      g_VotingResult.finalDecision = SIGNAL_NEUTRAL;
      g_VotingResult.finalScore = 0;
      g_VotingResult.isHighQuality = false;
      g_VotingResult.recommendation = "⚠️ WAIT - NO CLEAR SIGNAL";
      g_VotingResult.decisionReason = StringFormat("Buy:%.1f%% Sell:%.1f%% - Insufficient consensus", 
                                                   buyPercentage, sellPercentage);
   }
   
   //--- نمایش نتیجه
   Print("║                                                        ║");
   Print(StringFormat("║ Total Votes:       %d (Buy:%d Sell:%d Neutral:%d)  ║", 
                     g_VotingResult.totalVotes, totalBuyVotes, totalSellVotes, totalNeutralVotes));
   Print(StringFormat("║ Buy Percentage:    %.2f%%                           ║", buyPercentage));
   Print(StringFormat("║ Sell Percentage:   %.2f%%                           ║", sellPercentage));
   Print(StringFormat("║ Consensus Level:   %.2f%%                           ║", g_VotingResult.consensusLevel));
   Print(StringFormat("║ Final Confidence:  %.2f%%                           ║", g_VotingResult.finalConfidence));
   Print(StringFormat("║ Final Score:       %.2f/100                         ║", g_VotingResult.finalScore));
   Print(StringFormat("║ Quality:           %s                              ║", 
                     g_VotingResult.isHighQuality ? "HIGH ✅" : "MODERATE ⚠️"));
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ 🎯 DECISION:       %-30s ║", EnumToString(g_VotingResult.finalDecision)));
   Print(StringFormat("║ 💡 RECOMMENDATION: %-30s ║", g_VotingResult.recommendation));
   Print(StringFormat("║ 📝 REASON:         %-30s ║", g_VotingResult.decisionReason));
   Print("╚════════════════════════════════════════════════════════╝");
}


//+------------------------------------------------------------------+
//| 🎯 تابع اصلی سیستم رای‌گیری چند لایه                            |
//+------------------------------------------------------------------+
bool MultiLayerVotingSystem()
{
   Print("");
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║     🗳️ MULTI-LAYER VOTING SYSTEM ACTIVATED            ║");
   Print("╚════════════════════════════════════════════════════════╝");
   Print("");
   
   //--- لایه 1: رای‌گیری از اندیکاتورها
   VoteLayer1_Indicators();
   
   Print("");
   
   //--- لایه 2: رای‌گیری از استراتژی‌ها
   VoteLayer2_Strategies();
   
   Print("");
   
   //--- لایه 3: فیلترهای امنیتی
   bool securityPassed = SecurityFilters_Layer3();
   
   if(!securityPassed)
   {
      Print("");
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║  ⛔ SECURITY FILTERS FAILED - NO TRADING ALLOWED      ║");
      Print("╚════════════════════════════════════════════════════════╝");
      
      g_VotingResult.finalDecision = SIGNAL_NEUTRAL;
      g_VotingResult.finalScore = 0;
      g_VotingResult.recommendation = "REJECTED BY SECURITY FILTERS";
      
      return false;
   }
   
   Print("");
   
   //--- تصمیم‌گیری نهایی
   FinalDecisionAlgorithm();
   
   Print("");
   
   return true;
}


//+------------------------------------------------------------------+
//| 📤 بخش سیستم باز کردن معامله - TRADE OPENING SYSTEM            |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 📊 محاسبه دقیق TP و SL                                          |
//+------------------------------------------------------------------+
bool CalculateTradeLevels(ENUM_ORDER_TYPE orderType, double &entryPrice, double &tp, double &sl)
{
   //--- رفرش قیمت‌ها
   if(!symbolInfo.RefreshRates())
   {
      Print("❌ Failed to refresh rates");
      return false;
   }
   
   //--- تعیین قیمت ورود
   if(orderType == ORDER_TYPE_BUY)
   {
      entryPrice = symbolInfo.Ask();
   }
   else if(orderType == ORDER_TYPE_SELL)
   {
      entryPrice = symbolInfo.Bid();
   }
   else
   {
      Print("❌ Invalid order type");
      return false;
   }
   
   //--- محاسبه فاصله به پوینت
   double tpDistance = InpTakeProfitPoint * g_Point * 10; // × 10 برای تبدیل
   double slDistance = InpStopLossPoint * g_Point * 10;
   
   //--- محاسبه سطوح
   if(orderType == ORDER_TYPE_BUY)
   {
      tp = entryPrice + tpDistance;
      sl = entryPrice - slDistance;
   }
   else // SELL
   {
      tp = entryPrice - tpDistance;
      sl = entryPrice + slDistance;
   }
   
   //--- نرمال‌سازی
   tp = NormalizePrice(tp);
   sl = NormalizePrice(sl);
   
   //--- بررسی اعتبار سطوح
   if(!ValidateStopLevels(orderType, entryPrice, sl, tp))
   {
      Print("❌ Invalid TP/SL levels");
      return false;
   }
   
   //--- نمایش اطلاعات
   Print("📊 Trade Levels Calculated:");
   Print("   ├─ Order Type: ", EnumToString(orderType));
   Print("   ├─ Entry Price: ", DoubleToString(entryPrice, g_Digits));
   Print("   ├─ Take Profit: ", DoubleToString(tp, g_Digits), " (+", DoubleToString(tpDistance/g_Point, 1), " points)");
   Print("   ├─ Stop Loss: ", DoubleToString(sl, g_Digits), " (-", DoubleToString(slDistance/g_Point, 1), " points)");
   Print("   └─ Risk/Reward: 1:", DoubleToString(tpDistance/slDistance, 2));
   
   return true;
}


//+------------------------------------------------------------------+
//| 🔓 باز کردن معامله جدید - نسخه اصلاح شده و امن                 |
//+------------------------------------------------------------------+
bool OpenNewTrade(ENUM_SIGNAL_TYPE signal)
{
   //--- بررسی سیگنال
   if(signal != SIGNAL_BUY && signal != SIGNAL_SELL)
   {
      Print("❌ Invalid signal type for opening trade");
      return false;
   }
   
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║           📤 ATTEMPTING TO OPEN NEW TRADE             ║");
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Signal: %-45s ║", EnumToString(signal)));
   Print(StringFormat("║ Current Trades: %d/%d                              ║", 
                     g_TodayTrades, InpMaxTradesPerDay));
   Print("╚════════════════════════════════════════════════════════╝");
   
   //--- تعیین نوع سفارش
   ENUM_ORDER_TYPE orderType = (signal == SIGNAL_BUY) ? ORDER_TYPE_BUY : ORDER_TYPE_SELL;
   
   //--- رفرش قیمت‌ها
   if(!symbolInfo.RefreshRates())
   {
      Print("❌ FAILED: Cannot refresh rates");
      return false;
   }
   
   //--- تعیین قیمت ورود
   double entryPrice = 0;
   if(orderType == ORDER_TYPE_BUY)
      entryPrice = symbolInfo.Ask();
   else
      entryPrice = symbolInfo.Bid();
   
   Print("📊 Entry Price: ", DoubleToString(entryPrice, g_Digits));
   
   //--- محاسبه فاصله به پوینت
   double tpDistance = InpTakeProfitPoint * g_Point * 10;
   double slDistance = InpStopLossPoint * g_Point * 10;
   
   Print("📏 TP Distance: ", DoubleToString(tpDistance, g_Digits), 
         " (", InpTakeProfitPoint, " points)");
   Print("📏 SL Distance: ", DoubleToString(slDistance, g_Digits), 
         " (", InpStopLossPoint, " points)");
   
   //--- محاسبه سطوح
   double tp = 0, sl = 0;
   
   if(orderType == ORDER_TYPE_BUY)
   {
      tp = entryPrice + tpDistance;
      sl = entryPrice - slDistance;
   }
   else
   {
      tp = entryPrice - tpDistance;
      sl = entryPrice + slDistance;
   }
   
   //--- نرمال‌سازی
   tp = NormalizeDouble(tp, g_Digits);
   sl = NormalizeDouble(sl, g_Digits);
   
   Print("🎯 Take Profit: ", DoubleToString(tp, g_Digits));
   Print("🛑 Stop Loss: ", DoubleToString(sl, g_Digits));
   
   //--- بررسی Stop Level
   double minStopLevel = symbolInfo.StopsLevel() * g_Point;
   
   if(orderType == ORDER_TYPE_BUY)
   {
      if(sl > 0 && (entryPrice - sl) < minStopLevel)
      {
         Print("❌ FAILED: SL too close. Min distance: ", minStopLevel, 
               " Current: ", (entryPrice - sl));
         return false;
      }
      if(tp > 0 && (tp - entryPrice) < minStopLevel)
      {
         Print("❌ FAILED: TP too close. Min distance: ", minStopLevel, 
               " Current: ", (tp - entryPrice));
         return false;
      }
   }
   else
   {
      if(sl > 0 && (sl - entryPrice) < minStopLevel)
      {
         Print("❌ FAILED: SL too close. Min distance: ", minStopLevel, 
               " Current: ", (sl - entryPrice));
         return false;
      }
      if(tp > 0 && (entryPrice - tp) < minStopLevel)
      {
         Print("❌ FAILED: TP too close. Min distance: ", minStopLevel, 
               " Current: ", (entryPrice - tp));
         return false;
      }
   }
   
   //--- نرمال‌سازی حجم
   double lotSize = NormalizeDouble(InpLotSize, 2);
   
   //--- بررسی حجم مجاز
   if(lotSize < symbolInfo.LotsMin())
   {
      Print("❌ FAILED: Lot size too small. Min: ", symbolInfo.LotsMin(), 
            " Requested: ", lotSize);
      return false;
   }
   
   if(lotSize > symbolInfo.LotsMax())
   {
      Print("❌ FAILED: Lot size too large. Max: ", symbolInfo.LotsMax(), 
            " Requested: ", lotSize);
      return false;
   }
   
   Print("💼 Lot Size: ", DoubleToString(lotSize, 2));
   
   //--- بررسی مارجین
   double requiredMargin = 0;
   if(!OrderCalcMargin(orderType, InpTradeSymbol, lotSize, entryPrice, requiredMargin))
   {
      Print("❌ FAILED: Cannot calculate margin");
      return false;
   }
   
   double freeMargin = accountInfo.FreeMargin();
   
   Print("💰 Required Margin: $", DoubleToString(requiredMargin, 2));
   Print("💰 Free Margin: $", DoubleToString(freeMargin, 2));
   
   if(freeMargin < requiredMargin)
   {
      Print("❌ FAILED: Insufficient margin!");
      return false;
   }
   
   //--- ساخت کامنت
   string comment = StringFormat("%s_%s_T%d", 
                                 InpEA_Name, 
                                 InpEA_Version,
                                 g_TodayTrades + 1);
   
   Print("📝 Comment: ", comment);
   
   //--- تنظیمات Trade
   trade.SetExpertMagicNumber(InpMagicNumber);
   trade.SetDeviationInPoints(50);
   trade.SetTypeFilling(ORDER_FILLING_FOK);
   
   //--- تلاش برای باز کردن معامله
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("⏳ Sending order to broker...");
   
   bool result = false;
   
   if(orderType == ORDER_TYPE_BUY)
   {
      result = trade.Buy(lotSize, InpTradeSymbol, 0, sl, tp, comment);
   }
   else
   {
      result = trade.Sell(lotSize, InpTradeSymbol, 0, sl, tp, comment);
   }
   
   //--- بررسی دقیق نتیجه
   uint resultCode = trade.ResultRetcode();
   
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("📊 BROKER RESPONSE:");
   Print("   ├─ Result: ", result ? "TRUE" : "FALSE");
   Print("   ├─ Return Code: ", resultCode);
   Print("   ├─ Description: ", trade.ResultRetcodeDescription());
   Print("   ├─ Deal: ", trade.ResultDeal());
   Print("   ├─ Order: ", trade.ResultOrder());
   Print("   ├─ Volume: ", trade.ResultVolume());
   Print("   └─ Price: ", trade.ResultPrice());
   
   if(result && (resultCode == TRADE_RETCODE_DONE || resultCode == TRADE_RETCODE_PLACED))
   {
      ulong ticket = trade.ResultOrder();
      
      if(ticket > 0)
      {
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         Print("✅ ✅ ✅ TRADE OPENED SUCCESSFULLY! ✅ ✅ ✅");
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         Print("   ├─ Ticket: #", ticket);
         Print("   ├─ Type: ", EnumToString(orderType));
         Print("   ├─ Volume: ", DoubleToString(lotSize, 2), " lots");
         Print("   ├─ Entry: ", DoubleToString(entryPrice, g_Digits));
         Print("   ├─ TP: ", DoubleToString(tp, g_Digits));
         Print("   ├─ SL: ", DoubleToString(sl, g_Digits));
         Print("   ├─ Comment: ", comment);
         Print("   └─ Time: ", TimeToString(TimeCurrent(), TIME_DATE|TIME_MINUTES|TIME_SECONDS));
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         
         //--- ذخیره اطلاعات
         g_LastTradeTime = TimeCurrent();
         g_CurrentTicket = ticket;
         g_CurrentTP = tp;
         g_CurrentSL = sl;
         
         return true;
      }
      else
      {
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         Print("❌ ❌ ❌ TRADE FAILED - NO TICKET! ❌ ❌ ❌");
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         Print("⚠️ Broker returned success but no ticket created!");
         Print("⚠️ This is unusual - check broker connection");
         return false;
      }
   }
   else
   {
      Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
      Print("❌ ❌ ❌ TRADE FAILED! ❌ ❌ ❌");
      Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
      Print("❌ Error Code: ", resultCode);
      Print("❌ Description: ", trade.ResultRetcodeDescription());
      Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
      
      //--- توضیح خطاهای رایج
      switch(resultCode)
      {
         case TRADE_RETCODE_REJECT:
            Print("⚠️ Reason: Request rejected by broker");
            break;
         case TRADE_RETCODE_INVALID:
            Print("⚠️ Reason: Invalid request");
            break;
         case TRADE_RETCODE_INVALID_VOLUME:
            Print("⚠️ Reason: Invalid volume");
            break;
         case TRADE_RETCODE_INVALID_PRICE:
            Print("⚠️ Reason: Invalid price");
            break;
         case TRADE_RETCODE_INVALID_STOPS:
            Print("⚠️ Reason: Invalid SL/TP");
            break;
         case TRADE_RETCODE_NO_MONEY:
            Print("⚠️ Reason: Not enough money");
            break;
         case TRADE_RETCODE_PRICE_OFF:
            Print("⚠️ Reason: Price changed");
            break;
         case TRADE_RETCODE_CONNECTION:
            Print("⚠️ Reason: No connection to broker");
            break;
      }
      
      return false;
   }
}


//+------------------------------------------------------------------+
//| 📥 بخش مدیریت معاملات باز - POSITION MANAGEMENT                |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 🔄 مدیریت Trailing Stop                                         |
//+------------------------------------------------------------------+
void ManageTrailingStop()
{
   if(!InpUseTrailingStop)
      return;
   
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!position.SelectByIndex(i))
         continue;
      
      if(position.Symbol() != InpTradeSymbol)
         continue;
      
      if(position.Magic() != InpMagicNumber)
         continue;
      
      //--- دریافت اطلاعات پوزیشن
      ulong ticket = position.Ticket();
      ENUM_POSITION_TYPE posType = position.PositionType();
      double openPrice = position.PriceOpen();
      double currentSL = position.StopLoss();
      double currentTP = position.TakeProfit();
      
      double currentPrice = (posType == POSITION_TYPE_BUY) ? symbolInfo.Bid() : symbolInfo.Ask();
      
      //--- محاسبه سود فعلی به پوینت
      double profitPoints = 0;
      
      if(posType == POSITION_TYPE_BUY)
         profitPoints = (currentPrice - openPrice) / g_Point;
      else
         profitPoints = (openPrice - currentPrice) / g_Point;
      
      //--- بررسی آیا سود به حد شروع Trailing رسیده
      double trailingStart = InpTrailingStart * 10; // تبدیل به پوینت واقعی
      
      if(profitPoints < trailingStart)
         continue; // هنوز به حد شروع نرسیده
      
      //--- محاسبه SL جدید
      double newSL = 0;
      double trailingDistance = InpTrailingStep * 10 * g_Point;
      
      if(posType == POSITION_TYPE_BUY)
      {
         newSL = currentPrice - trailingDistance;
         newSL = NormalizePrice(newSL);
         
         // SL جدید باید بالاتر از SL فعلی باشد
         if(currentSL > 0 && newSL <= currentSL)
            continue;
         
         // SL جدید باید بالاتر از قیمت ورود باشد (برای حفظ سود)
         if(newSL <= openPrice)
            continue;
      }
      else // SELL
      {
         newSL = currentPrice + trailingDistance;
         newSL = NormalizePrice(newSL);
         
         // SL جدید باید پایین‌تر از SL فعلی باشد
         if(currentSL > 0 && newSL >= currentSL)
            continue;
         
         // SL جدید باید پایین‌تر از قیمت ورود باشد
         if(newSL >= openPrice)
            continue;
      }
      
      //--- تغییر SL
      if(trade.PositionModify(ticket, newSL, currentTP))
      {
         Print("✅ Trailing Stop Updated:");
         Print("   ├─ Ticket: #", ticket);
         Print("   ├─ Type: ", EnumToString(posType));
         Print("   ├─ Profit: ", DoubleToString(profitPoints, 1), " points");
         Print("   ├─ Old SL: ", DoubleToString(currentSL, g_Digits));
         Print("   └─ New SL: ", DoubleToString(newSL, g_Digits));
      }
      else
      {
         Print("⚠️ Failed to update Trailing Stop for #", ticket);
      }
   }
}

//+------------------------------------------------------------------+
//| 📊 بررسی و به‌روزرسانی پوزیشن‌ها                                |
//+------------------------------------------------------------------+
void CheckOpenPositions()
{
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      if(!position.SelectByIndex(i))
         continue;
      
      if(position.Symbol() != InpTradeSymbol)
         continue;
      
      if(position.Magic() != InpMagicNumber)
         continue;
      
      //--- دریافت اطلاعات
      ulong ticket = position.Ticket();
      double profit = position.Profit();
      double openPrice = position.PriceOpen();
      ENUM_POSITION_TYPE posType = position.PositionType();
      
      double currentPrice = (posType == POSITION_TYPE_BUY) ? symbolInfo.Bid() : symbolInfo.Ask();
      
      //--- محاسبه سود/ضرر به پوینت
      double profitPoints = 0;
      if(posType == POSITION_TYPE_BUY)
         profitPoints = (currentPrice - openPrice) / g_Point;
      else
         profitPoints = (openPrice - currentPrice) / g_Point;
      
      //--- نمایش وضعیت (هر 20 تیک یک بار)
      static int tickCounter = 0;
      tickCounter++;
      
      if(tickCounter % 20 == 0)
      {
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
         Print("📊 Position Status #", ticket, ":");
         Print("   ├─ Type: ", EnumToString(posType));
         
         if(profit >= 0)
            Print("   ├─ P/L: ✅ $", DoubleToString(profit, 2), " (+", DoubleToString(profitPoints, 1), " points)");
         else
            Print("   ├─ P/L: ❌ $", DoubleToString(profit, 2), " (", DoubleToString(profitPoints, 1), " points)");
         
         Print("   ├─ Entry: ", DoubleToString(openPrice, g_Digits));
         Print("   ├─ Current: ", DoubleToString(currentPrice, g_Digits));
         
         if(InpUseDynamicSL)
         {
            int secondsToNext = InpMinutesReanalysis * 60 - (int)(TimeCurrent() - g_LastReanalysisTime);
            if(secondsToNext > 0)
               Print("   └─ 🧠 Next reanalysis in: ", secondsToNext, " seconds");
            else
               Print("   └─ 🧠 Reanalysis due now");
         }
         
         Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
      }
   }
}


//+------------------------------------------------------------------+
//| 🔍 بررسی معاملات بسته شده - بدون تکرار                         |
//+------------------------------------------------------------------+
void CheckClosedPositions()
{
   static ulong g_LastProcessedDeal = 0;  // آخرین Deal پردازش شده (STATIC مهمه!)
   
   //--- انتخاب تاریخچه از ابتدای روز تا الان
   datetime today_start = iTime(InpTradeSymbol, PERIOD_D1, 0);
   datetime now = TimeCurrent();
   
   if(!HistorySelect(today_start, now))
      return;
   
   int totalDeals = HistoryDealsTotal();
   
   if(totalDeals == 0)
      return;
   
   //--- فقط آخرین Deal جدید را بررسی کن
   for(int i = totalDeals - 1; i >= 0; i--)
   {
      ulong dealTicket = HistoryDealGetTicket(i);
      
      if(dealTicket == 0)
         continue;
      
      //--- اگر قبلاً پردازش شده، رد کن
      if(dealTicket <= g_LastProcessedDeal)
         break; // از بقیه Dealها رد شو
      
      //--- فقط معاملات این EA
      if(HistoryDealGetString(dealTicket, DEAL_SYMBOL) != InpTradeSymbol)
         continue;
      
      if(HistoryDealGetInteger(dealTicket, DEAL_MAGIC) != InpMagicNumber)
         continue;
      
      //--- فقط معاملات خروجی (بسته شده)
      ENUM_DEAL_ENTRY dealEntry = (ENUM_DEAL_ENTRY)HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
      
      if(dealEntry != DEAL_ENTRY_OUT)
         continue;
      
      //--- این یک Deal خروجی واقعی است!
      g_LastProcessedDeal = dealTicket; // ثبت کن که پردازش شد
      
      //--- دریافت اطلاعات
      double profit = HistoryDealGetDouble(dealTicket, DEAL_PROFIT);
      double swap = HistoryDealGetDouble(dealTicket, DEAL_SWAP);
      double commission = HistoryDealGetDouble(dealTicket, DEAL_COMMISSION);
      double totalProfit = profit + swap + commission;
      
      double volume = HistoryDealGetDouble(dealTicket, DEAL_VOLUME);
      ENUM_DEAL_TYPE dealType = (ENUM_DEAL_TYPE)HistoryDealGetInteger(dealTicket, DEAL_TYPE);
      datetime dealTime = (datetime)HistoryDealGetInteger(dealTicket, DEAL_TIME);
      double dealPrice = HistoryDealGetDouble(dealTicket, DEAL_PRICE);
      
      //--- نمایش
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║           📥 POSITION CLOSED (NEW!)                   ║");
      Print("╠════════════════════════════════════════════════════════╣");
      Print(StringFormat("║ Deal Ticket: #%-40d ║", dealTicket));
      Print(StringFormat("║ Type: %-47s ║", EnumToString(dealType)));
      Print(StringFormat("║ Volume: %.2f lots                                     ║", volume));
      Print(StringFormat("║ Close Price: %-38s ║", DoubleToString(dealPrice, g_Digits)));
      Print(StringFormat("║ Close Time: %-39s ║", TimeToString(dealTime, TIME_DATE|TIME_MINUTES)));
      Print("╠════════════════════════════════════════════════════════╣");
      Print(StringFormat("║ Profit: $%.2f                                        ║", profit));
      Print(StringFormat("║ Swap: $%.2f                                          ║", swap));
      Print(StringFormat("║ Commission: $%.2f                                    ║", commission));
      Print(StringFormat("║ Total P/L: $%.2f                                     ║", totalProfit));
      Print("╚════════════════════════════════════════════════════════╝");
      
      //--- به‌روزرسانی آمار (فقط یک بار!)
      UpdateTradeStatistics(totalProfit);
      
      //--- فقط یک Deal در هر بار - خروج
      break;
   }
}


//+------------------------------------------------------------------+
//| 🎯 تابع اصلی OnTick - قلب اجرایی ربات                           |
//+------------------------------------------------------------------+
void OnTick()
{
   //--- بررسی روز جدید و ریست آمار
   CheckAndResetDailyStats();
   
   //--- بررسی معاملات بسته شده
   CheckClosedPositions();
   
   //--- مدیریت پوزیشن‌های باز
   if(CountOpenPositions() > 0)
   {
      CheckOpenPositions();
      
      //--- *** این خط جدید را اضافه کنید ***
      ManageSmartDynamicSL();  // 🧠 SL هوشمند قبل از Trailing
      
      ManageTrailingStop();
      return; // اگر پوزیشن باز داریم، سیگنال جدید نمی‌گیریم
   }
   
   // ... بقیه کد

   //--- بررسی اتصال
   if(!TerminalInfoInteger(TERMINAL_CONNECTED))
   {
      static bool connectionWarning = false;
      if(!connectionWarning)
      {
         Print("⚠️ WARNING: No connection to trade server!");
         connectionWarning = true;
      }
      return;
   }
   
   //--- بررسی مجوز معامله
   if(!TerminalInfoInteger(TERMINAL_TRADE_ALLOWED))
   {
      static bool tradeWarning = false;
      if(!tradeWarning)
      {
         Print("⚠️ WARNING: AutoTrading is disabled in terminal!");
         Print("⚠️ Please enable AutoTrading (Ctrl+E or click AutoTrading button)");
         tradeWarning = true;
      }
      return;
   }
   
   //--- بررسی روز جدید و ریست آمار
   CheckAndResetDailyStats();
   
   //--- بررسی معاملات بسته شده
   CheckClosedPositions();
   
   // ... ادامه کد

   //--- بررسی روز جدید و ریست آمار
   CheckAndResetDailyStats();
   
   //--- بررسی معاملات بسته شده
   CheckClosedPositions();
   
   //--- مدیریت پوزیشن‌های باز
   if(CountOpenPositions() > 0)
   {
      CheckOpenPositions();
      ManageTrailingStop();
      return; // اگر پوزیشن باز داریم، سیگنال جدید نمی‌گیریم
   }
   
   //--- بررسی امکان باز کردن معامله جدید
   if(!CanOpenNewTrade())
      return;
   
   //--- محدودیت به تعداد تیک (برای کاهش بار)
   static int tickCount = 0;
   tickCount++;
   
   // هر 5 تیک یک بار سیگنال بگیریم (برای M1 مناسب است)
   if(tickCount % 5 != 0)
      return;
   
   //--- بررسی کندل جدید
   static datetime lastBarTime = 0;
   datetime currentBarTime = iTime(InpTradeSymbol, InpMainTF, 0);
   
   bool isNewBar = (currentBarTime != lastBarTime);
   
   if(isNewBar)
   {
      lastBarTime = currentBarTime;
      
      Print("");
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║              🕐 NEW CANDLE DETECTED                    ║");
      Print("╠════════════════════════════════════════════════════════╣");
      Print(StringFormat("║ Time: %s                                       ║", 
                        TimeToString(currentBarTime, TIME_DATE|TIME_MINUTES)));
      Print(StringFormat("║ Today's Trades: %d/%d                              ║", 
                        g_TodayTrades, InpMaxTradesPerDay));
      Print(StringFormat("║ Today's P/L: $%.2f                               ║", 
                        g_TodayProfit));
      Print("╚════════════════════════════════════════════════════════╝");
      Print("");
   }
   
   //--- فقط در کندل جدید سیگنال بگیریم
   if(!isNewBar)
      return;
   
   Print("🔄 Starting Analysis...");
   Print("");
   
   //--- گام 1: به‌روزرسانی اندیکاتورها
   if(!UpdateAllIndicators())
   {
      Print("⚠️ Failed to update indicators");
      return;
   }
   
   Print("");
   
   //--- گام 2: اجرای استراتژی‌ها
   if(!ExecuteAllStrategies())
   {
      Print("⚠️ Failed to execute strategies");
      return;
   }
   
   Print("");
   
   //--- گام 3: سیستم رای‌گیری چند لایه
   if(!MultiLayerVotingSystem())
   {
      Print("⚠️ Voting system failed or rejected signal");
      return;
   }
   
   Print("");
   
   //--- گام 4: تصمیم نهایی
   ENUM_SIGNAL_TYPE finalSignal = g_VotingResult.finalDecision;
   
   if(finalSignal == SIGNAL_NEUTRAL)
   {
      Print("⚪ Final Decision: NO TRADE - Waiting for better opportunity");
      return;
   }
   
   //--- بررسی کیفیت سیگنال
   if(!g_VotingResult.isHighQuality)
   {
      Print("⚠️ Signal quality is not high enough - Skipping");
      Print("   └─ Score: ", DoubleToString(g_VotingResult.finalScore, 2), 
            " Confidence: ", DoubleToString(g_VotingResult.finalConfidence, 2), "%");
      return;
   }
   
   //--- بررسی حداقل امتیاز
   if(g_VotingResult.finalScore < 75)
   {
      Print("⚠️ Final score too low: ", DoubleToString(g_VotingResult.finalScore, 2), " < 75");
      return;
   }
   
   //--- بررسی حداقل اطمینان
   if(g_VotingResult.finalConfidence < 70)
   {
      Print("⚠️ Final confidence too low: ", DoubleToString(g_VotingResult.finalConfidence, 2), "% < 70%");
      return;
   }
   
   Print("");
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║           ✅ HIGH QUALITY SIGNAL DETECTED!            ║");
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Signal: %-45s ║", EnumToString(finalSignal)));
   Print(StringFormat("║ Score: %.2f/100                                      ║", 
                     g_VotingResult.finalScore));
   Print(StringFormat("║ Confidence: %.2f%%                                   ║", 
                     g_VotingResult.finalConfidence));
   Print(StringFormat("║ Consensus: %.2f%%                                    ║", 
                     g_VotingResult.consensusLevel));
   Print(StringFormat("║ Recommendation: %-35s ║", 
                     g_VotingResult.recommendation));
   Print("╚════════════════════════════════════════════════════════╝");
   Print("");
   
   //--- گام 5: باز کردن معامله
   if(OpenNewTrade(finalSignal))
   {
      Print("");
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║          🎉 TRADE EXECUTED SUCCESSFULLY!              ║");
      Print("╚════════════════════════════════════════════════════════╝");
      Print("");
      
      //--- نمایش آمار به‌روز
      PrintCurrentStatistics();
   }
   else
   {
      Print("");
      Print("╔════════════════════════════════════════════════════════╗");
      Print("║          ❌ TRADE EXECUTION FAILED!                   ║");
      Print("╚════════════════════════════════════════════════════════╝");
      Print("");
   }
}


//+------------------------------------------------------------------+
//| 🛠️ توابع کمکی نهایی - FINAL HELPER FUNCTIONS                   |
//+------------------------------------------------------------------+

//+------------------------------------------------------------------+
//| 🔔 تابع نوتیفیکیشن (اختیاری)                                    |
//+------------------------------------------------------------------+
void SendTradeNotification(string message)
{
   // می‌توانید این تابع را فعال کنید برای دریافت اعلان
   // SendNotification(message);
   
   // یا ارسال به تلگرام/ایمیل
   // SendMail("EMITER_V1 Trade Alert", message);
}

//+------------------------------------------------------------------+
//| 📊 نمایش اطلاعات روی چارت                                       |
//+------------------------------------------------------------------+
void DisplayInfoOnChart()
{
   // می‌توانید از Comment یا ObjectCreate استفاده کنید
   
   string info = "";
   info += "═══════════════════════════════════════\n";
   info += "🤖 " + InpEA_Name + " v" + InpEA_Version + "\n";
   info += "═══════════════════════════════════════\n";
   info += "📅 Trades Today: " + IntegerToString(g_TodayTrades) + "/" + IntegerToString(InpMaxTradesPerDay) + "\n";
   info += "💰 Today's P/L: $" + DoubleToString(g_TodayProfit, 2) + "\n";
   info += "📊 Balance: $" + DoubleToString(accountInfo.Balance(), 2) + "\n";
   info += "📈 Equity: $" + DoubleToString(accountInfo.Equity(), 2) + "\n";
   info += "───────────────────────────────────────\n";
   info += "✅ Wins: " + IntegerToString(g_ConsecutiveWins) + "\n";
   info += "❌ Losses: " + IntegerToString(g_ConsecutiveLosses) + "\n";
   info += "───────────────────────────────────────\n";
   info += "🗳️ Last Signal: " + EnumToString(g_VotingResult.finalDecision) + "\n";
   info += "💯 Score: " + DoubleToString(g_VotingResult.finalScore, 1) + "\n";
   info += "🎯 Confidence: " + DoubleToString(g_VotingResult.finalConfidence, 1) + "%\n";
   info += "═══════════════════════════════════════\n";
   
   Comment(info);
}

//+------------------------------------------------------------------+
//| 🧹 پاکسازی کامنت از چارت                                        |
//+------------------------------------------------------------------+
void ClearChartComment()
{
   Comment("");
}

//+------------------------------------------------------------------+
//| 📝 لاگ کامل سیستم                                                |
//+------------------------------------------------------------------+
void LogSystemState()
{
   // برای دیباگ - می‌توانید به فایل بنویسید
   
   string logFile = "EMITER_V1_Log_" + TimeToString(TimeCurrent(), TIME_DATE) + ".txt";
   int fileHandle = FileOpen(logFile, FILE_WRITE|FILE_READ|FILE_TXT|FILE_ANSI, '\t');
   
   if(fileHandle != INVALID_HANDLE)
   {
      FileSeek(fileHandle, 0, SEEK_END);
      
      FileWrite(fileHandle, "═══════════════════════════════════════");
      FileWrite(fileHandle, "Time: " + TimeToString(TimeCurrent(), TIME_DATE|TIME_MINUTES|TIME_SECONDS));
      FileWrite(fileHandle, "Signal: " + EnumToString(g_VotingResult.finalDecision));
      FileWrite(fileHandle, "Score: " + DoubleToString(g_VotingResult.finalScore, 2));
      FileWrite(fileHandle, "Trades: " + IntegerToString(g_TodayTrades));
      FileWrite(fileHandle, "P/L: " + DoubleToString(g_TodayProfit, 2));
      FileWrite(fileHandle, "═══════════════════════════════════════");
      
      FileClose(fileHandle);
   }
}


//+------------------------------------------------------------------+
//| 📊 نمایش وضعیت SL پویا                                          |
//+------------------------------------------------------------------+
void PrintDynamicSLStatus()
{
   if(!InpUseDynamicSL)
      return;
   
   Print("╔════════════════════════════════════════════════════════╗");
   Print("║          🧠 DYNAMIC SL SYSTEM STATUS                  ║");
   Print("╠════════════════════════════════════════════════════════╣");
   Print(StringFormat("║ Active: %s                                          ║", 
                     InpUseDynamicSL ? "YES ✅" : "NO ❌"));
   Print(StringFormat("║ Reanalysis Interval: %d minute(s)                   ║", 
                     InpMinutesReanalysis));
   Print(StringFormat("║ Min Confirmations to Hold: %d/25                    ║", 
                     InpMinConfirmToHold));
   Print(StringFormat("║ Max Loss to Hold: %.1f points                       ║", 
                     InpMaxLossToHold));
   Print(StringFormat("║ Emergency SL: %.1f points                           ║", 
                     InpEmergencySL));
   Print(StringFormat("║ Min Score to Hold: %d/100                           ║", 
                     InpMinScoreToHold));
   Print(StringFormat("║ Close on Signal Change: %s                          ║", 
                     InpCloseOnSignalChange ? "YES" : "NO"));
   Print(StringFormat("║ Total Reanalyses: %d                                ║", 
                     g_ReanalysisCount));
   Print("╚════════════════════════════════════════════════════════╝");
}


//+------------------------------------------------------------------+
//| ⏰ تابع OnTimer (اختیاری)                                        |
//+------------------------------------------------------------------+
void OnTimer()
{
   // می‌توانید از این تابع برای به‌روزرسانی نمایش استفاده کنید
   DisplayInfoOnChart();
}


//+------------------------------------------------------------------+
//| 🎯 تابع OnInit کامل شده (نسخه نهایی)                           |
//+------------------------------------------------------------------+
int OnInit()
{
   Print("═══════════════════════════════════════════════════════");
   Print("🚀 Starting EMITER_V1 - Version: ", InpEA_Version);
   Print("═══════════════════════════════════════════════════════");
   
   //--- گام 1: بررسی نماد معاملاتی
   if(!InitSymbolParameters())
   {
      Print("❌ ERROR: Failed to initialize symbol parameters!");
      return(INIT_FAILED);
   }
   
   //--- گام 2: تنظیمات Trade Object
   trade.SetExpertMagicNumber(InpMagicNumber);
   trade.SetDeviationInPoints(50);
   trade.SetTypeFilling(ORDER_FILLING_FOK);
   trade.SetAsyncMode(false);
   trade.LogLevel(LOG_LEVEL_ERRORS);
   
   //--- گام 3: اولیه‌سازی اندیکاتورها
   if(!InitIndicators())
   {
      Print("❌ ERROR: Failed to initialize indicators!");
      return(INIT_FAILED);
   }
   
   //--- گام 4: تنظیم بافرهای اندیکاتورها
   if(!SetBufferArrays())
   {
      Print("❌ ERROR: Failed to set buffer arrays!");
      return(INIT_FAILED);
   }
   
   //--- گام 5: اولیه‌سازی متغیرهای آماری
   ResetDailyStats();
   g_StartDayBalance = accountInfo.Balance();
   g_CurrentDay = TimeCurrent();
   
   //--- گام 6: بررسی تنظیمات ورودی
   if(!ValidateInputParameters())
   {
      Print("❌ ERROR: Invalid input parameters!");
      return(INIT_FAILED);
   }
   
   //--- گام 7: نمایش اطلاعات
   PrintInitInfo();
   PrintDynamicSLStatus();  // نمایش تنظیمات SL پویا


//--- نمایش تنظیمات زمانی
if(InpUseTimeFilter)
{
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("⏰ TIME FILTER SETTINGS (Tehran Time):");
   Print("   ├─ Active: YES");
   Print(StringFormat("   ├─ Trading Hours: %02d:%02d - %02d:%02d", 
                     InpStartHour, InpStartMinute, 
                     InpEndHour, InpEndMinute));
   Print("   ├─ Active Days:");
   if(InpTradeMonday) Print("   │  ├─ Monday: ✅");
   if(InpTradeTuesday) Print("   │  ├─ Tuesday: ✅");
   if(InpTradeWednesday) Print("   │  ├─ Wednesday: ✅");
   if(InpTradeThursday) Print("   │  ├─ Thursday: ✅");
   if(InpTradeFriday) Print("   │  ├─ Friday: ✅");
   if(!InpTradeMonday) Print("   │  ├─ Monday: ❌");
   if(!InpTradeTuesday) Print("   │  ├─ Tuesday: ❌");
   if(!InpTradeWednesday) Print("   │  ├─ Wednesday: ❌");
   if(!InpTradeThursday) Print("   │  ├─ Thursday: ❌");
   if(!InpTradeFriday) Print("   │  ├─ Friday: ❌");
   Print("   └─ News Avoidance: ", InpAvoidNews ? "YES" : "NO");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
}
else
{
   Print("⏰ TIME FILTER: DISABLED (Trading 24/7)");
}
   
   //--- گام 8: تنظیم Timer (هر 1 ثانیه)
   EventSetTimer(1);
   
   Print("✅ EMITER_V1 initialized successfully!");
   Print("🎯 Target: 10 Trades × $1 = $10 Daily Profit");
   Print("🛡️ Risk Management: Active");
   Print("🗳️ Multi-Layer Voting: Active");
   Print("📊 Total Indicators: 15");
   Print("🎯 Total Strategies: 25");
   Print("═══════════════════════════════════════════════════════");
   
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| 🛑 تابع OnDeinit کامل شده (نسخه نهایی)                         |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   Print("═══════════════════════════════════════════════════════");
   Print("🛑 Stopping EMITER_V1...");
   
   //--- حذف Timer
   EventKillTimer();
   
   //--- پاکسازی کامنت
   ClearChartComment();
   
   //--- پاکسازی اندیکاتورها
   if(handle_EMA_Fast != INVALID_HANDLE) IndicatorRelease(handle_EMA_Fast);
   if(handle_EMA_Slow != INVALID_HANDLE) IndicatorRelease(handle_EMA_Slow);
   if(handle_RSI != INVALID_HANDLE) IndicatorRelease(handle_RSI);
   if(handle_Stoch != INVALID_HANDLE) IndicatorRelease(handle_Stoch);
   if(handle_MACD != INVALID_HANDLE) IndicatorRelease(handle_MACD);
   if(handle_ATR != INVALID_HANDLE) IndicatorRelease(handle_ATR);
   if(handle_BB != INVALID_HANDLE) IndicatorRelease(handle_BB);
   if(handle_CCI != INVALID_HANDLE) IndicatorRelease(handle_CCI);
   if(handle_ADX != INVALID_HANDLE) IndicatorRelease(handle_ADX);
   if(handle_WPR != INVALID_HANDLE) IndicatorRelease(handle_WPR);
   if(handle_MOM != INVALID_HANDLE) IndicatorRelease(handle_MOM);
   if(handle_SAR != INVALID_HANDLE) IndicatorRelease(handle_SAR);
   if(handle_OBV != INVALID_HANDLE) IndicatorRelease(handle_OBV);
   if(handle_AO != INVALID_HANDLE) IndicatorRelease(handle_AO);
   if(handle_DeMarker != INVALID_HANDLE) IndicatorRelease(handle_DeMarker);
   
   Print("✅ All indicators released");
   
   //--- نمایش آمار نهایی
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   Print("📊 FINAL SESSION STATISTICS:");
   Print("   ├─ Today's Trades: ", g_TodayTrades);
   Print("   ├─ Today's Profit: $", DoubleToString(g_TodayProfit, 2));
   Print("   ├─ Consecutive Wins: ", g_ConsecutiveWins);
   Print("   ├─ Consecutive Losses: ", g_ConsecutiveLosses);
   Print("   ├─ Final Balance: $", DoubleToString(accountInfo.Balance(), 2));
   Print("   └─ Final Equity: $", DoubleToString(accountInfo.Equity(), 2));
   
   double profitPercent = 0;
   if(g_StartDayBalance > 0)
      profitPercent = (g_TodayProfit / g_StartDayBalance) * 100;
   
   Print("   └─ ROI Today: ", DoubleToString(profitPercent, 2), "%");
   Print("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
   
   //--- دلیل توقف
   string reason_text;
   switch(reason)
   {
      case REASON_PROGRAM:     reason_text = "Expert manually stopped"; break;
      case REASON_REMOVE:      reason_text = "Expert removed from chart"; break;
      case REASON_RECOMPILE:   reason_text = "Expert recompiled"; break;
      case REASON_CHARTCHANGE: reason_text = "Chart symbol or period changed"; break;
      case REASON_CHARTCLOSE:  reason_text = "Chart closed"; break;
      case REASON_PARAMETERS:  reason_text = "Input parameters changed"; break;
      case REASON_ACCOUNT:     reason_text = "Account changed"; break;
      default:                 reason_text = "Unknown reason"; break;
   }
   
   Print("🔴 Reason: ", reason_text);
   Print("═══════════════════════════════════════════════════════");
}
