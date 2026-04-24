---
type: concept
created: 2026-04-24
updated: 2026-04-24
tags: [ai, agent, code-retrieval, grep, llm-tooling]
sources: 1
aliases: [Agentic Grep, Agentic Retrieval, 主动式检索, LLM 临场 grep]
---

# Agentic Grep(主动式 grep 检索)

> **让 LLM 自己临场 grep 代码库**,不预建索引、不走向量、每次现查。三件套工具:`Grep` + `Glob` + `Read`。比喻成**"AI 拿着你的键盘自己写 `rg` 命令"**——不是把文档切碎喂给它,而是让它像人类 senior 工程师一样,看命中、Read 上下文、再 grep。

## 概览

这是 Claudian(本仓 LLM)以及 Claude Code 等现代 AI 编码 agent 的主流代码检索方式,也是 Eureka 同事观察到的"机器人把相关代码连注释发给他"的底层机制。

它和 [[Wiki/Concepts/Methodology/Rag|RAG]] 是两种**完全不同**的检索哲学——对代码域,agentic grep 通常比 RAG 更准、更可靠;对自然语言长文档,RAG 仍有优势。

## 三件套工具

| 工具 | 本质 | 典型用法 |
|---|---|---|
| **Grep** | 底层是 ripgrep,字面/正则匹配文件内容 | 找 `MATERIAL_TWOSIDED` symbol、找 `#ifdef XXX` 所有出现 |
| **Glob** | 按文件名 pattern 找文件 | `**/MaterialExpression*.cpp`、`Engine/Shaders/**/*.ush` |
| **Read** | 按路径读文件(可指定行号) | 读 grep 命中行的上下文几十行 |

**关键点**:没有 embedding、没有向量库、没有预建索引。每次提问都是**临场** grep,看命中,再决定下一步 grep 什么、读哪一个文件——这就是 "agentic retrieval"。

## vs RAG 的对比

| 维度 | RAG | Agentic Grep |
|---|---|---|
| 匹配方式 | 向量空间语义近似 | 字面/正则 |
| 索引 | 需要预建、代码变更要 reindex | 无索引,每次现查 |
| 维护成本 | 持续(chunk 策略、embedding 模型选择、重建) | 几乎为 0 |
| 精确度(代码域) | 可能漏 symbol、可能幻觉片段 | `path:line` 硬证据 |
| 精确度(自然语言) | 强(能理解"猫"≈"喵星人") | 弱(字面不对就零命中) |
| 未知 symbol 容错 | 靠语义近似兜底 | ⚠️ 零命中,需要破解策略 |
| 适用 | 文档问答、语义检索 | 代码导航、字段级查询 |

⚠️ **陷阱**:agentic grep 依赖**关键词命中**。如果 LLM 不知道、也猜不出要 grep 什么词,它就卡住了。这是它相对 RAG 的真实弱点。

## 破解关键词依赖:四路策略

未知 symbol 场景(特别是项目自定义代码)下,按推荐度从高到低:

### 1. 锚点反推(最好用)

美术/策划不懂代码,但一定看过**某个字符串**:UI 勾选项文本、工具栏按钮文字、报错消息、日志 tag、控制台命令。这些字符串在代码里**一定存在**。

```
美术说的              → grep 什么
"材质里那个 XX 开关"  → LOCTEXT.*"XX"  /  DisplayName="XX"  /  ToolTip="XX"
"Log 里说 XX 失败"    → "XX"(字面,找 UE_LOG / ensureMsgf)
"r.XX 这个开关"       → "r.XX"(IConsoleVariable 注册点就在附近)
"按 F5 会怎样"        → EKeys::F5  /  输入绑定名
```

UE 约定稳定,所有玩家/美术可见文本都走 `LOCTEXT` / `NSLOCTEXT` / `FText` / `DisplayName` meta。grep 这些**几乎 100% 命中**。

### 2. 命名约定穷举

UE 和项目代码有稳定 prefix/pattern。同一概念同时试多个变体:

```bash
# 比如"那个 XX 渲染开关"
grep -i "xx"                          # 不区分大小写,广撒网
grep "bXx\|bXX"                       # UE bool 约定
grep "XX_\|_XX\|USE_XX\|ENABLE_XX"    # shader 宏约定
grep "r\.XX\|r\.Render\.XX"           # CVar 约定
grep "FXx\|UXx\|AXx\|EXx"             # UE 类型前缀
```

### 3. 种子爬行

命中一个——哪怕模糊相关——就能 Read 那附近 ±50 行,相邻的 `#include`、function name、注释里提到的概念,都是新 grep 种子。几跳之内通常能摸到目标。

LLM 看起来"知道要搜什么",很多时候前几步是偷偷试错——试失败的不显示,试成功的链接下去看起来很笃定。

### 4. 停下追问

上面都不行时,**不猜**是最重要的策略。典型问法:

- "这个开关在哪个面板看到?截图能发吗?"
- "你能在 .uasset 里看到它附带的文字吗?哪怕错别字也行"
- "这是 Engine 改的还是 Plugin 加的?大概哪个模块?"

一句自然语言比盲 grep 10 次省事。**宁可多问一轮,不能胡答**。

## Wiki 补齐关键词空白

本仓库的 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]天然补足 agentic grep 的弱点:

- Wiki 里已有的概念页 / 源摘要**本身就是词汇桥**——美术的自然语言 → wiki 页里的 alias / tag / 叙事 → 代码 symbol
- 半成熟期:机器人先 grep wiki,命中率陡升,答案还是人类复核过的
- 成熟期:高频问题直接命中 wiki,**几乎不用再 grep 代码**
- 具体做法:建 `Wiki/Concepts/Glossary/术语对照表.md`(美术常说 ↔ 代码 symbol ↔ 模块)

## LLM 训练数据先验的"作弊成分"

⚠️ 老实交代:Claudian 对 **UE 原生 symbol**能 grep 得准,是因为训练语料里见过——脑子里已有候选拼写(`DitheredLODTransition` / `DITHERED_LOD_TRANSITION` / `r.Mobile.AllowDitheredLODTransition`),所以第一次 grep 就命中。

换成项目自定义 symbol,这个先验完全不管用,**必须**走上面四路破解策略 + wiki 补齐。这是 agentic grep 应用到自有项目时的**核心现实**。

## 相关

- [[Wiki/Concepts/Methodology/Rag|RAG]] — 对照面,语义检索范式
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 补齐关键词空白的机制
- [[Wiki/Concepts/AIFoundations/Hallucination|幻觉]] — "未 grep 到证据不许作答"是核心部署纪律
- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — agentic retrieval 是 Agent 的核心能力之一
- [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot|给美术做代码问答机器人]] — 本概念在项目级的具体落地

## 引用来源

- 主题读本(推荐通读):[[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]]
- 原子 source:[[Wiki/Sources/AIFoundations/Code-retrieval-conversation]] (raw: [[Raw/Notes/代码检索与美术问答机器人对话]])

## 开放问题

- Agentic grep + RAG 混合检索:能否在不增加太多维护成本前提下,用 RAG 作为"未知关键词时的近义桥",grep 仍作为主力?
- 自定义代码的 ingest 自动化:有没有一套机制在 CI 阶段就把新增 `UPROPERTY` / 新增 shader 宏自动 append 到"术语对照表"?
- 对代码以外的结构化资产(`.uasset` / `.blueprint` 二进制)agentic grep 无解——是否需要专门的"蓝图 dump 成文本"工具链?
