---
type: synthesis
created: 2026-04-26
updated: 2026-04-26
tags: [aiagents, desktop-pet, mcp, agent-hub, tauri, live2d, reader]
sources: 4
aliases: [桌宠从零方案, Desktop Pet From Scratch]
---

# 桌宠 AI 入口的从零方案

> 本页是 **"做一个桌宠作为未来所有 AI 应用的入口"** 的主题读本——详细、精确、满满当当,一次读完即完整掌握这个议题的全部决策与施工蓝图,不需要跳转。
>
> 字段级查询见末尾 [[#深入阅读]]。

---

## 0. 这个议题要回答的问题

桌宠这个形态(Live2D 角色 + 常驻形象 + 系统托盘)在 2026 已经不是新东西。但当用户的目标从"陪伴"升级为**"统一入口承载所有 AI 应用"**时,几乎所有现成项目都会暴露架构错配。

本读本要回答四件事:

1. 现成项目(以 AIRI 为代表)为什么不够用?**它们在哪一层不契合"AI 入口"目标**?
2. "完全从零做"是否合理?如果不合理,**哪些必须自做、哪些必须装配**?
3. 推荐架构长什么样?**形象层 / Hub / MCP host** 三层各做什么?
4. 一个人业余怎么落地?**P0-P5 的分阶段路线 + 每阶段的可验证里程碑**是什么?

> [!abstract] 一句话主线
> 桌宠是壳,Hub 是脑,**MCP host 才是真正的"入口"语义**。先把这三层的边界画清楚,再决定每层用现成的还是自写。

叙事链:**目标拆解 → AIRI 评估 → 三档路线 → L2 装配 → 推荐架构 → 分阶段落地 → 避坑 → 决策回收**。

---

## 1. 目标拆解:什么叫"AI 应用入口"

"我想做个桌宠当 AI 入口"这句话可以是三种完全不同的东西。先看你想要的是哪种,后面的所有架构决策都不一样:

| 入口语义 | 用户体验 | 架构含义 |
|---|---|---|
| **A. 启动器** | 点桌宠 → 弹菜单 → 启动其他 app | 桌宠 = 美化版任务栏,无需 AI 能力 |
| **B. 单体 AI 助手** | 跟桌宠聊天,它能调 LLM 帮你做事 | 一个会说话的 ChatGPT 客户端 + 头像 |
| **C. AI 应用总线** ⭐ | 一个统一对话面 + 形象,**所有 AI 工具(代码问答 / 贴图 / Niagara 理解 / 笔记搜索 / ...)挂在它下面**,LLM 自动调度 | 桌宠 = MCP host,所有 AI 应用 = MCP server |

你已经在前面对话里明确了想要 C。这个目标的关键不在"桌宠"——而在**"统一 + 可热插拔 + 自动调度"**。Live2D 形象只是给这个总线一张人脸,让"召唤一只 AI 伴侣"比"打开一个工具栏"更有日常感。

所以本读本之后所有的判断,都按 C 来收敛。

> [!warning] 不要混淆"陪伴感"和"入口语义"
> 桌宠很容易做着做着滑向 B(变成单体 chatbot 加形象),因为做语音、做表情、做动作的反馈循环最爽。但**入口语义是 MCP**,这是工程主线。形象/语音是 V2 之后的事,先别碰。

---

## 2. AIRI 深度评估(为什么"还不够用")

[AIRI(moeru-ai/airi)](https://github.com/moeru-ai/airi) 是当前开源里最接近你目标的项目,值得先认真评估。

### 2.1 它强在哪

- **Provider 抽象**:用 `xsAI` 库统一封装 26+ LLM 厂商(OpenAI / Claude / Gemini / DeepSeek / 通义 / 豆包 / Kimi / Ollama / vLLM / ...),**换模型就改个 baseURL**。这件事 Open-LLM-VTuber / Fay 都没做这么彻底。
- **形象层完整**:Live2D + VRM 双轨,自动眨眼/视线/idle 微动作齐全。
- **跨平台**:`apps/stage-web`(浏览器)/ `apps/stage-tamagotchi`(Electron 桌面)/ `apps/stage-pocket`(PWA 移动)三端一套架构。
- **Agent 真跑通**:Minecraft agent 是完整的、Factorio agent 有 PoC,证明工具调用和长链 prompting 在工程上 work。
- **本地 + 云端都吃**:WebGPU 浏览器内推理 + Ollama / vLLM / SGLang / Transformers.js / Candle。

### 2.2 它不够的地方(对你的目标)

> [!warning] 三个错配
> 1. **MCP 不是一级公民** — 主仓 `plugins/` 五个插件(bilibili / claude-code / chess / homeassistant / web-extension)**没有 mcp-client**。姊妹仓 `moeru-ai/mcp-launcher`(Go,99%)定位是"MCP server 的 Ollama"——管 server 进程的,不是给 host 用的 client。AIRI 端要接 MCP 必须自写。
> 2. **VTuber/游戏躯壳** — 项目目标是"复刻 Neuro-sama",VTuber 直播 + 玩游戏是核心戏路。这些代码对"AI 应用入口"是负重。
> 3. **plugin 系统标 WIP** — 现有 plugins 走目录约定,没有稳定 SDK,API 还在变。深度 fork 会扛上游合并地狱。

### 2.3 评分回收

| 维度 | 评 |
|---|---|
| LLM 灵活度 | ⭐⭐⭐⭐⭐ |
| 形象层 | ⭐⭐⭐⭐⭐ |
| MCP / 工具调用通用性 | ⭐⭐ |
| 桌面常驻形态 | ⭐⭐⭐ |
| 架构契合度(对 AI 入口) | ⭐⭐ |
| 作为 fork 起点 | ⭐⭐⭐ |
| **作为装配库参考** | ⭐⭐⭐⭐⭐ |

最后一行是关键洞察:**AIRI 最有用的不是它的躯壳,而是它的依赖装配清单**。它选的 Live2D / VRM / xsAI / Drizzle / DuckDB 都是好东西,你应该把这些库直接拿到自做项目里,但不要把它的应用层架构带过来。

完整对比矩阵见 [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]]。

---

## 3. 三档路线:把"自做"分级

"自己做"在工程上是个含糊词,差 5 倍工作量:

| 档 | 做什么 | 工作量(一人业余) | 评 |
|---|---|---|---|
| **L1 完全从零** | 自写 Live2D 渲染 / LLM HTTP client / MCP 协议帧 | 6-12 月 | ❌ 没有任何理由,这些都有官方/事实标准库 |
| **L2 从零做壳 + 装配成熟库** ⭐ | 自写窗口/交互/Hub 架构,渲染/LLM/MCP 全用现成库 | 2-3 月 v0.1,半年成熟 | 推荐 |
| **L3 fork AIRI 改造** | 接受其架构,定向改 | 1-2 月 | 起步快,扛 VTuber 包袱 + 上游合并 |

### 3.1 为什么不选 L3

L3 看似省时间,但有两个隐性成本:

1. **每次 AIRI 发版你都要 merge**——它每 2-3 周一版,API 不稳定,你的定制代码会反复冲突。
2. **你最终是在 AIRI 框子里建偏离它愿景的东西**——AIRI 想做 Neuro-sama,你想做 MCP host,目标分叉一年后你的 fork 跟主仓没关系了,等于自己维护了一个 fork 的 AIRI。

L3 唯一合理的场景:**你打心眼里就想要 Neuro-sama 那种体验**,只是想加点工具调用。

### 3.2 为什么 L2 才合理

L2 的本质是:**只原创"桌宠形态的窗口/交互"和"Hub 的架构"两块,其他全部装配**。下面这张装配清单是手册级的:

| 模块 | 选型 | 你省了多少时间 |
|---|---|---|
| **桌面壳** | **Tauri 2.0**(Rust + WebView,体积 ~10MB)| ~2 周(对比 Electron 的体积/启动速度调优) |
| **形象渲染** | **pixi-live2d-display** + Cubism SDK(2D)<br>或 `@pixiv/three-vrm`(3D) | ~1 月(光是 Live2D motion 系统就能做爆) |
| **LLM 抽象** | **Vercel AI SDK**(`ai` + `@ai-sdk/openai` `@ai-sdk/anthropic` 等)| ~3 周(provider 抽象 + 流式 + tool calling 一次到位) |
| **MCP** | **`@modelcontextprotocol/sdk`** 官方 TS SDK | ~1 月(协议你不会想自己实现的) |
| **本地 DB / 记忆** | **SQLite**(better-sqlite3 / Tauri SQL plugin)+ `sqlite-vec` 或 `lancedb` | ~2 周 |
| **TTS / ASR**(可选) | edge-tts(免费)/ Azure / 火山;whisper.cpp / WebSpeech | ~1 周 |
| **UI 框架** | Vue 或 React + shadcn / Element Plus | — |
| **配置文件** | **复用 Claude Desktop 的 `claude_desktop_config.json` 格式** | — |

> [!tip] Vercel AI SDK 是关键选择
> 它的 `tools` 字段直接吃 JSON Schema——而 `@modelcontextprotocol/sdk` 拉到的 `inputSchema` **就是 JSON Schema**。这两者天然兼容,中间不需要写转换层。如果换 LangChain.js 或自写 LLM client,这条桥就要自己搭,工作量翻倍。

> [!tip] 配置照抄 Claude Desktop
> 你装的 MCP server 在 Cursor / Claude Desktop / 你的桌宠之间一份配置通用——这是你能做的最便宜、收益最大的兼容性投资。**绝对不要自定义配置格式**。

---

## 4. 推荐架构(三层独立)

```
┌──────────────────────────────────────────┐
│  Tauri 前端 (Vue/React + pixi-live2d)     │
│  ├─ 形象层:透明窗口 + 点透 + 托盘 + 快捷键 │
│  ├─ 对话层:聊天气泡 / 设置面板             │
│  └─ 通过 IPC 调后端                        │
├──────────────────────────────────────────┤
│  Tauri Rust 后端 (壳的核心)               │
│  ├─ 系统级:全局快捷键 / 剪贴板 / 屏幕监听  │
│  └─ 转发到 Node sidecar                   │
├──────────────────────────────────────────┤
│  Node sidecar (Hub 大脑) — 独立进程        │
│  ├─ LLM Router (Vercel AI SDK)           │
│  ├─ MCP Manager (官方 SDK) ←── 核心       │
│  │   ├─ 启动/管理 N 个 MCP server 子进程  │
│  │   └─ tools 聚合 → 喂给 LLM            │
│  ├─ Memory (SQLite + sqlite-vec)         │
│  ├─ Session/Context 管理                  │
│  └─ Event Bus (前端订阅状态变化)          │
└──────────────────────────────────────────┘
        ↓ stdio / SSE / HTTP
┌──────────────────────────────────────────┐
│  N 个 MCP server (子进程或远程)            │
│  filesystem / git / 你写的 vfx / 代码问答 │
└──────────────────────────────────────────┘
```

三个关键决策反复强调:

1. **MCP-first**:Hub 设计中心是 MCP host 数据流,不是 chat data flow。
2. **Hub 单独抽 sidecar**:桌宠只是它的一个前端,以后做 CLI / VS Code 插件 / Web 都共享同一个 Hub。
3. **形象层与 Hub 严格解耦**:今天 Live2D 明天 VRM 后天像素 sprite,Hub 一行不改。

> [!warning] sidecar 的争议
> 把 Hub 拆 sidecar 是过度工程吗?判断标准:**你 1 年内会不会真做 CLI / VS Code 入口**。会做 → 拆;不会做 → Hub 直接进 Tauri Rust 后端或前端 Web Worker。这条不是教条,按你的真实产能决定。

---

## 5. MCP host 是真正的工程重心

整个项目的复杂度不在 Live2D,不在 LLM 调用,而在 MCP host。这一节是手册的浓缩;详细施工见 [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]]。

### 5.1 五个职责

| # | 职责 | 谁负责 |
|---|---|---|
| 1 | server 进程管理(spawn/health/restart)| MCP Manager |
| 2 | tools 聚合(`tools/list` 拉合并 + 命名空间)| MCP Manager |
| 3 | tool calling 桥接(LLM `tool_calls` ↔ MCP `tools/call`)| LLM Router ↔ MCP Manager |
| 4 | 配置/UI(增删 server / 看 tools / 看日志)| 设置面板 |
| 5 | 安全/边界(确认弹窗 / 白名单 / 审计 / 沙箱)| MCP Manager + 前端 |

### 5.2 数据流

```
用户说话 → 桌宠前端
       → Hub: 1.拉 tools  2.调 LLM 带 tool def
                3.LLM 返 tool_calls
                4.MCP Manager 转发到对应 server
                5.收 tool result
                6.回灌 LLM
                7.LLM 出 final answer
       → 桌宠流式说话
```

关键点:tool_calls 可能多个并发,**要并行调用**;每个 turn 是一个完整回路。

### 5.3 命名空间防冲突

多个 MCP server 可能各有 `read_file` 这种通用名 tool。**必须**给 LLM 看到的 tool name 加 server 名前缀(如 `filesystem__read_file`)防冲突。这件事写错的代价是 tool call 被路由到错的 server,debug 起来很恼火,所以从第一天就加。

```typescript
// 在 MCP Manager 的 getAllTools 里
const fqn = `${serverName}__${toolName}`;
out[fqn] = { description, parameters: def.inputSchema, execute: ... };
```

### 5.4 Vercel AI SDK 的桥接

```typescript
import { streamText } from "ai";
import { openai } from "@ai-sdk/openai";

const tools = mcpManager.getAllTools();  // ← MCP tools 在这里
return streamText({
  model: openai("gpt-4o"),
  messages,
  tools,
  maxSteps: 10,  // 允许多轮 tool call
  onStepFinish: ({ toolCalls, toolResults }) => {
    events.emit("tool:step", { toolCalls, toolResults });  // 前端动画
  },
});
```

`maxSteps: 10` 是允许 LLM 在一个 turn 里反复调 tool 链(读文件 → 看到引用 → 再读另一个文件 → 总结)的关键。

### 5.5 安全边界

> [!warning] filesystem MCP 的根目录
> 用 `@modelcontextprotocol/server-filesystem` 时,`args` 必须显式指定根目录(如 `D:/Notes`),**永远不要给整个磁盘根**。一旦 LLM 被 prompt injection 攻击,它能 `read_file` 你整个家目录里所有 `.env`。

最低安全配置:

1. **首次调用确认**:同一 (server, tool) 第一次被 LLM 调用时弹气泡问"允许?[一次/总是/拒绝]",存白名单。
2. **能力白名单**:配置硬关 destructive tools(`write_file` / `execute_command`)默认禁用直到允许。
3. **审计日志**:所有 tool call + args + result 写本地 SQLite 可检索。
4. **超时**:每个 tool call 30s,超时取消。

### 5.6 调试三件套

| 工具 | 用途 |
|---|---|
| **MCP Inspector**(`npx @modelcontextprotocol/inspector`)| 官方调试 GUI,独立连任何 MCP server 看 tools 和 call,排除 server 端问题 |
| Hub sidecar stderr | 关键日志 |
| 桌宠开发者面板 | 实时看 tool_calls / result(类似 Chrome DevTools network) |

排错顺序:**Inspector 直连 server → Hub 是否拉到 tool → LLM 是否输出 tool_call → args schema 是否对得上**。

---

## 6. 分阶段落地路线

> [!info] 节奏
> 一人业余,每阶段 2-4 周。下面写的是"完成时你能跑给别人看什么",不是"代码量"。**关键节点判断:做完 P2 你能验证整条路对不对**。

| Phase | 目标 | 完成时你能演示 |
|---|---|---|
| **P0 壳** | Tauri 透明窗口 + 系统托盘 + 全局快捷键 + Live2D 角色站桌面会眨眼 | 一只什么都不会的桌宠,能用快捷键召唤/隐藏 |
| **P1 对话** | Vercel AI SDK + 1-2 个 LLM provider,流式聊天 | 会聊天的桌宠 ≈ ChatGPT 桌面版 + 脸 |
| **P2 MCP 接入** ⭐ | MCP Manager + tool calling 桥接,挂 1-2 个官方 MCP server(filesystem / fetch)跑通 | **"入口"愿景第一次成立**——桌宠能读你的笔记/抓网页 |
| **P3 自定义 MCP** | 把第一个你想要的 AI 应用(代码问答 / 笔记搜索 / 贴图工具)写成 MCP server 挂上去 | 真正的"我自己的 AI 入口" |
| **P4 记忆 + 设置 UI** | SQLite + 向量记忆;可视化管理 MCP server / 看审计日志 | 长期可用版 v1.0 |
| **P5+ 锦上添花** | TTS/ASR / 多角色 / 动作触发 / 皮肤系统 | 看心情 |

> [!warning] 不要颠倒顺序
> 1. **不要先做语音**——TTS/ASR 是时间黑洞且对核心价值贡献低,留 P5。
> 2. **不要先做花哨动画**——P0 一只会眨眼的呆角色就够了,动作系统边际收益最低。
> 3. **不要先做配置 UI**——P2 跑通完整 turn 之前,改 JSON 配置文件就够了。
> 4. **不要"以后再加 MCP"**——MCP 决定 Hub 数据流形状,P0 就要按 MCP-host 设计代码路径(可以 0 个 server,但路径要在)。

### 6.1 P0 的最小验收

- 双击桌宠图标启动,系统托盘有图标
- 主窗口透明、无边框、置顶,Live2D 角色显示在屏幕角落
- 角色每 5s 眨一次眼(Live2D 自动 motion)
- 全局快捷键(如 Ctrl+Space)切换角色显隐
- 右键托盘菜单:退出 / 设置(空)/ 关于

代码骨架级别——不超过 500 行 Rust + TS。

### 6.2 P1 的最小验收

- 点击角色弹聊天气泡(或快捷键召唤)
- 输入文字,流式打印 LLM 回复
- 设置面板填 API key,选 provider
- 多轮对话(简单 in-memory history,P4 再持久化)

### 6.3 P2 的最小验收(关键里程碑)

- 配置文件挂 `@modelcontextprotocol/server-filesystem`(根目录指 `D:/Notes`)
- 跟桌宠说"读一下我今天的笔记",LLM 调 `filesystem__list_directory` + `filesystem__read_file`,流式总结回话
- 桌宠思考表情(tool 执行中)+ 完成表情
- Hub 日志能看到完整 tool_call / result

> [!abstract] P2 跑通后你拿到了什么
> 你已经造出了**一个 MCP host 桌宠**。剩下的 P3-P5 都是把它做厚,核心架构问题已经解。如果到 P2 就别扭(LLM 调 tool 不准 / 流式卡 / 桥接复杂),回头看是不是某个层用错了库或选错了抽象——**别硬着头皮往下建**。

### 6.4 P3 的关键决策:第一个自定义 MCP server 选什么

候选(都已在你的 wiki 里设计过):

| 候选 | 复杂度 | 立竿见影感 | 推荐度 |
|---|---|---|---|
| 笔记搜索(MCP server 包 wiki + RAG)| 低 | 高(每天都用)| ⭐⭐⭐⭐⭐ |
| 代码问答(见 [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]]) | 中 | 中(场景特定)| ⭐⭐⭐⭐ |
| 特效贴图(见 [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]]) | 高 | 高(但 PoC 工作量大)| ⭐⭐⭐ |
| Niagara/Material 理解(见 [[Wiki/Syntheses/AIAgents/Niagara-material-ai-comprehension]]) | 高 | 中(场景窄)| ⭐⭐ |

**推荐"笔记搜索"作 P3**:工作量小、价值密度高、能验证你的 wiki 真的成了 LLM 的扩展记忆。后面的代码问答 / 贴图都可以复用同一套 MCP server 框架。

---

## 7. 核心避坑清单

> [!warning] 已在文中各处出现,这里集中归档
> 1. **MCP-first**——P0 就要按 MCP host 设计代码路径,不能"以后加"。
> 2. **配置兼容 Claude Desktop**——绝不自定义配置格式。
> 3. **命名空间防冲突**——多 server 同名 tool 是常态,前缀必加。
> 4. **filesystem MCP 必须限根**——永不给整盘根。
> 5. **首次 tool call 弹确认**——白名单存本地。
> 6. **不要先做语音/动画/UI**——P0-P2 是核心,P3-P4 是骨肉,P5+ 是装饰。
> 7. **Tauri 在 macOS 透明窗口有坑**——主用 Win 先别管 mac。
> 8. **Vercel AI SDK 国内 provider 质量参差**——必要时直接走 OpenAI 兼容 endpoint(豆包/通义/DeepSeek 都支持)。
> 9. **maxSteps 不要设太低**——5-10 比较合理,1-2 会让 LLM 中途断 tool 链。
> 10. **审计日志早做**——P2 就把 tool call 写 SQLite,后期 debug 救命。

---

## 8. 全景回看

```
┌────────────────────────────────────────────────────────┐
│  目标:C 类入口语义(AI 应用总线)                        │
│       ↓                                                 │
│  评估:AIRI 在 LLM/形象上五星,在 MCP/契合度上两星        │
│       → 不 fork(L3 不合)                               │
│       ↓                                                 │
│  路线:L2 自做壳 + 装配库                                │
│       ↓                                                 │
│  装配:Tauri / pixi-live2d / Vercel AI SDK / MCP SDK    │
│       / SQLite-vec / Claude Desktop 配置格式             │
│       ↓                                                 │
│  架构:三层独立(形象 / Hub / MCP host)+ Hub sidecar 化  │
│       ↓                                                 │
│  施工:P0 壳 → P1 对话 → P2 MCP host(关键里程碑) →      │
│        P3 自定义 server(笔记搜索) → P4 记忆+UI →        │
│        P5+ 装饰                                          │
│       ↓                                                 │
│  落地:你写的"代码问答 / 贴图 / Niagara 理解"全部包成     │
│        MCP server,桌宠 + Cursor + Claude Desktop 通用    │
└────────────────────────────────────────────────────────┘
```

---

## 9. 关键洞察

1. **MCP 是这个议题的"重力"**——所有架构决策都向它收敛,绕开它做出来的都是 chatbot 加形象。
2. **形象层是壳,Hub 是脑,但 MCP host 才是"入口"语义本身**——名字是桌宠,本质是你的私人 MCP host。
3. **AIRI 最值得借鉴的不是它的应用层,而是它的依赖装配清单**——那些库选得对,但怎么拼是你自己的事。
4. **Hub sidecar 化的真实价值取决于你 1 年内是否真做别的前端**——是教条还是必需,看你的产能,不看架构图。
5. **配置格式照抄 Claude Desktop 是杠杆最大的兼容性投资**——一份配置三个 host 通用,下游所有 MCP server 自动跨平台。
6. **Vercel AI SDK + MCP SDK 的 JSON Schema 天然兼容**是 L2 选型的核心理由——这条桥不用自写,选别的栈这条桥就要自搭。
7. **"以后再加 MCP" 是最大的反模式**——它决定整个 Hub 数据流的形状,后期改造等于推倒重来。
8. **P2 是验证整条路的最小回路**——做完它你就知道这个项目能不能走下去;之前做的都是 P2 的脚手架,之后做的都是把 P2 做厚。
9. **第一个自定义 MCP server 选"笔记搜索"**——工作量低、价值密度最高、能让你的 wiki 同时变成 LLM 的扩展记忆,一石二鸟。
10. **未来所有 AI 应用都包成 MCP server**——这是把"做一次,处处用"从口号变成架构,你的工作天然是向 Cursor / Claude Desktop / VS Code Cline 等所有 host 输出。

---

## 自检问题(读完回答)

下面这些题需要把本读本几个核心节(§1 入口语义 / §4 三层架构 / §5 MCP host / §6 阶段路线 / §7 避坑)串起来才能答得有根据。**不在文中给答案,留给你自测**。

1. **入口语义错配**:如果你跳过 §1 直接做"会聊天的桌宠"(B 类),做完后再加 MCP——具体哪几处代码会要返工?为什么"返工成本 = 推倒重来"而不是"加几个文件"?
2. **L2 vs L3 取舍**:什么样的具体场景下,L3 fork AIRI 比 L2 自做更优?反过来,什么场景下 L2 才是唯一合理选择?给两个判断标准。
3. **Hub sidecar 化的第一性原理**:为什么读本说"是不是过度工程取决于你 1 年内会不会真做 CLI/VS Code 前端"?如果你确定只做桌宠,sidecar 拆分还有什么剩余价值,还是真的应该合进 Tauri 后端?
4. **MCP-first 的连锁后果**:把"P0 就按 MCP host 设计代码路径"这条具体化——P0 阶段你 0 个 MCP server,代码里要存在什么对象/接口/数据流空架,才算"路径已存在"?省略它在 P2 阶段会触发什么具体的重构?
5. **Vercel AI SDK 选择的可替代性**:如果换成 LangChain.js,§5.4 那段桥接代码会变成什么样?新增的工作量主要在哪里?反过来如果换成自写 fetch + OpenAI 协议,工作量在哪儿?
6. **命名空间冲突的具体场景**:假设你同时挂了 `filesystem` 和 `git` 两个 server,LLM 看到 `read_file` 这个 tool 名(没加前缀)调它——具体会发生什么 bug?debug 时你怎么判断是路由到了错的 server?
7. **P2 跑不通的诊断顺序**:如果 P2 阶段 LLM 死活不调你挂的 filesystem tool,§5.6 三件套排错顺序为什么是"Inspector → Hub tools → LLM 输出 → schema",而不是反过来?每一步排除什么类别的 bug?
8. **P3 选笔记搜索 vs 代码问答的取舍**:为什么读本推荐 P3 选笔记搜索而不是代码问答机器人,即使你的最终目标里两个都要做?把 §6.4 表格里"复杂度"和"立竿见影感"两列的具体含义展开。
9. **第一性原理对话**:用 5 句话向不熟悉本议题的人解释"为什么桌宠的本质是 MCP host,而不是 Live2D 形象 + ChatGPT 客户端"。

---

## 这个议题留下的问题

- **桌宠形态是否真的比纯 menubar 工具或 CLI 体验更好**——P2 跑通后用 1 周看实际使用频次,如果你发现自己更频繁用 Cursor 而不是桌宠,可能要重新评估投入。→ P2 之后的 review 节点。
- **Hub sidecar 通信开销**——Tauri IPC + Node sidecar 的延迟在 LLM 流式场景下是否够低,要实测;P1 阶段就能验证。
- **MCP server 的本地资源占用**——挂 5-10 个 server 时,各自常驻一个 Node/Python 进程,内存可能爆。何时需要 mcp-launcher 这种集中进程管理器?→ P3-P4 之间观察。
- **角色 IP 与陪伴感**——Live2D 默认模型(Hiyori / Haru)够用还是要自己画/买?这件事影响"日用感"但不影响架构,放 P5。

## 下一步预告

- **真正动手**:从 P0 开始;另开一个仓库,跟随本读本 P0-P5。
- **可以先做的预备**:给本仓库的 wiki 写一个 MCP server(暴露 search_notes / read_note / list_recent 三个 tool),这样 P3 阶段能立刻挂上桌宠 → 自我闭环最快。
- **AIRI 持续观察**:`mcp-launcher` 的进展、`@proj-airi/server-runtime` 是否暴露稳定接口——如果 1 年后 AIRI 把 MCP 做成了一级公民,届时可重评是否合并/迁移。

---

## 深入阅读

### 本议题的原子页

- 概念:[[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]]
- 实体:[[Wiki/Entities/AIAgents/AIRI]]
- 综合(选型):[[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]]
- 综合(MCP 实施):[[Wiki/Syntheses/AIAgents/Mcp-host-implementation]]

### 前置议题

- [[Wiki/Concepts/AIFoundations/Mcp]] — MCP 协议本身,理解前置
- [[Wiki/Concepts/AIFoundations/Ai-agent]] — Agent 概念
- [[Wiki/Concepts/AIFoundations/Agent-skills]] — Skills 与 MCP server 的能力打包对比
- [[Wiki/Concepts/AIFoundations/Context-window]] — 为什么 MCP host 要做记忆和会话管理
- [[Readers/AIFoundations/AI 应用生态全景 2026]] — 整体技术栈背景

### 未来挂上桌宠的 AI 应用(都已设计成 MCP server 候选)

- [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]] / [[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]]
- [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]] / [[Readers/AIAgents/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]]
- [[Wiki/Syntheses/AIAgents/Niagara-material-ai-comprehension]] / [[Readers/AIAgents/给 AI 看懂 Niagara 和材质 - 正反两条管道]]

### 跨主题联系

- [[Wiki/Concepts/Methodology/Llm-wiki-方法论]] — 本仓 wiki 本身可作为桌宠的第一个自定义 MCP server(笔记搜索)
- [[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] — 生成式智能体研究脉络;桌宠是工具型 agent 而非社会模拟,但人格化形象的命题相通

---

*本读本由 [[Wiki/Entities/Claudian|Claudian]] 基于桌宠 AI 入口议题的 4 个原子页(1 概念 + 1 实体 + 2 综合)综合生成,2026-04-26。*
