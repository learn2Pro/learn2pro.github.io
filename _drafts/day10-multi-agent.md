# 多Agent编排：让一群Agent协同干活

> Agent深度解析系列 · Day 10

单个Agent再强，也有能力天花板。让它既写代码又做测试还管部署？结果大概率是三件事都做得半吊子。多Agent的核心逻辑和人类组织一样：**专业化分工 + 协调机制**。

## 为什么需要多Agent

单Agent的局限性体现在三个维度：**上下文窗口有限**（一个Agent塞不下所有领域知识）、**能力边界模糊**（什么都能做 = 什么都做不好）、**错误传播不可控**（一步错步步错，没有纠偏机制）。

多Agent系统的本质是把一个复杂任务分解成多个可独立执行的子任务，每个Agent专注自己的领域，通过明确的接口交互。这不是什么新概念——微服务架构的思想在Agent领域的复刻。

## 四种编排模式

**串行Pipeline**：A做完交给B，B做完交给C。最简单，延迟最高，适合有严格先后依赖的任务。典型场景：代码生成 → 代码审查 → 测试执行。

**并行Fan-out**：同一个任务分发给多个Agent并行处理，汇总结果。适合可拆分的独立任务。比如同时用不同策略搜索信息，最后merge。延迟低但成本翻倍。

**层级Hierarchical**：有一个Supervisor Agent负责分配任务、收集结果、做最终决策。下层Agent只管执行。清晰但Supervisor是单点瓶颈。

**协商Negotiation**：多个Agent之间互相讨论、辩论，达成共识。适合需要多角度评估的场景（比如代码审查中安全Agent和性能Agent观点冲突时的仲裁）。最灵活也最难控制。

## Supervisor vs P2P：怎么选

Supervisor模式（中心化）：一个"经理"Agent统筹全局，分配任务给"员工"Agent。优点是控制流清晰、易调试；缺点是Supervisor挂了全挂，且Supervisor本身需要很强的规划能力。

P2P模式（去中心化）：Agent之间直接通信，没有中心节点。优点是弹性好、无单点故障；缺点是协调开销大，容易出现消息风暴。

**实践建议**：大多数场景用Supervisor模式就够了。P2P适合Agent数量多（>5）、任务高度动态的场景。别为了架构优雅选P2P——调试地狱不是闹着玩的。

## 实战：代码交付Pipeline

一个典型的多Agent代码交付流水线：

1. **规划Agent**：接收需求，拆解为具体的代码任务，输出任务清单
2. **编码Agent**：根据任务清单逐个实现，输出代码变更
3. **审查Agent**：Review代码质量、安全漏洞、风格规范，输出审查意见
4. **测试Agent**：编写和运行测试，输出测试报告
5. **部署Agent**：通过CI/CD Pipeline部署到staging环境

关键点：每个Agent之间通过**结构化消息**（不是自然语言聊天）传递数据。审查Agent发现问题直接打回编码Agent，形成反馈循环，但要设置**最大重试次数**（通常3次），避免无限循环。

## 常见坑

**死循环**：Agent A让Agent B修改，B改完A又不满意，如此往复。解法：硬性设置最大轮次，超过就升级到人工。

**资源竞争**：多个Agent同时修改同一个文件。解法：用锁机制或者划分文件所有权。

**结果不一致**：并行Agent给出矛盾的结论。解法：需要一个仲裁机制（投票、Supervisor裁决、或者规则优先级）。

**成本爆炸**：每个Agent都用GPT-4，5个Agent跑一轮就是5倍token消耗。解法：模型分级——简单任务用小模型，只有关键决策用大模型。

## 设计原则

1. **明确边界**：每个Agent的输入输出格式、职责范围必须精确定义，模糊地带是bug的温床
2. **共享状态最小化**：Agent之间通过消息传递数据，而不是共享全局状态。共享越多耦合越深
3. **失败可恢复**：任何一个Agent失败不应该导致整个流程崩溃。要有重试、降级、人工兜底机制
4. **可观测**：每个Agent的输入输出、决策过程都要可追溯，否则出了问题根本没法debug

多Agent编排不是银弹。在引入复杂度之前，先问自己：**一个Agent加上好的Prompt真的搞不定吗？** 如果答案是"能"，那就别折腾多Agent。

## 参考资料

- [Codebridge多Agent编排指南](https://codebridge.tech/multi-agent-orchestration-guide)
- [AWS Agent评估实践](https://aws.amazon.com/blogs/machine-learning/best-practices-for-evaluating-ai-agents/)
- [LangGraph Multi-Agent文档](https://langchain-ai.github.io/langgraph/concepts/multi_agent/)
- [CrewAI框架](https://github.com/crewAIInc/crewAI)
