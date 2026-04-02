---
name: okx-eth-swap-strategy
description: "ETH USDT perpetual contract adaptive trading strategy. Use when user says: start strategy, run strategy, ETH strategy, swap strategy, contract strategy, check position, market analysis, execute round, or Chinese equivalents like 启动策略, 运行策略, 开始交易, ETH策略, 永续策略, 合约策略, 检查持仓, 市场分析, 执行一轮."
license: MIT
metadata:
  author: maomaoxia99
  version: "1.0.0"
  homepage: "https://github.com/maomaoxia99/research-skills"
---

# ETH USDT Perpetual Adaptive Trading Strategy

Multi-strategy adaptive perpetual contract trading skill for ETH-USDT-SWAP. AI automatically identifies market regime (trending vs ranging) and switches strategy accordingly. Uses local state file for cross-round memory.

## Prerequisites

```bash
npm install -g @okx_ai/okx-trade-cli
okx config init
```

Credentials must be configured: OKX_API_KEY, OKX_SECRET_KEY, OKX_PASSPHRASE.

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| instId | ETH-USDT-SWAP | Trading instrument |
| trendLever | 7 | Leverage in trending mode |
| rangeLever | 5 | Leverage in ranging mode |
| maxSingleRisk | 3% | Max loss per trade |
| maxPositionPct | 70% | Max margin usage |
| maxDrawdown | 25% | Max drawdown before pause |
| dailyLossLimit | 10% | Daily loss cap |
| consecutiveLossLimit | 3 | Consecutive losses before size reduction |
| atrMultiplier | 1.5 | Stop loss = ATR x this |
| moveStopThreshold | 5% | Move stop to breakeven when profit exceeds this |
| emergencyExitPct | 75% | Emergency exit if equity drops below initial x this |
| trendScoreThreshold | 3 | Min direction score for strong trend entry |
| rsiOverbought | 70 | RSI overbought level |
| rsiOversold | 30 | RSI oversold level |

## Execution Flow

Each round follows this exact sequence:

```
1. Read state file (~/.okx-strategy/state.json)
2. Pre-trade risk check (pause / reduce size?)
   -> If paused -> update state -> END
3. Data collection (15 data points)
4. Market regime assessment (mode + direction + score)
5. Trade decision (open / close / add / hold)
6. Order execution
7. Execution verification (confirm fills + positions)
8. Post-trade risk check (stop loss confirmation + trailing stop)
9. Update state file
10. Output round summary
```

## Step 1: Data Collection

Collect ALL of the following data before making any decision:

> Market data commands need no --profile. Trading commands MUST use --profile live.

| # | Data | Command |
|---|------|---------|
| 1 | Current price | `okx market ticker ETH-USDT-SWAP` |
| 2 | 1H candles | `okx market candles --instId ETH-USDT-SWAP --bar 1H --limit 100` |
| 3 | 4H candles | `okx market candles --instId ETH-USDT-SWAP --bar 4H --limit 50` |
| 4 | EMA12 (1H) | `okx market indicator --instId ETH-USDT-SWAP --indicator ema --bar 1H --period 12` |
| 5 | EMA26 (1H) | `okx market indicator --instId ETH-USDT-SWAP --indicator ema --bar 1H --period 26` |
| 6 | EMA12 (4H) | `okx market indicator --instId ETH-USDT-SWAP --indicator ema --bar 4H --period 12` |
| 7 | EMA26 (4H) | `okx market indicator --instId ETH-USDT-SWAP --indicator ema --bar 4H --period 26` |
| 8 | MACD | `okx market indicator --instId ETH-USDT-SWAP --indicator macd --bar 1H` |
| 9 | RSI(14) | `okx market indicator --instId ETH-USDT-SWAP --indicator rsi --bar 1H` |
| 10 | Bollinger Bands | `okx market indicator --instId ETH-USDT-SWAP --indicator bb --bar 1H` |
| 11 | ATR(14) | `okx market indicator --instId ETH-USDT-SWAP --indicator atr --bar 1H` |
| 12 | Funding rate | `okx market funding-rate --instId ETH-USDT-SWAP` |
| 13 | Open interest | `okx market open-interest --instId ETH-USDT-SWAP` |
| 14 | Account balance | `okx account balance --profile live` |
| 15 | Current position | `okx swap positions --instId ETH-USDT-SWAP --profile live` |

Use 4H for overall direction, 1H for entry timing.

## Step 2: Market Regime Assessment

### 2.1 Mode Identification

| Mode | Conditions | Strategy |
|------|-----------|----------|
| **Strong trend** | EMA12 > EMA26 with widening gap + MACD histogram increasing + BB bandwidth expanding | Follow trend, no counter-trade |
| **Weak trend** | EMA crossover but MACD momentum fading + BB narrowing | Light position, ready to exit |
| **Ranging** | Price oscillating around BB middle + MACD near zero alternating + ATR low | Buy low sell high within range |
| **Pre-breakout** | BB bandwidth at 20-period low + OI rising + ATR low | Wait for breakout, no entry |

### 2.2 Direction Scoring

| Signal | Long weight | Short weight |
|--------|------------|--------------|
| EMA12 > EMA26 (1H) | +1 | — |
| EMA12 > EMA26 (4H) | +2 | — |
| EMA12 < EMA26 (1H) | — | +1 |
| EMA12 < EMA26 (4H) | — | +2 |
| MACD golden cross | +1 | — |
| MACD death cross | — | +1 |
| RSI < 30 | +1 (oversold bounce) | — |
| RSI > 70 | — | +1 (overbought pullback) |
| Funding rate > 0.1% | — | +1 (overcrowded long) |
| Funding rate < -0.1% | +1 (overcrowded short) | — |

**Direction score** = long weight - short weight:
- >= 3: Strong long
- 1~2: Mild long
- -2~0: Neutral / wait
- <= -3: Strong short

### 2.3 Output

Write assessment to state file:

```json
{
  "timestamp": "2026-04-10T08:00:00Z",
  "mode": "trending|ranging|weak_trend|pre_breakout",
  "direction": "long|short|neutral",
  "directionScore": 4,
  "confidence": "high|medium|low",
  "keyLevels": {
    "support": 1820,
    "resistance": 1900,
    "bbUpper": 1905,
    "bbLower": 1815,
    "bbMiddle": 1860
  }
}
```

## Step 3: Trade Decision

### 3.1 Entry Conditions (No Position)

| Market Mode | Conditions | Action |
|-------------|-----------|--------|
| **Strong trend long** | directionScore >= 3 + RSI 35-65 + price near EMA12 | Open long, 7x lever |
| **Strong trend short** | directionScore <= -3 + RSI 35-65 + price near EMA12 | Open short, 7x lever |
| **Ranging long** | mode=ranging + price at BB lower band + RSI < 35 | Open long, 5x lever |
| **Ranging short** | mode=ranging + price at BB upper band + RSI > 65 | Open short, 5x lever |
| **Weak trend** | directionScore 1~2 + MACD aligned | Light follow, 5x lever |
| **Pre-breakout / neutral** | None of above met | No trade, wait |

### 3.2 Position Management (Has Position)

| Scenario | Conditions | Action |
|----------|-----------|--------|
| **Trend add** | Holding long + trend strengthening + unrealized PnL > 2% + total margin < 70% | Add 50% of original size |
| **Trend partial exit** | Holding long + RSI > 75 or MACD divergence | Close 50% (keep base) |
| **Trend full exit** | Holding long + EMA12 crosses below EMA26 (1H) + MACD death cross | Close all |
| **Range exit** | Holding long + price hits BB upper or profit 2-3% | Close all |
| **Direction reversal** | Holding long + directionScore becomes <= -3 | Close all (don't reverse immediately, wait next round) |
| **Stop loss** | Unrealized loss >= 3% | Close all |

> Short side logic is the mirror of long side.

### 3.3 Position Sizing

```
Risk amount = available balance x 3%
Stop distance = ATR(14) x 1.5
Position size = risk amount / stop distance
Actual leverage = min(calculated, mode lever cap)
```

Example (300U capital):
- Risk = 300 x 3% = 9U
- ATR = 20U, stop distance = 30U
- Size = 9 / 30 = 0.3 ETH
- 7x lever margin = 0.3 x 1850 / 7 = 79U (26% of capital, safe)

## Step 4: Order Execution

### 4.1 Set Leverage (only when changed)

```bash
okx swap leverage --instId ETH-USDT-SWAP --lever 7 --mgnMode cross --profile live
```

### 4.2 Open Position

```bash
# Open long with TP/SL
okx swap place --instId ETH-USDT-SWAP \
  --side buy --posSide long \
  --ordType market --sz 0.3 \
  --tdMode cross \
  --tpTriggerPx 1920 --tpOrdPx -1 \
  --slTriggerPx 1795 --slOrdPx -1 \
  --profile live

# Open short with TP/SL
okx swap place --instId ETH-USDT-SWAP \
  --side sell --posSide short \
  --ordType market --sz 0.3 \
  --tdMode cross \
  --tpTriggerPx 1780 --tpOrdPx -1 \
  --slTriggerPx 1905 --slOrdPx -1 \
  --profile live
```

> `--tpOrdPx -1` and `--slOrdPx -1` = market execution on trigger.

### 4.3 Close Position

```bash
# Partial close (50%)
okx swap place --instId ETH-USDT-SWAP \
  --side sell --posSide long \
  --ordType market --sz 0.15 \
  --tdMode cross --profile live

# Full close
okx swap close --instId ETH-USDT-SWAP --mgnMode cross --posSide long --profile live
```

### 4.4 Add to Position

```bash
okx swap place --instId ETH-USDT-SWAP \
  --side buy --posSide long \
  --ordType market --sz 0.15 \
  --tdMode cross \
  --slTriggerPx 1810 --slOrdPx -1 \
  --profile live
```

### 4.5 Verify Execution

After EVERY order, verify:

```bash
okx swap fills --instId ETH-USDT-SWAP --profile live
okx swap positions --instId ETH-USDT-SWAP --profile live
```

Update state file with actual fill data.

## Step 5: Risk Control

### 5.1 Per-Trade Risk

| Rule | Parameter | Implementation |
|------|-----------|---------------|
| **Hard stop** | 3% loss | Attached via --slTriggerPx on order, runs on OKX server |
| **Dynamic stop distance** | ATR(14) x 1.5 | Calculated each round |
| **Trailing stop** | Move stop to breakeven when profit > 5% | Check each round, amend existing stop |
| **Max single margin** | <= 35% of available balance | Pre-trade check |

### 5.2 Account-Level Risk

| Rule | Threshold | Action |
|------|-----------|--------|
| **Max total margin** | <= 70% of available balance | Block new positions |
| **Max drawdown** | >= 25% from peak | Close all + pause 2 rounds |
| **Consecutive losses** | 3 in a row | Halve next position size |
| **Daily loss limit** | >= 10% | Stop new positions for the day |
| **Funding rate guard** | > 0.3% against position | Warn and suggest close |

### 5.3 Risk Check Flow

**Pre-decision gate:**
1. Read state file -> calculate current drawdown
2. If pause triggered -> skip round, update state only
3. If consecutive losses -> mark reducedSize mode
4. Check daily loss -> if exceeded, only allow close operations

**Post-execution guard:**
1. After open -> confirm stop loss order exists (okx swap algo orders)
2. While holding -> check if trailing stop needed
3. After close -> update PnL, win rate, peak balance
4. Anomaly -> if OKX position data differs from state, trust OKX data

### 5.4 Emergency Exit

Immediately close ALL positions if ANY:
- Account equity < 75% of initial balance
- OKX API returns 3 consecutive errors
- Any position unrealized loss > 5% (stop loss backup)

```bash
okx swap close --instId ETH-USDT-SWAP --mgnMode cross --posSide long --profile live
okx swap close --instId ETH-USDT-SWAP --mgnMode cross --posSide short --profile live
```

## State File

Path: `~/.okx-strategy/state.json`

Initialize on first run if not exists. Always read at start, write at end.

```json
{
  "lastRun": "2026-04-10T08:00:00Z",
  "account": {
    "initialBalance": 300,
    "currentBalance": 312.5,
    "availableBalance": 233.5
  },
  "position": {
    "side": "long",
    "posSide": "long",
    "size": 0.3,
    "entryPx": 1850,
    "lever": 7,
    "unrealizedPnl": 6.3,
    "margin": 79
  },
  "riskMetrics": {
    "totalPnl": 12.5,
    "totalPnlPct": 4.17,
    "maxDrawdown": -3.2,
    "peakBalance": 315,
    "winRate": 0.6,
    "tradeCount": 5,
    "consecutiveLosses": 0,
    "dailyPnl": 3.2,
    "tradingPaused": false,
    "pauseUntil": null,
    "reducedSize": false
  },
  "marketState": {
    "mode": "trending",
    "direction": "long",
    "directionScore": 4,
    "confidence": "high"
  },
  "tradeHistory": [
    {
      "time": "2026-04-09T14:00:00Z",
      "action": "open_long",
      "price": 1850,
      "size": 0.3,
      "pnl": null
    }
  ]
}
```

## Round Summary Output

After each round, output:

```
Strategy Round Summary | 2026-04-10 08:00 UTC
Market: Strong Trend UP | Score: +4 | ETH: $1,862
Position: Long 0.3 ETH @ $1,850 | 7x | PnL +$3.6 (+1.2%)
Action: Hold (trend continues, no add/close trigger)
Risk: SL $1,795 | TP $1,920 | Drawdown -1.1%
Account: $312.5 (+4.17%) | Win rate 60% | 5 trades
Next: 30 minutes
```

## Execution Frequency

| Scenario | Interval | Reason |
|----------|----------|--------|
| Has position | Every 30 min | Monitor closely, adjust stops |
| No position | Every 1 hour | Wait for entry signal |
| High volatility | Every 15 min | ATR spikes 150%+ triggers fast mode |

Use `/loop 30m` or `/schedule` for automated execution.
