# Uniswap V2 Router 合约分析

**合约地址**: `0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D`  
**合约名**: UniswapV2Router02  
**部署网络**: Ethereum Mainnet  
**分析日期**: 2026-05-26  
**分析工具**: Hermes Agent (手动分析 + 已有知识)

---

## 一句话概括

这是 Uniswap V2 的「用户入口合约」。你不需要直接跟底层的交易对合约（Pair）交互，通过 Router 就能完成**兑换、添加流动性、移除流动性**的所有操作。

---

## 合约在 Uniswap 生态中的位置

```
    用户/前端
        │
        ▼
┌───────────────────────┐
│  UniswapV2Router02    │  ← 这个合约！
│  (0x7a25...F2488D)    │
└──────┬────────────────┘
       │ 调用
       ▼
┌──────────────┐    ┌──────────────┐
│ UniswapV2Pair │    │ UniswapV2Pair │  (USDC/ETH, DAI/ETH, ...)
│   (USDC/ETH)  │    │   (DAI/ETH)   │
└──────────────┘    └──────────────┘
       │
       ▼
┌──────────────┐
│ UniswapV2Factory│  (创建新的 Pair)
└──────────────┘
       │
       ▼
┌──────────────┐
│     WETH     │  (ETH 的 ERC-20 包装版本)
└──────────────┘
```

**Router 自己不存钱**——每次操作前需要先 approve，操作后资金回到你的地址。

---

## 核心函数分组

### 1. 兑换（Swap）— 最常用的功能

| 函数 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `swapExactTokensForTokens` | 我要花多少TokenA | 最少收到多少TokenB | 精确输入，滑点保护 |
| `swapTokensForExactTokens` | 最少花多少TokenA | 我必须要多少TokenB | 精确输出 |
| `swapExactETHForTokens` | ETH → Token | 最少收到多少Token | 用 ETH 买 Token |
| `swapExactTokensForETH` | Token → ETH | 最少收到多少ETH | 卖 Token 换 ETH |
| `swapTokensForExactETH` | Token → ETH | 必须是这么多ETH | 精确 ETH 数量 |
| `swapETHForExactTokens` | ETH → Token | 必须是这么多Token | 精确 Token 数量 |

**核心算法**：`x * y = k`（恒定乘积做市商）
```
TokenA * TokenB = k（常数）
价格 = TokenB数量 / TokenA数量
```

### 2. 添加流动性（Add Liquidity）

| 函数 | 说明 |
|------|------|
| `addLiquidity` | 添加 TokenA + TokenB，收到 LP Token |
| `addLiquidityETH` | 添加 Token + ETH，收到 LP Token |

LP Token 代表你在池子里的份额，可以拿去质押挖矿。

### 3. 移除流动性（Remove Liquidity）

| 函数 | 说明 |
|------|------|
| `removeLiquidity` | 烧毁 LP Token，拿回 TokenA + TokenB |
| `removeLiquidityETH` | 烧毁 LP Token，拿回 Token + ETH |

### 4. 辅助函数

| 函数 | 说明 |
|------|------|
| `getAmountsOut` | **只读查询**：输入一定量TokenA，能换多少TokenB |
| `getAmountsIn` | **只读查询**：要得到一定量TokenB，需要多少TokenA |
| `quote` | 给定等值流动性，计算两个 Token 的数量 |
| `factory()` / `WETH()` | 返回关联的 Factory 和 WETH 地址 |

---

## 一个真实的 swap 调用会发生什么？

```
swapExactETHForTokens(
  amountOutMin: 1000000,        // 最少收到 100 万个小单位（防滑点）
  path: [WETH, USDC],           // 兑换路径：ETH → USDC
  to: 0xStephen的地址,           // 收到 USDC 的地址
  deadline: 1716799999           // 超过这个时间放弃（防 pending 交易被抢跑）
)
{value: 1 ETH}                  // 附带 1 ETH

执行流程：
1. Router 把 1 ETH 包装成 1 WETH
2. Router 把 1 WETH 发给 WETH/USDC Pair
3. Pair 用 x*y=k 算出应该给多少 USDC
4. Pair 把 USDC 发给 Router
5. Router 把 USDC 转发给 to 地址
6. 完成！
```

---

## 安全设计要点

| 机制 | 解决什么问题 |
|------|-------------|
| `amountOutMin` | **滑点保护**：如果池子价格波动太大，交易失败 |
| `deadline` | **抢跑保护**：如果交易在 mempool 里待太久，过期不执行 |
| `path[]` | **多跳路由**：ETH → USDC → DAI，一次完成跨币兑换 |
| `to` 参数 | 输出地址和调用者可以不同（支持合约间调用） |

---

## 为什么选这个合约来读？

1. **它是以太坊上被调用最多的合约之一**（几千万次交互）
2. **结构清晰**：Router → Factory → Pair 三层架构，是智能合约设计的经典案例
3. **跟 Week 2 方向直接相关**：你学 Dev Tooling，理解 DEX 架构是基本功。未来你可以写一个 MCP Tool 来自动分析这类合约。
4. **代码可验证**：Etherscan 上开源，Solidity 0.6.6，~400 行

---

## 如果你是 Dev Tooling Agent，你能做什么？

- 写一个 MCP Server，暴露 `explain_contract(address)` 工具
- 自动拉取 Etherscan 源码 → 解析 ABI → 解释每个函数
- 或者更高级：用 Slither 对合约做静态分析，输出安全报告
- 这就是你 Week 2 proposal 的潜在方向！

---

## 延伸

- Uniswap V3 用集中流动性替代了 x*y=k
- Uniswap V4 引入了 Hook 机制（Singleton 合约 + Hook 合约）
- 这些演进本身就是很好的 Week 2 研究方向
