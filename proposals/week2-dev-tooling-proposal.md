# Week 2 Proposal: AI-Powered Smart Contract Analyzer

**学员**: Always-Stephen  
**Week**: 2 — AI × Web3 交叉研究与方向选择  
**方向**: Dev Tooling / Agent Workflow  
**日期**: 2026-05-26  
**状态**: 初稿（待提交）

---

## 方向选择

**我选择的方向：Dev Tooling / Agent Workflow**

选择理由：
1. 我已经在用 Hermes Agent（一个 MCP 驱动的 AI Agent），可以直接在这个平台上做扩展
2. Dev Tooling 产出的工具是**可用的实际产品**，适合 Hackathon 展示
3. 跟我的技能背景（会基础脚本，用过 AI 工具）最匹配
4. 合约分析是 Web3 开发者的刚需，市场验证充分

---

## 项目构想：Smart Contract Analyzer MCP Server

### 一句话描述

一个 MCP Server，让 AI Agent 能自动拉取、解析、解释以太坊智能合约，输出人类可读的分析报告。

### 核心功能

| 功能 | 输入 | 输出 | 实现方式 |
|------|------|------|---------|
| 合约信息查询 | 合约地址 | 合约名、ABI、源码、部署信息 | Etherscan API |
| 函数逐行解释 | 合约地址 + 函数名 | 自然语言解释每个函数的作用 | LLM + ABI 解析 |
| 安全初筛 | 合约地址 | 风险提示（可增发、可暂停、owner特权） | 规则引擎 + ABI 模式匹配 |
| 调用流程图 | 合约地址 | Mermaid 流程图（合约间调用关系） | ABI 分析 + 图生成 |

### 技术栈

```
用户 (Hermes Agent)
    │ MCP Protocol (JSON-RPC 2.0)
    ▼
┌─────────────────────────────────┐
│  Smart Contract Analyzer        │
│  (MCP Server — Python)          │
│                                 │
│  ┌─────────┐  ┌──────────┐     │
│  │ Etherscan│  │  ABI     │     │
│  │ API 拉取 │  │ 解析器    │     │
│  └─────────┘  └──────────┘     │
│  ┌─────────┐  ┌──────────┐     │
│  │ 安全规则 │  │ 报告生成  │     │
│  │ 引擎     │  │ (Markdown)│    │
│  └─────────┘  └──────────┘     │
└─────────────────────────────────┘
```

### MCP Tools 设计

```json
{
  "tools": [
    {
      "name": "get_contract_info",
      "description": "获取合约基本信息（名称、ABI、编译器版本、源码）",
      "input": {"address": "0x..."}
    },
    {
      "name": "explain_function",
      "description": "用自然语言解释合约中指定函数的作用",
      "input": {"address": "0x...", "function_name": "swapExactTokensForTokens"}
    },
    {
      "name": "audit_check",
      "description": "对合约做快速安全扫描，输出风险提示",
      "input": {"address": "0x..."}
    },
    {
      "name": "generate_call_graph",
      "description": "生成合约间的调用关系图",
      "input": {"address": "0x..."}
    }
  ]
}
```

### 使用演示（预期效果）

```
User: 帮我分析 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D

Hermes (通过 MCP Server):
→ 合约: UniswapV2Router02
→ 编译器: Solidity 0.6.6
→ 36 个函数，6 大类：
   - 6 个 swap 函数（兑换 Token）
   - 2 个 addLiquidity 函数（添加流动性）
   - 2 个 removeLiquidity 函数（移除流动性）
   - 3 个只读查询函数
   - ...
→ 安全初筛：
   ✅ 无代理模式（不可升级）
   ✅ 无 owner 特权函数
   ⚠️ deadline 参数需注意 MEV 风险
```

---

## 为什么这个方向有价值？

### 1. 市场需求
- Etherscan 是 Web3 开发者每天必开的网站
- 但读合约源码仍然需要专业知识（Solidity、EVM、安全模式）
- AI 可以把「读合约」的门槛从「需要 Solidity 知识」降到「有地址就行」

### 2. 跟 MCP/A2A 的结合
- **MCP**：这是项目的核心——合约分析器作为一个 MCP Server，任意 AI Agent 都能调用
- **A2A**（未来扩展）：多个 Agent 协作——分析 Agent + 审计 Agent + 报告 Agent

### 3. Hackathon 友好
- 2-3 天能出 MVP（Etherscan API + ABI 解析 + LLM 解释）
- Demo 效果好（输入地址 → 输出报告，直观）
- 有实际用户价值

---

## Week 2 执行计划

| 日期 | 任务 |
|------|------|
| 5/26 (今天) | ✅ 方向确定 + Proposal 初稿 |
| 5/27 (周三) | 搭建 MCP Server 骨架（Python + MCP SDK），接入 Etherscan API |
| 5/28 (周四) | 实现 `get_contract_info` + `explain_function` 两个 Tool |
| 5/29 (周五) | Week 2 例会提交 + 展示 Demo |

---

## 风险和缓解

| 风险 | 缓解方案 |
|------|---------|
| Etherscan API 限流（5次/秒免费） | 加缓存层，重复查询不过 API |
| 合约未开源（无源码） | 降级到 ABI-only 分析（函数签名 + 输入输出） |
| 复杂合约（几百个函数）解释太啰嗦 | 分优先级：先解释高频交互函数 |

---

## 学习目标

通过这个项目，我预期学会：
1. MCP 协议的实际编程（一个真实可用的 MCP Server）
2. 以太坊智能合约的常见模式和安全隐患
3. 如何把 AI Agent 用到 Web3 开发工作流中

---

## 反馈请在此留言

> (待 WCB 导师 / 同学反馈)
