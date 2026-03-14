# Agent架构全景：一张图看懂Agent系统设计

> Agent深度解析系列 · Day 2

2024年我们还在讨论"LLM能做什么"，2026年问题已经变成"Agent该怎么搭"。架构决定了Agent的能力上限，选错架构，再强的模型也白搭。

## 四大核心模块

一个完整的Agent系统由四个模块构成，缺一不可：

- **LLM核心**：大脑，负责理解意图、推理决策、生成输出。它不是Agent本身，只是Agent的引擎。
- **Tools（工具）**：手和脚，让Agent能搜索、计算、调用API、操作文件。没有工具的Agent就是个聊天机器人。
- **Memory（记忆）**：短期记忆是对话上下文，长期记忆是向量数据库或外部存储。没有记忆，Agent每次对话都是失忆状态。
- **Planning（规划）**：将复杂任务拆解为可执行步骤的能力。这是Agent区别于简单问答的关键——它能制定计划、执行计划、根据反馈调整计划。

这四个模块的关系很清晰：Planning决定做什么，LLM负责怎么想，Tools负责怎么做，Memory负责记住做过什么。

## ReAct：推理与行动的交替循环

ReAct（Reasoning + Acting）是目前最主流的Agent执行模式，核心思想极简：

1. **Thought**：模型先推理当前状态，决定下一步该做什么
2. **Action**：调用一个工具执行操作
3. **Observation**：获取工具返回的结果
4. 回到第1步，直到任务完成

这个循环的精妙之处在于：每一步行动都有推理支撑，每一次推理都基于真实观察。它避免了纯推理的"幻想"问题，也避免了盲目行动的浪费。实际工程中，ReAct几乎是所有Agent框架的默认模式——LangChain、AutoGPT、OpenAI Assistants API，底层逻辑都是它。

## 架构全景：信息流怎么走

用一句话描述Agent的信息流：

```
用户请求 → Agent（LLM + Planning） → 工具调用 → 外部环境（API/DB/Web）
    ↑                                              |
    └──────────── 结果返回 + Memory更新 ←──────────┘
```

用户发出请求，Agent的Planning模块将任务拆解，LLM逐步推理并选择合适的Tool执行，Tool与外部环境交互后返回结果，Agent根据结果决定是继续执行下一步还是返回最终答案。整个过程中Memory持续更新，为后续决策提供上下文。

## 单Agent vs 多Agent：什么时候该上多Agent

**单Agent**适合任务边界清晰、工具数量有限的场景。一个Agent配10-20个工具，覆盖一个垂直领域，够用且简单。

**多Agent**适合以下场景：
- 工具太多（50+），单Agent选择困难
- 任务需要不同"专业角色"协作（研究、写作、审核）
- 需要并行处理多个子任务

多Agent架构有几种常见变种：

| 模式 | 特点 | 适用场景 |
|------|------|----------|
| **Router Agent** | 一个路由Agent将请求分发给专业Agent | 客服系统、多领域问答 |
| **Supervisor Agent** | 一个监督Agent协调多个Worker Agent | 复杂项目管理、多步骤工作流 |
| **Hierarchical** | 多层级Agent树，逐级分解任务 | 企业级复杂流程 |
| **Peer-to-Peer** | Agent之间平等协作，无中心节点 | 去中心化场景 |

我的观点：**先从单Agent开始，遇到瓶颈再拆**。过早引入多Agent会带来通信开销、状态同步、错误传播等一系列工程问题。多Agent不是银弹，是在单Agent确实不够用时的解决方案。

## 工程实践中的关键取舍

架构选型之外，还有几个工程层面的决策：

- **同步 vs 异步**：简单任务同步等待，复杂任务异步执行+回调通知
- **确定性 vs 灵活性**：工作流引擎（如Dify）提供确定性，纯Agent提供灵活性，实际项目往往需要混合
- **成本控制**：每一步推理都消耗Token，Planning阶段用强模型，执行阶段可以用轻量模型

Agent架构不是一成不变的。2025年的最佳实践可能在2026年就被新范式替代。但核心原则不变：**让LLM专注于推理，让工具负责执行，让记忆提供上下文，让规划驱动全局。**

## 参考资料

- [LLM Powered Autonomous Agents - Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [AI Agent Architecture - ByteBytego](https://blog.bytebytego.com/p/ai-agent-architecture)
- [Building AI Agents - AWS](https://aws.amazon.com/blogs/machine-learning/build-ai-agents/)
