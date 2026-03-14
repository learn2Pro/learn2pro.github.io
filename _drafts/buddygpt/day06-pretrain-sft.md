# 预训练与SFT：从文字接龙到对话能力

> 从零训练大模型系列 · Day 6

大模型的训练其实分两步：第一步是"文字接龙"（预训练），第二步是"学会说话"（SFT）。这两步看似简单，但从一个只会续写文本的模型，到一个能正经回答问题的助手，中间的转变比你想象的更有意思。

这篇文章记录BuddyGPT从预训练到SFT的完整过程，包括数据选择、训练配置、踩过的坑，以及一些个人思考。

## 预训练：教模型"读书"

### 什么是Next Token Prediction？

预训练的目标极其简单：**给定前面的文字，预测下一个token**。

```
输入：  "今天天气"
目标：  "真"

输入：  "今天天气真"
目标：  "好"
```

就是这么朴素。没有问答、没有对话、没有任何"智能"的味道。模型就像一个疯狂读书的学生，读完整个互联网，学会了"什么文字通常跟在什么文字后面"。

但神奇的是，当这个"文字接龙"玩到极致——几十亿参数、几万亿token——模型就突然"涌现"出了推理能力、常识理解、甚至数学能力。这大概是深度学习最让人惊叹的地方。

### 数据：35B Token的中文语料

BuddyGPT的预训练数据构成：

```
数据来源                      规模           语言
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Ultra-FineWeb              1T + 120B zh    英文+中文
Firefly                    4.7B            中文
Chinese-Instruct           100B            中文
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
合计约 35B Token（实际使用量）
```

为什么选这些数据？几个考虑：

1. **Ultra-FineWeb** 是目前质量最好的开源预训练语料之一，经过严格的质量过滤
2. **Firefly** 提供了高质量的中文文本
3. **Chinese-Instruct** 覆盖了大量中文场景

35B Token在大模型界不算多（LLaMA用了2T），但对于0.7B的小模型来说，这个数据量已经足够模型收敛了。

### Tokenizer：借用Qwen的词表

Tokenizer的选择是一个容易被忽视但非常关键的决策。BuddyGPT直接使用了Qwen的Tokenizer：

```python
from transformers import AutoTokenizer

# Qwen的tokenizer：151669个token的词表
# 对中文特别友好：一个中文词通常只需要1-2个token
# 对比GPT-2的tokenizer：一个中文字可能要3-4个token
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-0.5B")

# 示例：
# "你好世界" → Qwen: [4个token]
# "你好世界" → GPT2: [8-12个token]
```

为什么不自己训练Tokenizer？因为训练一个好的Tokenizer需要大量的语料统计，而Qwen已经在海量中文数据上训练过了，它的词表对中文非常友好。站在巨人的肩膀上，没什么不好意思的。

### 训练配置

```yaml
# ptrain.yaml - accelerate分布式训练配置
mixed_precision: bf16              # 用bf16半精度，省显存、加速度
gradient_accumulation_steps: 20    # 梯度累积20步，等效增大batch_size
```

```python
# pretrain.py 核心训练循环（简化版）

# 训练参数
seq_len = 1024          # 每条样本1024个token
batch_size_per_gpu = 1  # 每个GPU一条样本（显存有限）
grad_accum = 20         # 梯度累积20步
# 等效batch_size = 1 × 20 × n_gpus
# 每步处理的token数 = 1024 × 20 × n_gpus ≈ 64k tokens

# 学习率调度：cosine scheduler
# 从peak逐渐降低到接近0，像cos曲线一样平滑下降
scheduler = get_cosine_schedule_with_warmup(
    optimizer,
    num_warmup_steps=1000,      # 前1000步warmup：学习率从0线性增长
    num_training_steps=total_steps
)

# 训练循环
for batch in dataloader:
    # 前向传播：给定前n-1个token，预测第2到第n个token
    outputs = model(input_ids=batch["input_ids"], 
                    labels=batch["labels"])
    loss = outputs.loss
    
    # 反向传播 + 梯度累积
    loss.backward()
    
    if step % grad_accum == 0:
        optimizer.step()       # 更新参数
        scheduler.step()       # 更新学习率
        optimizer.zero_grad()  # 清空梯度
```

### Loss下降：模型在学东西吗？

判断预训练是否正常，最重要的指标就是**loss曲线**。BuddyGPT的loss变化大致是：

```
Loss
4.0 ┤
    │ ╲
3.5 ┤  ╲ ← 3.5766（0.1B tokens处）
    │   ╲
3.0 ┤    ╲╲
    │      ╲╲
2.5 ┤        ╲╲╲
    │           ╲╲╲
2.0 ┤              ╲╲╲╲╲╲╲──────── ← 逐渐收敛
    ├──────────────────────────────
    0      5B     10B    20B    35B  Tokens
```

如果loss一直不降、剧烈震荡、或者突然飙升，说明训练出了问题（学习率太大、数据有问题、梯度爆炸等）。BuddyGPT的loss整体平稳下降，说明模型确实在从数据中学到东西。

我们在WandB上监控整个训练过程——loss曲线、学习率变化、梯度范数，这些都是判断训练是否健康的关键信号。

## SFT：从"文字接龙"到"对话"

### 预训练模型的尴尬

预训练结束后，你得到了一个很"聪明"但很"没礼貌"的模型：

```
你：请介绍一下Python
模型：是一种编程语言。Python的创始人是Guido van Rossum。
     Python的名字来自于Monty Python。Python是一种解释型语言。
     Python是一种面向对象的语言...
     （像百科全书一样平铺直叙，停不下来）
```

它会续写文本，但不会"对话"。它不知道什么时候该停，不知道用户想要什么格式的回答，也不知道什么是"有帮助"的回复。

### ChatML格式：教模型理解角色

SFT的核心是用**对话格式的数据**来微调模型。BuddyGPT使用ChatML格式：

```python
# SFT数据格式：ChatML
messages = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好！有什么可以帮你的？"}
]

# apply_chat_template会把它转成这样的文本：
# <|im_start|>user
# 你好<|im_end|>
# <|im_start|>assistant
# 你好！有什么可以帮你的？<|im_end|>

text = tokenizer.apply_chat_template(messages, tokenize=False)

# 关键点：训练时只计算assistant回复部分的loss
# user部分作为输入，不参与loss计算
# 这样模型学的是"如何回复"，而不是"如何提问"
```

`<|im_start|>`和`<|im_end|>`是特殊token，告诉模型"这里开始/结束了一个角色的发言"。这些特殊token是Qwen Tokenizer自带的，这也是我们选择Qwen词表的好处之一。

### SFT数据集

```
数据集                规模        特点
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Chinese-Instruct-Lite 精选        高质量中文指令
Belle 2M              200万条     中文通用对话
MOSS                  大规模      多样化中文对话
ShareGPT              社区贡献    真实用户对话
Alpaca                5万条       经典指令数据
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
合计约 400万条指令数据
```

### RoPE内插：把上下文窗口从1024拉到4096

预训练时我们用的是`seq_len=1024`，但对话场景经常需要更长的上下文。直接推理超过1024的文本，模型会"困惑"——因为它从没见过1024之后的位置编码。

解决方案是**RoPE内插（interpolation）**：

```python
# RoPE内插的直觉：
# 原来：位置0, 1, 2, ..., 1023 对应频率f(0), f(1), ..., f(1023)
# 内插后：位置0, 1, 2, ..., 4095 被压缩映射到原来的频率范围
# 等于把"尺子的刻度变密"了，原来的1024格现在要装4096个位置

# 在模型配置中设置：
config.rope_scaling = {
    "type": "linear",      # 线性内插
    "factor": 4.0           # 4倍扩展：1024 → 4096
}
```

这个技巧非常实用：不需要重新预训练，只在SFT阶段做一次内插，就能显著扩展上下文长度。

## 个人思考

预训练和SFT的关系，我觉得可以这样理解：

**预训练像是上学**——系统地学习世界知识、语言规律、逻辑关系。学的东西很多但不太"实用"。

**SFT像是入职培训**——学会在特定场景下如何应用知识，用什么格式、什么语气、什么方式来回答问题。

两者缺一不可。没有预训练的SFT就像给文盲做职业培训——格式学会了但内容空洞。没有SFT的预训练就像一个满腹经纶的学者不会说人话——知识有了但不知道怎么输出。

BuddyGPT的35B Token预训练 + 400万条SFT，规模虽然不大，但完整地走了一遍"从语料到对话"的全流程。过程中最大的感受是：**数据质量比数据量更重要**。一条高质量的对话数据，抵得过十条噪声数据。

## 参考资料

- [Training language models to follow instructions - OpenAI InstructGPT论文](https://arxiv.org/abs/2203.02155)
- [Ultra-FineWeb数据集](https://huggingface.co/datasets/HuggingFaceFW/fineweb)
- [Qwen2.5技术报告](https://arxiv.org/abs/2412.15115)
- [ChatML格式说明](https://github.com/openai/openai-python/blob/main/chatml.md)
- [RoFormer: Enhanced Transformer with Rotary Position Embedding](https://arxiv.org/abs/2104.09864)
- [BuddyGPT项目地址](https://github.com/learn2pro/buddygpt)
