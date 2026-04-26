---
type: synthesis
created: 2026-04-25
updated: 2026-04-25
tags: [aiagents, mcp, mcp-host, desktop-pet, implementation]
sources: 1
aliases: [MCP host 手册, MCP Host Implementation, MCP 接入手册]
---

# 桌宠侧 MCP host 实施手册

> 把"桌宠是 MCP host"这件事拆成可施工的工程蓝图。议题概念见 [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]];整体路线见 [[Readers/AIAgents/桌宠 AI 入口的从零方案]] § P2。本页是 P2 单独成册的执行手册。

## 0. 为什么 MCP 不能后置

> [!warning] 这是整本手册的"为什么"
> 桌宠最容易踩的坑——先做"chat + 形象",觉得"工具调用之后再加"。但 MCP 决定 Hub **整个数据流的形状**:tool 注册回路、function calling schema 桥接、tool result 回灌上下文、设置 UI 的数据模型。等你单体 chatbot 写完再回来改,几乎等于推倒 Hub。所以从 P0 就要按 MCP-host 设计;早期可以**只挂 0 个 MCP server**,但代码路径必须存在。

## 1. MCP host 的五个职责

| 职责 | 说明 | 谁负责 |
|---|---|---|
| **1. server 进程管理** | spawn / 连接 / health-check / restart 各个 MCP server | MCP Manager(本手册主角) |
| **2. tools 聚合** | 从所有连上的 server 拉 `tools/list`,合并 + 命名空间防冲突 | MCP Manager |
| **3. tool calling 桥接** | LLM 输出 `tool_calls` → 路由到对应 server 的 `tools/call` → 结果回灌对话 | LLM Router ↔ MCP Manager |
| **4. 配置/UI** | 用户增删 server、查看可用 tools、看实时日志 | 设置面板 |
| **5. 安全/边界** | tool 调用确认、能力白名单、审计日志 | MCP Manager + 前端确认 UI |

## 2. 配置格式(强制照抄 Claude Desktop)

复用 Claude Desktop / Cursor / Windsurf 等已有的 MCP 客户端配置格式,你装的 MCP server 在多个 host 间无缝通用:

```jsonc
// claude_desktop_config.json 兼容格式
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
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    },
    // 你自己写的 MCP server
    "vfx-texture-tool": {
      "command": "node",
      "args": ["./mcp-servers/vfx-texture/dist/index.js"],
      "env": { "TEXTURE_CACHE_DIR": "D:/AI/cache" }
    }
  }
}
```

> [!tip] 兼容性投资
> 用户配过的 Claude Desktop / Cursor 的 MCP server 可以一键导入你的桌宠;你写的 MCP server 任何 host 都能用。这是单个最便宜、收益最大的设计决策。

## 3. 推荐技术栈(基于 L2 路线)

| 模块 | 选型 | 理由 |
|---|---|---|
| **MCP SDK** | `@modelcontextprotocol/sdk`(官方 TS) | 协议完整,stdio + SSE + HTTP transport 全支持 |
| **Hub runtime** | Node 20+(独立 sidecar 进程) | TS 生态统一,与 Vercel AI SDK 互通 |
| **进程管理** | Node 自带 `child_process.spawn`(stdio MCP)+ EventSource(SSE)+ fetch(HTTP) | 无需额外 lib |
| **LLM 端 tool 桥接** | Vercel AI SDK 的 `tool()` + `streamText({ tools })` | 自动 function calling schema |
| **配置管理** | 一个 JSON 文件 + chokidar 热重载 | 改配置无需重启桌宠 |
| **前端通信** | Tauri IPC / WebSocket 推 server-status / tool-call 事件 | 实时更新 UI |

## 4. 数据流时序

```
┌─用户──┐     ┌─桌宠前端─┐     ┌─Hub sidecar──┐     ┌─MCP server─┐
│ 说话  │────>│ "Q"      │────>│ 1. 拉 tools  │<───│ tools/list │
└──────┘     └──────────┘     │ 2. 调 LLM    │     └────────────┘
                              │   带 tool def │
                              │ 3. LLM 返     │
                              │   tool_calls  │
                              │ 4. 转发 call  │───>│ tools/call  │
                              │ 5. 收 result  │<───│ result      │
                              │ 6. 回灌 LLM   │     └────────────┘
                              │ 7. final answer│
                              └────┬──────────┘
                                   ▼
                              桌宠说话(流式)
```

关键点:**步骤 1-7 是一个回合的单次 turn**;tool_calls 可能多个并发,要并行调用并各自等待。

## 5. MCP Manager 实现骨架(伪代码)

```typescript
// hub/mcp/manager.ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { SSEClientTransport } from "@modelcontextprotocol/sdk/client/sse.js";

interface ServerConfig {
  command?: string; args?: string[]; env?: Record<string, string>;  // stdio
  url?: string;  // sse / http
}

interface ConnectedServer {
  name: string;
  client: Client;
  tools: Map<string, ToolDef>;  // tool name → schema
}

class McpManager {
  private servers = new Map<string, ConnectedServer>();

  async connect(name: string, cfg: ServerConfig) {
    const transport = cfg.command
      ? new StdioClientTransport({ command: cfg.command, args: cfg.args, env: cfg.env })
      : new SSEClientTransport(new URL(cfg.url!));
    const client = new Client({ name: "desktop-pet", version: "0.1.0" }, { capabilities: {} });
    await client.connect(transport);

    // 拉 tools
    const { tools } = await client.listTools();
    const toolMap = new Map(tools.map(t => [t.name, t]));
    this.servers.set(name, { name, client, tools: toolMap });
    this.emit("server:connected", { name, toolCount: tools.length });
  }

  /** 给 LLM 用:聚合所有 server 的 tools,带命名空间前缀防冲突 */
  getAllTools(): Record<string, AiSdkTool> {
    const out: Record<string, AiSdkTool> = {};
    for (const [serverName, server] of this.servers) {
      for (const [toolName, def] of server.tools) {
        const fqn = `${serverName}__${toolName}`;  // 命名空间
        out[fqn] = {
          description: def.description,
          parameters: def.inputSchema,  // JSON Schema
          execute: async (args) => {
            const res = await server.client.callTool({ name: toolName, arguments: args });
            return res.content;
          },
        };
      }
    }
    return out;
  }

  async disconnect(name: string) { /* ... */ }
  async reload(configPath: string) { /* chokidar 监听 → diff → 增删 */ }
}
```

> [!warning] 命名空间防冲突
> 多个 MCP server 可能各有 `read_file` 这种通用名 tool。**必须**加 server name 前缀(`filesystem__read_file`)防冲突,LLM 也能从前缀理解 tool 归属。

## 6. 与 LLM Router 的接合(Vercel AI SDK)

```typescript
// hub/llm/router.ts
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";
import { mcpManager } from "../mcp/manager";

export async function chat(messages: Message[]) {
  const tools = mcpManager.getAllTools();
  return streamText({
    model: openai("gpt-4o"),  // 或任意 provider
    messages,
    tools,                    // ← MCP tools 在这里桥接进来
    maxSteps: 10,             // 允许多轮 tool call
    onStepFinish: ({ toolCalls, toolResults }) => {
      // 推送给前端用于"桌宠正在调 tool X"动画/气泡
      events.emit("tool:step", { toolCalls, toolResults });
    },
  });
}
```

Vercel AI SDK 的 `tools` 字段直接吃 JSON Schema,`@modelcontextprotocol/sdk` 拉到的 `inputSchema` 就是 JSON Schema——天然兼容,**这是选 Vercel AI SDK 的关键理由**。

## 7. 前端体感(桌宠层)

| 事件 | 桌宠表现 |
|---|---|
| `server:connected` | 托盘冒泡"X 已就绪" |
| `tool:step` 开始 | 桌宠思考表情/思考气泡 |
| `tool:step` 完成 | 表情恢复,气泡显示 tool 名 |
| `tool:error` | 桌宠困惑/抱歉表情 + 错误日志 |
| `text-stream` | 流式打字气泡 |

> [!tip]
> 不需要为每个 tool 设计专属动画——通用"思考中/已完成/出错了"三态就够好。形象层是用户感知"AI 在工作"的反馈通道,不是 tool 详情显示器。

## 8. 安全/边界

1. **首次调用确认**:同一 (server, tool) 第一次被 LLM 调用时,弹气泡问用户"允许吗?[一次 / 总是 / 拒绝]"——存白名单。
2. **能力白名单**:配置里可硬关掉某些 tool(如 `filesystem.write_file` 默认禁用直到允许)。
3. **审计日志**:所有 tool call + args + result 写本地 SQLite,可视化检索。
4. **沙箱**:filesystem / shell 类 server 通过 `args` 限制根目录,**永远不要给整个磁盘根**。
5. **超时**:每个 tool call 有超时(默认 30s),超了取消并回 LLM "tool timeout"。

## 9. 调试/诊断

| 工具 | 用途 |
|---|---|
| **MCP Inspector**(`npx @modelcontextprotocol/inspector`) | 官方调试 GUI,可独立连任何 MCP server 看 tools/list/call,排除 server 端问题 |
| Hub sidecar 的 stderr | 关键日志直接打印 |
| 桌宠开发者面板 | 实时看 tool_calls 和 result(类似 Chrome DevTools network) |

> [!tip] debug 顺序
> tool 调用出问题时,排查顺序:**(1)** Inspector 直连 server 看 tool 是否正常 → **(2)** 看 Hub 是否拉到了 tool(getAllTools 输出)→ **(3)** 看 LLM 是否输出了 tool_call(streamText debug)→ **(4)** 看 args schema 是否对得上。

## 10. 用 AIRI 做底座的三条路线(对比)

| 路线 | 做什么 | 优点 | 缺点 |
|---|---|---|---|
| **A 写 airi-plugin-mcp-client** | 仿 `airi-plugin-claude-code` 目录,引入 `@modelcontextprotocol/sdk`,把 MCP tools 注册到 AIRI 的 LLM 调用链 | 干净、可 PR 上游、复用 AIRI 形象层 | 受 AIRI plugin SDK(WIP)不稳定影响 |
| **B 用 mcp-launcher 作前置 daemon** | 同组织 Go 项目,常驻后台管理 MCP server,AIRI 端写轻量 client | server 进程管理它扛 | 多本地依赖,launcher 自己也很新 |
| **C prompt 层模拟** | 不接真 MCP,把工具能力以 OpenAPI/自定义 schema 塞 prompt | 1 天出 demo | 不是真 MCP,生态不通用,生产用不上 |

如果选 L3 fork AIRI,推荐路线 A。

## 11. 自做 L2 的 MCP 集成顺序

P2 阶段(2-4 周)细分:

1. **P2.1** Hub sidecar 起来,加 MCP Manager 骨架(暂不连 server),`getAllTools()` 返回 `{}`
2. **P2.2** 接 Vercel AI SDK + 一个 LLM provider,确保"无 tool 时纯聊天"流式跑通
3. **P2.3** 挂第一个官方 MCP server(`@modelcontextprotocol/server-filesystem`,只读模式),验证 tools/list 能被 LLM 看到
4. **P2.4** 跑通一次完整 turn:LLM 调 `filesystem__read_file` 读到内容并回话
5. **P2.5** 加第二个 server(`mcp-server-fetch`),验证多 server 命名空间不冲突
6. **P2.6** 配置文件热重载 + 设置 UI 可视化增删 server
7. **P2.7** 调用确认 + 白名单 + 审计

> [!warning] 不要先写 P2.6/P2.7
> 先把 P2.4 跑通——这是验证整条链路的最小回路,没跑通就别去做配置 UI 这种边角工程,会上下文割裂。

## 12. 把"未来 AI 应用"包装成 MCP server 的好处

> [!abstract]
> 你的"代码问答机器人"、"特效贴图工具"、"Niagara 理解器"等(见 [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]] / [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]] / [[Wiki/Syntheses/AIAgents/Niagara-material-ai-comprehension]])如果都包装成 MCP server,**不仅能在桌宠里用**——Cursor / Claude Desktop / VS Code (with Cline) / 任何未来的 host 都能复用。这把"做一次,处处用"从口号变成真实架构。

实现模式:每个 AI 应用 = 一个 MCP server 进程,暴露若干 tools。桌宠是其 host,但不是唯一 host。

## 相关

- [[Wiki/Concepts/AIFoundations/Mcp]] — MCP 协议本身
- [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] — 议题概念
- [[Wiki/Entities/AIAgents/AIRI]] — fork 路线参考
- [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] — 选型矩阵
- [[Readers/AIAgents/桌宠 AI 入口的从零方案]] — 主题读本(整体路线)
- [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]] / [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]] / [[Wiki/Syntheses/AIAgents/Niagara-material-ai-comprehension]] — 未来要包装成 MCP server 的候选 AI 应用
