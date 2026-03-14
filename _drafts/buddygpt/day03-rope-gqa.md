# RoPE旋转位置编码与GQA分组注意力

> 从零训练大模型系列 · Day 3

## Transformer的"失忆症"：为什么需要位置编码？

来做个思想实验。把下面两句话喂给一个没有位置编码的Transformer：

- "小明打了小红"
- "小红打了小明"

在Attention计算中，每个token和其他所有token做交互。但如果没有位置信息，模型看到的是**同一组token的同一组交互**——"小明"和"打了"的注意力权重不会因为顺序不同而改变。换句话说，模型分不清"谁打了谁"。

这就是位置编码存在的意义：**告诉模型每个token在序列中的位置。**

早期方案包括：
- **绝对位置编码**（原始Transformer）：给每个位置一个固定的向量，加到Embedding上
- **可学习位置编码**（BERT/GPT-2）：位置向量作为可训练参数

这些方案的问题是：**训练时见过的最大长度就是天花板。** 训练长度512，推理时给它1024的文本，效果就崩了。

## RoPE：用旋转编码位置

RoPE（Rotary Position Embedding）是苏剑林在2021年提出的方案，现在几乎所有主流大模型都在用。它的核心思想用一句话概括：**不要把位置信息加到向量上，而是通过旋转向量来编码位置。**

### 从二维旋转说起

假设我们有一个二维向量 `(x₁, x₂)`，想编码它在位置 `m` 的信息。RoPE的做法是把这个向量旋转 `mθ` 角度：

```
旋转前: (x₁, x₂)
旋转后: (x₁·cos(mθ) - x₂·sin(mθ),  x₁·sin(mθ) + x₂·cos(mθ))
```

用复数表示更优雅：`(x₁ + ix₂) × e^(imθ) = 旋转后的复数`

### 为什么旋转能编码相对位置？

这是RoPE最精妙的地方。看Q和K的内积：

```
q在位置m: q' = q × e^(imθ)
k在位置n: k' = k × e^(inθ)

内积: q' · k'* = q · k* × e^(i(m-n)θ)
                              ^^^^^^^^
                        只依赖于位置差 (m-n)！
```

位置m的Query和位置n的Key做内积，结果只和它们的相对距离 `m-n` 有关，与绝对位置无关。这就是"旋转位置编码天然支持相对位置"的数学本质。

### 推广到高维

实际的hidden维度不是2，而是64、128这样的数。RoPE的做法是：**每两个维度一组，分别用不同频率旋转。**

```
维度:  [d₀,d₁] [d₂,d₃] [d₄,d₅] ... [d₆₂,d₆₃]
频率:    θ₀      θ₁      θ₂    ...    θ₃₁

其中 θᵢ = 1 / (base^(2i/dim))
base = 10000 (Llama2) 或 100000 (Llama3/BuddyGPT)
```

低维度对用高频旋转（变化快，捕捉近距离关系），高维度对用低频旋转（变化慢，捕捉远距离关系）。base越大，频率整体越低，能支持的序列长度就越长——这就是BuddyGPT选择 `rope_theta=100000` 的原因。

### RoPE代码实现

```python
class RotaryEmbedding(nn.Module):
    """旋转位置编码"""
    def __init__(self, dim, base=100000.0):
        super().__init__()
        # 计算每个维度对的频率
        # dim=64时，生成32个频率值: θ₀, θ₁, ..., θ₃₁
        inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
        self.register_buffer("inv_freq", inv_freq)

    def forward(self, x, position_ids):
        # position_ids: [B, L]，每个位置的编号 0,1,2,...
        # inv_freq: [dim//2]

        # 外积：每个位置 × 每个频率 = [B, L, dim//2]
        freqs = torch.einsum("bi,j->bij", position_ids.float(), self.inv_freq)

        # 拼接两份，因为cos和sin要作用于完整维度
        emb = torch.cat((freqs, freqs), dim=-1)  # [B, L, dim]

        return emb.cos(), emb.sin()  # 返回cos和sin缓存


def apply_rotary_pos_emb(q, k, cos, sin):
    """
    对Q和K应用旋转位置编码
    q, k: [B, n_heads, L, head_dim]
    cos, sin: [B, L, head_dim] → 需要unsqueeze到[B, 1, L, head_dim]
    """
    cos = cos.unsqueeze(1)  # [B, 1, L, dim]
    sin = sin.unsqueeze(1)

    # 旋转的核心操作
    # rotate_half: [x₁,x₂,x₃,x₄,...] → [-x₂,x₁,-x₄,x₃,...]
    # 这等价于对每个2D子空间做旋转
    q_embed = q * cos + rotate_half(q) * sin
    k_embed = k * cos + rotate_half(k) * sin

    return q_embed, k_embed


def rotate_half(x):
    """将向量的前半部分和后半部分交换并取负，模拟复数乘法"""
    x1 = x[..., :x.shape[-1] // 2]   # 前半: [x₁, x₂, ...]
    x2 = x[..., x.shape[-1] // 2:]   # 后半: [x₃, x₄, ...]
    return torch.cat((-x2, x1), dim=-1)  # [-x₃, -x₄, ..., x₁, x₂, ...]
```

## GQA：在速度和质量之间找到平衡

### 从MHA到MQA再到GQA

传统的Multi-Head Attention（MHA）中，每个Query head都有自己独立的Key head和Value head。如果有16个head，就有16组Q、16组K、16组V。

```
MHA (Multi-Head Attention):      16 Q heads,  16 K heads,  16 V heads
                                 每个Q有自己独立的K和V
                                 参数多，KV Cache大

MQA (Multi-Query Attention):     16 Q heads,   1 K head,    1 V head
                                 所有Q共享同一个K和V
                                 参数少，KV Cache小，但质量下降明显

GQA (Grouped Query Attention):   16 Q heads,   8 K heads,   8 V heads
                                 每2个Q共享一组K和V
                                 折中方案：质量接近MHA，速度接近MQA
```

用图来理解：

```
MHA:  Q₀─K₀  Q₁─K₁  Q₂─K₂  ... Q₁₅─K₁₅     (16组独立的KV)

MQA:  Q₀─┐   Q₁─┐   Q₂─┐   ... Q₁₅─┐
         └──K₀───┘       └────────────┘         (1组共享的KV)

GQA:  Q₀─┐       Q₂─┐       Q₄─┐             Q₁₄─┐
      Q₁─┴─K₀    Q₃─┴─K₁    Q₅─┴─K₂   ...   Q₁₅─┴─K₇
      (每2个Q共享1组KV，共8组)
```

BuddyGPT使用 `num_attention_heads=16, num_key_value_heads=8` 的GQA配置。

### repeat_kv：让KV匹配Q的数量

在实际计算中，我们需要让K和V的head数量和Q匹配。做法很简单——把每组KV复制几份：

```python
def repeat_kv(hidden_states, n_rep):
    """
    把KV的head数量通过复制扩展到和Q一样多
    hidden_states: [B, n_kv_heads, L, head_dim]
    n_rep: 每组KV需要复制的次数 (= n_heads // n_kv_heads)
    """
    if n_rep == 1:
        return hidden_states  # MHA的情况，不需要复制

    batch, n_kv_heads, seq_len, head_dim = hidden_states.shape

    # 在head维度上增加一个维度，然后expand复制
    hidden_states = hidden_states[:, :, None, :, :]  # [B, n_kv, 1, L, dim]
    hidden_states = hidden_states.expand(batch, n_kv_heads, n_rep, seq_len, head_dim)

    # reshape回标准形状
    return hidden_states.reshape(batch, n_kv_heads * n_rep, seq_len, head_dim)
```

注意：`expand` 不会真的复制数据（零内存开销），只是创建了一个"视图"。实际的内存节省发生在KV Cache中——缓存8组KV比缓存16组省了一半的显存。

### GQA省了多少显存？

以BuddyGPT 0.3B为例，序列长度2048：

```
KV Cache 每层大小 = 2(K+V) × n_kv_heads × seq_len × head_dim × 2(fp16字节)

MHA (16 KV heads): 2 × 16 × 2048 × 64 × 2 = 8,388,608 bytes ≈ 8MB/层
GQA ( 8 KV heads): 2 ×  8 × 2048 × 64 × 2 = 4,194,304 bytes ≈ 4MB/层

24层总计:
MHA: 8MB × 24 = 192MB
GQA: 4MB × 24 = 96MB  → 节省50%!
```

在更大的模型和更长的序列下，这个差距会更加显著。对于70B模型+4096长度，GQA可以节省GB级别的显存。

### SdpaAttention：硬件加速

BuddyGPT使用PyTorch内置的 `scaled_dot_product_attention` (SDPA) 来计算注意力，它会自动选择最优的底层实现：

```python
class SdpaAttention(nn.Module):
    def forward(self, hidden_states, attention_mask=None, position_ids=None):
        B, L, _ = hidden_states.shape

        # 投影Q、K、V
        q = self.q_proj(hidden_states).view(B, L, self.n_heads, self.head_dim).transpose(1, 2)
        k = self.k_proj(hidden_states).view(B, L, self.n_kv_heads, self.head_dim).transpose(1, 2)
        v = self.v_proj(hidden_states).view(B, L, self.n_kv_heads, self.head_dim).transpose(1, 2)

        # 应用RoPE旋转位置编码
        cos, sin = self.rotary_emb(v, position_ids)
        q, k = apply_rotary_pos_emb(q, k, cos, sin)

        # GQA: 复制KV以匹配Q的head数
        k = repeat_kv(k, self.n_heads // self.n_kv_heads)
        v = repeat_kv(v, self.n_heads // self.n_kv_heads)

        # 使用PyTorch的SDPA，自动选择FlashAttention/Math/Memory-efficient实现
        # is_causal=True 自动应用因果mask（下三角），无需手动构造mask
        attn_output = F.scaled_dot_product_attention(
            q, k, v,
            attn_mask=attention_mask,
            is_causal=True  # Decoder-Only必须用因果注意力
        )

        # 合并多头输出
        attn_output = attn_output.transpose(1, 2).reshape(B, L, -1)
        return self.o_proj(attn_output)
```

`F.scaled_dot_product_attention` 会自动选择FlashAttention v2（如果硬件支持），它的内存复杂度从O(L²)降到O(L)，速度提升2-4倍。不需要你手动实现FlashAttention，PyTorch帮你搞定了。

## 个人思考

RoPE和GQA是现代大模型的两个基础组件。理解它们之后，你会发现大模型的很多设计都遵循一个共同的哲学：**在数学上找到优雅的简化，在工程上找到性价比最高的trade-off。**

RoPE用旋转编码位置，数学上等价于相对位置编码，但实现比传统的相对位置编码简单得多。GQA在MHA和MQA之间找到了甜蜜点，几乎不损失质量但大幅节省资源。

这种"简化但不简单"的设计哲学，值得我们在其他工程问题中借鉴。

## 参考资料

- [RoFormer: Enhanced Transformer with Rotary Position Embedding](https://arxiv.org/abs/2104.09864) - 苏剑林的RoPE论文
- [GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints](https://arxiv.org/abs/2305.13245)
- [Fast Transformer Decoding: One Write-Head is All You Need](https://arxiv.org/abs/1911.02150) - MQA原始论文
- [FlashAttention: Fast and Memory-Efficient Exact Attention](https://arxiv.org/abs/2205.14135)
- [苏剑林的博客：让研究人员绞尽脑汁的Transformer位置编码](https://spaces.ac.cn/archives/8130)
