---
title: "我对大模型的认知"
date: 2023-10-11 20:18:00
tags:
- LLM
categories:
- Technology
---

## 背景
 - 2017.06 transformer模型架构论文发布
 - 2018.06 gpt-1模型发布，参数量1.17亿
 - 2019.02 gpt-2模型发布，参数量15亿，提出zero-shot，few-shot概念，decoder-only的模型通过prompt能做更多事情 
 - 2020.05 gpt-3模型发布，参数量1750亿，大力出奇迹
 - 2022.03 instruct-gpt, 基于RLHF对齐人类偏好
 - 2022.11 chatgpt横空出世，让我们感觉到agi真的要来了

当前各类AI+应用如雨后春笋般出现，AI+Law，AI+Data，AI+Market，AI+Assistant，AI+Math，很多人都觉得大模型是人类历史上另外一个iphone时刻，因为在其上可以用新的逻辑构建非常多应用，重塑当前的生态，连openai都觉得未来重点发展方向是agent，如最近出现的code-interpreter
 什么是Agent，简单点说就是基于大模型的能力，完成特定任务的智能体，可以辅助人类解决一类问题，比如个人助理，帮你订机票，帮你订酒店等等
 ![agent](https://lilianweng.github.io/posts/2023-06-23-agent/agent-overview.png)
 它主要包含几个部分tools, planning, memory, action; 

1. tools包含一系列的工具，比如搜索，日历，计算器，可以获取外部数据或者能力，因为这些能力通常是大模型不擅长的计算
2. planning用于规划问题解决路径，通过大模型的理解能力和判断，规划用户的问题需要哪些tool来进行解决，比如用户想看些时政新闻，此时就会用到搜索引擎来进行搜索，然后整理成结构化数据展示
3. memory用于记录历史步骤结果，以及用户的上下文，方便快速准确的识别用户意图
4. action就是具体操作了，使用相应工具执行即可
5. 当然planning的模块也包含很多设计内容，如何做反思，如何实现COT，如何拆解任务，是比较核心的模块

## 几点预判

1. 未来所有与人交互的产品，都会变成自然语言的模式，因为这是最符合人类习惯，而大模型恰恰能做到这件事情
2. 做底层大模型的公司不会太多，因为这是一个零和游戏，强者更强，反而做上层应用，toB，toC都大有前景，这里面有非常多细分的机会
3. 可遇见的未来，所有产品都会被重塑，这反而是创业者的机会，因为大厂的认知现在也是初级阶段，创业者的优势是可以更快学习，适应，调整，更新
4. 硬件上会有很多改变，UI的交互不是必须了，语音的能力会加强，输入法会成为历史
5. 不需要太多APP了，很多基础能力互联网已经建设好，比如物流，外卖，电商，最后只需要一个Assistant
6. 必须要重新学习，用新的思路来做事，以前觉得不可能的事情，现在可以做了，比如电子宠物，比真实的宠物更体贴，更自愈

～ 先说到这儿，必须要抓紧行动了

## reference
 - https://lilianweng.github.io/posts/2023-06-23-agent/
 - https://medium.com/@gerardo.pdm/karpathy-the-potential-and-challenges-of-ai-agents-f53c55734050
 - https://www.wolframalpha.com/input?i=exp%28x%29+from+0+to+1

