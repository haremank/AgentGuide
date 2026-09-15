---
type: 教程
status: 已发布
level: 进阶
topic:
  - Agent
  - 框架工具
  - 多智能体
---

# 2026 Agent 框架与协议生态地图

> 框架选型不是选"最强的",而是选"和你 harness 设计最匹配的"。本文用一张地图回答:2026 年做 Agent 开发,框架层、协议层、评测层各自的事实标准是什么,哪些旧答案已经过时。

## 一图总览

```text
┌─────────────────────────────────────────────────────┐
│ 应用层: 你的 Agent = Model + Harness(见 27 篇)      │
├─────────────────────────────────────────────────────┤
│ 编排框架: LangGraph / OpenAI Agents SDK /            │
│          MS Agent Framework / Google ADK / Pydantic AI│
├─────────────────────────────────────────────────────┤
│ 协议层:  MCP(工具接入) | A2A(跨智能体协作)          │
├─────────────────────────────────────────────────────┤
│ 模型层:  Claude 4.5 / GPT-5.x / Gemini 3 /           │
│          DeepSeek / Qwen / Kimi K2 / GLM / MiniMax   │
└─────────────────────────────────────────────────────┘
```

## 框架层:谁在做什么

### LangChain / LangGraph

- 2025 年 LangChain 1.0 发布,**旧 AgentExecutor 已废弃**;现代写法是 `create_agent` + LangGraph 状态图
- LangGraph 成为复杂编排(循环、分支、checkpoint、human-in-the-loop)的事实标准
- 选型建议:需要细粒度状态机与持久化时选它;简单 tool-loop 不要引入,直接手写(见 examples/minimal-agent-loop)

### OpenAI Agents SDK

- 核心原语:Agent(指令+工具)、Handoffs(任务移交)、Guardrails(输入输出护栏)、Sessions(会话状态)
- 定位轻量,适合"单 agent + 少量移交"的straightforward 场景
- 注意:Swift 版生态与 Python 版并行,招 Python 后端为主的团队用 Python 版

### Microsoft Agent Framework

- **AutoGen 与 Semantic Kernel 已合并**为此项目,AutoGen 作为独立品牌停止演进
- 企业 .NET/Azure 生态优先考虑;开源社区的新教程基本不再基于 AutoGen 0.2

### Google ADK / Pydantic AI / 其他

- Google ADK:与 Gemini / Vertex 深度绑定,用 GCP 全家桶时值得看
- Pydantic AI:类型安全、轻量,适合强类型 Python 团队
- CrewAI:角色扮演式多智能体,教程多但生产透明度一般,建议认知级别了解

### 过时信号(面试别再这么答)

- ✗ "我熟悉 LangChain 的 AgentExecutor 和 initialize_agent"——已废弃
- ✗ 用 AutoGen 0.2 group chat API 描述你的多智能体方案——品牌已并入 MS Agent Framework
- ✗ 把 CrewAI 角色分工当作多智能体最佳实践——见下文单上下文派争论

## 协议层:MCP 与 A2A

### MCP(Model Context Protocol)

- 2024 年底发布,2025 年成为工具接入事实标准:registry、OAuth 授权、Streamable HTTP 传输相继落地
- 解决"M×N 集成"问题:工具方实现一次 server,所有支持 MCP 的客户端都能用
- 工程要点:工具描述质量决定模型调用质量(见 27 篇 L3 工具层);高危工具要有 permission tier

### A2A(Agent2Agent)

- Google 发起,2025 年捐赠给 Linux Foundation,定位于**跨厂商智能体互操作**(Agent Card 声明能力、任务生命周期、异步长任务)
- 与 MCP 互补而非竞争:MCP 连接"agent ↔ 工具/资源",A2A 连接"agent ↔ agent"
- 2026 年现状:MCP 已大规模落地;A2A 生态在早期adopt 阶段,面试中讲清两者边界即可加分

## 派别之争:多智能体还是单上下文?

这是高级岗面试的必备辩证题:

- **多智能体编排派**:子任务独立性强、需要并行与上下文隔离时,orchestrator-worker 有效(Anthropic multi-agent research system 是代表案例)
- **单上下文派**(Cognition "Don't Build Multi-Agents" 论点):两个 agent 协作会带来上下文割裂与决策不一致;能用压缩、记忆、子任务化解决就不要拆多 agent
- **工程共识**:先榨干单 agent(好的上下文工程 + 工具设计),出现真实瓶颈(上下文放不下、需要并行)再上多智能体,且要设计好状态汇总与冲突消解

## 模型层速览(2026)

- 闭源:Claude 4.5 系(coding/长任务)、GPT-5.x、Gemini 3(多模态长上下文)
- 开源/国产:DeepSeek V3.x/R1、Qwen3 系、Kimi K2、GLM 系列、MiniMax M2——agentic 能力与工具调用已普遍可用
- 选型逻辑:能力上限(难任务)vs 成本/延迟(高频调用)vs 部署约束(合规/私有化)三层路由,参考 25 篇 post-training 与模型系列笔记

## 选型决策树

```text
你的 Agent 需要什么?
├── 简单 tool loop(≤5 工具,单会话)→ 手写 loop,不引框架
├── 复杂状态机/checkpoint/HITL → LangGraph
├── 轻量多 agent + 移交 → OpenAI Agents SDK
├── .NET/Azure 企业栈 → MS Agent Framework
├── GCP/Gemini 栈 → Google ADK
├── 工具要给多个客户端复用 → 实现 MCP server
└── 跨厂商 agent 互操作 → 关注 A2A
```

## 学习路径建议

1. 先读 [Agent Harness Engineering](./27-agent-harness-engineering.md) 建立 harness 视角——框架只是 harness 的封装
2. 手写 [minimal agent loop](../../examples/) 再用框架重写一遍,体会框架替你做了什么
3. 按上方决策树为一个真实需求选型,写清"为什么不用另外两个"
4. 面试复述:框架层(LangGraph/Agents SDK/MS AF)+ 协议层(MCP/A2A)+ 派别之争(单上下文 vs 编排)三段式

## 参考来源

- LangChain 1.0 与 AgentExecutor 迁移公告;OpenAI Agents SDK 官方文档
- Microsoft Agent Framework 发布说明(AutoGen + Semantic Kernel 合并)
- Cognition: Don't Build Multi-Agents;Anthropic: Building Effective Agents / multi-agent research system
- Linux Foundation: A2A 项目公告;Anthropic: Model Context Protocol 规范
