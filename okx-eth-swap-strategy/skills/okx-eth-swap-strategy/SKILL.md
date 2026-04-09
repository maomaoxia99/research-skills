---
name: okx-eth-swap-strategy
description: >-
  ETH USDT perpetual trading skill for OKX Agent Trade Kit. Use when the user
  asks to run an ETH swap strategy, evaluate ETH-USDT-SWAP, execute one round,
  check whether to long short or no-trade, manage an ETH perpetual position, or
  optimize an ETH perpetual trading workflow with explicit entry exit and risk
  rules.
license: MIT
metadata:
  author: "maomaoxia99"
  version: "1.0.0"
  homepage: "https://github.com/maomaoxia99/research-skills"
---

# ETH USDT Perpetual Strategy

This skill is for `ETH-USDT-SWAP` on OKX. It combines trend-following and range trading, with an emphasis on clean execution, selective entries, and controlled risk.

This skill is for **USDT-margined perpetuals only** and assumes all qualifying orders are routed through Agent Trade Kit / `okx` CLI, not manual clicking.

## Core Principles

1. Trade one instrument only: `ETH-USDT-SWAP`
2. Prefer `no-trade` over low-quality entries
3. Use small USDT sizing with `--tgtCcy quote_ccy`
4. Default leverage is `3x`; only use `4x` in very clean trend continuation
5. Every position must have a stop-loss plan before entry
6. Never add to a losing position
7. Protect capital first, then push for returns

## Scope

This skill is designed to:

- trade `USDT` perpetual swaps only
- use Agent Trade Kit / `okx` CLI workflow
- contain explicit open / close / reduce / no-trade logic
- contain explicit risk controls
- support repeated “execute one round” operation

## Profile Rules

- Use the actual profile name returned by `okx config show`
- Example profile names may look like `okx-demo` and `okx-prod`
- Do not assume profile names are literally `demo` or `live`
- Use a demo or simulated profile for testing or rehearsal
- Use a real-funds profile only when the user explicitly wants the live trading account
- Never assume a live profile silently

## Preflight Check

Before any trade decision that could lead to an order, verify:

```bash
okx account config --profile <profile_name>
okx account balance --ccy USDT --profile <profile_name>
okx account positions --instType SWAP --instId ETH-USDT-SWAP --profile <profile_name>
```

Read `posMode` from account config:

- if `posMode = long_short_mode`, use `--posSide long` / `--posSide short`
- if `posMode = net_mode`, do not use the example long/short commands as-is
- this skill assumes `long_short_mode` for directional long/short management
- if the account is `net_mode`, stop and ask the user to either switch to `long_short_mode` or explicitly request a net-mode adaptation

If account configuration is unavailable, do not place orders.

## Account Assumptions

Default trading posture:

- target style: aggressive but controlled
- max concurrent positions: `1`
- preferred trading window: 15m execution with 1H/4H confirmation
- order size is chosen by the user or inferred conservatively from available balance

## Step 1 数据采集

Collect all of the following before every decision:

```bash
okx market ticker ETH-USDT-SWAP
okx market candles ETH-USDT-SWAP --bar 15m --limit 120
okx market candles ETH-USDT-SWAP --bar 1H --limit 100
okx market candles ETH-USDT-SWAP --bar 4H --limit 60
okx market indicator ema ETH-USDT-SWAP --bar 1H --params 20
okx market indicator ema ETH-USDT-SWAP --bar 1H --params 50
okx market indicator ema ETH-USDT-SWAP --bar 4H --params 20
okx market indicator ema ETH-USDT-SWAP --bar 4H --params 50
okx market indicator rsi ETH-USDT-SWAP --bar 1H --params 14
okx market indicator macd ETH-USDT-SWAP --bar 1H --params 12,26,9
okx market indicator atr ETH-USDT-SWAP --bar 1H --params 14
okx market indicator bb ETH-USDT-SWAP --bar 1H --params 20,2
okx market funding-rate ETH-USDT-SWAP
okx market open-interest --instType SWAP --instId ETH-USDT-SWAP
okx account balance --ccy USDT --profile <profile_name>
okx account positions --instType SWAP --instId ETH-USDT-SWAP --profile <profile_name>
```

Output a compact snapshot:

```markdown
## Market Snapshot
- Price:
- 15m structure:
- 1H structure:
- 4H structure:
- RSI(14):
- MACD state:
- ATR:
- Bollinger location:
- Funding:
- Open interest:
- Current position:
```

## Step 2 情绪评估

Translate the market into one of four states:

| State | Meaning | Trading posture |
|---|---|---|
| `trend-long` | 1H and 4H both bullish | look for pullback long |
| `trend-short` | 1H and 4H both bearish | look for pullback short |
| `range` | 1H mixed, 4H flat, BB contained | fade edges with smaller size |
| `no-trade` | mixed or unstable | do nothing |

### Direction Score

Use the following scoring model:

| Signal | Long | Short |
|---|---:|---:|
| 4H EMA20 > EMA50 | +2 | 0 |
| 4H EMA20 < EMA50 | 0 | +2 |
| 1H EMA20 > EMA50 | +1 | 0 |
| 1H EMA20 < EMA50 | 0 | +1 |
| MACD positive and rising | +1 | 0 |
| MACD negative and falling | 0 | +1 |
| RSI 40-60 | +0.5 to the leading direction only | +0.5 to the leading direction only |
| Funding too crowded long | 0 | +1 |
| Funding too crowded short | +1 | 0 |

Interpretation:

- long score >= 3 and short score <= 1 -> `trend-long`
- short score >= 3 and long score <= 1 -> `trend-short`
- neither side wins, but BB range is clean -> `range`
- otherwise -> `no-trade`

RSI handling note:

- if the long side is already leading, RSI 40-60 can strengthen the long case slightly
- if the short side is already leading, RSI 40-60 can strengthen the short case slightly
- never add neutral-zone RSI points to both sides at the same time

Always output:

```markdown
## Regime Assessment
- State:
- Long score:
- Short score:
- Confidence: low / medium / high
- Best action: long / short / range fade / no-trade
```

## Step 3 AI 判断逻辑

### 3.1 开仓逻辑

#### Trend Long

Open long only if all are true:

- regime = `trend-long`
- price is above 4H EMA20
- 15m or 1H pulls back without breaking key support
- RSI is between `40` and `62`
- funding is not extremely positive
- no existing position

#### Trend Short

Open short only if all are true:

- regime = `trend-short`
- price is below 4H EMA20
- 15m or 1H bounces into resistance and fails
- RSI is between `38` and `60`
- funding is not extremely negative
- no existing position

#### Range Long

Open small long only if all are true:

- regime = `range`
- price is near Bollinger lower band or recent support
- RSI < `35`
- open interest is not expanding aggressively downward
- no existing position

#### Range Short

Open small short only if all are true:

- regime = `range`
- price is near Bollinger upper band or recent resistance
- RSI > `65`
- open interest is not expanding aggressively upward
- no existing position

#### No-Trade

Do not trade when any are true:

- regime = `no-trade`
- price is between key levels without edge
- ATR suddenly expands without structure
- funding is extreme
- account drawdown protection is active

### 3.2 平仓逻辑

Close the full position when any are true:

- stop-loss is hit
- take-profit target is hit
- trend setup invalidates on 1H close
- regime flips strongly against the position
- end-of-day risk cap is hit

### 3.3 减仓逻辑

Reduce 50% when:

- position reaches first target around `+3%` to `+4%`
- RSI becomes stretched
- momentum weakens while still in profit

After partial take profit, remaining size can run toward second target only if structure still supports it.

### 3.3A Re-entry Logic

Allow one controlled re-entry after a stop-out only when all are true:

- the previous trade was stopped out recently
- regime is still `trend-long` or `trend-short`
- direction score still supports the same side
- the higher-timeframe thesis remains intact
- daily loss guard has not been hit

Re-entry rules:

- maximum re-entry count per direction signal: `1`
- re-entry size = `75%` of the original entry size
- re-entry must use the same or lower leverage than the first attempt
- do not re-enter range-fade setups after a stop-out
- if the re-entry also fails, stop trading that direction until a new regime appears

### 3.4 加仓逻辑

Only add once, and only when all are true:

- existing position is already in profit by at least `+2%`
- original direction remains valid
- pullback retest holds
- total open notional remains within risk cap

Add size must be at most `25%` of the initial USDT size. Never add to a losing trade.

## Step 4 下单执行

### 4.1 Position Sizing Model

Use setup quality and available balance to determine size:

| Setup Quality | Leverage | Suggested Capital Usage |
|---|---:|---:|
| Range fade | 2x to 3x | 8% to 12% of available USDT |
| Normal trend | 3x | 12% to 20% of available USDT |
| Strong trend continuation | 4x max | 20% to 25% of available USDT |

Sizing rules:

- if account equity is unknown, do not place orders
- do not exceed one open ETH perpetual position at a time
- reduce new size after drawdown or unstable market conditions
- if the user specifies a notional amount in USDT, use that value directly with `--tgtCcy quote_ccy`
- if sizing is inferred automatically, round down and stay conservative

Automatic sizing ladder:

- `range` setup -> use `10%` of available USDT
- `trend-long` / `trend-short` normal quality -> use `15%` of available USDT
- highest-quality continuation -> use `20%` of available USDT
- defensive mode -> cap new entries at `8%` of available USDT

Hard caps:

- never exceed `25%` of available USDT on a fresh entry
- never exceed one add-on per position
- add-on size capped at `25%` of the initial notional size

### 4.2 Set Leverage

Use the official command style:

```bash
okx swap leverage --instId ETH-USDT-SWAP --lever 3 --mgnMode cross --profile <profile_name>
```

### 4.3 Open Position

Use `--tgtCcy quote_ccy` so the order size is expressed in `USDT`, which is safer and easier to control.

If your local OKX CLI environment uses audit tags, you may append:

```bash
--tag agentTradeKit
```

The exact tag handling can vary by environment, so treat it as an optional integration detail rather than a mandatory public CLI requirement.

### 4.4 Execution Check

Before any write command, output:

```markdown
## Execution Check
- Profile:
- posMode:
- Pair: ETH-USDT-SWAP
- Regime:
- Direction:
- Entry type:
- Available USDT:
- Order size in USDT:
- Leverage:
- Stop-loss:
- Take-profit:
- Reason to trade:
- Invalidation:
```

If any of the above is missing, do not place the order.

TP/SL calculation template:

- long stop-loss = `entryPx - ATR(1H) x 1.5`
- long take-profit 1 = `entryPx + ATR(1H) x 2.5`
- long take-profit 2 = `entryPx + ATR(1H) x 3.0`
- short stop-loss = `entryPx + ATR(1H) x 1.5`
- short take-profit 1 = `entryPx - ATR(1H) x 2.5`
- short take-profit 2 = `entryPx - ATR(1H) x 3.0`

Only place an order after converting the current ATR into explicit trigger prices.

Long example:

```bash
okx swap place --instId ETH-USDT-SWAP \
  --side buy \
  --posSide long \
  --ordType market \
  --sz <usdt_size> \
  --tgtCcy quote_ccy \
  --tdMode cross \
  --tpTriggerPx <tp_price> --tpOrdPx=-1 \
  --slTriggerPx <sl_price> --slOrdPx=-1 \
  --profile <profile_name>
```

Short example:

```bash
okx swap place --instId ETH-USDT-SWAP \
  --side sell \
  --posSide short \
  --ordType market \
  --sz <usdt_size> \
  --tgtCcy quote_ccy \
  --tdMode cross \
  --tpTriggerPx <tp_price> --tpOrdPx=-1 \
  --slTriggerPx <sl_price> --slOrdPx=-1 \
  --profile <profile_name>
```

### 4.5 Reduce Position

For partial take profit, reduce using a smaller market order in the opposite direction:

```bash
okx swap place --instId ETH-USDT-SWAP \
  --side sell \
  --posSide long \
  --ordType market \
  --sz <reduce_usdt_size> \
  --tgtCcy quote_ccy \
  --tdMode cross \
  --profile <profile_name>
```

Mirror the logic for short positions.

Precision note:

- when reducing, prefer reading the actual live contract size from `account positions`
- if precise half-close behavior matters, use half of the real position size directly instead of relying on a fresh `quote_ccy` conversion
- use `quote_ccy` reduction only when exact contract sizing is unavailable and a small rounding difference is acceptable

### 4.6 Close Position

Use the official close command:

```bash
okx swap close --instId ETH-USDT-SWAP --mgnMode cross --posSide long --profile <profile_name>
okx swap close --instId ETH-USDT-SWAP --mgnMode cross --posSide short --profile <profile_name>
```

### 4.7 Trailing Stop And Stop Upgrade

If a position is already open and profit expands meaningfully, upgrade protection rather than widening risk.

Use this flow:

```bash
okx swap algo orders --instId ETH-USDT-SWAP --profile <profile_name>

okx swap algo amend \
  --instId ETH-USDT-SWAP \
  --algoId <algo_id> \
  --newSlTriggerPx <new_stop_price> \
  --newSlOrdPx=-1 \
  --profile <profile_name>
```

Upgrade rules:

- when unrealized profit reaches roughly `+1R`, move stop closer to entry
- when first take-profit is taken, move stop to breakeven or small profit
- do not loosen the stop once tightened

For fresh protection on an existing position, use a close-only algo order:

```bash
okx swap algo place --instId ETH-USDT-SWAP \
  --side <buy|sell> \
  --ordType conditional \
  --sz <usdt_size_or_contracts> \
  --tdMode cross \
  --posSide <long|short> \
  --reduceOnly \
  --slTriggerPx <sl_price> --slOrdPx=-1 \
  --profile <profile_name>
```

Rules:

- for a long position, protection side must be `sell`
- for a short position, protection side must be `buy`
- `--reduceOnly` must be present for protection algos
- `--sz` must not exceed the actual live position size
- if position data cannot be verified from OKX, do not place or amend the protection order

### 4.8 Execution Verification

After every write action, verify:

```bash
okx swap orders --instId ETH-USDT-SWAP --profile <profile_name>
okx account positions --instType SWAP --instId ETH-USDT-SWAP --profile <profile_name>
okx swap fills --instId ETH-USDT-SWAP --profile <profile_name>
```

For audit and troubleshooting, later review bills and closing records from OKX to confirm the position was opened through Agent Trade Kit in your environment.

## Step 5 风控规则

### 5.1 单笔风险

- every trade must define stop-loss before entry
- risk per trade should target roughly `1.5%` to `2.5%` of account equity
- first target should generally be at least `1.5R`
- avoid entries where stop distance is too wide relative to expected target

### 5.2 账户级风险

| Rule | Threshold | Action |
|---|---|---|
| Max daily loss | 6% of account equity | stop opening new trades for the day |
| Max total drawdown | 12% from recent peak | reduce to defensive mode |
| Hard stop mode | 15% from recent peak | pause all new trades temporarily |
| Consecutive losses | 3 | next trade size cut by 50% |
| Winning streak | 3 | do not increase leverage automatically |

### 5.3 Defensive Mode

Switch to defensive mode when:

- equity drops materially from recent peak
- 2 losses occur within one session
- market enters unstable no-trade conditions

Defensive mode rules:

- leverage capped at `2x`
- entry size capped at `8%` of available USDT
- range trades disabled
- only A-grade trend setups allowed

Exit defensive mode when either is true:

- two consecutive profitable trades are completed
- account equity recovers to at least `95%` of peak balance

### 5.4 Emergency Exit

Immediately flatten if any of these happen:

- exchange data and local position understanding diverge materially
- unrealized loss exceeds expected stop-loss due to gap or fast move
- API errors make position management unreliable

### 5.5 Time Stop

If a position remains open for too long without meaningful progress, prefer capital efficiency over hope.

Time-stop rule:

- if holding time exceeds `8 hours`
- and unrealized PnL stays between `-0.5%` and `+0.5%`
- and no fresh regime improvement appears

Then close the position unless a compelling new signal has appeared in the current round.

## Local State

Mandatory local file:

`~/.okx-strategy/eth-swap-state.json`

Initialize the directory first if needed:

```bash
mkdir -p ~/.okx-strategy
```

Track lightweight but actionable fields:

```json
{
  "initialBalance": null,
  "peakBalance": null,
  "currentBalance": null,
  "currentMode": "normal",
  "consecutiveLosses": 0,
  "dailyLoss": 0,
  "reEntryCount": 0,
  "reEntryDirection": null,
  "lastTradeDate": null,
  "lastRegime": null,
  "lastAction": null,
  "marketState": {
    "state": null,
    "longScore": null,
    "shortScore": null,
    "confidence": null
  },
  "position": {
    "side": null,
    "entryPx": null,
    "lever": null,
    "sizeMode": null,
    "sizeValue": null,
    "stopLossPx": null,
    "takeProfitPx": null,
    "algoId": null
  },
  "tradeHistory": [
    {
      "time": null,
      "action": null,
      "price": null,
      "size": null,
      "pnl": null,
      "regime": null
    }
  ]
}
```

If the file is missing, re-initialize it. If local state conflicts with OKX position data, trust OKX.

State maintenance rules:

- reset `dailyLoss` to `0` at the start of each new UTC day
- use `lastTradeDate` to detect day boundaries
- when regime or direction changes, reset `reEntryCount` to `0` and `reEntryDirection` to `null`
- after a full close, set all `position` fields to `null` and append the completed trade to `tradeHistory`
- keep `tradeHistory` entries structurally consistent across rounds

Update these fields after each round:

- `marketState` after regime assessment
- `position` after any open, reduce, stop amendment, or close
- `tradeHistory` after every completed trade
- `peakBalance` whenever equity makes a new high
- `reEntryCount` and `reEntryDirection` for the active regime and direction
- `lastTradeDate` whenever a trade is opened, reduced, or closed

## Main Prompt Template

Use prompts like:

`检查 ETH-USDT-SWAP 的 15m/1H/4H 结构、资金费率、持仓和波动，如果满足这份 skill 的 A 级开仓条件，就用 <profile_name> 账户执行一笔用户指定或按余额保守推导的 3x 趋势单并带止盈止损；如果不满足，就输出 no-trade 和原因。`

`执行一轮 ETH 永续策略。先给 Market Snapshot 和 Regime Assessment，再决定 long short 还是 no-trade。`

## Review / Recap

When asked to review results:

```bash
okx account balance --ccy USDT --profile <profile_name>
okx account positions --instType SWAP --instId ETH-USDT-SWAP --profile <profile_name>
okx swap fills --instId ETH-USDT-SWAP --limit 100 --archive --profile <profile_name>
```

Then summarize:

- total PnL
- win rate
- average winner vs average loser
- best regime
- worst mistake
- whether size or leverage should be adjusted

## Execution Frequency

- no position: every `1H`
- open position: every `30m`
- high volatility and active position: every `15m`

Do not force trades just because a round is scheduled. `No-trade` is a valid winning action.
