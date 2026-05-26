# MCP vs A2A — 深度对比

**日期**: 2026-05-26  
**来源**: MCP 官方文档 (modelcontextprotocol.io) + A2A 官方 README (github.com/a2aproject/A2A)  
**作者**: Stephen + Hermes Agent

---

## 一句话总结

| | MCP | A2A |
|------|-----|-----|
| **全称** | Model Context Protocol | Agent2Agent Protocol |
| **谁做的** | Anthropic (2024.11) | Google (2025.04) |
| **解决什么** | Agent 怎么调用工具和数据 | Agent 之间怎么协作 |
| **类比** | USB-C 协议——统一外设接口 | HTTP 协议——统一服务间通信 |
| **方向** | **纵向**：Agent ↔ Tool | **横向**：Agent ↔ Agent |

---

## 架构对比

```
MCP 架构（Agent ↔ Tool）:
┌─────────────┐     JSON-RPC      ┌──────────────┐
│  MCP Client  │ ◄──────────────► │  MCP Server   │
│  (你的Agent) │   (stdio/HTTP)   │  (Tool/API)   │
└─────────────┘                   └──────────────┘
  暴露: 无                          暴露: Tools, Resources, Prompts
  看到: 工具的完整接口              看到: 无（不感知Agent）

A2A 架构（Agent ↔ Agent）:
┌─────────────┐     JSON-RPC      ┌──────────────┐
│  A2A Client  │ ◄──────────────► │  A2A Server   │
│  (Agent A)   │   (HTTP + SSE)   │  (Agent B)    │
└─────────────┘                   └──────────────┘
  暴露: Task请求 + 输入数据         暴露: Agent Card（能力描述）
  看到: Agent B 的公开能力          看到: Agent A 的 Task 状态
```

**关键区别**：MCP Server 把自己完全暴露给 Agent（Tools/Resources/Prompts），A2A Server 只暴露一张「Agent Card」——内部 memory、tool、逻辑完全隐藏。

---

## 协议层面

| 维度 | MCP | A2A |
|------|-----|-----|
| **传输** | stdio（本地）/ HTTP SSE（远程） | HTTP(S) + SSE |
| **RPC** | JSON-RPC 2.0 | JSON-RPC 2.0 |
| **发现机制** | 无（客户端硬编码 server） | **Agent Card** — 自描述能力 |
| **核心原语** | `tools/list`, `tools/call`, `resources/read`, `prompts/get` | `tasks/send`, `tasks/get`, `tasks/cancel` |
| **流式** | 支持 | 支持（SSE + Push Notification） |
| **安全** | 依赖传输层（OAuth 2.0 可选） | 内建认证、授权方案 |
| **状态** | 无状态（每次调用独立） | 有状态（Task 生命周期管理） |
| **规范成熟度** | 2025-03 发布 1.0 spec | 仍在积极开发中 |

---

## 核心概念拆解

### MCP — 三个原语

1. **Tools**：Agent 可调用的函数。比如 `read_file`、`execute_code`、`web_search`。
   - 类比：函数签名 + 描述
   - Hermes 的工具集就是典型的 MCP Tools

2. **Resources**：Agent 可读取的上下文数据。比如文件内容、数据库记录。
   - 类比：只读文件系统
   - `resource://docs/api-reference` → 返回 API 文档内容

3. **Prompts**：预定义的提示模板，帮助用户和 Agent 交互。
   - 类比：Slash Command 模板
   - `/summarize` → 展开成完整 prompt

### A2A — 任务驱动

1. **Agent Card**：每个 A2A Agent 的公开展示页。
   ```json
   {
     "name": "CodeReviewAgent",
     "description": "Reviews pull requests",
     "url": "https://agent.example.com",
     "capabilities": {"streaming": true, "pushNotifications": true},
     "skills": [{"id": "code_review", "name": "Code Review"}]
   }
   ```

2. **Task**：A2A 的工作单元，有完整生命周期：
   ```
   submitted → working → input-required → working → completed
                     ↘ failed
                     ↘ canceled
   ```
   - 支持长时间运行（可能几小时）
   - 支持中间状态查询
   - 支持流式输出 + 推送通知

3. **协作模式**：
   - **Sequential**：Agent A 完成任务 → 输出传给 Agent B
   - **Hierarchical**：Orchestrator Agent 分派子任务给 Specialist Agents
   - **Peer-to-peer**：两个 Agent 对等协商

---

## 什么时候用哪个？

| 场景 | 用 MCP | 用 A2A |
|------|--------|--------|
| Agent 需要调用一个 API | ✅ | ❌ |
| Agent 需要读取本地文件 | ✅ | ❌ |
| 让两个不同团队的 Agent 协作 | ❌ | ✅ |
| Orchestrator 分派任务给多个 Specialist | 可以但不优雅 | ✅ |
| 连接 GitHub、Slack、数据库 | ✅ MCP Server | ❌ |
| 构建多 Agent 工作流（如代码审查 pipeline） | ❌ | ✅ |
| Hermes 调你的工具 | ✅ 你已经在用 | ❌ |

---

## 互补关系

MCP 和 A2A **不是竞争关系，是互补关系**：

```
         A2A (Agent ↔ Agent)
    ┌──────────────────────────────┐
    │                              │
    ▼                              ▼
 ┌──────┐  MCP   ┌──────┐      ┌──────┐  MCP   ┌──────┐
 │Agent A│◄─────►│Tools │      │Agent B│◄─────►│Tools │
 └──────┘       └──────┘      └──────┘       └──────┘
```

- **MCP**：每个 Agent 通过 MCP 获得工具能力（读文件、调 API、搜网页）
- **A2A**：Agent 之间通过 A2A 协作（一个 Agent 把代码审查任务委托给另一个）

**实际项目中的组合**：
1. 用 MCP 给 Hermes 接入 GitHub、Etherscan、文件系统
2. 用 A2A 让 Hermes 把「审查智能合约」的任务委托给一个专门的安全审计 Agent
3. 安全审计 Agent 自己通过 MCP 调用 Slither、Mythril 等工具

---

## 对 Stephen 的 Week 2 Proposal 启发

你的方向是 **Dev Tooling / Agent Workflow**，这意味着：

- **MCP** 是你的「切入层」——你已经通过 Hermes Agent 在用了。可以深入研究如何给 Hermes 写自定义 MCP Server（比如一个能读链上合约的 MCP Tool）。
- **A2A** 是你的「扩展层」——Week 3+ 可以探索多 Agent 协作。比如「一个 Agent 写合约 + 一个 Agent 审查 + 一个 Agent 部署」。

---

## 参考链接

- MCP 官网: https://modelcontextprotocol.io
- MCP 规范: https://spec.modelcontextprotocol.io
- A2A GitHub: https://github.com/a2aproject/A2A
- A2A 官网: https://a2a-protocol.org
- A2A 课程: https://goo.gle/dlai-a2a (DeepLearning.AI)
