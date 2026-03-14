# 模型架构深度拆解：从Embedding到输出层

> 从零训练大模型系列 · Day 2

## 开篇：为什么是Decoder-Only？

在Transformer的世界里，有三种主流架构：

```
Encoder-Only:    BERT系列 → 擅长理解，不擅长生成
Encoder-Decoder: T5/BART  → 理解+生成，但结构复杂
Decoder-Only:    GPT系列  → "只管生成"，但效果出奇地好
```

你可能会问：Decoder-Only不是只能"续写"吗？为什么能胜任几乎所有NLP任务？

关键洞察是：**所有任务都可以转化为"续写"。** 翻译？"请翻译：Hello → "，模型续写"你好"。摘要？"请总结以下文章：...→"，模型续写摘要。问答？更不用说了。

而且Decoder-Only架构有一个巨大的工程优势：**KV Cache。** 在自回归生成时，每生成一个token，之前token的Key和Value可以缓存复用，不需要重复计算。这让推理速度大幅提升。Encoder-Decoder架构在这方面就没这么方便了。

BuddyGPT采用的就是Decoder-Only架构，以Llama2/3为蓝本。

## BuddyGPT 0.3B 模型结构全貌

先看整体结构：

```
输入 token_ids [B, L]
       │
       v
┌─────────────────────┐
│  Embedding Layer     │  vocab_size=151669, hidden_size=1024
│  token → 1024维向量  │
└──────────┬──────────┘
           │
           v
┌─────────────────────────────────────┐
│  DecoderLayer × 24                  │  ← 重复24次
│  ┌───────────────────────────────┐  │
│  │ RMSNorm                       │  │
│  │ → GQA Attention (16Q, 8KV)   │  │
│  │ → Residual Add               │  │
│  ├───────────────────────────────┤  │
│  │ RMSNorm                       │  │
│  │ → GateMLP (SwiGLU)           │  │
│  │ → Residual Add               │  │
│  └───────────────────────────────┘  │
└──────────┬──────────────────────────┘
           │
           v
┌─────────────────────┐
│  RMSNorm (最终)      │
└──────────┬──────────┘
           │
           v
┌─────────────────────┐
│  lm_head (线性层)    │  1024 → 151669 (与Embedding共享权重)
│  → logits           │
└─────────────────────┘
```

接下来我们逐层拆解。

## Embedding层：从离散到连续

Embedding做的事情很简单：把一个整数（token ID）映射到一个高维向量。

BuddyGPT使用Qwen的Tokenizer，词表大小为151669。也就是说，每个token会被映射到一个1024维的向量。Embedding层的参数矩阵大小是 `151669 × 1024`，大约1.55亿个参数——这已经占了0.3B模型总参数的一半以上！

这也是为什么BuddyGPT使用了 **Weight Tying**（权重共享）：让Embedding层和最后的lm_head共享同一个权重矩阵。这样不仅省了一半的Embedding参数，还有正则化的效果——输入和输出用同一套"词向量字典"。

## DecoderLayer：模型的核心积木

每个DecoderLayer由两部分组成：注意力模块和前馈网络，各自前面有一个RMSNorm，后面有残差连接。

### 为什么用RMSNorm而不是LayerNorm？

LayerNorm的计算：先减均值，再除标准差，最后乘gamma加beta。
RMSNorm的计算：只除RMS（均方根），然后乘gamma。

```python
# LayerNorm (简化版)
mean = x.mean(dim=-1, keepdim=True)
var = x.var(dim=-1, keepdim=True)
output = (x - mean) / sqrt(var + eps) * gamma + beta

# RMSNorm (简化版)
rms = sqrt(x.pow(2).mean(dim=-1, keepdim=True) + eps)
output = x / rms * gamma
```

RMSNorm省略了"减均值"和bias，计算更快。实践证明，在大模型场景下，RMSNorm和LayerNorm效果几乎一样，但速度快了10-15%。Llama系列都用的RMSNorm，BuddyGPT也跟进了这个选择。

### GQA Attention：16Q/8KV

这里先简单提一下，Day 3会详细展开。BuddyGPT 0.3B用16个Query head和8个Key/Value head。也就是说，每2个Q head共享一组K和V。这比标准的Multi-Head Attention（每个Q都有独立的K、V）省了一半的KV参数和KV Cache。

### GateMLP（SwiGLU）：FFN的进化

传统Transformer的FFN长这样：

```
FFN(x) = ReLU(x · W1) · W2
```

BuddyGPT使用的GateMLP用了SwiGLU激活函数：

```python
# GateMLP (SwiGLU) 实现
class GateMLP(nn.Module):
    def __init__(self, hidden_size, intermediate_size):
        super().__init__()
        # 三个线性层，没有bias
        self.gate_proj = nn.Linear(hidden_size, intermediate_size, bias=False)  # 门控
        self.up_proj   = nn.Linear(hidden_size, intermediate_size, bias=False)  # 上投影
        self.down_proj = nn.Linear(intermediate_size, hidden_size, bias=False)  # 下投影

    def forward(self, x):
        # SwiGLU: silu(gate) * up，然后降维
        gate = F.silu(self.gate_proj(x))  # silu = x * sigmoid(x)，即"Swish"激活
        up = self.up_proj(x)              # 线性变换，不加激活
        return self.down_proj(gate * up)  # 逐元素相乘后降维回hidden_size
```

为什么SwiGLU比ReLU/GELU更好？

```
激活函数对比：
ReLU:   max(0, x)           → 简单粗暴，但会"杀死"负值信息
GELU:   x · Φ(x)           → 更平滑，BERT/GPT-2用它
SwiGLU: silu(xW1) * (xW2)  → 门控机制，让模型自己学习要保留什么信息
```

SwiGLU引入了额外的gate投影，参数量增加了50%（三个矩阵 vs 两个），但效果显著提升。Google在PaLM论文中验证了这一点，Llama系列也采用了它。代价就是FFN部分的计算量增加了，但这个trade-off目前被认为是值得的。

## 参数量计算

让我们手动算一下0.3B模型的参数量：

```
各部分参数量计算（hidden=1024, inter=2048, layers=24, vocab=151669）
═══════════════════════════════════════════════════════════════
Embedding:           151669 × 1024           = 155,309,056
                    (lm_head共享，不重复计算)

每层DecoderLayer:
  Attention:
    Q proj:          1024 × 1024             =   1,048,576
    K proj:          1024 × 512              =     524,288  (8个KV head)
    V proj:          1024 × 512              =     524,288
    O proj:          1024 × 1024             =   1,048,576
  FFN (SwiGLU):
    gate_proj:       1024 × 2048             =   2,097,152
    up_proj:         1024 × 2048             =   2,097,152
    down_proj:       2048 × 1024             =   2,097,152
  RMSNorm × 2:      1024 × 2                =       2,048
  ──────────────────────────────────────────
  每层合计:                                  ≈   9,439,232

24层合计:            9,439,232 × 24          = 226,541,568
最终RMSNorm:         1024                    =       1,024
───────────────────────────────────────────────────────────
总计:                155,309,056 + 226,541,568 + 1,024
                   ≈ 381,851,648  ≈ 0.38B
```

注意：由于Weight Tying，lm_head不额外占参数。实际参数量约0.38B，我们习惯叫它"0.3B级别"。

## BuddyGPTConfig关键参数

```python
BuddyGPTConfig(
    vocab_size=151669,        # 词表大小，用Qwen的Tokenizer
    hidden_size=1024,         # 每个token的向量维度
    intermediate_size=2048,   # FFN中间层维度（通常是hidden_size的2-4倍）
    num_hidden_layers=24,     # Decoder层数
    num_attention_heads=16,   # Query的注意力头数
    num_key_value_heads=8,    # Key/Value的注意力头数（GQA: 16/8=每2个Q共享1组KV）
    tie_word_embeddings=True, # Embedding和lm_head共享权重
    rope_theta=100000.0       # RoPE的base频率（越大，支持的序列长度越长）
)
```

每个参数的选择都不是随意的。`hidden_size=1024` 在0.3B级别是标准选择；`intermediate_size=2048` 是hidden_size的2倍（Llama用的是2.68倍，但更小的模型可以低一些）；`rope_theta=100000.0` 是Llama3的选择，比Llama2的10000.0支持更长的上下文。

## 个人思考

拆解完模型架构，最大的感受是：**大模型的架构其实很"朴素"。** 没有什么魔法，就是Embedding → 一堆相似的层 → 线性输出。真正的创新在于每个组件的细节优化：RMSNorm比LayerNorm快一点，SwiGLU比ReLU好一点，GQA比MHA省一点。这些"一点点"累积起来，就是质的飞跃。

另一个感受是：**参数量的分布很不均匀。** Embedding占了将近一半的参数，但它的"信息密度"远低于Attention和FFN层。这也是为什么很多优化手段（比如Weight Tying、量化）都优先从Embedding入手。

## 参考资料

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762) - Transformer原始论文
- [LLaMA: Open and Efficient Foundation Language Models](https://arxiv.org/abs/2302.13971) - Llama架构
- [GLU Variants Improve Transformer](https://arxiv.org/abs/2002.05202) - SwiGLU的理论基础
- [Root Mean Square Layer Normalization](https://arxiv.org/abs/1910.07467) - RMSNorm论文
- [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245)
