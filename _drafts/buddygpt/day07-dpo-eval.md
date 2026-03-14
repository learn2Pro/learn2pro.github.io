# DPO对齐与评估：让模型说人话

> 从零训练大模型系列 · Day 7

经过预训练和SFT，BuddyGPT已经能进行基本的对话了。但你有没有遇到过这种情况：模型的回答"技术上没错"，但就是让你觉得不舒服？

比如你问"怎么减肥"，模型回复"你可以不吃饭"——这句话不算错，但显然不是一个好回答。SFT教会了模型对话的格式，但没有教会它什么是"好"的回答。

这就是为什么我们需要**对齐（Alignment）**：让模型的输出符合人类偏好。

## 为什么SFT还不够？

SFT后的模型有几个典型问题：

1. **回答质量不稳定**：同一个问题，有时回答得很好，有时一塌糊涂
2. **可能生成不当内容**：模型从互联网数据中学到了各种东西，包括有害信息
3. **不够"人类"**：回答可能啰嗦、跑题、或者用一种机器人的方式表达

根本原因是：SFT只教了模型"看到问题A，应该回答B"，但没有教它"B比C好，C比D好"这样的**偏好排序**。

## PPO vs DPO：两条对齐之路

### PPO（Proximal Policy Optimization）

PPO是OpenAI在ChatGPT中使用的方法，流程大致是：

```
PPO流程（复杂版）：

1. 先训练一个Reward Model（奖励模型）
   人类标注：回答A比回答B好 → RM学会给回答打分

2. 用RL训练LLM
   LLM生成回答 → RM打分 → 用PPO算法更新LLM参数
   
   ┌─────────┐   生成回答   ┌──────────┐
   │  LLM    │ ──────────→ │ Reward   │
   │ (策略)   │ ←────────── │  Model   │
   └─────────┘   返回奖励   └──────────┘
        ↑
        │ PPO更新
        ↓
   ┌─────────┐
   │ 参考模型  │ ← 防止LLM偏离太远
   └─────────┘
```

问题是：这个过程涉及4个模型（LLM、参考模型、RM、价值模型），训练不稳定，调参痛苦，工程复杂度极高。

### DPO（Direct Preference Optimization）

DPO的核心洞察是：**我们其实可以跳过Reward Model，直接用偏好数据优化LLM**。

```
DPO流程（简化版）：

偏好数据：
  问题："推荐一本书"
  chosen（好回答）："《百年孤独》，一部魔幻现实主义经典..."
  rejected（差回答）："书很多啊，你自己去书店看看"

                直接优化
  偏好数据 ──────────────→ LLM
  
  不需要Reward Model！不需要RL循环！
```

### DPO的核心公式（直觉版）

DPO的loss函数说的其实是一件很直觉的事：

```
L_DPO = -log σ(β × (好回答的得分提升 - 差回答的得分提升))

翻译成人话：
1. 计算模型给"好回答"打的分（相对于参考模型的变化）
2. 计算模型给"差回答"打的分（相对于参考模型的变化）
3. 让 (1) 尽可能大于 (2)
4. β控制优化的力度

效果：模型越来越倾向于生成chosen风格的回答
      同时越来越远离rejected风格的回答
```

简单说就是：**让模型学会"A比B好"，而不仅仅是"A是标准答案"**。这比SFT提供了更丰富的信号。

## BuddyGPT的DPO实践

### 数据准备

```
DPO数据集               特点
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FuseChat-3.0-DPO-Data   多模型融合偏好数据
HC3-Chinese             人类vs ChatGPT对比
UltraFeedback           高质量多维度偏好标注
```

每条DPO数据包含三个部分：prompt（问题）、chosen（好回答）、rejected（差回答）。

### 训练代码

DPO的代码出人意料地简单，感谢HuggingFace的`trl`库：

```python
from trl import DPOTrainer, DPOConfig
from transformers import AutoModelForCausalLM, AutoTokenizer

# 加载SFT后的模型作为起点
model = AutoModelForCausalLM.from_pretrained("buddygpt-0.7b-sft")
tokenizer = AutoTokenizer.from_pretrained("buddygpt-0.7b-sft")

# DPO训练配置
training_args = DPOConfig(
    output_dir="buddygpt-0.7b-dpo",    # 输出目录
    per_device_train_batch_size=2,       # 每GPU batch_size=2（DPO需要同时处理chosen和rejected，显存翻倍）
    gradient_accumulation_steps=16,      # 梯度累积16步
    bf16=True,                           # bf16精度
    max_grad_norm=1.0,                   # 梯度裁剪，防止训练不稳定
    max_length=1024,                     # 最大序列长度
    learning_rate=5e-7,                  # 学习率要比SFT小很多！
)

# 加载偏好数据集
# 数据格式：{"prompt": "...", "chosen": "...", "rejected": "..."}
ds = load_dataset("dpo_data")

# 创建DPOTrainer并开始训练
trainer = DPOTrainer(
    model=model,                # 要优化的模型
    args=training_args,         # 训练参数
    processing_class=tokenizer, # tokenizer
    train_dataset=ds,           # 偏好数据
    # DPOTrainer会自动创建参考模型（冻结的SFT模型副本）
)

trainer.train()  # 开始训练！就这么简单
```

DPO训练的一个关键点是：**学习率要非常小**（通常是SFT的1/10甚至更低）。因为我们不希望模型忘掉SFT学到的对话能力，只是在此基础上微调偏好。

## 评估：到底学得怎么样？

训练完了，总要检验一下成果。大模型的评估本身就是一个大课题，BuddyGPT使用了标准的benchmark评估体系。

### 评估工具和指标

```bash
# 使用lm-eval-harness进行评估
# 这是开源社区最广泛使用的LLM评估框架

lm_eval --model hf \
    --model_args pretrained=learn2pro/buddygpt-0.4b-base-zh,dtype="bfloat16" \
    --tasks cmmlu,mmlu,ceval \  # 三个benchmark
    --batch_size 8 \             # 评估batch_size
    --num_fewshot 2              # 2-shot：给2个示例再回答
```

三个Benchmark的定位：

| Benchmark | 语言 | 内容 | 评估什么 |
|-----------|------|------|----------|
| CMMLU | 中文 | 67个学科领域 | 中文知识和推理 |
| MMLU | 英文 | 57个学科领域 | 英文知识和推理 |
| CEval | 中文 | 52个考试科目 | 中文应试能力 |

### BuddyGPT的评估结果

```
模型                    CMMLU    CEval    MMLU
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BuddyGPT 0.1B         25.38     -        -      ← 基本随机猜（4选1=25%）
BuddyGPT 0.3B         25.60    28.60     -      ← 略好于随机
Qwen2.5-0.5B           41.44     -        -      ← 差距明显
DeepSeek-V3            88.80     -        -      ← 仰望星空
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

让我们直面这个结果：

**BuddyGPT 0.1B的CMMLU 25.38%，基本等于随机猜测**（4个选项随机选，期望就是25%）。0.3B略好一点，但提升有限。

对比Qwen2.5-0.5B的41.44%，差距是显著的。而DeepSeek-V3的88.8%更是另一个维度的存在。

这个结果说明了几件事：

1. **规模很重要**：0.1B → 0.3B的提升微乎其微，说明在这个量级，模型容量还不足以存储和推理复杂知识
2. **数据质量和量级很重要**：Qwen用了远超BuddyGPT的数据量和更好的数据质量
3. **架构优化有天花板**：MOE等架构创新能帮忙，但不能弥补数据和规模的差距

## 反思：小模型的意义在哪里？

看到这个跑分结果，你可能会想：那训练BuddyGPT的意义是什么？

我的回答是：**小模型的价值不在于跑分，在于理解原理**。

就像学编程时写的第一个"Hello World"不是为了替代专业软件，训练一个0.7B的小模型是为了：

1. **理解完整的训练流程**：从数据处理、预训练、SFT到DPO，每一步是怎么工作的
2. **建立直觉**：loss应该是多少才正常？学习率设多大？数据要怎么处理？
3. **动手实践**：读100篇论文不如自己跑一次训练，看到loss曲线下降的那一刻，你就真的理解了
4. **为更大的模型做准备**：等你有了更多资源，这些经验会直接迁移

DeepSeek-V3的88.8%很厉害，但如果你不理解从25%到88%之间发生了什么，那个数字对你来说就只是一个数字。

## 下一步

BuddyGPT的旅程还在继续。接下来的方向：

- **更多、更好的数据**：数据质量是投入产出比最高的改进方向
- **更大的模型**：尝试1B+规模，看看scaling law在小模型上的表现
- **更好的评估**：除了选择题benchmark，还需要更多主观评估（人工测试、开放问答等）

从零训练一个大模型，最大的收获不是模型本身，而是对整个Pipeline的深刻理解。这些理解，才是你在大模型时代最有价值的能力。

## 参考资料

- [DPO: Direct Preference Optimization论文](https://arxiv.org/abs/2305.18290)
- [PPO: Proximal Policy Optimization论文](https://arxiv.org/abs/1707.06347)
- [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)
- [CMMLU Benchmark](https://github.com/haonan-li/CMMLU)
- [trl库文档](https://huggingface.co/docs/trl)
- [BuddyGPT项目地址](https://github.com/learn2pro/buddygpt)
