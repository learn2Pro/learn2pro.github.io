# Agent规划与推理：思维链的力量

> Agent深度解析系列 · Day 7

一个能调用工具的LLM不算Agent，一个能**想清楚再行动**的LLM才算。规划与推理是Agent的大脑皮层——决定了它是盲目试错还是有策略地解决问题。

## Chain-of-Thought：想一步说一步

2022年Wei等人的论文奠定了基础：在prompt中加入"Let's think step by step"，模型的推理准确率就能大幅提升。这不是魔法，而是迫使模型将隐式推理过程**外显化**，减少跳步带来的错误积累。

CoT的核心洞察：**语言模型的推理能力受限于输出空间，而非知识空间**。它可能"知道"答案，但直接输出会出错；分步写出来，每一步都能得到前面步骤的"提示"，准确率自然上升。

对Agent而言，CoT不只是提升准确率的技巧，而是**规划的基础范式**——先想、再做、边做边想。

## 任务分解：从大问题到小步骤

**Tree of Thoughts（ToT）**：把线性的思维链扩展为树状搜索。每一步生成多个候选思路，评估后选择最优分支，必要时回溯。适合有明确评估标准的问题（如数学证明、代码生成），但计算开销大，实际生产中用得不多。

**LLM+P**：让LLM生成PDDL（规划域定义语言）格式的问题描述，然后交给传统规划器求解。思路很好——用LLM做自然语言到形式化的翻译，用经典算法做规划。但受限于PDDL的表达能力，只适合结构化程度高的任务。

实际上，2026年最常见的任务分解方式还是最直接的：**让LLM把大任务拆成子任务列表**，然后逐个执行。简单粗暴，但配合好的prompt效果不差。

## 反思机制：做完了回头看

**Reflexion**：Agent执行任务后，将结果和反馈输入一个"反思"环节，生成经验总结，存入记忆。下次遇到类似任务时，先检索历史反思。核心是把**试错的代价转化为知识资产**。

**Self-Refine**：更轻量的方案——生成初始输出后，自己评估、自己修改，迭代几轮。不需要外部反馈，纯靠模型自我审视。实测中，代码生成和写作任务的效果提升明显，但有"自我强化偏见"的风险——模型可能越改越偏。

反思机制的关键限制：**模型不知道自己不知道什么**。如果初始推理方向就错了，反思可能只是在错误方向上做微调。

## ReAct：推理与行动的交织

ReAct（Reasoning + Acting）是当前Agent最主流的执行范式：

```
Thought: 用户需要查找最近的财报数据，我应该搜索SEC网站
Action: search("AAPL 10-Q 2026 Q1")
Observation: 找到了Apple 2026 Q1财报链接...
Thought: 拿到了链接，现在需要提取关键指标
Action: fetch(url)
Observation: 营收$124.3B，同比增长8.7%...
Thought: 数据齐了，可以生成分析报告
```

ReAct的精髓在于**Thought步骤不是装饰**——它迫使模型在每次行动前明确推理意图，在每次观察后更新认知。去掉Thought直接Action→Observation，效果会显著下降。

## 层级规划：大任务的管理艺术

Microsoft的CORPGEN提出了多层规划架构：

- **战略层**：理解最终目标，拆分为里程碑
- **战术层**：每个里程碑分解为具体任务序列
- **执行层**：每个任务选择工具、生成参数、处理异常

这种层级结构在复杂Agent系统中越来越常见。Claude Code的"Agent Teams"、CrewAI的Manager Agent，本质上都是层级规划的实现。

## 什么时候该推理，什么时候该硬编码

一条实用的判断标准：**如果流程固定且可枚举，硬编码；如果需要适应未知情况，让Agent推理。**

报销审批流程？硬编码。每一步该做什么、需要什么字段，都是确定的，让Agent"自由推理"只会增加出错概率。

帮用户调研一个开放话题？交给Agent推理。搜索什么、读哪些内容、如何整合，这些决策无法预先穷举。

2026年的趋势是**混合模式**：用确定性的代码编排整体流程（LangGraph的图），在需要灵活判断的节点上调用LLM推理。既有可控性，又有灵活性。

## 2026变化：推理模型的跃升

OpenAI o1/o3、DeepSeek R1这类推理模型的出现，让Agent的规划能力有了质的飞跃。这些模型在输出前进行了长时间的内部推理（hidden chain-of-thought），在数学、编程、复杂逻辑任务上远超传统模型。

对Agent的影响是直接的：**规划步骤的质量大幅提升**。以前需要精心设计prompt才能让模型做好任务分解，现在推理模型"天生"就擅长这个。代价是速度更慢、成本更高，但在高价值任务上完全值得。

我的判断：2026年下半年，Agent框架会开始区分"规划模型"和"执行模型"——用推理模型做规划和决策，用快速模型做简单的工具调用和文本生成。

## 参考资料

- [Wei et al. Chain-of-Thought Prompting Elicits Reasoning in Large Language Models](https://arxiv.org/abs/2201.11903)
- [IBM: What is Agentic Reasoning?](https://www.ibm.com/think/topics/agentic-reasoning)
- [Microsoft CORPGEN: Multi-agent Task Planning](https://www.microsoft.com/en-us/research/publication/corpgen/)
- [Yao et al. ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)
- [Shinn et al. Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
