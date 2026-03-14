# Tool Use：Agent的手和脚

> Agent深度解析系列 · Day 3

LLM再聪明，也只是一个"嘴强王者"——它能说，但不能做。Tool Use是让Agent从"会说话的模型"变成"能干活的系统"的关键跳跃。

## 为什么Agent必须有工具

LLM有三个硬伤，靠模型本身无法解决：

1. **知识截止**：训练数据有时间边界，问它今天的股价，它只能编。接上搜索工具，问题消失。
2. **不能精确计算**：让GPT算`17 * 384`，它可能算对也可能算错。接上代码解释器，100%准确。
3. **不能操作外部系统**：它不能发邮件、不能改数据库、不能下单购物。工具是它与真实世界交互的唯一通道。

一句话总结：**LLM负责决策，工具负责执行。** Agent的价值不在于模型多强，而在于它能调动多少工具、多可靠地使用这些工具。

## Function Calling：工具调用的核心机制

Function Calling是目前主流的工具调用实现，流程分四步：

**第一步：描述（Describe）**
开发者用JSON Schema描述每个工具的名称、用途和参数。这些描述会作为系统提示词注入LLM上下文。

```json
{
  "name": "get_weather",
  "description": "获取指定城市的当前天气",
  "parameters": {
    "type": "object",
    "properties": {
      "city": { "type": "string", "description": "城市名称" }
    },
    "required": ["city"]
  }
}
```

**第二步：选择（Select）**
用户说"北京今天天气怎么样"，LLM分析意图后，输出结构化的工具调用请求，而非自然语言回答。

**第三步：调用（Execute）**
应用层拿到LLM的调用请求，执行实际的API调用，获取真实数据。

**第四步：返回（Return）**
将工具执行结果喂回LLM，LLM基于真实数据生成最终回答。

关键点：**LLM从不直接执行工具**。它只输出"我想调用什么工具、传什么参数"，实际执行权在应用层。这是安全设计——LLM是建议者，不是执行者。

## 工具设计三原则

工具的质量直接决定Agent的可靠性。设计不好的工具，LLM要么不会用，要么用错：

**原则一：描述清晰**
工具描述就是给LLM看的说明书。模糊的描述 = 错误的调用。`"处理数据"` 是烂描述，`"将CSV文件按指定列排序并返回前N行"` 才是好描述。

**原则二：参数明确**
每个参数的类型、范围、默认值都要标注清楚。枚举值比自由文本好——`enum: ["asc", "desc"]` 远比 `"排序方向"` 更不容易出错。

**原则三：错误可恢复**
工具调用会失败，这是常态。好的工具在失败时返回有意义的错误信息，让LLM能理解失败原因并重试或换一种方式。返回`"error: 404 not found"` 远好于直接崩溃。

## 实战中的典型工具链

不同类型的Agent配备不同的工具链：

- **搜索类**：Web搜索、知识库检索、数据库查询。RAG本质上也是一种工具调用。
- **代码执行**：Python/JS沙箱，让Agent能写代码并立即验证结果。这是最强大的工具类型——等于给了Agent一台电脑。
- **文件操作**：读写文件、创建文档、管理目录。Coding Agent的核心能力。
- **浏览器控制**：Playwright/Puppeteer驱动，让Agent能浏览网页、填表单、抓数据。复杂但威力巨大。
- **通信类**：发邮件、发消息、调用第三方API。这类工具通常需要更严格的权限控制。

## 2026趋势：从碎片化到标准化

2025年以前，每个AI平台有自己的工具定义格式。OpenAI一套，Anthropic一套，Google又一套。开发者写一个工具要适配N个平台。

2026年，**MCP（Model Context Protocol）** 正在统一这个混乱局面。MCP定义了一套标准的工具描述和调用协议，写一次工具，所有支持MCP的平台都能用。这个趋势对开发者是巨大利好——工具变成了可复用的资产，而不是平台绑定的代码。

下一篇我们专门聊MCP。

## 参考资料

- [Function Calling - OpenAI API Documentation](https://platform.openai.com/docs/guides/function-calling)
- [Tool Use (Function Calling) - Anthropic Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
