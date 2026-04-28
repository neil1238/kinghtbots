# 1. Create app
npx create-react-app bot
cd bot
npm install recharts lucide-react

# 2. Replace src/App.jsx with ultimate-complete-bot-fixed.jsx

# 3. Run
npm start

# 4. Open http://localhost:3000 & paste your Deriv token
import React, { useState, useRef, useEffect } from "react";
import {
  LineChart,
  Line,
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  ComposedChart,
  Area,
  Legend,
} from "recharts";
import {
  TrendingUp,
  TrendingDown,
  AlertCircle,
  Play,
  Pause,
  Settings,
  LogOut,
  Activity,
  Zap,
  Shield,
  BarChart3,
  Maximize2,
} from "lucide-react";

export default function UltimateTradeBot() {
  // ==================== STATE ====================
  const [token, setToken] = useState("");
  const [connected, setConnected] = useState(false);
  const [isTrading, setIsTrading] = useState(false);
  const [profit, setProfit] = useState(0);
  const [status, setStatus] = useState("Idle");
  const [chartData, setChartData] = useState([]);
  const [trades, setTrades] = useState([]);
  const [expandedChart, setExpandedChart] = useState("price");
  const [selectedTimeframe, setSelectedTimeframe] = useState("1m");

  const [stats, setStats] = useState({
    wins: 0,
    losses: 0,
    winRate: 0,
    totalTrades: 0,
    avgWin: 0,
    avgLoss: 0,
    profitFactor: 0,
    sharpeRatio: 0,
    maxDrawdown: 0,
    currentDrawdown: 0,
  });

  const [indicators, setIndicators] = useState({
    rsi: 50,
    macd: 0,
    macdSignal: 0,
    bb_upper: 0,
    bb_middle: 0,
    bb_lower: 0,
    ema12: 0,
    ema26: 0,
    momentum: 0,
    atr: 0,
  });

  const [equityHistory, setEquityHistory] = useState([]);
  const [drawdownHistory, setDrawdownHistory] = useState([]);
  const [rsiHistory, setRsiHistory] = useState([]);
  const [macdHistory, setMacdHistory] = useState([]);

  // ==================== BOT STATE ====================
  const ws = useRef(null);
  const fast = useRef([]);
  const slow = useRef([]);
  const prices = useRef([]);
  const returns = useRef([]);
  const lastTrade = useRef(0);
  const stake = useRef(10);
  const lossStreak = useRef(0);
  const sessionProfit = useRef(0);
  const peakProfit = useRef(0);
  const winStreak = useRef(0);
  const openPrice = useRef(0);
  const highPrice = useRef(0);
  const lowPrice = useRef(0);

  // ==================== CONFIG ====================
  const STOP_LOSS = -150;
  const TAKE_PROFIT = 300;
  const COOLDOWN = 7000;
  const MAX_STAKE = 60;
  const MIN_STAKE = 5;

  // ==================== INDICATORS ====================
  const EMA = (arr, p = 10) => {
    if (!arr || arr.length === 0) return 0;
    let k = 2 / (p + 1);
    let ema = arr[0] || 0;
    for (let v of arr) {
      ema = v * k + ema * (1 - k);
    }
    return ema;
  };

  const SMA = (arr, period = 20) => {
    if (arr.length < period) return arr[arr.length - 1] || 0;
    return arr.slice(-period).reduce((a, b) => a + b, 0) / period;
  };

  const RSI = (arr, p = 14) => {
    if (arr.length < p) return 50;
    let gains = 0,
      losses = 0;
    for (let i = Math.max(0, arr.length - p - 1); i < arr.length - 1; i++) {
      const diff = arr[i + 1] - arr[i];
      if (diff > 0) gains += diff;
      else losses -= diff;
    }
    const avgGain = gains / p;
    const avgLoss = losses / p;
    const rs = avgGain / (avgLoss || 0.0001);
    return 100 - 100 / (1 + rs);
  };

  const MACD = (arr) => {
    const ema12 = EMA(arr, 12);
    const ema26 = EMA(arr, 26);
    const macd = ema12 - ema26;
    const signal = EMA([macd], 9);
    return { macd, signal: signal || ema12, histogram: macd - (signal || ema12) };
  };

  const Bollinger = (arr, period = 20, stdDev = 2) => {
    const sma = SMA(arr, period);
    const variance =
      arr.slice(-period).reduce((sum, val) => sum + Math.pow(val - sma, 2), 0) /
      period;
    const std = Math.sqrt(variance);
    return {
      upper: sma + std * stdDev,
      middle: sma,
      lower: sma - std * stdDev,
    };
  };

  const ATR = (arr, period = 14) => {
    if (arr.length < period) return 0;
    const trueRanges = [];
    for (let i = 1; i < arr.length; i++) {
      const tr = Math.max(arr[i] - arr[i - 1], Math.abs(arr[i]));
      trueRanges.push(tr);
    }
    const atr = trueRanges.slice(-period).reduce((a, b) => a + b, 0) / period;
    return atr;
  };

  const momentum = (arr, period = 10) => {
    if (arr.length < period) return 0;
    return (
      ((arr[arr.length - 1] - arr[arr.length - 1 - period]) /
        arr[arr.length - 1 - period]) *
      100
    );
  };

  const trend = (arr) => {
    if (arr.length < 5) return null;
    const recent = arr.slice(-5);
    const avg = recent.reduce((a, b) => a + b) / recent.length;
    return arr[arr.length - 1] > avg ? "UP" : "DOWN";
  };

  // ==================== CONNECT ====================
  const connect = () => {
    if (!token) {
      setStatus("❌ Please enter API token");
      return;
    }

    ws.current = new WebSocket("wss://ws.derivws.com/websockets/v3");

    ws.current.onopen = () => {
      ws.current.send(JSON.stringify({ authorize: token }));
      setStatus("🔄 Connecting...");
    };

    ws.current.onmessage = (msg) => {
      const data = JSON.parse(msg.data);

      if (data.authorize) {
        setConnected(true);
        setStatus("✅ Connected to Deriv");
        ws.current.send(JSON.stringify({ ticks: "R_100", subscribe: 1 }));
        return;
      }

      if (data.error) {
        setStatus(`❌ Error: ${data.error.message}`);
        return;
      }

      if (isTrading) {
        handleBot(data);
      }
    };

    ws.current.onerror = () => {
      setStatus("❌ Connection failed");
      setConnected(false);
    };

    ws.current.onclose = () => {
      setConnected(false);
      setStatus("Disconnected");
    };
  };

  // ==================== BOT LOGIC ====================
  const handleBot = (data) => {
    if (data.msg_type === "tick") {
      const price = parseFloat(data.tick.quote);

      // Initialize candle
      if (openPrice.current === 0) {
        openPrice.current = price;
        highPrice.current = price;
        lowPrice.current = price;
      }

      // Track price movement
      if (price > highPrice.current) highPrice.current = price;
      if (price < lowPrice.current) lowPrice.current = price;

      // Store prices
      fast.current.push(price);
      prices.current.push(price);

      if (fast.current.length > 100) fast.current.shift();
      if (prices.current.length > 100) prices.current.shift();

      // Slow moving average
      if (fast.current.length % 3 === 0) {
        slow.current.push(price);
        if (slow.current.length > 50) slow.current.shift();
      }

      // Calculate indicators
      const ema12 = EMA(prices.current, 12);
      const ema26 = EMA(prices.current, 26);
      const rsi = RSI(prices.current, 14);
      const { macd, signal: macdSignal, histogram } = MACD(prices.current);
      const bb = Bollinger(prices.current, 20);
      const mom = momentum(prices.current, 10);
      const atr = ATR(prices.current, 14);
      const tFast = trend(fast.current);
      const tSlow = trend(slow.current);

      // Update indicators state
      setIndicators({
        rsi: parseFloat(rsi.toFixed(2)),
        macd: parseFloat(macd.toFixed(4)),
        macdSignal: parseFloat(macdSignal.toFixed(4)),
        bb_upper: parseFloat(bb.upper.toFixed(2)),
        bb_middle: parseFloat(bb.middle.toFixed(2)),
        bb_lower: parseFloat(bb.lower.toFixed(2)),
        ema12: parseFloat(ema12.toFixed(2)),
        ema26: parseFloat(ema26.toFixed(2)),
        momentum: parseFloat(mom.toFixed(2)),
        atr: parseFloat(atr.toFixed(4)),
      });

      // Update main chart
      setChartData((prev) => [
        ...prev.slice(-150),
        {
          i: prev.length,
          price: parseFloat(price.toFixed(2)),
          ema12: parseFloat(ema12.toFixed(2)),
          ema26: parseFloat(ema26.toFixed(2)),
          bb_upper: parseFloat(bb.upper.toFixed(2)),
          bb_lower: parseFloat(bb.lower.toFixed(2)),
        },
      ]);

      // Update RSI history
      setRsiHistory((prev) => [
        ...prev.slice(-150),
        {
          i: prev.length,
          rsi: parseFloat(rsi.toFixed(2)),
        },
      ]);

      // Update MACD history
      setMacdHistory((prev) => [
        ...prev.slice(-150),
        {
          i: prev.length,
          macd: parseFloat(macd.toFixed(4)),
          signal: parseFloat(macdSignal.toFixed(4)),
        },
      ]);

      // ==================== STRATEGY ====================
      let signal_type = null;
      let confidence = 0;

      if (
        tFast === "UP" &&
        tSlow === "UP" &&
        price > ema12 &&
        price > ema26 &&
        rsi > 40 &&
        rsi < 70 &&
        macd > macdSignal &&
        mom > 0 &&
        price > bb.lower
      ) {
        signal_type = "CALL";
        confidence = 0.95;
      }

      if (
        tFast === "DOWN" &&
        tSlow === "DOWN" &&
        price < ema12 &&
        price < ema26 &&
        rsi < 60 &&
        rsi > 30 &&
        macd < macdSignal &&
        mom < 0 &&
        price < bb.upper
      ) {
        signal_type = "PUT";
        confidence = 0.95;
      }

      if (price < bb.lower && rsi < 30 && tFast === "UP") {
        signal_type = "CALL";
        confidence = 0.75;
      }

      if (price > bb.upper && rsi > 70 && tFast === "DOWN") {
        signal_type = "PUT";
        confidence = 0.75;
      }

      // ==================== EXECUTE TRADE ====================
      const now = Date.now();
      if (
        signal_type &&
        confidence > 0.5 &&
        now - lastTrade.current > COOLDOWN &&
        sessionProfit.current > STOP_LOSS &&
        sessionProfit.current < TAKE_PROFIT
      ) {
        placeTrade(signal_type, confidence);
        lastTrade.current = now;
      }
    }

    // ==================== HANDLE RESULTS ====================
    if (data.msg_type === "proposal_open_contract") {
      const contract = data.proposal_open_contract;
      if (contract.is_sold) {
        const result = contract.profit || 0;
        sessionProfit.current += result;
        setProfit(sessionProfit.current);

        // Track returns for Sharpe ratio
        returns.current.push(result);
        if (returns.current.length > 100) returns.current.shift();

        // Update peak for drawdown
        if (sessionProfit.current > peakProfit.current) {
          peakProfit.current = sessionProfit.current;
        }

        // Update equity history
        setEquityHistory((prev) => [
          ...prev.slice(-99),
          {
            i: prev.length,
            equity: parseFloat(sessionProfit.current.toFixed(2)),
          },
        ]);

        // Track drawdown
        const currentDrawdown = peakProfit.current - sessionProfit.current;
        setDrawdownHistory((prev) => [
          ...prev.slice(-99),
          {
            i: prev.length,
            drawdown: currentDrawdown,
          },
        ]);

        // Calculate stats
        setStats((prev) => {
          const isWin = result > 0;
          const newWins = isWin ? prev.wins + 1 : prev.wins;
          const newLosses = isWin ? prev.losses : prev.losses + 1;
          const newTotal = newWins + newLosses;

          const newAvgWin = isWin
            ? (prev.avgWin * prev.wins + result) / newWins
            : prev.avgWin;
          const newAvgLoss = !isWin
            ? (prev.avgLoss * prev.losses + result) / newLosses
            : prev.avgLoss;

          // Sharpe ratio
          const avgReturn = returns.current.reduce((a, b) => a + b, 0) / (returns.current.length || 1);
          const variance = returns.current.reduce((sum, r) => sum + Math.pow(r - avgReturn, 2), 0) / (returns.current.length || 1);
          const stdDev = Math.sqrt(variance);
          const sharpeRatio = stdDev !== 0 ? (avgReturn / stdDev) * Math.sqrt(252) : 0;

          const newStats = {
            wins: newWins,
            losses: newLosses,
            totalTrades: newTotal,
            avgWin: parseFloat(newAvgWin.toFixed(2)),
            avgLoss: parseFloat(newAvgLoss.toFixed(2)),
            profitFactor: newAvgLoss !== 0 ? parseFloat((newAvgWin / Math.abs(newAvgLoss)).toFixed(2)) : 0,
            winRate: ((newWins / newTotal) * 100).toFixed(1),
            sharpeRatio: parseFloat(sharpeRatio.toFixed(2)),
            maxDrawdown: 0,
            currentDrawdown: parseFloat(currentDrawdown.toFixed(2)),
          };

          return newStats;
        });

        // Log trade
        setTrades((prev) => [
          {
            id: Date.now(),
            type: contract.contract_type,
            result: result > 0 ? "✅ WIN" : "❌ LOSS",
            profit: parseFloat(result.toFixed(2)),
            time: new Date().toLocaleTimeString(),
            stake: stake.current,
          },
          ...prev.slice(0, 14),
        ]);

        // ==================== RISK MANAGEMENT ====================
        if (result > 0) {
          winStreak.current++;
          lossStreak.current = 0;
          if (winStreak.current > 3 && stake.current < MAX_STAKE) {
            stake.current = Math.min(stake.current + 5, MAX_STAKE);
          }
        } else {
          lossStreak.current++;
          winStreak.current = 0;
          stake.current = Math.max(Math.ceil(stake.current * 1.5), MIN_STAKE);
          if (lossStreak.current > 6) {
            setIsTrading(false);
            setStatus("⚠️ Loss streak - Auto paused (6+ losses)");
          }
        }

        // Reset candle
        openPrice.current = 0;
        highPrice.current = 0;
        lowPrice.current = 0;
      }
    }
  };

  // ==================== PLACE TRADE ====================
  const placeTrade = (type, conf) => {
    if (!ws.current) return;
    ws.current.send(
      JSON.stringify({
        buy: 1,
        price: stake.current,
        parameters: {
          amount: stake.current,
          basis: "stake",
          contract_type: type,
          currency: "USD",
          duration: 1,
          duration_unit: "m",
          symbol: "R_100",
        },
      })
    );
  };

  const toggleTrading = () => {
    setIsTrading(!isTrading);
    setStatus(isTrading ? "⏸️ Paused" : "▶️ Trading...");
  };

  const disconnect = () => {
    if (ws.current) ws.current.close();
    setConnected(false);
    setIsTrading(false);
    setToken("");
    setStatus("Disconnected");
  };

  // ==================== RENDER ====================
  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-950 via-slate-900 to-slate-800 text-white p-4 md:p-6 overflow-x-hidden">
      {/* HEADER */}
      <div className="mb-6">
        <div className="flex justify-between items-center mb-4 flex-wrap gap-2">
          <div>
            <h1 className="text-3xl md:text-5xl font-black bg-gradient-to-r from-cyan-400 via-blue-400 to-purple-400 bg-clip-text text-transparent">
              🔥 ULTIMATE TRADE BOT
            </h1>
            <p className="text-slate-400 text-xs md:text-sm mt-1">
              AI Trading Engine with Advanced Charts
            </p>
          </div>
          <div className="flex gap-2">
            {connected && (
              <button
                onClick={toggleTrading}
                className={`px-3 md:px-6 py-2 md:py-3 rounded-lg font-bold flex items-center gap-2 transition-all text-sm md:text-base ${
                  isTrading
                    ? "bg-red-600 hover:bg-red-700 shadow-lg shadow-red-500/50"
                    : "bg-green-600 hover:bg-green-700 shadow-lg shadow-green-500/50"
                }`}
              >
                {isTrading ? <Pause size={18} /> : <Play size={18} />}
                <span className="hidden md:inline">{isTrading ? "STOP" : "START"}</span>
              </button>
            )}
            {connected && (
              <button
                onClick={disconnect}
                className="px-3 md:px-4 py-2 md:py-3 rounded-lg bg-red-500/20 hover:bg-red-500/30 border border-red-500/50 transition"
              >
                <LogOut size={18} />
              </button>
            )}
          </div>
        </div>

        {/* AUTH */}
        {!connected && (
          <div className="bg-gradient-to-r from-slate-800 to-slate-850 rounded-xl p-4 md:p-6 border border-slate-700/50 mb-6">
            <div className="flex flex-col md:flex-row gap-3">
              <input
                type="password"
                placeholder="Paste your Deriv API Token..."
                value={token}
                onChange={(e) => setToken(e.target.value)}
                onKeyPress={(e) => e.key === "Enter" && connect()}
                className="flex-1 px-4 py-3 rounded-lg bg-slate-700 border border-slate-600 text-white placeholder-slate-400 focus:outline-none focus:ring-2 focus:ring-cyan-500 text-sm"
              />
              <button
                onClick={connect}
                className="px-6 py-3 bg-gradient-to-r from-cyan-500 to-blue-500 hover:from-cyan-600 hover:to-blue-600 rounded-lg font-bold shadow-lg shadow-cyan-500/50 whitespace-nowrap text-sm"
              >
                Connect
              </button>
            </div>
            <p className="text-xs text-slate-400 mt-2">
              Get token:{" "}
              <a
                href="https://hub.deriv.com/tradershub/home"
                target="_blank"
                rel="noopener noreferrer"
                className="text-cyan-400 hover:underline font-semibold"
              >
                hub.deriv.com
              </a>
            </p>
          </div>
        )}

        {/* KPI CARDS */}
        <div className="grid grid-cols-2 md:grid-cols-4 gap-2 md:gap-4">
          <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
            <p className="text-slate-400 text-xs uppercase mb-1">Status</p>
            <p className="text-xs md:text-lg font-bold truncate">{status}</p>
          </div>
          <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
            <p className="text-slate-400 text-xs uppercase mb-1">P&L</p>
            <p
              className={`text-xs md:text-lg font-bold ${
                profit >= 0 ? "text-green-400" : "text-red-400"
              }`}
            >
              ${profit.toFixed(2)}
            </p>
          </div>
          <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
            <p className="text-slate-400 text-xs uppercase mb-1">Win Rate</p>
            <p className="text-xs md:text-lg font-bold text-cyan-400">{stats.winRate}%</p>
          </div>
          <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
            <p className="text-slate-400 text-xs uppercase mb-1">Trades</p>
            <p className="text-xs md:text-lg font-bold text-blue-400">{stats.totalTrades}</p>
          </div>
        </div>
      </div>

      {/* MAIN CHARTS */}
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-4 md:gap-6 mb-6">
        {/* PRICE CHART */}
        <div className="lg:col-span-2 bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <div className="flex justify-between items-center mb-4">
            <h2 className="text-base md:text-xl font-bold flex items-center gap-2">
              <TrendingUp className="text-cyan-400" size={20} />
              Price Action
            </h2>
          </div>
          <ResponsiveContainer width="100%" height={280}>
            <ComposedChart data={chartData}>
              <CartesianGrid strokeDasharray="3 3" stroke="#334155" />
              <XAxis dataKey="i" stroke="#94a3b8" />
              <YAxis stroke="#94a3b8" />
              <Tooltip
                contentStyle={{
                  backgroundColor: "#1e293b",
                  border: "1px solid #475569",
                  borderRadius: "8px",
                }}
              />
              <Legend />
              <Line
                type="monotone"
                dataKey="price"
                stroke="#06b6d4"
                dot={false}
                strokeWidth={2}
                name="Price"
              />
              <Line
                type="monotone"
                dataKey="ema12"
                stroke="#f59e0b"
                dot={false}
                strokeWidth={2}
                strokeDasharray="5 5"
                name="EMA12"
              />
              <Line
                type="monotone"
                dataKey="ema26"
                stroke="#ef4444"
                dot={false}
                strokeWidth={2}
                strokeDasharray="5 5"
                name="EMA26"
              />
            </ComposedChart>
          </ResponsiveContainer>
        </div>

        {/* INDICATORS PANEL */}
        <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <h2 className="text-base md:text-xl font-bold mb-4 flex items-center gap-2">
            <Zap className="text-yellow-400" size={20} />
            Indicators
          </h2>
          <div className="space-y-2 text-xs md:text-sm overflow-y-auto max-h-80">
            <div className="flex justify-between">
              <span className="text-slate-400">RSI(14)</span>
              <span
                className={`font-bold ${
                  indicators.rsi < 30
                    ? "text-green-400"
                    : indicators.rsi > 70
                    ? "text-red-400"
                    : "text-cyan-400"
                }`}
              >
                {indicators.rsi}
              </span>
            </div>
            <div className="flex justify-between">
              <span className="text-slate-400">MACD</span>
              <span className="font-bold text-blue-400">{indicators.macd}</span>
            </div>
            <div className="flex justify-between">
              <span className="text-slate-400">EMA12</span>
              <span className="font-bold text-orange-400">{indicators.ema12}</span>
            </div>
            <div className="flex justify-between">
              <span className="text-slate-400">EMA26</span>
              <span className="font-bold text-red-400">{indicators.ema26}</span>
            </div>
            <div className="flex justify-between">
              <span className="text-slate-400">Momentum</span>
              <span
                className={`font-bold ${
                  indicators.momentum > 0 ? "text-green-400" : "text-red-400"
                }`}
              >
                {indicators.momentum}%
              </span>
            </div>
            <div className="border-t border-slate-700 pt-2 mt-2">
              <p className="text-slate-400 text-xs mb-2">Bollinger Bands:</p>
              <div className="space-y-1">
                <div className="flex justify-between text-xs">
                  <span>Upper</span>
                  <span className="text-purple-400">{indicators.bb_upper}</span>
                </div>
                <div className="flex justify-between text-xs">
                  <span>Lower</span>
                  <span className="text-purple-400">{indicators.bb_lower}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      {/* SECONDARY CHARTS */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 md:gap-6 mb-6">
        {/* RSI CHART */}
        <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <h2 className="text-base md:text-lg font-bold mb-4 flex items-center gap-2">
            <BarChart3 className="text-amber-400" size={20} />
            RSI Momentum
          </h2>
          <ResponsiveContainer width="100%" height={220}>
            <LineChart data={rsiHistory}>
              <CartesianGrid strokeDasharray="3 3" stroke="#334155" />
              <XAxis dataKey="i" stroke="#94a3b8" />
              <YAxis domain={[0, 100]} stroke="#94a3b8" />
              <Tooltip
                contentStyle={{
                  backgroundColor: "#1e293b",
                  border: "1px solid #475569",
                }}
              />
              <Line
                type="monotone"
                dataKey="rsi"
                stroke="#f59e0b"
                dot={false}
                strokeWidth={2}
                name="RSI"
              />
            </LineChart>
          </ResponsiveContainer>
        </div>

        {/* MACD CHART */}
        <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <h2 className="text-base md:text-lg font-bold mb-4 flex items-center gap-2">
            <TrendingDown className="text-blue-400" size={20} />
            MACD Convergence
          </h2>
          <ResponsiveContainer width="100%" height={220}>
            <ComposedChart data={macdHistory}>
              <CartesianGrid strokeDasharray="3 3" stroke="#334155" />
              <XAxis dataKey="i" stroke="#94a3b8" />
              <YAxis stroke="#94a3b8" />
              <Tooltip
                contentStyle={{
                  backgroundColor: "#1e293b",
                  border: "1px solid #475569",
                }}
              />
              <Line
                type="monotone"
                dataKey="macd"
                stroke="#06b6d4"
                dot={false}
                strokeWidth={2}
                name="MACD"
              />
              <Line
                type="monotone"
                dataKey="signal"
                stroke="#8b5cf6"
                dot={false}
                strokeWidth={2}
                name="Signal"
              />
            </ComposedChart>
          </ResponsiveContainer>
        </div>
      </div>

      {/* EQUITY & PERFORMANCE */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 md:gap-6 mb-6">
        {/* EQUITY CURVE */}
        <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <h2 className="text-base md:text-lg font-bold mb-4">📈 Equity Curve</h2>
          <ResponsiveContainer width="100%" height={220}>
            <AreaChart data={equityHistory}>
              <CartesianGrid strokeDasharray="3 3" stroke="#334155" />
              <XAxis dataKey="i" stroke="#94a3b8" />
              <YAxis stroke="#94a3b8" />
              <Tooltip
                contentStyle={{
                  backgroundColor: "#1e293b",
                  border: "1px solid #475569",
                }}
              />
              <Area
                type="monotone"
                dataKey="equity"
                fill="#06b6d4"
                stroke="#06b6d4"
                isAnimationActive={false}
              />
            </AreaChart>
          </ResponsiveContainer>
        </div>

        {/* DRAWDOWN */}
        <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50">
          <h2 className="text-base md:text-lg font-bold mb-4">📉 Drawdown</h2>
          <ResponsiveContainer width="100%" height={220}>
            <AreaChart data={drawdownHistory}>
              <CartesianGrid strokeDasharray="3 3" stroke="#334155" />
              <XAxis dataKey="i" stroke="#94a3b8" />
              <YAxis stroke="#94a3b8" />
              <Tooltip
                contentStyle={{
                  backgroundColor: "#1e293b",
                  border: "1px solid #475569",
                }}
              />
              <Area
                type="monotone"
                dataKey="drawdown"
                fill="#ef4444"
                stroke="#ef4444"
                isAnimationActive={false}
              />
            </AreaChart>
          </ResponsiveContainer>
        </div>
      </div>

      {/* PERFORMANCE STATS */}
      <div className="grid grid-cols-2 md:grid-cols-4 gap-2 md:gap-4 mb-6">
        <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
          <p className="text-slate-400 text-xs uppercase mb-1">Avg Win</p>
          <p className="text-xs md:text-lg font-bold text-green-400">
            ${stats.avgWin.toFixed(2)}
          </p>
        </div>
        <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
          <p className="text-slate-400 text-xs uppercase mb-1">Avg Loss</p>
          <p className="text-xs md:text-lg font-bold text-red-400">
            ${stats.avgLoss.toFixed(2)}
          </p>
        </div>
        <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
          <p className="text-slate-400 text-xs uppercase mb-1">Profit Factor</p>
          <p className="text-xs md:text-lg font-bold text-cyan-400">
            {stats.profitFactor}
          </p>
        </div>
        <div className="bg-slate-800/50 rounded-lg p-2 md:p-4 border border-slate-700/50">
          <p className="text-slate-400 text-xs uppercase mb-1">Sharpe Ratio</p>
          <p className="text-xs md:text-lg font-bold text-blue-400">
            {stats.sharpeRatio}
          </p>
        </div>
      </div>

      {/* TRADE LOG */}
      <div className="bg-gradient-to-br from-slate-800 to-slate-850 rounded-xl p-3 md:p-6 border border-slate-700/50 mb-6">
        <h2 className="text-base md:text-xl font-bold mb-4 flex items-center gap-2">
          <Activity className="text-yellow-400" size={20} />
          Trade History ({stats.totalTrades})
        </h2>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-2 max-h-64 overflow-y-auto">
          {trades.length === 0 ? (
            <p className="text-slate-400 col-span-full text-center py-8 text-sm">
              No trades yet. Click START to begin.
            </p>
          ) : (
            trades.map((trade) => (
              <div
                key={trade.id}
                className="p-2 md:p-3 bg-slate-700/30 rounded-lg border border-slate-600/20"
              >
                <div className="flex justify-between items-start mb-2">
                  <div>
                    <p className="font-semibold text-xs md:text-sm">{trade.type}</p>
                    <p className="text-xs text-slate-400">{trade.time}</p>
                  </div>
                  <p className="text-xs md:text-sm font-bold">{trade.result}</p>
                </div>
                <div className="flex justify-between text-xs">
                  <span className="text-slate-400">Stake: ${trade.stake}</span>
                  <span
                    className={
                      trade.profit > 0 ? "text-green-400" : "text-red-400"
                    }
                  >
                    {trade.profit > 0 ? "+" : ""} ${trade.profit}
                  </span>
                </div>
              </div>
            ))
          )}
        </div>
      </div>

      {/* FOOTER */}
      <div className="bg-gradient-to-r from-slate-800 to-slate-850 rounded-xl p-3 md:p-4 border border-slate-700/50 flex flex-col md:flex-row justify-between items-start md:items-center gap-3 text-xs md:text-sm">
        <div className="flex items-center gap-3">
          <div
            className={`w-2 h-2 rounded-full animate-pulse ${
              isTrading ? "bg-green-500" : "bg-red-500"
            }`}
          />
          <span className="text-slate-300">
            {isTrading ? "▶ TRADING" : "⏸ PAUSED"}
          </span>
        </div>
        <span className="text-slate-400">
          {new Date().toLocaleTimeString()}
        </span>
      </div>
    </div>
  );
}
