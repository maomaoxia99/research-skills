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

> **IMPORTANT: All trading commands (swap, account) MUST include `--profile live` for real trading. Market data commands do not need --profile. Never omit `--profile live` on any order placement, position query, or account balance check.**


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
