#property copyright "Your Name"
#property link      "https://www.mql5.com"
#property version   "1.02"
#property strict
#property description "NAS100-Optimized Multi-Timeframe EMA Trading System"

//+------------------------------------------------------------------+
//| Constants and Error Definitions                                    |
//+------------------------------------------------------------------+
#define ERR_REQUOTE         136
#define ERR_INVALID_PRICE   131
#define ERR_INVALID_STOPS   130
#define ERR_PRICE_CHANGED       10006
#define ERR_INVALID_TRADE_VOLUME 10023
#define ERR_TRADE_NOT_ALLOWED   10014
#define ERR_MARKET_CLOSED       10020

//+------------------------------------------------------------------+
//| Perform safety checks before trading                              |
//+------------------------------------------------------------------+
bool PerformSafetyChecks()
{
    // Check if max daily trades reached
    if(Market.dailyTradeCount >= MaxDailyTrades)
    {
        Print("Maximum daily trades reached");
        return false;
    }
    
    // Check drawdown limits
    if(!isWithinDrawdownLimits())
    {
        Print("Drawdown limits exceeded");
        return false;
    }
    
    // Manage recovery mode if needed
    ManageRecoveryMode();
    
    return true;
}

//+------------------------------------------------------------------+
//| Verify indicator handles                                          |
//+------------------------------------------------------------------+
bool VerifyIndicatorHandles()
{
    // Check if any indicator handles are invalid
    return (Handles.fast_ema_m5 != INVALID_HANDLE &&
            Handles.slow_ema_m5 != INVALID_HANDLE &&
            Handles.fast_ema_m15 != INVALID_HANDLE &&
            Handles.slow_ema_m15 != INVALID_HANDLE &&
            Handles.fast_ema_h1 != INVALID_HANDLE &&
            Handles.slow_ema_h1 != INVALID_HANDLE &&
            Handles.atr != INVALID_HANDLE);
}

//+------------------------------------------------------------------+
//| Risk Management Parameters - Optimized for NAS100                  |
//+------------------------------------------------------------------+
input group "Risk Management"
input double RiskPercent = 0.5;           // Risk per trade (%) - Reduced for NAS100
input double MaxDailyDD = 2.0;            // Maximum daily drawdown (%)
input double MaxWeeklyDD = 4.0;           // Maximum weekly drawdown (%)
input double MaxSpreadPoints = 30;         // Maximum allowed spread in points for NAS100
input double MaxMarginPercent = 15.0;      // Maximum margin usage (%)

input group "Trade Management"
input int ATR_Period = 14;                // ATR Period for dynamic SL
input double ATR_Multiplier = 2.0;        // ATR Multiplier for SL calculation
int MaxDailyTrades = 2;                   // Changed from input to regular variable
input double MinWinRate = 45.0;           // Minimum required win rate (%)
input int WinRateCalculationPeriod = 20;  // Number of trades to calculate win rate
input double InitialSLPoints = 100;       // Initial Stop Loss in Points for NAS100

//+------------------------------------------------------------------+
//| Trading Parameters - Adjusted for NAS100                          |
//+------------------------------------------------------------------+
input group "Moving Averages"
input int FastEMA_M5 = 8;                 // 5M Fast EMA Period
input int SlowEMA_M5 = 21;                // 5M Slow EMA Period
input int FastEMA_M15 = 13;               // 15M Fast EMA Period
input int SlowEMA_M15 = 34;               // 15M Slow EMA Period
input int FastEMA_H1 = 21;                // 1H Fast EMA Period
input int SlowEMA_H1 = 55;                // 1H Slow EMA Period

input group "Trade Exit"
double ActivationPoints = 50;             // Changed from input to regular variable
double PartialProfitPct = 0.8;      // Profit percentage to trigger partial close
input double PartialClosePct = 0.5;             // Changed from input to regular variable
input double MinProfitToClose = 50;       // Minimum points in profit for early exit
input double MaxStopLevel = 200;          // Maximum stop loss in points

//+------------------------------------------------------------------+
//| Trading Time Filters - Adjusted for NAS100                        |
//+------------------------------------------------------------------+
input group "Time Filters"
input string SessionStart = "13:00";       // Session Start Time (Pre-market)
input string SessionEnd = "20:30";         // Session End Time (NY Close)
input bool MondayFilter = true;           // Enable Monday Filter
input bool FridayFilter = true;           // Enable Friday Filter
input bool TradePreMarket = true;         // Allow pre-market trading

//+------------------------------------------------------------------+
//| Volatility Filter - Adjusted for NAS100                           |
//+------------------------------------------------------------------+
input group "Volatility"
input bool UseVolatilityFilter = true;    // Enable Volatility Filter
input double MaxATRMultiplier = 2.5;      // Maximum ATR multiplier for volatility
input double MinATRMultiplier = 0.4;      // Minimum ATR multiplier for volatility
input double VolatilityThreshold = 50;    // Maximum ATR value for normal session
input double PreMarketVolThreshold = 75;  // Maximum ATR value for pre-market

//+------------------------------------------------------------------+
//| Global Variables                                                   |
//+------------------------------------------------------------------+
// Indicator handles
struct IndicatorHandles {
    int fast_ema_m5;
    int slow_ema_m5;
    int fast_ema_m15;
    int slow_ema_m15;
    int fast_ema_h1;
    int slow_ema_h1;
    int atr;
} Handles;

// Market variables
struct MarketVariables {
    double point;
    double pip;
    datetime sessionStartTime;
    datetime sessionEndTime;
    double initialBalance;
    double weeklyInitialBalance;
    datetime lastResetTime;
    datetime weeklyResetTime;
    int dailyTradeCount;
    datetime lastTradeTime;
    int totalTrades;
    int winningTrades;
    bool isPreMarket;
    double baseSpreadPoints;
    double baseRiskPercent;
} Market;

//+------------------------------------------------------------------+
//| Trade History Structure                                           |
//+------------------------------------------------------------------+
struct TradeHistory {
    datetime openTime;
    bool isWin;
    double profit;
    double lots;
    double entryPrice;
    double exitPrice;
    double stopLoss;
    double takeProfit;
    ENUM_ORDER_TYPE type;
    string comment;
};

TradeHistory tradeHistory[];

//+------------------------------------------------------------------+
//| Session State Structure                                           |
//+------------------------------------------------------------------+
struct SessionState {
    bool isTrading;
    bool isPreMarket;
    double currentSpread;
    double currentATR;
    double volatilityRatio;
    int marketDirection;
    datetime lastUpdate;
} Session;

//+------------------------------------------------------------------+
//| Risk State Structure                                              |
//+------------------------------------------------------------------+
struct RiskState {
    double currentRisk;
    double maxPositionSize;
    double availableMargin;
    double marginLevel;
    double equityPercent;
    bool isHighVolatility;
    bool isExcessiveSpread;
} Risk;

//+------------------------------------------------------------------+
//| Expert initialization function                                     |
//+------------------------------------------------------------------+
int OnInit()
{
    // Verify symbol is NAS100
    if(_Symbol != "NAS100" && _Symbol != "US100" && _Symbol != "USTEC")
    {
        Print("Error: EA is optimized for NAS100/US100 only");
        return(INIT_FAILED);
    }

    // Initialize indicator handles with error checking
    if(!InitializeIndicators())
    {
        Print("Failed to initialize indicators");
        return(INIT_FAILED);
    }
    
    // Initialize market variables
    if(!InitializeMarketVariables())
    {
        Print("Failed to initialize market variables");
        return(INIT_FAILED);
    }
    
    // Validate and initialize session times
    if(!InitializeSessionTimes())
    {
        Print("Failed to initialize session times");
        return(INIT_FAILED);
    }
    
    // Initialize risk parameters
    if(!InitializeRiskParameters())
    {
        Print("Failed to initialize risk parameters");
        return(INIT_FAILED);
    }
    
    // Initialize trade history array
    InitializeTradeHistory();
    
    // Create timer for periodic checks
    EventSetTimer(60);
    
    return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Initialize all indicators                                          |
//+------------------------------------------------------------------+
bool InitializeIndicators()
{
    // Initialize M5 EMAs
    Handles.fast_ema_m5 = iMA(_Symbol, PERIOD_M5, FastEMA_M5, 0, MODE_EMA, PRICE_CLOSE);
    Handles.slow_ema_m5 = iMA(_Symbol, PERIOD_M5, SlowEMA_M5, 0, MODE_EMA, PRICE_CLOSE);
    
    // Initialize M15 EMAs
    Handles.fast_ema_m15 = iMA(_Symbol, PERIOD_M15, FastEMA_M15, 0, MODE_EMA, PRICE_CLOSE);
    Handles.slow_ema_m15 = iMA(_Symbol, PERIOD_M15, SlowEMA_M15, 0, MODE_EMA, PRICE_CLOSE);
    
    // Initialize H1 EMAs
    Handles.fast_ema_h1 = iMA(_Symbol, PERIOD_H1, FastEMA_H1, 0, MODE_EMA, PRICE_CLOSE);
    Handles.slow_ema_h1 = iMA(_Symbol, PERIOD_H1, SlowEMA_H1, 0, MODE_EMA, PRICE_CLOSE);
    
    // Initialize ATR
    Handles.atr = iATR(_Symbol, PERIOD_CURRENT, ATR_Period);
    
    // Verify all handles
    if(Handles.fast_ema_m5 == INVALID_HANDLE || 
       Handles.slow_ema_m5 == INVALID_HANDLE ||
       Handles.fast_ema_m15 == INVALID_HANDLE || 
       Handles.slow_ema_m15 == INVALID_HANDLE ||
       Handles.fast_ema_h1 == INVALID_HANDLE || 
       Handles.slow_ema_h1 == INVALID_HANDLE ||
       Handles.atr == INVALID_HANDLE)
    {
        Print("Error creating indicators. Error code: ", GetLastError());
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Initialize market variables                                        |
//+------------------------------------------------------------------+
bool InitializeMarketVariables()
{
    // Initialize basic market variables
    Market.point = SymbolInfoDouble(_Symbol, SYMBOL_POINT);
    Market.pip = Market.point;  // For NAS100, point and pip are the same
    
    // Store base values for risk management
    Market.baseSpreadPoints = MaxSpreadPoints;
    Market.baseRiskPercent = RiskPercent;
    
    // Initialize tracking variables
    Market.initialBalance = AccountInfoDouble(ACCOUNT_BALANCE);
    Market.weeklyInitialBalance = Market.initialBalance;
    Market.dailyTradeCount = 0;
    Market.lastTradeTime = 0;
    Market.totalTrades = 0;
    Market.winningTrades = 0;
    
    // Initialize session state
    ResetSessionState();
    
    // Validate symbol-specific parameters
    if(!ValidateSymbolParameters())
    {
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Initialize session times                                           |
//+------------------------------------------------------------------+
bool InitializeSessionTimes()
{
    string start_arr[], end_arr[];
    if(StringSplit(SessionStart, ':', start_arr) != 2 || 
       StringSplit(SessionEnd, ':', end_arr) != 2)
    {
        Print("Invalid time format. Use HH:MM");
        return false;
    }
    
    MqlDateTime time_struct;
    TimeToStruct(TimeCurrent(), time_struct);
    
    // Validate hour and minute values
    int start_hour = (int)StringToInteger(start_arr[0]);
    int start_min = (int)StringToInteger(start_arr[1]);
    int end_hour = (int)StringToInteger(end_arr[0]);
    int end_min = (int)StringToInteger(end_arr[1]);
    
    if(start_hour < 0 || start_hour > 23 || end_hour < 0 || end_hour > 23 ||
       start_min < 0 || start_min > 59 || end_min < 0 || end_min > 59)
    {
        Print("Invalid time values");
        return false;
    }
    
    // Set session times
    time_struct.hour = start_hour;
    time_struct.min = start_min;
    Market.sessionStartTime = StructToTime(time_struct);
    
    time_struct.hour = end_hour;
    time_struct.min = end_min;
    Market.sessionEndTime = StructToTime(time_struct);
    
    // Validate session times for NAS100
    if(!ValidateSessionTimes())
    {
        return false;
    }
    
    PrintFormat("NAS100 Session hours initialized: %s - %s (Including pre-market)", 
                SessionStart, SessionEnd);
    return true;
}

//+------------------------------------------------------------------+
//| Initialize risk parameters                                         |
//+------------------------------------------------------------------+
bool InitializeRiskParameters()
{
    // Initialize risk state
    Risk.currentRisk = RiskPercent;
    Risk.maxPositionSize = 0;
    Risk.isHighVolatility = false;
    Risk.isExcessiveSpread = false;
    
    // Calculate initial margin levels
    Risk.availableMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
    Risk.marginLevel = AccountInfoDouble(ACCOUNT_MARGIN_LEVEL);
    Risk.equityPercent = AccountInfoDouble(ACCOUNT_EQUITY) / Market.initialBalance * 100;
    
    // Validate risk parameters
    if(!ValidateRiskParameters())
    {
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Initialize trade history                                           |
//+------------------------------------------------------------------+
void InitializeTradeHistory()
{
    ArrayResize(tradeHistory, WinRateCalculationPeriod);
    for(int i = 0; i < WinRateCalculationPeriod; i++)
    {
        tradeHistory[i].openTime = 0;
        tradeHistory[i].isWin = false;
        tradeHistory[i].profit = 0;
        tradeHistory[i].lots = 0;
        tradeHistory[i].entryPrice = 0;
        tradeHistory[i].exitPrice = 0;
        tradeHistory[i].stopLoss = 0;
        tradeHistory[i].takeProfit = 0;
        tradeHistory[i].type = ORDER_TYPE_BUY;
        tradeHistory[i].comment = "";
    }
}

//+------------------------------------------------------------------+
//| Reset session state                                               |
//+------------------------------------------------------------------+
void ResetSessionState()
{
    Session.isTrading = false;
    Session.isPreMarket = false;
    Session.currentSpread = 0;
    Session.currentATR = 0;
    Session.volatilityRatio = 0;
    Session.marketDirection = 0;
    Session.lastUpdate = 0;
}

//+------------------------------------------------------------------+
//| Validate symbol parameters                                         |
//+------------------------------------------------------------------+
bool ValidateSymbolParameters()
{
    // Check symbol digits
    if(SymbolInfoInteger(_Symbol, SYMBOL_DIGITS) != 2)
    {
        Print("Warning: Unexpected decimal places for NAS100");
    }
    
    // Validate minimum stops level
    double minStops = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * Market.point;
    if(minStops > MaxStopLevel * Market.point)
    {
        PrintFormat("Error: Minimum stops level (%.1f) exceeds maximum stop level (%.1f)",
                   minStops/Market.point, MaxStopLevel);
        return false;
    }
    
    // Validate contract specifications
    double contractSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
    double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    
    if(contractSize <= 0 || tickValue <= 0)
    {
        Print("Error: Invalid contract specifications");
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Validate session times                                            |
//+------------------------------------------------------------------+
bool ValidateSessionTimes()
{
    // Ensure session start is before end
    if(Market.sessionEndTime <= Market.sessionStartTime)
    {
        Print("Error: Session end time must be after start time");
        return false;
    }
    
    // Validate pre-market session
    MqlDateTime start_struct;
    TimeToStruct(Market.sessionStartTime, start_struct);
    
    if(TradePreMarket && (start_struct.hour < 13 || 
       (start_struct.hour == 13 && start_struct.min < 0)))
    {
        Print("Error: Pre-market session must start at or after 13:00");
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Validate risk parameters                                          |
//+------------------------------------------------------------------+
bool ValidateRiskParameters()
{
    // Validate risk percent
    if(RiskPercent <= 0 || RiskPercent > 2.0)
    {
        Print("Error: Risk percent must be between 0 and 2%");
        return false;
    }
    
    // Validate drawdown limits
    if(MaxDailyDD <= 0 || MaxWeeklyDD <= 0)
    {
        Print("Error: Invalid drawdown limits");
        return false;
    }
    
    // Validate margin usage
    if(MaxMarginPercent <= 0 || MaxMarginPercent > 50)
    {
        Print("Error: Invalid margin usage percentage");
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                   |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
    EventKillTimer();
    
    // Release all indicator handles
    IndicatorRelease(Handles.fast_ema_m5);
    IndicatorRelease(Handles.slow_ema_m5);
    IndicatorRelease(Handles.fast_ema_m15);
    IndicatorRelease(Handles.slow_ema_m15);
    IndicatorRelease(Handles.fast_ema_h1);
    IndicatorRelease(Handles.slow_ema_h1);
    IndicatorRelease(Handles.atr);
    
    // Reset market variables
    ResetSessionState();
    
    // Log final statistics
    LogFinalStatistics();
}

//+------------------------------------------------------------------+
//| Check for new bar                                                  |
//+------------------------------------------------------------------+
bool isNewBar()
{
    static datetime lastBar;
    datetime currentBar = iTime(_Symbol, PERIOD_M5, 0);
    
    if(lastBar != currentBar)
    {
        lastBar = currentBar;
        return true;
    }
    return false;
}

//+------------------------------------------------------------------+
//| Trading session check                                              |
//+------------------------------------------------------------------+
bool isInSession()
{
    static datetime lastSessionCheck = 0;
    static bool lastSessionStatus = false;
    
    datetime current_time = TimeCurrent();
    
    // Only check every minute to reduce processing
    if(current_time - lastSessionCheck < 60) 
        return lastSessionStatus;
        
    lastSessionCheck = current_time;
    
    MqlDateTime time_struct;
    TimeToStruct(current_time, time_struct);
    
    // Weekend check
    if(time_struct.day_of_week == 0 || time_struct.day_of_week == 6)
    {
        lastSessionStatus = false;
        return false;
    }
    
    // Check day of week filters
    if(MondayFilter && time_struct.day_of_week == 1)
    {
        lastSessionStatus = false;
        return false;
    }
    if(FridayFilter && time_struct.day_of_week == 5)
    {
        lastSessionStatus = false;
        return false;
    }
    
    // Define NAS100 sessions
    bool isPreMarket = (time_struct.hour == 13) || 
                      (time_struct.hour == 14 && time_struct.min < 30);
    bool isMainSession = (time_struct.hour == 14 && time_struct.min >= 30) ||
                        (time_struct.hour > 14 && time_struct.hour < 20) ||
                        (time_struct.hour == 20 && time_struct.min <= 30);
    
    // Update session state
    Session.isPreMarket = isPreMarket;
    Session.isTrading = TradePreMarket ? (isPreMarket || isMainSession) : isMainSession;
    
    lastSessionStatus = Session.isTrading;
    return lastSessionStatus;
}

//+------------------------------------------------------------------+
//| Check spread conditions                                            |
//+------------------------------------------------------------------+
bool isSpreadOK()
{
    Session.currentSpread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) * Market.point;
    double maxAllowedSpread = Session.isPreMarket ? 
                             MaxSpreadPoints * 1.5 : MaxSpreadPoints;
    
    if(Session.currentSpread > maxAllowedSpread * Market.point)
    {
        PrintFormat("Spread too high: %.1f > %.1f points", 
                   Session.currentSpread/Market.point, maxAllowedSpread);
        Session.isTrading = false;
        Risk.isExcessiveSpread = true;
        return false;
    }
    
    Risk.isExcessiveSpread = false;
    return true;
}

//+------------------------------------------------------------------+
//| Check NAS100 specific volatility                                  |
//+------------------------------------------------------------------+
bool isVolatilityAcceptable()
{
    if(!UseVolatilityFilter) return true;
    
    double atr[];
    ArraySetAsSeries(atr, true);
    
    // Get more ATR values for better volatility analysis
    if(CopyBuffer(Handles.atr, 0, 0, ATR_Period * 2, atr) <= 0)
    {
        PrintFormat("%s: Failed to copy ATR data. Error: %d", __FUNCTION__, GetLastError());
        return false;
    }
    
    // Store current ATR
    Session.currentATR = atr[0];
    
    // Calculate volatility ratio
    double historicalATR = 0;
    for(int i = 1; i < ATR_Period; i++)
    {
        historicalATR += atr[i];
    }
    historicalATR /= (ATR_Period - 1);
    
    Session.volatilityRatio = Session.currentATR / historicalATR;
    
    // Get volatility threshold based on session
    double maxATRThreshold = Session.isPreMarket ? PreMarketVolThreshold : VolatilityThreshold;
    
    // Check if volatility is too high
    if(Session.currentATR > maxATRThreshold)
    {
        PrintFormat("NAS100 volatility too high: %.1f points (Max: %.1f)", 
                   Session.currentATR, maxATRThreshold);
        Risk.isHighVolatility = true;
        return false;
    }
    
    // Check if volatility is too low
    if(Session.currentATR < maxATRThreshold * 0.2)
    {
        PrintFormat("NAS100 volatility too low: %.1f points", Session.currentATR);
        return false;
    }
    
    Risk.isHighVolatility = false;
    return true;
}

//+------------------------------------------------------------------+
//| Get trend direction with enhanced validation                       |
//+------------------------------------------------------------------+
int getTrendDirection(int fast_handle, int slow_handle, ENUM_TIMEFRAMES timeframe)
{
    double fast_ema[], slow_ema[];
    ArraySetAsSeries(fast_ema, true);
    ArraySetAsSeries(slow_ema, true);
    
    // Get more data points for better trend confirmation
    const int REQUIRED_BARS = 5;
    
    if(CopyBuffer(fast_handle, 0, 0, REQUIRED_BARS, fast_ema) <= 0)
    {
        PrintFormat("%s: Failed to copy fast EMA data. Error: %d", __FUNCTION__, GetLastError());
        return 0;
    }
    
    if(CopyBuffer(slow_handle, 0, 0, REQUIRED_BARS, slow_ema) <= 0)
    {
        PrintFormat("%s: Failed to copy slow EMA data. Error: %d", __FUNCTION__, GetLastError());
        return 0;
    }
    
    // Enhanced trend detection
    int trendStrength = 0;
    double minTrendMovement = Session.isPreMarket ? 15 * Market.point : 10 * Market.point;
    
    for(int i = 0; i < REQUIRED_BARS-1; i++)
    {
        if(fast_ema[i] > slow_ema[i] && fast_ema[i] - slow_ema[i] > minTrendMovement)
            trendStrength++;
        else if(fast_ema[i] < slow_ema[i] && slow_ema[i] - fast_ema[i] > minTrendMovement)
            trendStrength--;
    }
    
    // Calculate EMA slopes
    double fastSlope = (fast_ema[0] - fast_ema[1]) / Market.point;
    double slowSlope = (slow_ema[0] - slow_ema[1]) / Market.point;
    
    // Calculate acceleration
    double fastAcceleration = ((fast_ema[0] - fast_ema[1]) - (fast_ema[1] - fast_ema[2])) / Market.point;
    
    // Log trend analysis
    PrintFormat("NAS100 Trend Analysis - TF: %s, Strength: %d, FastSlope: %.1f, Accel: %.1f",
                EnumToString(timeframe), trendStrength, fastSlope, fastAcceleration);
    
    // Enhanced trend confirmation for NAS100
    if(trendStrength >= 3 && fastSlope > 0 && slowSlope > 0 && fastAcceleration >= 0)
        return 1;    // Strong uptrend
    if(trendStrength <= -3 && fastSlope < 0 && slowSlope < 0 && fastAcceleration <= 0)
        return -1;   // Strong downtrend
    return 0;        // No clear trend
}

//+------------------------------------------------------------------+
//| Check entry signals with enhanced validation                       |
//+------------------------------------------------------------------+
bool checkEntrySignals(int trend)
{
    if(trend == 0)
    {
        PrintFormat("%s: No clear trend detected", __FUNCTION__);
        return false;
    }
    
    double fast_ema_m5[], slow_ema_m5[];
    ArraySetAsSeries(fast_ema_m5, true);
    ArraySetAsSeries(slow_ema_m5, true);
    
    // Get more data points for signal confirmation
    const int REQUIRED_BARS = 5;
    
    if(CopyBuffer(Handles.fast_ema_m5, 0, 0, REQUIRED_BARS, fast_ema_m5) <= 0 ||
       CopyBuffer(Handles.slow_ema_m5, 0, 0, REQUIRED_BARS, slow_ema_m5) <= 0)
    {
        PrintFormat("%s: Failed to copy EMA data. Error: %d", __FUNCTION__, GetLastError());
        return false;
    }
    
    // Enhanced signal detection
    bool crossOver = false;
    bool crossUnder = false;
    
    // Confirm the crossover with multiple bars
    if(fast_ema_m5[1] <= slow_ema_m5[1] && fast_ema_m5[0] > slow_ema_m5[0])
    {
        crossOver = ValidateCrossover(fast_ema_m5, slow_ema_m5, true);
    }
    else if(fast_ema_m5[1] >= slow_ema_m5[1] && fast_ema_m5[0] < slow_ema_m5[0])
    {
        crossUnder = ValidateCrossover(fast_ema_m5, slow_ema_m5, false);
    }
    
    // Add momentum confirmation
    double momentum = (fast_ema_m5[0] - fast_ema_m5[REQUIRED_BARS-1]) / Session.currentATR;
    bool momentumAligned = (trend == 1 && momentum > 0.5) || (trend == -1 && momentum < -0.5);
    
    // Log signal analysis
    PrintFormat("Signal Analysis - Trend: %d, Cross: %s, Under: %s, Momentum: %.2f",
                trend, crossOver ? "true" : "false", crossUnder ? "true" : "false", momentum);
    
    return ((trend == 1 && crossOver) || (trend == -1 && crossUnder)) && momentumAligned;
}

//+------------------------------------------------------------------+
//| Validate crossover signals                                         |
//+------------------------------------------------------------------+
bool ValidateCrossover(const double& fast[], const double& slow[], bool isOver)
{
    // Calculate momentum of the crossover
    double momentum = MathAbs(fast[0] - slow[0]) / Market.point;
    
    // Calculate average momentum over previous bars
    double avgMomentum = 0;
    for(int i = 1; i < ArraySize(fast); i++)
    {
        avgMomentum += MathAbs(fast[i] - slow[i]);
    }
    avgMomentum /= (ArraySize(fast) - 1);
    avgMomentum /= Market.point;
    
    // Adjust minimum momentum based on session
    double minMomentumRatio = Session.isPreMarket ? 0.6 : 0.5;
    
    // Check if momentum is strong enough
    if(momentum < avgMomentum * minMomentumRatio)
    {
        PrintFormat("Weak crossover momentum: %.2f < %.2f", momentum, avgMomentum * minMomentumRatio);
        return false;
    }
    
    // Check for false signals
    if(isOver)
    {
        for(int i = 2; i < ArraySize(fast); i++)
        {
            if(fast[i] > slow[i])
            {
                PrintFormat("Recent previous crossover detected, avoiding false signal");
                return false;
            }
        }
    }
    else
    {
        for(int i = 2; i < ArraySize(fast); i++)
        {
            if(fast[i] < slow[i])
            {
                PrintFormat("Recent previous crossover detected, avoiding false signal");
                return false;
            }
        }
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Check market conditions                                            |
//+------------------------------------------------------------------+
bool areMarketConditionsValid()
{
    // Check trading session
    if(!isInSession())
    {
        Print("Outside trading session");
        return false;
    }
    
    // Check spread
    if(!isSpreadOK())
    {
        Print("Spread conditions not met");
        return false;
    }
    
    // Check volatility
    if(!isVolatilityAcceptable())
    {
        Print("Volatility conditions not met");
        return false;
    }
    
    // Check for news times
    if(isNewsTime())
    {
        Print("During news time");
        return false;
    }
    
    // Check gap conditions
    if(!checkGapConditions())
    {
        Print("Gap conditions not met");
        return false;
    }
    
    // Adjust risk parameters based on conditions
    adjustRiskParameters();
    
    return true;
}

//+------------------------------------------------------------------+
//| Check for major market gaps                                        |
//+------------------------------------------------------------------+
bool checkGapConditions()
{
    if(!Session.isPreMarket)
        return true;
        
    double currentPrice = SymbolInfoDouble(_Symbol, SYMBOL_BID);
    double previousClose = iClose(_Symbol, PERIOD_D1, 1);
    
    // Calculate gap size
    double gapSize = MathAbs(currentPrice - previousClose);
    
    // Log gap information
    if(gapSize > 50 * Market.point)
    {
        PrintFormat("Large pre-market gap detected: %.1f points", gapSize/Market.point);
        
        // Adjust risk for large gaps
        if(gapSize > 100 * Market.point)
        {
            Risk.currentRisk = Market.baseRiskPercent * 0.5;
            return false;
        }
        else
        {
            Risk.currentRisk = Market.baseRiskPercent * 0.75;
        }
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Adjust risk parameters based on market conditions                  |
//+------------------------------------------------------------------+
void adjustRiskParameters()
{
    // Reset to base values
    Risk.currentRisk = Market.baseRiskPercent;
    
    // Adjust for session
    if(Session.isPreMarket)
    {
        Risk.currentRisk *= 0.5;
    }
    
    // Adjust for volatility
    if(Session.volatilityRatio > MaxATRMultiplier)
    {
        Risk.currentRisk *= 0.75;
    }
    
    // Adjust for spread
    if(Session.currentSpread > Market.baseSpreadPoints * Market.point)
    {
        Risk.currentRisk *= 0.8;
    }
    
    // Adjust for recent performance
    double winRate = calculateWinRate();
    if(winRate < MinWinRate)
    {
        Risk.currentRisk *= 0.8;
    }
    
    // Calculate maximum position size
    Risk.maxPositionSize = calculateMaxPositionSize();
}

//+------------------------------------------------------------------+
//| Calculate current win rate                                         |
//+------------------------------------------------------------------+
double calculateWinRate()
{
    if(Market.totalTrades == 0)
        return 100.0;
        
    int recentWins = 0;
    int countedTrades = 0;
    
    for(int i = 0; i < WinRateCalculationPeriod && i < Market.totalTrades; i++)
    {
        if(tradeHistory[i].isWin)
            recentWins++;
        countedTrades++;
    }
    
    return countedTrades > 0 ? (double)recentWins/countedTrades * 100 : 100.0;
}

//+------------------------------------------------------------------+
//| Calculate maximum allowable position size                          |
//+------------------------------------------------------------------+
double calculateMaxPositionSize()
{
    double accountEquity = AccountInfoDouble(ACCOUNT_EQUITY);
    double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    double maxRiskAmount = accountEquity * (Risk.currentRisk/100);
    
    // Calculate base position size
    double maxPositionSize = maxRiskAmount / (VolatilityThreshold * tickValue);
    
    // Apply additional limits
    double maxLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
    double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
    double lotStep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
    
    // Normalize position size
    maxPositionSize = NormalizeDouble(MathFloor(maxPositionSize/lotStep) * lotStep, 2);
    maxPositionSize = MathMax(minLot, MathMin(maxLot, maxPositionSize));
    
    return maxPositionSize;
}

//+------------------------------------------------------------------+
//| Verify position size with enhanced NAS100 checks                   |
//+------------------------------------------------------------------+
bool verifyPositionSize(double lots, double price)
{
    if(lots <= 0)
    {
        PrintFormat("%s: Invalid position size %.2f", __FUNCTION__, lots);
        return false;
    }
    
    // Get account and symbol information
    double accountEquity = AccountInfoDouble(ACCOUNT_EQUITY);
    double freeMargin = AccountInfoDouble(ACCOUNT_MARGIN_FREE);
    double marginRequired = SymbolInfoDouble(_Symbol, SYMBOL_MARGIN_INITIAL);
    double contractSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
    
    // Calculate required margin with extra safety buffer for NAS100
    double marginBuffer = Session.isPreMarket ? 1.5 : 1.2;  // Higher buffer in pre-market
    double requiredMargin = marginRequired * lots * contractSize * price * marginBuffer;
    
    // Enhanced margin checks
    if(requiredMargin > freeMargin)
    {
        PrintFormat("Margin Check Failed - Required: %.2f, Free: %.2f, Usage: %.1f%%",
                   requiredMargin, freeMargin, (requiredMargin/freeMargin) * 100);
        return false;
    }
    
    // Check margin percentage against maximum allowed
    double marginPercentage = (requiredMargin / accountEquity) * 100;
    if(marginPercentage > MaxMarginPercent)
    {
        PrintFormat("Margin Percentage Too High - Current: %.1f%%, Maximum: %.1f%%",
                   marginPercentage, MaxMarginPercent);
        return false;
    }
    
    // Additional volatility-based position size check
    double volBasedMaxLots = accountEquity * 0.0001 * (1.0 / Session.volatilityRatio);
    if(lots > volBasedMaxLots)
    {
        PrintFormat("Position size too large for current volatility: %.2f > %.2f",
                   lots, volBasedMaxLots);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Check stuck trades with NAS100 specific monitoring                 |
//+------------------------------------------------------------------+
void CheckStuckTrades()
{
    datetime currentTime = TimeCurrent();
    
    for(int i = PositionsTotal() - 1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(!PositionSelectByTicket(ticket)) continue;
        
        datetime openTime = (datetime)PositionGetInteger(POSITION_TIME);
        int tradeDuration = (int)(currentTime - openTime);
        
        // Get position details
        double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
        double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
        double positionProfit = PositionGetDouble(POSITION_PROFIT);
        
        // Check for trades open too long
        int maxDuration = Session.isPreMarket ? 7200 : 14400; // 2 hours pre-market, 4 hours main
        if(tradeDuration > maxDuration)
        {
            PrintFormat("Warning: Position %d open for %d hours", 
                       ticket, tradeDuration/3600);
            ClosePosition(ticket, "Time limit exceeded");
            continue;
        }
        
        // Check for trades with no movement
        double priceMovement = MathAbs(currentPrice - openPrice);
        if(priceMovement < Session.currentATR * 0.1) // Less than 10% of ATR movement
        {
            if(tradeDuration > 1800) // 30 minutes
            {
                PrintFormat("Warning: Position %d showing minimal movement for %d minutes", 
                          ticket, tradeDuration/60);
                ClosePosition(ticket, "No significant price movement");
                continue;
            }
        }
        
        // Check for trades in loss with increasing volatility
        if(positionProfit < 0 && Session.volatilityRatio > MaxATRMultiplier)
        {
            PrintFormat("Warning: Loss position %d in high volatility", ticket);
            ClosePosition(ticket, "High volatility protection");
            continue;
        }
    }
}

//+------------------------------------------------------------------+
//| Refresh market rates with validation                               |
//+------------------------------------------------------------------+
bool RefreshRates()
{
    // Get fresh market data
    MqlTick last_tick;
    if(!SymbolInfoTick(_Symbol, last_tick))
    {
        Print("Failed to get current tick data");
        return false;
    }
    
    // Validate bid/ask
    if(last_tick.bid <= 0 || last_tick.ask <= 0)
    {
        Print("Invalid bid/ask prices");
        return false;
    }
    
    // Check spread
    double currentSpread = (last_tick.ask - last_tick.bid) / Market.point;
    if(currentSpread > MaxSpreadPoints * 2)
    {
        PrintFormat("Extreme spread detected: %.1f points", currentSpread);
        return false;
    }
    
    // Check data freshness
    if(TimeCurrent() - last_tick.time > 5)
    {
        Print("Market data may be stale");
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Manage existing trades with enhanced monitoring                    |
//+------------------------------------------------------------------+
void ManageExistingTrades()
{
    static datetime lastCheck = 0;
    datetime currentTime = TimeCurrent();
    
    // Update market conditions
    UpdateSessionState();
    
    // Quick check for extreme conditions
    if(Session.volatilityRatio > MaxATRMultiplier * 1.5 || 
       Session.currentSpread > MaxSpreadPoints * 2)
    {
        CloseAllPositions("Extreme market conditions");
        return;
    }
    
    // Standard management tasks
    monitorTrades();
    checkAndUpdateStops();
    manageDynamicTP();
    
    // Additional checks every minute
    if(currentTime - lastCheck >= 60)
    {
        CheckStuckTrades();
        checkForExit();
        lastCheck = currentTime;
    }
    
    // Session transition check
    static bool wasPreMarket = false;
    if(wasPreMarket != Session.isPreMarket)
    {
        AdjustPositionsForSessionChange();
        wasPreMarket = Session.isPreMarket;
    }
}

//+------------------------------------------------------------------+
//| Adjust positions for session change                                |
//+------------------------------------------------------------------+
void AdjustPositionsForSessionChange()
{
    // If transitioning to main session, adjust stops to be less conservative
    if(!Session.isPreMarket)
    {
        for(int i = PositionsTotal() - 1; i >= 0; i--)
        {
            ulong ticket = PositionGetTicket(i);
            if(!PositionSelectByTicket(ticket)) continue;
            
            double currentSL = PositionGetDouble(POSITION_SL);
            ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
            
            // Recalculate stop loss with main session parameters
            double newSL = calculateATRStopLoss(
                posType == POSITION_TYPE_BUY ? ORDER_TYPE_BUY : ORDER_TYPE_SELL
            );
            
            if(newSL != currentSL)
            {
                ModifyPosition(ticket, newSL, 0);
            }
        }
    }
}

//+------------------------------------------------------------------+
//| Check for news impact times                                        |
//+------------------------------------------------------------------+
bool isNewsTime()
{
    MqlDateTime time_struct;
    TimeToStruct(TimeCurrent(), time_struct);
    
    // Define major news times for NAS100
    int newsHours[] = {8, 10, 12, 14, 15}; // Key US market hours
    
    // Check if current hour is in news hours
    bool isNewsHour = false;
    for(int i = 0; i < ArraySize(newsHours); i++)
    {
        if(time_struct.hour == newsHours[i])
        {
            isNewsHour = true;
            break;
        }
    }
    
    if(!isNewsHour) 
        return false;
    
    // Avoid trading 15 minutes before and after news for NAS100
    return (time_struct.min >= 45 || time_struct.min <= 15);
}

//+------------------------------------------------------------------+
//| Reset daily monitoring values                                      |
//+------------------------------------------------------------------+
void ResetDailyValues()
{
    Market.dailyTradeCount = 0;
    Market.initialBalance = AccountInfoDouble(ACCOUNT_BALANCE);
    Risk.currentRisk = Market.baseRiskPercent;
    Session.currentATR = 0;
    Session.volatilityRatio = 0;
    Risk.isHighVolatility = false;
    Risk.isExcessiveSpread = false;
    
    PrintFormat("Daily values reset - New balance reference: %.2f", 
                Market.initialBalance);
}

//+------------------------------------------------------------------+
//| Check drawdown limits                                              |
//+------------------------------------------------------------------+
bool isWithinDrawdownLimits()
{
    double currentEquity = AccountInfoDouble(ACCOUNT_EQUITY);
    
    // Check daily drawdown
    double dailyDD = (Market.initialBalance - currentEquity) / Market.initialBalance * 100;
    if(dailyDD > MaxDailyDD)
    {
        PrintFormat("Daily drawdown limit reached: %.2f%%", dailyDD);
        return false;
    }
    
    // Check weekly drawdown
    double weeklyDD = (Market.weeklyInitialBalance - currentEquity) / 
                     Market.weeklyInitialBalance * 100;
    if(weeklyDD > MaxWeeklyDD)
    {
        PrintFormat("Weekly drawdown limit reached: %.2f%%", weeklyDD);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Calculate position size for NAS100                                 |
//+------------------------------------------------------------------+
double calculatePositionSize(double slDistance)
{
    if(slDistance <= 0)
    {
        PrintFormat("%s: Invalid stop loss distance %.5f", __FUNCTION__, slDistance);
        return 0;
    }
    
    // Get account information
    double accountEquity = AccountInfoDouble(ACCOUNT_EQUITY);
    double riskAmount = accountEquity * (Risk.currentRisk / 100);
    
    // Calculate value per point for NAS100
    double contractSize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_CONTRACT_SIZE);
    double tickValue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
    double pointValue = tickValue / SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
    
    // Calculate base position size
    double positionSize = riskAmount / (slDistance * pointValue);
    
    // Normalize position size
    double minLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
    double maxLot = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
    double lotStep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
    
    positionSize = NormalizeDouble(MathFloor(positionSize / lotStep) * lotStep, 2);
    positionSize = MathMax(minLot, MathMin(maxLot, positionSize));
    
    // Additional safety limits for NAS100
    if(Session.isPreMarket)
    {
        positionSize = MathMin(positionSize, Risk.maxPositionSize * 0.5);
    }
    else
    {
        positionSize = MathMin(positionSize, Risk.maxPositionSize);
    }
    
    // Final margin check
    if(!verifyMarginRequirements(positionSize))
    {
        return 0;
    }
    
    return positionSize;
}

//+------------------------------------------------------------------+
//| Verify margin requirements                                         |
//+------------------------------------------------------------------+
bool verifyMarginRequirements(double lots)
{
    if(lots <= 0)
        return false;
        
    double margin = AccountInfoDouble(ACCOUNT_MARGIN);
    double equity = AccountInfoDouble(ACCOUNT_EQUITY);
    double marginRequired = SymbolInfoDouble(_Symbol, SYMBOL_MARGIN_INITIAL) * lots;
    
    // Check if we have enough free margin
    if(marginRequired > AccountInfoDouble(ACCOUNT_MARGIN_FREE))
    {
        PrintFormat("Insufficient free margin - Required: %.2f, Available: %.2f",
                   marginRequired, AccountInfoDouble(ACCOUNT_MARGIN_FREE));
        return false;
    }
    
    // Check margin level after potential trade
    double newMarginLevel = equity / (margin + marginRequired) * 100;
    if(newMarginLevel < 200) // Minimum 200% margin level
    {
        PrintFormat("Margin level too low after trade: %.2f%%", newMarginLevel);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Calculate ATR-based stop loss                                      |
//+------------------------------------------------------------------+
double calculateATRStopLoss(ENUM_ORDER_TYPE orderType)
{
    double atr[];
    ArraySetAsSeries(atr, true);
    
    if(CopyBuffer(Handles.atr, 0, 0, 1, atr) <= 0)
    {
        PrintFormat("%s: Failed to get ATR value. Error: %d", __FUNCTION__, GetLastError());
        return 0;
    }
    
    double atrValue = atr[0];
    if(atrValue <= 0)
    {
        Print("Invalid ATR value");
        return 0;
    }
    
    // Get minimum stop level from broker
    double minStopLevel = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * Market.point;
    
    // Calculate stop loss distance based on session
    double atrMultiplier = Session.isPreMarket ? ATR_Multiplier * 1.5 : ATR_Multiplier;
    double stopDistance = MathMax(atrValue * atrMultiplier, minStopLevel * 1.5);
    
    // Apply maximum stop loss limit
    stopDistance = MathMin(stopDistance, MaxStopLevel * Market.point);
    
    // Get current price
    double price = (orderType == ORDER_TYPE_BUY) ? 
        SymbolInfoDouble(_Symbol, SYMBOL_ASK) : 
        SymbolInfoDouble(_Symbol, SYMBOL_BID);
    
    // Calculate stop loss price
    double stopLoss = (orderType == ORDER_TYPE_BUY) ? 
        price - stopDistance : 
        price + stopDistance;
    
    return NormalizeDouble(stopLoss, _Digits);
}

//+------------------------------------------------------------------+
//| Calculate take profit levels                                       |
//+------------------------------------------------------------------+
double calculateTakeProfit(ENUM_ORDER_TYPE orderType, double entryPrice, double stopLoss)
{
    double stopDistance = MathAbs(entryPrice - stopLoss);
    double tpDistance = stopDistance * 1.5; // Risk:Reward 1:1.5
    
    // Adjust based on volatility
    if(Session.volatilityRatio > 1.5)
    {
        tpDistance = stopDistance * 1.2; // More conservative in high volatility
    }
    
    // Calculate take profit price
    double takeProfit = (orderType == ORDER_TYPE_BUY) ? 
        entryPrice + tpDistance : 
        entryPrice - tpDistance;
    
    return NormalizeDouble(takeProfit, _Digits);
}

//+------------------------------------------------------------------+
//| Execute trade with comprehensive error handling                    |
//+------------------------------------------------------------------+
bool ExecuteTrade(ENUM_ORDER_TYPE orderType, double lots, double price, double sl, double tp)
{
    MqlTradeRequest request = {};
    MqlTradeResult result = {};
    
    // Fill trade request
    request.action = TRADE_ACTION_DEAL;
    request.symbol = _Symbol;
    request.volume = lots;
    request.type = orderType;
    request.price = price;
    request.sl = sl;
    request.tp = tp;
    request.deviation = 10;
    request.magic = 123456;
    request.comment = Session.isPreMarket ? "NAS100 Pre-Market" : "NAS100 Main";
    request.type_filling = ORDER_FILLING_FOK;
    
    // Execute trade with retry mechanism
    for(int attempt = 0; attempt < 3; attempt++)
    {
        if(!OrderSend(request, result))
        {
            int error = GetLastError();
            PrintFormat("OrderSend failed with error #%d on attempt %d", error, attempt + 1);
            
            if(error == ERR_REQUOTE || error == ERR_INVALID_PRICE)
            {
                // Update price and retry
                if(!RefreshRates()) continue;
                request.price = (orderType == ORDER_TYPE_BUY) ? 
                    SymbolInfoDouble(_Symbol, SYMBOL_ASK) : 
                    SymbolInfoDouble(_Symbol, SYMBOL_BID);
            }
            else if(error == ERR_INVALID_STOPS)
            {
                // Adjust stop loss and try again
                double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * Market.point;
                if(orderType == ORDER_TYPE_BUY)
                    request.sl = request.price - minStop;
                else
                    request.sl = request.price + minStop;
            }
            else
            {
                Sleep(1000); // Wait before retry
                continue;
            }
        }
        else
        {
            break;
        }
    }
    
    if(result.retcode != TRADE_RETCODE_DONE)
    {
        PrintFormat("Trade execution failed with retcode #%d", result.retcode);
        return false;
    }
    
    // Log successful trade
    LogTradeExecution(result.order, orderType, lots, price, sl, tp);
    
    return true;
}

//+------------------------------------------------------------------+
//| Log trade execution details                                        |
//+------------------------------------------------------------------+
void LogTradeExecution(ulong ticket, ENUM_ORDER_TYPE type, double lots, 
                      double price, double sl, double tp)
{
    PrintFormat("NAS100 Trade Executed - Ticket: %d", ticket);
    PrintFormat("Type: %s, Lots: %.2f", 
                type == ORDER_TYPE_BUY ? "Buy" : "Sell", lots);
    PrintFormat("Entry: %.2f, SL: %.2f, TP: %.2f", price, sl, tp);
    PrintFormat("Session: %s", Session.isPreMarket ? "Pre-Market" : "Main");
    PrintFormat("Spread: %.1f, ATR: %.1f", 
                Session.currentSpread/Market.point, Session.currentATR);
    
    // Update trade history
    updateTradeHistory(ticket, type, lots, price, sl, tp);
}

//+------------------------------------------------------------------+
//| Update trade history                                               |
//+------------------------------------------------------------------+
void updateTradeHistory(ulong ticket, ENUM_ORDER_TYPE type, double lots, 
                       double price, double sl, double tp)
{
    // Shift array to make room for new trade
    for(int i = ArraySize(tradeHistory) - 1; i > 0; i--)
    {
        tradeHistory[i] = tradeHistory[i-1];
    }
    
    // Add new trade
    tradeHistory[0].openTime = TimeCurrent();
    tradeHistory[0].type = type;
    tradeHistory[0].lots = lots;
    tradeHistory[0].entryPrice = price;
    tradeHistory[0].stopLoss = sl;
    tradeHistory[0].takeProfit = tp;
    tradeHistory[0].isWin = false; // Will be updated on close
    tradeHistory[0].comment = Session.isPreMarket ? "Pre-Market" : "Main";
    
    // Update trade counters
    Market.totalTrades++;
    Market.dailyTradeCount++;
}

//+------------------------------------------------------------------+
//| Main trade function                                                |
//+------------------------------------------------------------------+
bool trade(ENUM_ORDER_TYPE orderType)
{
    // Verify market conditions
    if(!areMarketConditionsValid())
    {
        return false;
    }
    
    // Get current price
    double price = (orderType == ORDER_TYPE_BUY) ? 
        SymbolInfoDouble(_Symbol, SYMBOL_ASK) : 
        SymbolInfoDouble(_Symbol, SYMBOL_BID);
    
    // Calculate stop loss
    double sl = calculateATRStopLoss(orderType);
    if(sl == 0)
    {
        PrintFormat("%s: Failed to calculate valid stop loss", __FUNCTION__);
        return false;
    }
    
    // Calculate take profit
    double tp = calculateTakeProfit(orderType, price, sl);
    
    // Calculate position size
    double slDistance = MathAbs(price - sl);
    double lots = calculatePositionSize(slDistance);
    if(lots == 0)
    {
        PrintFormat("%s: Position size calculation failed", __FUNCTION__);
        return false;
    }
    
    // Execute trade
    if(!ExecuteTrade(orderType, lots, price, sl, tp))
    {
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Monitor active trades                                              |
//+------------------------------------------------------------------+
void monitorTrades()
{
    static datetime lastCheck = 0;
    datetime currentTime = TimeCurrent();
    
    // Check every 15 seconds for NAS100
    if(currentTime - lastCheck < 15) return;
    lastCheck = currentTime;
    
    double totalProfit = 0;
    int openPositions = 0;
    
    for(int i = PositionsTotal() - 1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(!PositionSelectByTicket(ticket))
        {
            PrintFormat("Failed to select position %d. Error: %d", ticket, GetLastError());
            continue;
        }
        
        // Check if we need to close due to market conditions
        if(!CheckContinuePosition(ticket))
        {
            ClosePosition(ticket, "Market condition change");
            continue;
        }
        
        // Update trailing stop
        UpdateTrailingStop(ticket);
        
        // Check partial close conditions
        CheckPartialClose(ticket);
        
        totalProfit += PositionGetDouble(POSITION_PROFIT);
        openPositions++;
    }
    
    // Log trading statistics
    if(openPositions > 0)
    {
        LogTradingStatistics(openPositions, totalProfit);
    }
}

//+------------------------------------------------------------------+
//| Check if position should continue                                  |
//+------------------------------------------------------------------+
bool CheckContinuePosition(ulong ticket)
{
    if(!PositionSelectByTicket(ticket))
        return false;
        
    datetime openTime = (datetime)PositionGetInteger(POSITION_TIME);
    double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
    ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
    
    // Check max trade duration
    int maxDuration = Session.isPreMarket ? 7200 : 14400; // 2 hours pre-market, 4 hours main
    if(TimeCurrent() - openTime > maxDuration)
    {
        PrintFormat("Position %d exceeded maximum duration", ticket);
        return false;
    }
    
    // Check for trend reversal
    int currentTrend = getTrendDirection(Handles.fast_ema_h1, Handles.slow_ema_h1, PERIOD_H1);
    if((posType == POSITION_TYPE_BUY && currentTrend == -1) ||
       (posType == POSITION_TYPE_SELL && currentTrend == 1))
    {
        PrintFormat("Trend reversal detected for position %d", ticket);
        return false;
    }
    
    // Check for extreme volatility
    if(Session.volatilityRatio > MaxATRMultiplier * 1.5)
    {
        PrintFormat("Extreme volatility detected: %.2f", Session.volatilityRatio);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Update trailing stop                                               |
//+------------------------------------------------------------------+
void UpdateTrailingStop(ulong ticket)
{
    if(!PositionSelectByTicket(ticket))
        return;
        
    double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
    double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
    double currentSL = PositionGetDouble(POSITION_SL);
    ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
    
    // Calculate points in profit
    double profitPoints = (posType == POSITION_TYPE_BUY) ?
                         currentPrice - openPrice :
                         openPrice - currentPrice;
    
    // Only trail if in sufficient profit
    if(profitPoints < ActivationPoints)
        return;
        
    // Calculate new stop loss based on ATR
    double atr[];
    ArraySetAsSeries(atr, true);
    if(CopyBuffer(Handles.atr, 0, 0, 1, atr) <= 0)
        return;
        
    // Adjust trailing distance based on session and profit
    double trailMultiplier = Session.isPreMarket ? 2.0 : 1.5;
    if(profitPoints > ActivationPoints * 2)
        trailMultiplier *= 0.8; // Tighter trail when in good profit
        
    double trailDistance = atr[0] * trailMultiplier;
    
    // Calculate new stop loss
    double newSL;
    if(posType == POSITION_TYPE_BUY)
    {
        newSL = currentPrice - trailDistance;
        if(newSL > currentSL)
        {
            ModifyPosition(ticket, newSL, 0);
        }
    }
    else
    {
        newSL = currentPrice + trailDistance;
        if(newSL < currentSL || currentSL == 0)
        {
            ModifyPosition(ticket, newSL, 0);
        }
    }
}

//+------------------------------------------------------------------+
//| News event handling for NAS100                                     |
//+------------------------------------------------------------------+
bool isHighImpactNewsTime()
{
    MqlDateTime time_struct;
    TimeToStruct(TimeCurrent(), time_struct);
    
    // Define critical news times for NAS100 (Eastern Time)
    int highImpactHours[] = {
        8,  // Market Open
        10, // Key Economic Data
        14, // Fed Speeches
        16  // Market Close
    };
    
    // Check current hour
    for(int i = 0; i < ArraySize(highImpactHours); i++)
    {
        if(time_struct.hour == highImpactHours[i])
        {
            // Avoid trading 15 minutes before and after news
            if(time_struct.min >= 45 || time_struct.min <= 15)
            {
                PrintFormat("High impact news period - Hour: %d, Minute: %d",
                           time_struct.hour, time_struct.min);
                return true;
            }
        }
    }
    
    return false;
}

//+------------------------------------------------------------------+
//| Gap analysis and monitoring                                        |
//+------------------------------------------------------------------+
struct GapInfo {
    bool exists;
    double size;
    bool isUpGap;
    datetime gapTime;
};

GapInfo AnalyzeGaps()
{
    GapInfo gap = {false, 0, false, 0};
    
    // Only check for gaps during pre-market
    if(!Session.isPreMarket)
        return gap;
        
    double previousClose = iClose(_Symbol, PERIOD_D1, 1);
    double currentPrice = SymbolInfoDouble(_Symbol, SYMBOL_BID);
    
    // Calculate gap size
    double gapSize = MathAbs(currentPrice - previousClose);
    
    // Check if gap is significant (more than 20 points)
    if(gapSize > 20 * Market.point)
    {
        gap.exists = true;
        gap.size = gapSize;
        gap.isUpGap = currentPrice > previousClose;
        gap.gapTime = TimeCurrent();
        
        PrintFormat("NAS100 Gap Detected - Size: %.1f points, Direction: %s",
                   gapSize/Market.point,
                   gap.isUpGap ? "Up" : "Down");
    }
    
    return gap;
}

//+------------------------------------------------------------------+
//| Advanced drawdown tracking                                         |
//+------------------------------------------------------------------+
struct DrawdownInfo {
    double currentDD;
    double maxDD;
    double recoveryLevel;
    datetime maxDDTime;
    bool isAcceptable;
};

DrawdownInfo TrackDrawdown()
{
    DrawdownInfo dd;
    
    // Calculate current drawdown
    double equity = AccountInfoDouble(ACCOUNT_EQUITY);
    double peak = MathMax(Market.initialBalance, equity);
    
    dd.currentDD = (peak - equity) / peak * 100;
    
    // Update max drawdown if needed
    static double globalMaxDD = 0;
    static datetime maxDDTime = 0;
    
    if(dd.currentDD > globalMaxDD)
    {
        globalMaxDD = dd.currentDD;
        maxDDTime = TimeCurrent();
    }
    
    dd.maxDD = globalMaxDD;
    dd.maxDDTime = maxDDTime;
    
    // Calculate recovery needed
    dd.recoveryLevel = equity * (1 + dd.currentDD/100);
    
    // Check if drawdown is acceptable
    double maxAllowedDD = Session.isPreMarket ? MaxDailyDD * 0.7 : MaxDailyDD;
    dd.isAcceptable = dd.currentDD <= maxAllowedDD;
    
    // Log drawdown status if significant
    if(dd.currentDD > maxAllowedDD * 0.8)
    {
        PrintFormat("Drawdown Alert - Current: %.2f%%, Max: %.2f%%, Recovery Needed: %.2f",
                   dd.currentDD, dd.maxDD, dd.recoveryLevel - equity);
    }
    
    return dd;
}

//+------------------------------------------------------------------+
//| Recovery mode management                                           |
//+------------------------------------------------------------------+
void ManageRecoveryMode()
{
    static bool inRecoveryMode = false;
    DrawdownInfo dd = TrackDrawdown();
    
    // Check if we should enter recovery mode
    if(dd.currentDD > MaxDailyDD * 0.8)
    {
        if(!inRecoveryMode)
        {
            PrintFormat("Entering recovery mode - Drawdown: %.2f%%", dd.currentDD);
            inRecoveryMode = true;
            
            // Adjust risk parameters
            Risk.currentRisk *= 0.5;
            MaxDailyTrades = MathMax(1, MaxDailyTrades - 1);
        }
    }
    else if(dd.currentDD < MaxDailyDD * 0.5)
    {
        // Exit recovery mode
        if(inRecoveryMode)
        {
            PrintFormat("Exiting recovery mode - Drawdown recovered to: %.2f%%", dd.currentDD);
            inRecoveryMode = false;
            
            // Restore normal risk parameters
            Risk.currentRisk = Market.baseRiskPercent;
            MaxDailyTrades = Market.dailyTradeCount + 2;
        }
    }
    
    // Additional safety checks in recovery mode
    if(inRecoveryMode)
    {
        // Close positions if drawdown continues to worsen
        if(dd.currentDD > MaxDailyDD * 0.9)
        {
            CloseAllPositions("Maximum drawdown protection");
        }
        
        // More conservative trade management
        ActivationPoints *= 1.5;
        PartialClosePct = 0.75;
    }
}

//+------------------------------------------------------------------+
//| Volatility trend analysis                                          |
//+------------------------------------------------------------------+
struct VolatilityTrend {
    bool isIncreasing;
    bool isExcessive;
    double trendStrength;
    string recommendation;
};

VolatilityTrend AnalyzeVolatilityTrend()
{
    VolatilityTrend vt = {false, false, 0, ""};
    
    double atr[];
    ArraySetAsSeries(atr, true);
    
    // Get more ATR values for trend analysis
    if(CopyBuffer(Handles.atr, 0, 0, 10, atr) <= 0)
        return vt;
        
    // Calculate trend
    double shortTermAvg = (atr[0] + atr[1] + atr[2]) / 3;
    double longTermAvg = 0;
    for(int i = 3; i < 10; i++)
        longTermAvg += atr[i];
    longTermAvg /= 7;
    
    // Calculate trend strength
    vt.trendStrength = (shortTermAvg - longTermAvg) / longTermAvg * 100;
    vt.isIncreasing = vt.trendStrength > 5;
    vt.isExcessive = shortTermAvg > Session.currentATR * 1.5;
    
    // Generate recommendation
    if(vt.isExcessive)
    {
        vt.recommendation = "Reduce position sizes";
        Risk.currentRisk *= 0.75;
    }
    else if(vt.isIncreasing)
    {
        vt.recommendation = "Monitor closely";
    }
    else
    {
        vt.recommendation = "Normal trading";
    }
    
    return vt;
}

//+------------------------------------------------------------------+
//| Market depth analysis for NAS100                                   |
//+------------------------------------------------------------------+
bool AnalyzeMarketDepth()
{
    MqlBookInfo book[];
    bool gotBook = MarketBookGet(_Symbol, book);
    if(!gotBook || ArraySize(book) < 10)
        return true;  // Continue if can't get market depth
        
    double buyVolume = 0, sellVolume = 0;
    
    // Calculate volume imbalance
    for(int i = 0; i < ArraySize(book); i++)
    {
        if(book[i].type == BOOK_TYPE_SELL)
            sellVolume += book[i].volume;
        else if(book[i].type == BOOK_TYPE_BUY)
            buyVolume += book[i].volume;
    }
    
    // Check for significant imbalance
    double imbalanceRatio = buyVolume / sellVolume;
    if(imbalanceRatio > 2 || imbalanceRatio < 0.5)
    {
        PrintFormat("Significant order imbalance detected: %.2f", imbalanceRatio);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Check partial close conditions                                     |
//+------------------------------------------------------------------+
void CheckPartialClose(ulong ticket)
{
    if(!PositionSelectByTicket(ticket))
        return;
        
    double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
    double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
    double currentVolume = PositionGetDouble(POSITION_VOLUME);
    ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
    
    // Calculate profit percentage
    double profitPoints = (posType == POSITION_TYPE_BUY) ?
                         currentPrice - openPrice :
                         openPrice - currentPrice;
                         
    // Adjust partial close threshold based on session
    double partialThreshold = Session.isPreMarket ? 
                             ActivationPoints * 0.8 : 
                             ActivationPoints;
    
    if(profitPoints >= partialThreshold)
    {
        // Calculate volume to close
        double closeVolume = currentVolume * PartialClosePct;
        
        // Execute partial close
        if(closeVolume >= SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN))
        {
            ExecutePartialClose(ticket, closeVolume, posType);
        }
    }
}

//+------------------------------------------------------------------+
//| Execute partial position close                                     |
//+------------------------------------------------------------------+
bool ExecutePartialClose(ulong ticket, double volume, ENUM_POSITION_TYPE posType)
{
    MqlTradeRequest request = {};
    MqlTradeResult result = {};
    
    request.action = TRADE_ACTION_DEAL;
    request.position = ticket;
    request.symbol = _Symbol;
    request.volume = volume;
    request.type = posType == POSITION_TYPE_BUY ? ORDER_TYPE_SELL : ORDER_TYPE_BUY;
    request.price = SymbolInfoDouble(_Symbol, posType == POSITION_TYPE_BUY ? SYMBOL_BID : SYMBOL_ASK);
    request.deviation = 10;
    
    for(int attempt = 0; attempt < 3; attempt++)
    {
        if(!OrderSend(request, result))
        {
            int error = GetLastError();
            PrintFormat("Partial close failed with error #%d on attempt %d", error, attempt + 1);
            
            if(error == ERR_REQUOTE || error == ERR_INVALID_PRICE)
            {
                if(!RefreshRates()) continue;
                request.price = SymbolInfoDouble(_Symbol, 
                    posType == POSITION_TYPE_BUY ? SYMBOL_BID : SYMBOL_ASK);
            }
            else
            {
                Sleep(1000);
                continue;
            }
        }
        else
        {
            PrintFormat("Partial close executed - Volume: %.2f", volume);
            return true;
        }
    }
    
    return false;
}

//+------------------------------------------------------------------+
//| Modify position stop loss/take profit                              |
//+------------------------------------------------------------------+
bool ModifyPosition(ulong ticket, double newSL, double newTP)
{
    if(!PositionSelectByTicket(ticket))
        return false;
        
    MqlTradeRequest request = {};
    MqlTradeResult result = {};
    
    request.action = TRADE_ACTION_SLTP;
    request.symbol = _Symbol;
    request.sl = newSL;
    request.tp = newTP;
    request.position = ticket;
    
    for(int attempt = 0; attempt < 3; attempt++)
    {
        if(!OrderSend(request, result))
        {
            int error = GetLastError();
            PrintFormat("Position modification failed with error #%d on attempt %d", 
                       error, attempt + 1);
            
            if(error == ERR_INVALID_STOPS)
            {
                // Validate stop level
                if(!IsValidStopLevel(newSL, 
                    (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE)))
                {
                    return false;
                }
            }
            
            Sleep(1000);
            continue;
        }
        else
        {
            PrintFormat("Position modified - SL: %.2f, TP: %.2f", newSL, newTP);
            return true;
        }
    }
    
    return false;
}

//+------------------------------------------------------------------+
//| Close position                                                     |
//+------------------------------------------------------------------+
bool ClosePosition(ulong ticket, string reason)
{
    if(!PositionSelectByTicket(ticket))
        return false;
        
    MqlTradeRequest request = {};
    MqlTradeResult result = {};
    
    request.action = TRADE_ACTION_DEAL;
    request.position = ticket;
    request.symbol = _Symbol;
    request.volume = PositionGetDouble(POSITION_VOLUME);
    request.type = PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY ? 
                  ORDER_TYPE_SELL : ORDER_TYPE_BUY;
    request.price = SymbolInfoDouble(_Symbol, 
                   request.type == ORDER_TYPE_BUY ? SYMBOL_ASK : SYMBOL_BID);
    request.deviation = 10;
    
    bool success = OrderSend(request, result);
    
    if(success)
    {
        // Update trade history
        double profit = PositionGetDouble(POSITION_PROFIT);
        PrintFormat("Position %d closed - Reason: %s, Profit: %.2f", 
                   ticket, reason, profit);
                   
        UpdateClosedTradeStats(ticket, profit > 0);
    }
    else
    {
        PrintFormat("Failed to close position %d. Error: %d", 
                   ticket, GetLastError());
    }
    
    return success;
}

//+------------------------------------------------------------------+
//| Update statistics for closed trade                                 |
//+------------------------------------------------------------------+
void UpdateClosedTradeStats(ulong ticket, bool isWin)
{
    // Find trade in history
    for(int i = 0; i < ArraySize(tradeHistory); i++)
    {
        if(tradeHistory[i].openTime > 0)
        {
            tradeHistory[i].isWin = isWin;
            if(isWin) Market.winningTrades++;
            break;
        }
    }
    
    // Update win rate calculation
    double winRate = (double)Market.winningTrades / Market.totalTrades * 100;
    PrintFormat("Updated win rate: %.2f%% (%d/%d)", 
                winRate, Market.winningTrades, Market.totalTrades);
}

//+------------------------------------------------------------------+
//| Log trading statistics                                            |
//+------------------------------------------------------------------+
void LogTradingStatistics(int openPositions, double totalProfit)
{
    PrintFormat("\n=== NAS100 Trading Statistics ===");
    PrintFormat("Session: %s", Session.isPreMarket ? "Pre-Market" : "Main");
    PrintFormat("Open Positions: %d", openPositions);
    PrintFormat("Total Current Profit: %.2f", totalProfit);
    PrintFormat("Daily Trades: %d/%d", Market.dailyTradeCount, MaxDailyTrades);
    
    double equity = AccountInfoDouble(ACCOUNT_EQUITY);
    double balance = AccountInfoDouble(ACCOUNT_BALANCE);
    double margin = AccountInfoDouble(ACCOUNT_MARGIN);
    
    PrintFormat("Account Balance: %.2f", balance);
    PrintFormat("Current Equity: %.2f", equity);
    PrintFormat("Daily Return: %.2f%%", 
                ((equity - Market.initialBalance) / Market.initialBalance) * 100);
    
    if(margin > 0)
        PrintFormat("Margin Level: %.2f%%", (equity/margin) * 100);
        
    PrintFormat("Current Spread: %.1f", Session.currentSpread/Market.point);
    PrintFormat("Current ATR: %.1f", Session.currentATR);
    PrintFormat("Volatility Ratio: %.2f", Session.volatilityRatio);
}

//+------------------------------------------------------------------+
//| Validate stop loss level                                          |
//+------------------------------------------------------------------+
bool IsValidStopLevel(double stopLevel, ENUM_POSITION_TYPE posType)
{
    double minDistance = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * Market.point;
    double currentPrice = SymbolInfoDouble(_Symbol, 
        posType == POSITION_TYPE_BUY ? SYMBOL_BID : SYMBOL_ASK);
    
    if(posType == POSITION_TYPE_BUY)
    {
        if(currentPrice - stopLevel < minDistance)
        {
            PrintFormat("Stop loss too close: %.2f < %.2f points", 
                       currentPrice - stopLevel, minDistance);
            return false;
        }
    }
    else
    {
        if(stopLevel - currentPrice < minDistance)
        {
            PrintFormat("Stop loss too close: %.2f < %.2f points", 
                       stopLevel - currentPrice, minDistance);
            return false;
        }
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Handle trade execution errors                                      |
//+------------------------------------------------------------------+
bool HandleTradeError(int error_code, ulong ticket = 0)
{
    switch(error_code)
    {
        case ERR_REQUOTE:
        case ERR_PRICE_CHANGED:
            RefreshRates();
            return true;  // Try again
            
        case ERR_INVALID_STOPS:
            if(ticket > 0)
            {
                AdjustStopLevels(ticket);
                return true;
            }
            break;
            
        case ERR_INVALID_TRADE_VOLUME:
            PrintFormat("Invalid volume - Check margin requirements");
            break;
            
        case ERR_TRADE_NOT_ALLOWED:
            PrintFormat("Trading not allowed - Check expert settings");
            break;
            
        case ERR_MARKET_CLOSED:
            PrintFormat("Market closed");
            Session.isTrading = false;
            break;
            
        case ERR_TRADE_DISABLED:
            PrintFormat("Trading disabled");
            Session.isTrading = false;
            break;
            
        default:
            PrintFormat("Trade error %d occurred", error_code);
            break;
    }
    return false;
}

//+------------------------------------------------------------------+
//| Adjust stop levels for broker requirements                         |
//+------------------------------------------------------------------+
void AdjustStopLevels(ulong ticket)
{
    if(!PositionSelectByTicket(ticket))
        return;
        
    double minStop = SymbolInfoInteger(_Symbol, SYMBOL_TRADE_STOPS_LEVEL) * Market.point;
    double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
    double currentSL = PositionGetDouble(POSITION_SL);
    ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
    
    // Calculate valid stop level
    double newSL;
    if(posType == POSITION_TYPE_BUY)
    {
        newSL = MathMin(currentSL, currentPrice - minStop);
    }
    else
    {
        newSL = MathMax(currentSL, currentPrice + minStop);
    }
    
    // Modify position with new stop
    ModifyPosition(ticket, newSL, PositionGetDouble(POSITION_TP));
}

//+------------------------------------------------------------------+
//| Check order filling conditions                                     |
//+------------------------------------------------------------------+
bool CheckFillConditions(ENUM_ORDER_TYPE orderType, double price, double volume)
{
    // Get current market depth
    MqlBookInfo book[];
    if(!MarketBookGet(_Symbol, book))
        return true;  // Continue if can't get market depth
        
    double availableVolume = 0;
    double weightedPrice = 0;
    
    // Calculate available volume and weighted price
    for(int i = 0; i < ArraySize(book); i++)
    {
        if((orderType == ORDER_TYPE_BUY && book[i].type == BOOK_TYPE_SELL) ||
           (orderType == ORDER_TYPE_SELL && book[i].type == BOOK_TYPE_BUY))
        {
            availableVolume += book[i].volume;
            weightedPrice += book[i].price * book[i].volume;
        }
    }
    
    if(availableVolume > 0)
        weightedPrice /= availableVolume;
    
    // Check if enough liquidity
    if(availableVolume < volume)
    {
        PrintFormat("Warning: Low liquidity - Available: %.2f, Required: %.2f",
                   availableVolume, volume);
        return false;
    }
    
    // Check for significant price deviation
    double priceDeviation = MathAbs(weightedPrice - price) / price * 100;
    if(priceDeviation > 0.1)  // More than 0.1% deviation
    {
        PrintFormat("Warning: Significant price deviation: %.2f%%", priceDeviation);
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Advanced performance tracking                                      |
//+------------------------------------------------------------------+
struct PerformanceMetrics {
    double sharpeRatio;
    double winRate;
    double avgWin;
    double avgLoss;
    double profitFactor;
    double expectancy;
};

//+------------------------------------------------------------------+
//| Calculate performance metrics                                      |
//+------------------------------------------------------------------+
PerformanceMetrics CalculatePerformanceMetrics()
{
    PerformanceMetrics metrics;
    double totalWins = 0, totalLosses = 0;
    double sumWins = 0, sumLosses = 0;
    double returns[];
    
    // Initialize arrays
    ArrayResize(returns, Market.totalTrades);
    
    // Calculate trade metrics
    for(int i = 0; i < Market.totalTrades && i < ArraySize(tradeHistory); i++)
    {
        if(tradeHistory[i].isWin)
        {
            totalWins++;
            sumWins += tradeHistory[i].profit;
        }
        else
        {
            totalLosses++;
            sumLosses += MathAbs(tradeHistory[i].profit);
        }
        
        // Calculate return for Sharpe ratio
        returns[i] = tradeHistory[i].profit / Market.initialBalance * 100;
    }
    
    // Calculate metrics
    metrics.winRate = Market.totalTrades > 0 ? totalWins / Market.totalTrades * 100 : 0;
    metrics.avgWin = totalWins > 0 ? sumWins / totalWins : 0;
    metrics.avgLoss = totalLosses > 0 ? sumLosses / totalLosses : 0;
    metrics.profitFactor = sumLosses > 0 ? sumWins / sumLosses : 0;
    
    // Calculate Sharpe Ratio
    double avgReturn = 0, stdDev = 0;
    for(int i = 0; i < Market.totalTrades; i++)
        avgReturn += returns[i];
    avgReturn /= Market.totalTrades;
    
    for(int i = 0; i < Market.totalTrades; i++)
        stdDev += MathPow(returns[i] - avgReturn, 2);
    stdDev = MathSqrt(stdDev / Market.totalTrades);
    
    metrics.sharpeRatio = stdDev > 0 ? avgReturn / stdDev : 0;
    
    // Calculate expectancy
    metrics.expectancy = (metrics.winRate/100 * metrics.avgWin) - 
                        ((1-metrics.winRate/100) * metrics.avgLoss);
    
    return metrics;
}

//+------------------------------------------------------------------+
//| Log detailed performance statistics                                |
//+------------------------------------------------------------------+
void LogDetailedPerformance()
{
    PerformanceMetrics metrics = CalculatePerformanceMetrics();
    
    PrintFormat("\n=== Detailed NAS100 Performance Analysis ===");
    PrintFormat("Win Rate: %.2f%%", metrics.winRate);
    PrintFormat("Average Win: %.2f", metrics.avgWin);
    PrintFormat("Average Loss: %.2f", metrics.avgLoss);
    PrintFormat("Profit Factor: %.2f", metrics.profitFactor);
    PrintFormat("Sharpe Ratio: %.2f", metrics.sharpeRatio);
    PrintFormat("Expectancy: %.2f", metrics.expectancy);
    
    // Session analysis
    double preMarketWinRate = CalculateSessionWinRate(true);
    double mainSessionWinRate = CalculateSessionWinRate(false);
    PrintFormat("Pre-Market Win Rate: %.2f%%", preMarketWinRate);
    PrintFormat("Main Session Win Rate: %.2f%%", mainSessionWinRate);
    
    // Volatility analysis
    PrintFormat("Current Volatility Ratio: %.2f", Session.volatilityRatio);
    PrintFormat("Average ATR: %.1f", Session.currentATR);
}

//+------------------------------------------------------------------+
//| Calculate session-specific win rate                                |
//+------------------------------------------------------------------+
double CalculateSessionWinRate(bool preMarket)
{
    int totalTrades = 0;
    int winningTrades = 0;
    
    for(int i = 0; i < ArraySize(tradeHistory); i++)
    {
        if(tradeHistory[i].openTime == 0)
            break;
            
        MqlDateTime time;
        TimeToStruct(tradeHistory[i].openTime, time);
        bool isPreMarketTrade = (time.hour == 13) || 
                               (time.hour == 14 && time.min < 30);
        
        if(isPreMarketTrade == preMarket)
        {
            totalTrades++;
            if(tradeHistory[i].isWin)
                winningTrades++;
        }
    }
    
    return totalTrades > 0 ? (double)winningTrades/totalTrades * 100 : 0;
}

//+------------------------------------------------------------------+
//| Check for market reversal                                         |
//+------------------------------------------------------------------+
bool checkMarketReversal()
{
    // Compare current trend with previous trend
    static int lastTrend = 0;
    int currentTrend = getTrendDirection(Handles.fast_ema_h1, Handles.slow_ema_h1, PERIOD_H1);
    
    // Check for volatility spike
    bool volatilitySpike = Session.volatilityRatio > MaxATRMultiplier * 1.5;
    
    // Check for significant price movement
    double close[];
    ArraySetAsSeries(close, true);
    
    if(CopyClose(_Symbol, PERIOD_H1, 0, 5, close) <= 0)
    {
        return false;
    }
    
    // Calculate price range
    double highestHigh = close[ArrayMaximum(close)];
    double lowestLow = close[ArrayMinimum(close)];
    double priceRange = (highestHigh - lowestLow) / Session.currentATR;
    
    // Detect reversal conditions
    bool trendReversal = (lastTrend != 0 && currentTrend != 0 && lastTrend != currentTrend);
    bool significantPriceMove = priceRange > 2.0; // More than 2 ATRs
    
    // Update last trend
    if(currentTrend != 0)
        lastTrend = currentTrend;
    
    // Log reversal conditions
    if(trendReversal || (volatilitySpike && significantPriceMove))
    {
        PrintFormat("Market Reversal Detected - Trend Change: %d, Volatility Spike: %s, Price Range: %.2f ATRs", 
                   currentTrend, volatilitySpike ? "Yes" : "No", priceRange);
        return true;
    }
    
    return false;
}

//+------------------------------------------------------------------+
//| Close all open positions                                          |
//+------------------------------------------------------------------+
void CloseAllPositions(string reason)
{
    PrintFormat("Closing all positions - Reason: %s", reason);
    
    for(int i = PositionsTotal() - 1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(!PositionSelectByTicket(ticket))
            continue;
        
        // Close the position
        ClosePosition(ticket, reason);
    }
}

//+------------------------------------------------------------------+
//| Check for early exit conditions                                   |
//+------------------------------------------------------------------+
void checkForExit()
{
    for(int i = PositionsTotal() - 1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(!PositionSelectByTicket(ticket))
            continue;
        
        double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
        double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
        double currentProfit = PositionGetDouble(POSITION_PROFIT);
        ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
        
        // Calculate profit points
        double profitPoints = (posType == POSITION_TYPE_BUY) ? 
            (currentPrice - openPrice) / Market.point : 
            (openPrice - currentPrice) / Market.point;
        
        // Early exit conditions
        bool earlyExitSignal = false;
        
        // Exit if profit reaches threshold but not hitting full take profit
        if(profitPoints >= MinProfitToClose && 
           profitPoints < (PositionGetDouble(POSITION_TP) - openPrice) / Market.point)
        {
            // Check for trend change
            int currentTrend = getTrendDirection(
                Handles.fast_ema_m15, 
                Handles.slow_ema_m15, 
                PERIOD_M15
            );
            
            if((posType == POSITION_TYPE_BUY && currentTrend == -1) ||
               (posType == POSITION_TYPE_SELL && currentTrend == 1))
            {
                earlyExitSignal = true;
            }
        }
        
        // Exit on extreme volatility
        if(Session.volatilityRatio > MaxATRMultiplier * 1.5)
        {
            earlyExitSignal = true;
        }
        
        // Close position if early exit conditions met
        if(earlyExitSignal)
        {
            ClosePosition(ticket, "Early exit triggered");
        }
    }
}

//+------------------------------------------------------------------+
//| Manage dynamic take profit                                        |
//+------------------------------------------------------------------+
void manageDynamicTP()
{
    for(int i = PositionsTotal() - 1; i >= 0; i--)
    {
        ulong ticket = PositionGetTicket(i);
        if(!PositionSelectByTicket(ticket))
            continue;
        
        double currentPrice = PositionGetDouble(POSITION_PRICE_CURRENT);
        double openPrice = PositionGetDouble(POSITION_PRICE_OPEN);
        double currentTP = PositionGetDouble(POSITION_TP);
        ENUM_POSITION_TYPE posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
        
        // Calculate profit points and ATR
        double profitPoints = (posType == POSITION_TYPE_BUY) ? 
            (currentPrice - openPrice) / Market.point : 
            (openPrice - currentPrice) / Market.point;
        
        // Adjust TP based on volatility and profit
        if(profitPoints >= ActivationPoints)
        {
            double dynamicTPMultiplier = Session.isPreMarket ? 1.2 : 1.5;
            
            // Base new TP on current price and volatility
            double newTP;
            if(posType == POSITION_TYPE_BUY)
            {
                newTP = currentPrice + (Session.currentATR * dynamicTPMultiplier);
            }
            else
            {
                newTP = currentPrice - (Session.currentATR * dynamicTPMultiplier);
            }
            
            // Modify TP if significantly different
            if(MathAbs(newTP - currentTP) > 10 * Market.point)
            {
                ModifyPosition(ticket, PositionGetDouble(POSITION_SL), newTP);
            }
        }
        
        // Tighten TP in high volatility
        if(Session.volatilityRatio > MaxATRMultiplier * 1.2)
        {
            double tightTP;
            if(posType == POSITION_TYPE_BUY)
            {
                tightTP = currentPrice + (Session.currentATR * 1.0);
            }
            else
            {
                tightTP = currentPrice - (Session.currentATR * 1.0);
            }
            
            ModifyPosition(ticket, PositionGetDouble(POSITION_SL), tightTP);
        }
    }
}

//+------------------------------------------------------------------+
//| Expert tick function                                               |
//+------------------------------------------------------------------+
void OnTick()
{
    // Reset daily counters if needed
    static datetime lastDay = 0;
    datetime currentTime = TimeCurrent();
    MqlDateTime current_struct;
    TimeToStruct(currentTime, current_struct);
    
    datetime today_start = StringToTime(StringFormat("%04d.%02d.%02d 00:00:00", 
        current_struct.year, current_struct.mon, current_struct.day));
        
    if(lastDay < today_start)
    {
        ResetDailyValues();
        lastDay = currentTime;
    }
    
    // Update session state
    UpdateSessionState();
    
    // Primary risk checks
    if(!PerformSafetyChecks())
    {
        return;
    }
    
    // Monitor and manage existing trades
    if(PositionsTotal() > 0)
    {
        ManageExistingTrades();
        return;
    }
    
    // Check for new trading opportunities on new M5 bar
    if(!isNewBar())
    {
        return;
    }
    
    // Look for new trade opportunities
    CheckForNewNAS100Trades();
}

//+------------------------------------------------------------------+
//| Update session state                                              |
//+------------------------------------------------------------------+
void UpdateSessionState()
{
    MqlDateTime time_struct;
    TimeToStruct(TimeCurrent(), time_struct);
    
    // Update pre-market status
    Session.isPreMarket = (time_struct.hour == 13) || 
                         (time_struct.hour == 14 && time_struct.min < 30);
    
    // Update spread
    Session.currentSpread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD) * Market.point;
    
    // Update ATR
    double atr[];
    ArraySetAsSeries(atr, true);
    if(CopyBuffer(Handles.atr, 0, 0, 1, atr) > 0)
    {
        Session.currentATR = atr[0];
    }
    
    // Update volatility ratio
    if(Session.currentATR > 0)
    {
        double atr_prev[];
        ArraySetAsSeries(atr_prev, true);
        if(CopyBuffer(Handles.atr, 0, 1, ATR_Period, atr_prev) > 0)
        {
            double avgATR = 0;
            for(int i = 0; i < ATR_Period; i++)
                avgATR += atr_prev[i];
            avgATR /= ATR_Period;
            
            Session.volatilityRatio = Session.currentATR / avgATR;
        }
    }
}

//+------------------------------------------------------------------+
//| Check for new NAS100 trade opportunities                          |
//+------------------------------------------------------------------+
void CheckForNewNAS100Trades()
{
    // Check consecutive losses
    if(CheckConsecutiveLosses())
    {
        Print("Maximum consecutive losses reached");
        return;
    }
    
    // Get trends across timeframes
    int trend_m5 = getTrendDirection(Handles.fast_ema_m5, 
                                   Handles.slow_ema_m5, PERIOD_M5);
    int trend_m15 = getTrendDirection(Handles.fast_ema_m15, 
                                    Handles.slow_ema_m15, PERIOD_M15);
    int trend_h1 = getTrendDirection(Handles.fast_ema_h1, 
                                   Handles.slow_ema_h1, PERIOD_H1);
    
    // Different trend confirmation requirements based on session
    if(Session.isPreMarket)
    {
        // Stricter requirements for pre-market
        if(trend_m5 == 0 || trend_m15 == 0 || trend_h1 == 0 || 
           trend_m5 != trend_m15 || trend_m15 != trend_h1)
        {
            return;
        }
    }
    else
    {
        // Main session requirements
        if(trend_m5 == 0 || trend_m15 == 0 || trend_m5 != trend_m15)
        {
            return;
        }
    }
    
    // Entry signal check
    if(ValidateEntrySignal(trend_m5))
    {
        if(trend_m5 == 1)
        {
            trade(ORDER_TYPE_BUY);
        }
        else if(trend_m5 == -1)
        {
            trade(ORDER_TYPE_SELL);
        }
    }
}

//+------------------------------------------------------------------+
//| Validate entry signal                                              |
//+------------------------------------------------------------------+
bool ValidateEntrySignal(int trend)
{
    if(trend == 0) return false;
    
    // Price data arrays
    double close[], high[], low[];
    ArraySetAsSeries(close, true);
    ArraySetAsSeries(high, true);
    ArraySetAsSeries(low, true);
    
    // Get recent price data
    int bars = Session.isPreMarket ? 10 : 5; // More bars for pre-market confirmation
    if(CopyClose(_Symbol, PERIOD_M5, 0, bars, close) <= 0 ||
       CopyHigh(_Symbol, PERIOD_M5, 0, bars, high) <= 0 ||
       CopyLow(_Symbol, PERIOD_M5, 0, bars, low) <= 0)
    {
        return false;
    }
    
    // Calculate price momentum
    double momentum = (close[0] - close[bars-1]) / Session.currentATR;
    
    // Check for momentum alignment with trend
    if((trend == 1 && momentum < 0.5) || (trend == -1 && momentum > -0.5))
    {
        return false;
    }
    
    // Check for overextension
    double avgRange = 0;
    for(int i = 1; i < bars; i++)
    {
        avgRange += high[i] - low[i];
    }
    avgRange /= (bars - 1);
    
    if(MathAbs(close[0] - close[bars-1]) > avgRange * 2)
    {
        return false; // Price too extended
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Check for consecutive losses                                       |
//+------------------------------------------------------------------+
bool CheckConsecutiveLosses()
{
    int maxConsecutiveLosses = Session.isPreMarket ? 2 : 3;
    int consecutiveLosses = 0;
    
    for(int i = 0; i < ArraySize(tradeHistory); i++)
    {
        if(tradeHistory[i].openTime == 0) break;
        
        if(!tradeHistory[i].isWin)
            consecutiveLosses++;
        else
            break;
    }
    
    return (consecutiveLosses >= maxConsecutiveLosses);
}

//+------------------------------------------------------------------+
//| Timer event handler                                                |
//+------------------------------------------------------------------+
void OnTimer()
{
    static datetime lastPeriodicCheck = 0;
    datetime currentTime = TimeCurrent();
    
    // Perform checks every 5 minutes
    if(currentTime - lastPeriodicCheck >= 300)
    {
        // Check system health
        if(!VerifySystemHealth())
        {
            Print("System health check failed");
            return;
        }
        
        // Log performance metrics
        LogPerformanceMetrics();
        
        lastPeriodicCheck = currentTime;
    }
}

//+------------------------------------------------------------------+
//| Verify system health                                              |
//+------------------------------------------------------------------+
bool VerifySystemHealth()
{
    // Check indicator handles
    if(!VerifyIndicatorHandles())
    {
        Print("Reinitializing indicators");
        return InitializeIndicators();
    }
    
    // Check connection status
    if(!TerminalInfoInteger(TERMINAL_CONNECTED))
    {
        Print("Terminal not connected");
        return false;
    }
    
    // Check trading context
    if(!MQLInfoInteger(MQL_TRADE_ALLOWED))
    {
        Print("Trading not allowed");
        return false;
    }
    
    // Verify market data freshness
    MqlTick last_tick;
    if(!SymbolInfoTick(_Symbol, last_tick))
    {
        Print("Failed to get current tick data");
        return false;
    }
    
    if(TimeCurrent() - last_tick.time > 5)
    {
        Print("Warning: Market data may be stale");
        return false;
    }
    
    return true;
}

//+------------------------------------------------------------------+
//| Log performance metrics                                            |
//+------------------------------------------------------------------+
void LogPerformanceMetrics()
{
    double balance = AccountInfoDouble(ACCOUNT_BALANCE);
    double equity = AccountInfoDouble(ACCOUNT_EQUITY);
    
    PrintFormat("\n=== NAS100 Performance Metrics ===");
    PrintFormat("Session Type: %s", Session.isPreMarket ? "Pre-Market" : "Main");
    PrintFormat("Total Trades Today: %d", Market.dailyTradeCount);
    PrintFormat("Win Rate: %.2f%%", 
                Market.totalTrades > 0 ? 
                ((double)Market.winningTrades/Market.totalTrades)*100 : 0);
    
    // Daily performance
    double dailyReturn = ((equity - Market.initialBalance) / Market.initialBalance) * 100;
    PrintFormat("Daily Return: %.2f%%", dailyReturn);
    
    // Market conditions
    PrintFormat("Current Spread: %.1f points", Session.currentSpread/Market.point);
    PrintFormat("ATR: %.1f points", Session.currentATR);
    PrintFormat("Volatility Ratio: %.2f", Session.volatilityRatio);
    
    // Risk metrics
    PrintFormat("Current Risk Level: %.2f%%", Risk.currentRisk);
    if(AccountInfoDouble(ACCOUNT_MARGIN) > 0)
    {
        PrintFormat("Margin Level: %.2f%%", 
                   AccountInfoDouble(ACCOUNT_MARGIN_LEVEL));
    }
}

//+------------------------------------------------------------------+
//| Log final statistics                                              |
//+------------------------------------------------------------------+
void LogFinalStatistics()
{
    PrintFormat("\n=== NAS100 EA Final Statistics ===");
    PrintFormat("Total Trades: %d", Market.totalTrades);
    PrintFormat("Winning Trades: %d", Market.winningTrades);
    PrintFormat("Win Rate: %.2f%%", 
                Market.totalTrades > 0 ? 
                ((double)Market.winningTrades/Market.totalTrades)*100 : 0);
    
    double finalEquity = AccountInfoDouble(ACCOUNT_EQUITY);
    double totalReturn = ((finalEquity - Market.initialBalance) / 
                         Market.initialBalance) * 100;
    
    PrintFormat("Total Return: %.2f%%", totalReturn);
    PrintFormat("Final Balance: %.2f", finalEquity);
}
