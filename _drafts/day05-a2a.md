# A2A协议：Agent之间怎么对话

> Agent深度解析系列 · Day 5

MCP解决了"Agent怎么用工具"，但还有一个问题没解决：**Agent怎么跟Agent协作？** 当你的研究Agent找到了数据，怎么把任务交给写作Agent？写作Agent完成后，怎么通知审核Agent来检查？A2A就是为这个场景设计的协议。

## A2A vs MCP：不是竞争，是分工

先把定位说清楚，这是最容易混淆的地方：

| 维度 | MCP | A2A |
|------|-----|-----|
| 解决什么 | Agent如何使用工具 | Agent如何与Agent协作 |
| 类比 | USB接口 | HTTP协议 |
| 通信方向 | Agent → Tool（单向控制） | Agent ↔ Agent（双向对等） |
| 谁知道对方 | Agent知道工具的Schema | Agent知道对方的Agent Card |
| 核心场景 | 调API、查数据库、执行代码 | 委派任务、协同工作、跨团队协作 |

一句话：**MCP是Agent的"手"，A2A是Agent的"嘴"。** 手用来操作工具，嘴用来跟同事沟通。两者互补，不是替代。

## 起源：Google主导，百家响应

A2A由Google在2025年4月发布，起初叫"Agent2Agent Protocol"。与MCP不同，A2A从一开始就强调多厂商参与——首批合作伙伴包括Salesforce、SAP、Atlassian、ServiceNow等企业软件巨头。

到2026年2月，已有超过100家企业宣布支持A2A。Google同样将A2A提交给Linux基金会AAIF管理，与MCP成为"兄弟协议"。

## 三个核心概念

A2A的设计围绕三个概念构建：

### Agent Card（能力名片）

每个Agent发布一个JSON格式的Agent Card，描述自己能做什么。类似于人类的名片+简历：

```json
{
  "name": "research-agent",
  "description": "深度互联网调研，返回结构化报告",
  "url": "https://agents.example.com/research",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "web-research",
      "name": "Web深度调研",
      "description": "基于关键词进行多源调研并生成报告"
    }
  ]
}
```

Agent Card通常托管在 `/.well-known/agent.json`，其他Agent可以自动发现和读取。这个设计借鉴了OAuth的`.well-known`约定，简洁有效。

### Task（任务）

A2A中的核心交互单元。一个Agent向另一个Agent发起Task，Task有完整的生命周期：

```
submitted → working → completed
                   → failed
                   → input-required（需要更多信息）
```

`input-required` 状态很重要——它允许Agent之间进行多轮协商。"我需要你明确一下目标市场是哪个区域" → "聚焦东南亚" → 继续执行。这不是简单的请求-响应，而是真正的协作对话。

### Message（消息）

Task内部的通信载体。每条Message可以包含多个Part：文本、文件、结构化数据、甚至交互式表单。这种设计让Agent之间的信息交换非常灵活——不只是传文字，可以传数据集、传图表、传中间结果。

## 三层协议栈：2026的行业共识

到2026年初，行业逐渐形成了一个三层协议栈共识：

```
┌─────────────────────────┐
│    A2A（Agent间通信）     │  ← Agent发现、任务委派、协作
├─────────────────────────┤
│    MCP（工具调用）        │  ← 工具描述、调用、结果返回
├─────────────────────────┤
│   WebMCP（Web操作）      │  ← 浏览器控制、页面交互
└─────────────────────────┘
```

这三层各司其职：WebMCP让Agent能操作网页，MCP让Agent能调用API和工具，A2A让Agent能互相协作。一个企业级Agent系统，三层都需要。

## 企业场景：为什么A2A是刚需

单Agent能处理80%的简单任务，但企业场景往往需要多个专业Agent协作：

**场景：智能内容工厂**
1. 用户向"项目经理Agent"提交需求："写一篇关于MCP协议的技术博客"
2. 项目经理Agent通过A2A将调研任务委派给"研究Agent"
3. 研究Agent完成调研，通过A2A将结构化资料发送给"写作Agent"
4. 写作Agent生成初稿，通过A2A提交给"审核Agent"
5. 审核Agent提出修改意见，写作Agent修订后返回终稿

每个Agent都是独立服务，可能由不同团队维护、部署在不同基础设施上。没有A2A这样的标准协议，这种跨团队、跨系统的Agent协作就只能靠硬编码集成——脆弱、不可扩展、维护噩梦。

**场景：企业采购流程**
- 需求Agent（收集采购需求）→ 供应商Agent（筛选供应商）→ 合规Agent（检查合规性）→ 审批Agent（走审批流）→ 财务Agent（处理付款）

每个环节是不同部门的Agent，A2A让它们像流水线一样衔接。

## 我的观点

A2A目前还处于早期阶段，实际落地案例远少于MCP。但它解决的是一个真实且重要的问题：**当Agent数量从1个变成10个、100个时，Agent间的通信标准是绕不过去的基础设施。**

值得注意的是，A2A和MCP都在AAIF旗下，未来很可能深度融合——一个Agent通过MCP使用工具，通过A2A与其他Agent协作，对外表现为一个统一的能力体。这种"协议栈"的思路，跟互联网的TCP/IP分层模型如出一辙。

## 参考资料

- [Agent2Agent Protocol - Google](https://google.github.io/A2A/)
- [A2A vs MCP: Understanding Agent Communication - DEV Community](https://dev.to/composio/mcp-vs-a2a-the-complete-guide-to-ai-agent-protocols-3k0p)
- [The Rise of Agentic AI Architecture - InfoQ](https://www.infoq.com/articles/agentic-ai-architecture/)
- [A2A Joins Linux Foundation AAIF](https://www.linuxfoundation.org/press/a2a-protocol-joins-aaif)
