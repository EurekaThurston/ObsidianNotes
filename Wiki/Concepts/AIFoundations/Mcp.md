---
type: concept
created: 2026-04-19
updated: 2026-04-26
tags: [ai, agent, protocol, tooling]
sources: 1
aliases: [MCP, Model Context Protocol, 模型上下文协议]
---

# MCP（Model Context Protocol）

> AI 世界的 **USB-C 接口**——让工具只实现一次，就能被任何 AI 应用调用。

## 为什么需要 MCP

### 一个能立刻代入的场景

你打开 ChatGPT 网页,问它:**"帮我读一下我电脑上某某文件,总结一下。"**

它会说:"很抱歉,我无法访问你的本地文件……"

这是 LLM 的根本限制:**它只会说话,不会做事**。它的能力被锁在一个文本对话框里——看不到你电脑上的文件、查不了实时网页、调不了你电脑上的程序、操作不了 GitHub / Figma / 数据库。

把"做事"的能力还给 LLM,有几种解法:

| 解法 | 做法 | 缺点 |
|---|---|---|
| **A. 复制粘贴** | 你手动把文件内容粘进对话框 | 累、上下文塞不下、不能动态查;LLM 只能"读到"不能"操作" |
| **B. [[Wiki/Concepts/Methodology/Rag\|RAG]]** | 把数据预处理成向量库,LLM 查询时检索 | 只能"读",不能"写";只能取文本片段,不能调一个动作(如"重命名文件""提交代码") |
| **C. Function Calling**(2023 OpenAI) | 让 LLM 输出结构化 JSON 说"我要调 read_file('xxx')",程序解析后执行 | **每家 LLM 格式都不一样**——给 GPT 写的工具 Claude 用不了,反过来也一样;N 个模型 × M 个工具 = N×M 套定制集成 |
| **D. MCP**(2024-11 Anthropic 推出) ⭐ | 把"工具"做成独立小程序,通过统一协议跟任何 LLM 对话 | — |

> [!abstract] 一句话
> **MCP = 给 LLM 装手脚的统一插头**。把"读文件 / 调 git / 操作 Figma / 查数据库"做成一个个小程序(MCP server),任何支持 MCP 的 AI 应用(Claude Desktop / Cursor / 自研 agent)都能即插即用。

### Function Calling → MCP 的演进

2023 年 OpenAI 推出 **Function Calling**,首次让 LLM 可以"调外部工具"。但每家 AI 公司的接口格式都不一样,工具开发者要给 GPT 写一遍、给 Claude 写一遍、给 Gemini 写一遍——**这就是 MCP 要消除的"N × M 集成难题"**。

**类比**:USB-C 之前,鼠标 PS/2、打印机并口、手机各种充电线。USB-C 统一了所有。MCP 对 AI 工具做的是同样的事。

---

## 三个角色

```
[ MCP Server ]   ← 真正干活的小程序(读文件 / 查 git / 操作 Figma...)
       ↑
       │  (MCP 协议:统一格式)
       ↓
[ MCP Client ]   ← Host 内部的"插座",负责跟 Server 说协议
       ↑
       │
[ MCP Host  ]    ← 用户实际看到的 AI 应用(Claude Desktop / Cursor / IDE / 自研 agent)
```

具体代入:

- **MCP Host(宿主)**:发起请求的 AI 应用——用户实际打开的窗口。例:Claude Desktop、Cursor、Windsurf、VS Code Cline。
- **MCP Client(客户端)**:Host 内部一段代码,负责按 MCP 协议跟外部 server 对话。普通用户看不见它,Host 开发者要实现一次。
- **MCP Server(服务器)**:对外提供工具或数据的独立小程序。可以是官方写的(filesystem / git / fetch),也可以是第三方写的(GitHub / Figma / Notion / Slack),也可以是你自己写的业务系统。

> [!abstract] 杠杆从哪儿来
> **工具开发者只需写一次 MCP Server,所有支持 MCP 的 AI 都能用。Host 开发者只需实现一次 MCP Client,就能挂上全世界的 MCP Server。** 写 server 的人和写 host 的人都得到了"一次劳动 N 次回报"的杠杆——这是 MCP 真正的价值。

---

## MCP server 长什么样(具体到能摸)

它就是个普通小程序(Node / Python / Go 都行),启动后通过标准输入输出收发 JSON。一个最小可运行的 server 大概这样(TypeScript):

```typescript
// my-server/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import * as fs from "fs";

const server = new Server({ name: "my-tool", version: "0.1.0" });

// 1. 声明你能提供哪些 tool
server.setRequestHandler("tools/list", async () => ({
  tools: [{
    name: "read_file",
    description: "读取指定路径的文件内容",
    inputSchema: {
      type: "object",
      properties: { path: { type: "string" } },
      required: ["path"],
    },
  }],
}));

// 2. 实现 tool 真正干啥
server.setRequestHandler("tools/call", async (req) => {
  if (req.params.name === "read_file") {
    const text = fs.readFileSync(req.params.arguments.path, "utf-8");
    return { content: [{ type: "text", text }] };
  }
});

// 3. 通过 stdio 跟 Host 说话
server.connect(new StdioServerTransport());
```

100 行内写完就能跑。**关键性质**:这个 server 不知道是谁在调它——今天 Claude Desktop 调它,明天 Cursor 调它,后天某个新出的 AI 应用调它,**server 一行代码不用改**。这就是"协议"的力量。

---

## Host 怎么找到 server(发现机制)

> [!abstract]
> **MCP 协议本身不规定"server 怎么被发现"——发现是 Host 的事**。Host 不去"找"server,你直接在配置文件里告诉 Host 启动命令,Host 启动时把 server spawn 起来用 stdio 通话。"位置"就是配置里那行 `command` + `args`。

### 配置文件:每个 Host 一个

主流 Host 的配置文件位置:

| Host | 配置文件 |
|---|---|
| Claude Desktop(Win) | `%APPDATA%\Claude\claude_desktop_config.json` |
| Claude Desktop(macOS) | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Cursor | `~/.cursor/mcp.json` 或项目级 `.cursor/mcp.json` |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| VS Code Cline | 通过插件 GUI 配,底层文件随版本 |
| 自研 Host | 自定;**强烈建议抄 Claude Desktop 格式**复用生态 |

> [!tip] 为什么大家都抄 Claude Desktop 格式
> Anthropic 推 MCP 时把 Claude Desktop 配置格式实质化成了"事实标准"。后来者复用同一格式 → 用户在多个 Host 之间一份配置通用 → 写 server 的人和用 server 的人都受益。这不是协议规定,是惯例。

### 配置内容(Claude Desktop 事实标准)

```jsonc
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "D:/Notes"]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "D:/Projects/MyRepo"]
    },
    "my-custom-tool": {
      "command": "node",
      "args": ["D:/code/my-server/dist/index.js"]
    }
  }
}
```

每个 server 一个 entry,`command` + `args` 就是告诉 Host 怎么启动它。

### Host 启动时序

```
1. Host 启动(用户双击应用)
2. Host 读 config 文件
3. 对每个 server entry:
   - spawn 子进程,执行 `command args...`
   - 跟该子进程的 stdin/stdout 建立 stdio 连接
   - 走 MCP 协议握手(initialize)
   - 调 tools/list 拉到该 server 提供的所有 tool
4. Host 把所有 server 的 tools 汇总,注册给 LLM
5. 用户开始聊天,LLM 输出 tool_call → Host 路由到对应 server
```

### 三种 server 描述方式

| 方式 | 配置写法 | 适用 |
|---|---|---|
| **本地可执行**(stdio) | `"command": "node", "args": ["/path/server.js"]` | 自写或本地构建的 server,路径明确 |
| **NPM 即装即用**(stdio) | `"command": "npx", "args": ["-y", "@scope/server-name"]` | 官方/社区 npm 包,首次自动下载,后续缓存 |
| **Python uvx**(stdio) | `"command": "uvx", "args": ["mcp-server-xxx"]` | Python 写的 server,uv 工具一行装 |
| **远程服务器**(SSE / HTTP) | `"url": "https://api.example.com/mcp"` | server 已部署在云上,Host 只是连 URL |

> [!tip]
> `npx -y` 和 `uvx` 是关键便利——用户**不用预装任何东西**,把那行配置抄进去,Host 启动时自动从 npm/PyPI 拉包并跑。所以官方文档的"安装步骤"经常就是"把这段 JSON 粘到配置里",没了。

### 当前的"发现"现状

> [!warning] 没有官方应用商店
> MCP 不像浏览器插件、VS Code Extension 有官方中央仓库。**用户必须知道某个 server 存在,手动加配置**。

现状的发现路径:

| 来源 | 说明 |
|---|---|
| **官方仓库** [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) | filesystem / git / fetch / sqlite / postgres / brave-search 等参考实现 |
| **Awesome list**(如 [awesome-mcp-servers](https://github.com/punkpeye/awesome-mcp-servers))| 社区收录,几百个 server,人工维护 |
| **第三方 registry** | Smithery、mcp-get、Pulse MCP 等;部分提供"一键安装"工具帮你改 Host 的配置文件 |
| **厂商自营** | GitHub 官方出 mcp-server-github、Notion 官方出 mcp-server-notion 等 |
| **集中式 launcher** | 如 [moeru-ai/mcp-launcher](https://github.com/moeru-ai/mcp-launcher) ——"MCP server 的 Ollama",管 server 进程 + 帮拉包 |

---

## 行业采纳

- 2024-11 Anthropic 发布开放规范
- OpenAI、Google DeepMind、Cursor、GitHub、Vercel 等陆续支持
- 2025 年起事实上已是行业标准——主流 AI Host 几乎都支持 MCP
- 官方 server 仓库 [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) 收录大量参考实现(filesystem / git / fetch / sqlite / postgres / brave-search / ...)
- 具体治理(是否捐赠给某基金会等组织细节)以 Anthropic 官方公告为准

---

## 与相关概念的区别

| 概念 | 定位 | 对比 |
|---|---|---|
| **Function Calling** | 各家专有的函数调用 | MCP 之前的做法,各不兼容 |
| **MCP** | 跨厂商标准协议 | 统一接口 |
| **[[Wiki/Concepts/AIFoundations/Agent-skills\|Skill]]** | 模块化专家知识包 | 告诉 AI"怎么做";MCP 告诉 AI"能做什么" |
| **[[Wiki/Concepts/Methodology/Rag\|RAG]]** | 检索式上下文注入 | 把数据"塞"给 AI;MCP 让 AI"主动去取/做" |
| **CLAUDE.md / AGENTS.md** | 项目级规则文件 | 针对当前项目的约定 |

简记:**MCP 是协议(管道),Skill 是知识(内容),RAG 是检索(被动塞),CLAUDE.md 是章程(规则)。**

---

## 相关

- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — MCP 是 Agent 连外部世界的主要手段
- [[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] — 与 MCP 互补:Skills 给知识,MCP 给工具
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 把数据塞给 AI 的另一条路径(检索),MCP 则是让 AI 主动去取/做

## 引用来源

- 主题读本(推荐通读):[[Readers/AIFoundations/AI 应用生态全景 2026]]
- 原子 source:[[Wiki/Sources/AIFoundations/AI-primer-v2]] (raw: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]])

## 开放问题

- MCP Server 的安全模型:本地 server 有文件系统访问权,远程 server 可能访问敏感 API——权限边界、首次调用确认、审计日志如何做?
- 多 MCP Server 并存时的工具冲突/命名空间问题(常用做法:`<server>__<tool>` 前缀)
- MCP server 的官方应用商店是否会出现——当前发现机制(见上"Host 怎么找到 server")是分散的(官方仓库 + awesome list + 第三方 registry + 厂商自营),用户体验对非技术人群仍门槛偏高
