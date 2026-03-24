---
name: okx-meme-antirug
description: >
  Use when user wants to scan, filter, buy, and auto-monitor Meme tokens on Solana/BSC.
  Triggers: meme 防rug、反rug、扫链买币、新币扫描、meme 定投、meme antirug、自动买 meme、Solana 新币、pump.fun 扫币、meme 过滤、翻倍出本、自动止损买币、监控价格自动卖出。
---

# okx-meme-antirug — Meme 防 Rug 过滤器 & 定投 Skill

## 策略概述

扫描链上新发行的 Meme 代币 → 多层防 Rug 过滤 → 自动买入 100 USDT → 翻倍卖出 50% 收回本金 → 剩余仓位零成本持有。

**依赖上游 Skills：**
- \`okx-dex-trenches\`：扫描新币、Dev 分析、Bundle 检测
- \`okx-dex-swap\`：报价查询、执行买卖 Swap
- \`okx-dex-market\`：实时价格监控、PnL 查询
- \`okx-agentic-wallet\`：签名 & 广播交易

---

## 完整工作流

### 阶段一：发现新代币

\`\`\`
onchainos memepump tokens \\
  --chain solana \\
  --stage NEW \\
  --has-x \\
  --has-telegram
\`\`\`

- 优先扫描 Solana 链（Meme 最活跃），备选 BSC
- 筛选条件：刚发布（NEW）、有 Twitter/X、有 Telegram
- 每次取前 20 个候选代币，逐一进入过滤阶段

---

### 阶段二：Anti-Rug 过滤评分

任意硬过滤不通过 → 直接跳过该代币，不浪费资金。

#### 【硬过滤 — 一票否决】

**① Dev 信誉检查**
\`\`\`
onchainos memepump token-dev-info --address <tokenAddress> --chain solana
\`\`\`
- \`rugPullCount\` 必须 = 0（任何跑路记录直接淘汰）
- \`abandonedCount\` ≤ 1

**② Bundle / Sniper 检测**
\`\`\`
onchainos memepump token-bundle-info --address <tokenAddress> --chain solana
\`\`\`
- Bundle 持仓占比 < 15%（超过说明机构砸盘风险大）
- Sniper 数量 < 5 个

**③ 代币详情检查**
\`\`\`
onchainos memepump token-details --address <tokenAddress> --chain solana
\`\`\`
- \`top10HoldingsPercent\` < 50%
- Bonding Curve 进度：10% ~ 85%

**④ 蜜罐验证（通过 Swap Quote）**
\`\`\`
onchainos swap quote \\
  --from <tokenAddress> \\
  --to 11111111111111111111111111111111 \\
  --amount <小额测试量> \\
  --chain solana
\`\`\`
- \`isHoneyPot\` = false
- \`taxRate\` < 10%

---

#### 【软评分 — 100 分制，≥ 60 分才买入】

| 维度 | 满分 | 规则 |
|------|------|------|
| 社交信号 | 20 | 有 Twitter 粉丝>1000 +15，有 Telegram +5 |
| Dev 历史 | 20 | 有过 ≥1 个成功项目（非跑路）+20 |
| 持仓分散度 | 20 | top10 < 30% +20，< 40% +10，< 50% +5 |
| Bonding Curve | 20 | 进度 30%-60% 满分，10-30% 或 60-85% 得 10 |
| Bundle 洁净度 | 20 | 占比 < 5% +20，< 10% +10，< 15% +5 |

> 综合评分 ≥ 60 分 → 进入买入阶段；< 60 分 → 跳过并记录原因

---

### 阶段三：买入执行（100 USDT）

Solana 链上 USDT 合约地址：\`Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB\`
wSOL 地址：\`So11111111111111111111111111111111111111112\`

\`\`\`bash
# Step 1: 获取买入报价（确认流动性 & 价格影响）
onchainos swap quote \\
  --from Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB \\
  --to <tokenAddress> \\
  --amount 100 \\
  --chain 501

# Step 2: 执行 Swap（meme 币建议滑点 8%）
onchainos swap swap \\
  --from Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB \\
  --to <tokenAddress> \\
  --amount 100 \\
  --chain 501 \\
  --wallet <yourWalletAddress> \\
  --slippage 8
\`\`\`

**买入成功后，立即执行阶段四（写入状态 + 创建监控任务）。**

---

### 阶段四：写入持仓状态 & 创建自动监控任务

买入成功后，从 swap 返回值中提取 \`entryPrice\`（fromToken.tokenUnitPrice）和 \`tokenAmount\`（toTokenAmount，UI 单位）。

#### 4a. 写入状态文件

\`\`\`bash
mkdir -p ~/.onchainos
cat > ~/.onchainos/meme-trade-<tokenSymbol>.json << EOF
{
  "tokenAddress": "<tokenAddress>",
  "symbol": "<tokenSymbol>",
  "chain": "501",
  "walletAddress": "<yourWalletAddress>",
  "entryPrice": <entryPrice>,
  "tokenAmount": "<tokenAmount>",
  "costBasis": 100,
  "entryTime": "<ISO8601>",
  "status": "active"
}
EOF
\`\`\`

#### 4b. 创建每 5 分钟自动监控的定时任务

使用 \`mcp__scheduled-tasks__create_scheduled_task\` 工具创建监控任务：

- **taskId**：\`meme-monitor-<tokenSymbol>\`
- **cronExpression**：\`*/5 * * * *\`（每 5 分钟）
- **prompt**：使用下方《监控任务 Prompt 模板》

> 如果 scheduled-tasks MCP 不可用，告知用户需手动回来确认价格。

---

### 阶段五：自动监控逻辑（定时任务运行时执行）

**监控任务 Prompt 模板**：

\`\`\`
## Meme Token 价格监控 & 自动出场

### 第一步：读取状态
读取 ~/.onchainos/meme-trade-<SYMBOL>.json。
- status = "waiting_entry" → 跳过
- status = "closed" → 跳过
- status = "active" 或 "partial_exit" → 继续

### 第二步：获取当前价格
onchainos swap quote \\
  --from <TOKEN_ADDRESS> \\
  --to So11111111111111111111111111111111111111112 \\
  --amount 1000000 \\
  --chain 501

### 第三步：出场判断

【翻倍出场】currentPrice >= entryPrice * 2.0：卖出 50%
【止损清仓】currentPrice <= entryPrice * 0.5：全部卖出
【72小时超时】提醒用户手动决策
【正常持有】继续持有
\`\`\`

**出场触发条件速查：**

| 条件 | 操作 | 滑点 |
|------|------|------|
| 价格 ≥ 入场价 × 2.0 | 卖出 50% | 10% |
| 价格 ≤ 入场价 × 0.5 | 全部止损 | 15% |
| 持仓 > 72 小时 | 提醒用户 | — |

---

## 收益逻辑（预期）

\`\`\`
假设 10 笔交易，每笔 100 USDT，过滤后胜率约 30%：
翻倍的 3 笔：卖出 50% 收回本金，剩余持仓若再涨 3x → 额外 +150 USDT/笔
止损的 7 笔：每笔最多亏 50 USDT
净收益预期 ≈ +100 USDT（总投入 1000 USDT）
\`\`\`

---

## 适用链

| 链 | Chain ID | 推荐度 |
|----|----------|--------|
| Solana | 501 | 主要 |
| BSC | 56 | 备选 |
| TRON | 195 | 可选 |

---

## 风险提示

> Meme 代币高风险，本 Skill 过滤系统可降低但无法消除 Rug 风险。
> 建议单钱包总仓位控制在可承受亏损范围内。
> 链上交易不可逆，执行前请确认钱包余额和 Gas 充足。
> 本 Skill 不构成投资建议，盈亏自负。
