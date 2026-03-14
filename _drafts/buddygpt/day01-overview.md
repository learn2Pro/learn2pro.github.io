# 从零训练一个中文大模型：BuddyGPT项目全景

> 从零训练大模型系列 · Day 1

## 为什么要从零训练？

2024年，大模型遍地开花。GPT-4、Claude、Gemini、Qwen、DeepSeek……随便拉一个出来都能聊天写代码。那我们为什么还要从零训练一个？

答案很简单：**理解原理比调API重要100倍。**

调API就像开车——你知道踩油门会走、踩刹车会停，但你不知道发动机怎么运转。从零训练就像自己造一辆车——哪怕造出来的是辆三轮车，你对"车"的理解也远超任何老司机。

我在做BuddyGPT的过程中，很多之前"知道但不理解"的概念突然就通了：
- 为什么Decoder-Only架构胜出？自己写一遍Attention就明白了
- KV Cache到底省了什么？自己跑一次推理就懂了
- RLHF为什么难？自己训一次DPO就知道数据有多重要了

所以，BuddyGPT不是为了和GPT-4竞争（那是痴人说梦），而是一个**学习项目**——通过亲手实现完整的训练流程，真正搞懂大模型的每一个环节。

## BuddyGPT总览：从0.1B到0.7B

BuddyGPT经历了三个版本的进化：

```
版本       参数量    架构特点              状态
─────────────────────────────────────────────────
0.1B      ~100M    基础Llama架构, MHA      ✅ 完成
0.3B      ~300M    GQA + SwiGLU           ✅ 完成
0.7B      ~700M    MLA/MOE (实验中)        🔧 进行中
```

为什么从小模型开始？因为**调试成本**。0.1B模型在单卡A100上几小时就能跑完一轮pretrain，快速验证代码是否正确。等流程跑通了，再逐步放大——这和软件工程里"先跑通再优化"的原则一模一样。

每个版本都发布到了HuggingFace Hub（`learn2pro/buddygpt-*`），任何人都可以下载体验。

## 训练全流程Pipeline

一个大模型从"什么都不会"到"能正常对话"，需要经过四个阶段：

```
┌─────────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐
│  Pretrain    │───>│   SFT   │───>│ RLHF/DPO │───>│  Eval   │
│  预训练      │    │ 监督微调 │    │ 对齐优化  │    │  评测   │
└─────────────┘    └─────────┘    └──────────┘    └─────────┘
  35B Token          400万条        偏好数据        CMMLU/MMLU
  学会"语言"        学会"对话"      学会"做人"      检验成果
```

**Pretrain（预训练）**：喂给模型海量文本，让它学会语言的基本规律。就像小孩子大量听、大量读，先建立语感。BuddyGPT用了约35B Token的数据。

**SFT（监督微调）**：用精心构造的"问-答"对训练，让模型学会按指令回答问题。相当于上学——从自由阅读变成有目标的学习。BuddyGPT用了约400万条SFT数据。

**RLHF/DPO（对齐优化）**：通过人类偏好数据，让模型学会给出更好的回答。DPO（Direct Preference Optimization）是RLHF的简化版，不需要单独训练奖励模型，实现更简洁。

**Eval（评测）**：用标准化的benchmark测试模型能力。BuddyGPT主要用CMMLU（中文）、MMLU（英文）和CEval来评测。

## 技术栈

```
核心框架:    PyTorch + HuggingFace Transformers
分布式训练:   Accelerate (DeepSpeed ZeRO Stage 2/3)
实验追踪:    WandB (Weights & Biases)
Tokenizer:  Qwen的Tokenizer (151669词表)
评测框架:    lm-evaluation-harness
代码管理:    Git + GitHub
模型发布:    HuggingFace Hub
```

选择HuggingFace Transformers作为基础框架，是因为它的生态最成熟——模型定义、数据加载、训练循环、模型发布都有现成的工具链。站在巨人的肩膀上，我们可以把精力集中在模型架构和训练策略上。

## 模型架构选择

BuddyGPT以Llama2/3的架构为基础，这是目前开源社区最主流的Decoder-Only架构。在此基础上，逐步引入更先进的技术：

- **GQA（Grouped Query Attention）**：减少KV Cache，加速推理
- **MLA（Multi-head Latent Attention）**：DeepSeek-V3的核心创新，进一步压缩KV Cache
- **MOE（Mixture of Experts）**：用更少的计算资源训练更大的模型

后续的文章会逐一深入讲解这些技术。

## 为什么选中文？

训练中文LLM有两个独特挑战：

**1. 数据问题**：高质量中文语料远少于英文。英文有Common Crawl、Wikipedia等海量数据源，中文的数据量和质量都要打折扣。BuddyGPT的预训练数据来自：
- 中文Wikipedia
- Firefly数据集
- Ultra-FineWeb（多语言高质量语料）

**2. Tokenizer问题**：中文的分词比英文复杂得多。英文基本以空格分词，中文没有天然分隔符。BuddyGPT直接采用了Qwen的Tokenizer（151669个token），省去了自己训练Tokenizer的麻烦，也保证了对中文的良好支持。

选择中文，一方面是因为自己的母语，另一方面也是想亲身体验中文LLM训练的痛点。

## 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    BuddyGPT 训练流程全景                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐                                       │
│  │      数据准备         │                                       │
│  │  Wikipedia (中文)     │                                       │
│  │  Firefly (对话数据)   │──┐                                    │
│  │  Ultra-FineWeb       │  │                                    │
│  └──────────────────────┘  │                                    │
│                            v                                    │
│  ┌──────────────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Tokenizer (Qwen)     │─>│ Pretrain     │─>│ SFT          │  │
│  │ vocab=151669         │  │ 35B Tokens   │  │ 400万条       │  │
│  └──────────────────────┘  └──────────────┘  └──────┬───────┘  │
│                                                      │          │
│                                                      v          │
│                            ┌──────────────┐  ┌──────────────┐  │
│                            │ Eval         │<─│ DPO          │  │
│                            │ CMMLU/MMLU   │  │ 偏好对齐      │  │
│                            │ CEval        │  │              │  │
│                            └──────────────┘  └──────────────┘  │
│                                                                 │
│  模型发布: HuggingFace Hub (learn2pro/buddygpt-*)               │
└─────────────────────────────────────────────────────────────────┘
```

## 接下来

这个系列会深入每一个环节：
- Day 2: 模型架构深度拆解
- Day 3: RoPE位置编码与GQA
- Day 4: MLA——DeepSeek-V3的核心创新
- Day 5+: 预训练、SFT、DPO、评测……

每一篇都会有代码、有推导、有踩坑经验。目标是让你读完之后，也能自己动手训一个。

## 个人思考

做BuddyGPT最大的感受是：**大模型没有想象中那么神秘。** 架构就是标准的Transformer，训练就是标准的梯度下降，数据就是标准的文本。真正的难点在于规模——如何高效处理TB级数据、如何在多卡上稳定训练、如何在有限算力下做出效果。

但对于学习目的来说，0.3B的模型已经足够让你理解所有核心概念。就像学开车不需要从法拉利开始，一辆教练车就够了。

## 参考资料

- [Llama 2: Open Foundation and Fine-Tuned Chat Models](https://arxiv.org/abs/2307.09288)
- [HuggingFace Transformers 文档](https://huggingface.co/docs/transformers)
- [BuddyGPT 模型合集 - HuggingFace Hub](https://huggingface.co/learn2pro)
- [DeepSpeed ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](https://arxiv.org/abs/1910.02054)
- [Direct Preference Optimization (DPO)](https://arxiv.org/abs/2305.18290)
