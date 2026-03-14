# MCP协议：Agent工具调用的USB标准

> Agent深度解析系列 · Day 4

USB出现之前，每种外设都有自己的接口——打印机用并口，鼠标用PS/2，扫描仪用SCSI。USB统一了一切。MCP对Agent工具生态做的，就是同样的事情。

## N×M问题：为什么需要MCP

假设市场上有10个AI平台（OpenAI、Anthropic、Google、Mistral……）和100个工具服务（GitHub、Slack、数据库、搜索引擎……）。没有统一协议的世界里，每个工具要为每个平台写一套适配代码，总共需要 10 × 100 = 1000 个集成。

MCP将这个问题简化为 **10 + 100 = 110**：每个平台实现一次MCP Client，每个工具实现一次MCP Server，就能互通。这就是"USB标准"的类比——设备厂商只需实现USB协议，不用关心插在哪台电脑上。

## 起源与治理

MCP由Anthropic在2024年11月发布，开源协议。2025年3月OpenAI宣布支持，此后Google、Amazon、Microsoft等主流厂商纷纷跟进。

2025年12月，Anthropic将MCP捐赠给Linux基金会下的AAIF（AI Alliance Infrastructure Foundation），转为社区治理。这一步至关重要——协议由中立组织管理，消除了"Anthropic私有标准"的顾虑，加速了行业采纳。

截至2026年初，MCP的Python和TypeScript SDK月下载量合计超过9700万次。

## 三层架构

MCP采用经典的三层架构，通信基于 **JSON-RPC 2.0**：

```
┌─────────────────────────────┐
│          Host               │  ← AI应用（Claude Desktop, IDE, 自定义App）
│  ┌───────┐  ┌───────┐      │
│  │Client1│  │Client2│ ...  │  ← 每个Client连一个Server
│  └───┬───┘  └───┬───┘      │
└──────┼──────────┼───────────┘
       │          │
  ┌────▼───┐ ┌───▼────┐
  │Server A│ │Server B│         ← MCP Server（工具提供方）
  └────────┘ └────────┘
```

- **Host**：AI应用层，管理生命周期和安全策略
- **Client**：协议实现层，维护与Server的1:1连接，处理协议协商
- **Server**：工具提供层，暴露具体能力给AI使用

传输层支持两种模式：本地进程用`stdio`，远程服务用`HTTP + SSE（Server-Sent Events）`。

## 四种核心能力

MCP Server可以暴露四种类型的能力：

**1. Resources（资源）**：只读数据源。类似REST的GET——文件内容、数据库记录、API数据。Agent可以读取但不能修改。

**2. Tools（工具）**：可执行的动作。这是最常用的能力——发邮件、创建PR、查询天气。每次调用都需要LLM主动发起。

**3. Prompts（提示模板）**：预定义的交互模板。比如"代码审查模板"、"SQL生成模板"，帮助用户快速启动特定场景。

**4. Sampling（采样）**：最特殊的能力——Server可以反向请求Host的LLM进行推理。这打开了"工具反过来用AI"的可能性，实现更复杂的交互模式。

## 代码示例：最小MCP Server

用TypeScript实现一个返回当前时间的MCP Server，不到30行：

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "time-server", version: "1.0.0" });

server.tool(
  "get_current_time",
  "获取指定时区的当前时间",
  { timezone: z.string().default("UTC").describe("时区，如 Asia/Shanghai") },
  async ({ timezone }) => ({
    content: [{
      type: "text",
      text: new Date().toLocaleString("zh-CN", { timeZone: timezone })
    }]
  })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

注册一个工具，定义参数schema，返回结果。任何支持MCP的AI应用都能直接调用。

## 2026新进展

**MCP Apps（2026年1月）**：Server不再只能返回文本，可以返回交互式UI组件。用户查询航班时，Agent可以直接展示一个可操作的航班选择界面，而不是一堆文字。这模糊了"工具调用"和"应用交互"的边界。

**WebMCP**：让Agent能像人一样操作Web页面——点击按钮、填写表单、导航页面。结合传统MCP的API调用能力，Agent的操作范围覆盖了几乎所有数字化系统。

**安全增强**：OAuth 2.1集成、权限分级、审计日志。企业级采用的必要条件。

## 我的判断

MCP已经赢了工具调用标准之争。不是因为它技术上完美（JSON-RPC在2026年其实有点过时），而是因为它足够简单、足够开放、足够早。在协议标准化的战争中，先发优势和生态规模比技术优雅更重要。

## 参考资料

- [Model Context Protocol - Official Documentation](https://modelcontextprotocol.io/)
- [MCP vs A2A: Complete Guide - DEV Community](https://dev.to/composio/mcp-vs-a2a-the-complete-guide-to-ai-agent-protocols-3k0p)
- [MCP Architecture Deep Dive - Clarifai](https://www.clarifai.com/blog/model-context-protocol-mcp)
- [Anthropic Donates MCP to Linux Foundation](https://www.linuxfoundation.org/press/anthropic-donates-model-context-protocol)
