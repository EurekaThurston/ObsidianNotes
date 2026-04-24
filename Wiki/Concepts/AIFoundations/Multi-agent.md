---
type: concept
created: 2026-04-21
updated: 2026-04-21
tags: [ai, agent, multi-agent, subagent, context-engineering]
sources: 1
aliases: [Multi-agent, Subagent, 多 Agent, 子 Agent, Agent 分工]
---

# Multi-agent / Subagent 架构

> 主 Agent 把任务拆给若干一次性子 Agent,每个子 Agent 在自己干净的上下文里干脏活,只返回压缩过的结论。**不是为了并行加速,是为了上下文隔离**。

## 第一性原理:上下文隔离,不是分工并行

第一次听"多 agent"容易想到多线程并行。但对 [[Wiki/Concepts/AIFoundations/Llm|LLM]] 而言,**最珍贵的资源不是算力,是上下文窗口**——而上下文有三个病:

1. **[[Wiki/Concepts/AIFoundations/Context-window|Context Rot]]**:窗口越满,对早期 token 注意力越弱、越矛盾、越忘指令。200K 塞到 150K 常比只填 30K 表现更差。
2. **Prompt Cache 失效成本**:Anthropic prompt cache TTL 5 分钟。上下文结构剧变要重建缓存,直接花钱。
3. **指令漂移**:主任务 system prompt 被子任务的搜索/浏览噪音稀释,模型"忘了自己是谁"。

> 多 agent 的真正动机是:**把脏活、大体量探索、临时性搜索结果,关进一次性上下文里,完事只把压缩结论吐回主 agent**。

"有效上下文"从单窗口 200K 变成 **N × 200K**,代价是 agent 间通信只能用压缩文字。类比操作系统"进程 + IPC":与其共享无限大内存,不如每进程独立地址空间 + 明确消息接口。

## 包工头隐喻

**主 agent = 项目经理 / 包工头**:不亲自读文件,派"一次性实习生"去读。实习生累死自己(读 40 文件、跑 20 次 grep、生成 80K token 中间思考),交上来 300 字报告后被"解雇"(释放上下文)。主 agent 上下文只长了 300 字,不是 80K。

## 子 Agent = 一次全新的 Claude 对话

每个子 agent 字面意义上就是:

- **不知道用户是谁**,不知道之前聊了啥
- **看不到主 agent 的 system prompt**(Claudian 的 Obsidian 规则子 agent 也看不到)
- 只能看到**派活那一刻主 agent 手写给它的 prompt**
- 干完返回**一条消息**,然后销毁

⚠️ **写派活 prompt 要像对刚入职同事**:背景 / 目标 / 已知信息 / 期望格式全写清楚,不能说"你懂的"——它真的啥都不懂。

⚠️ **强制要求压缩汇报**(如"200 字以内"):不是省 API 钱,是**防子 agent 把原始信息倾倒回主 agent 上下文,那就白隔离了**。

## 除了上下文隔离,还有四个收益

| 收益 | 说明 |
|---|---|
| **角色专用 system prompt + 限定工具集** | 规划 agent 写"只输出 plan 不写代码"且不给 Write;执行 agent 写"只按 plan 改不重新规划"且不给 WebSearch |
| **不同模型分工** | 规划/审查用贵大模型(Opus),grep/read 苦力用便宜快模型(Haiku)。单 agent 做不到——切模型对话就断 |
| **独立审视视角** | 审查 agent 看不到主 agent 思考过程,不被其推理锚定。同上下文里的"自我批评"易被之前推理影响 |
| **失败隔离** | 子 agent 卡住/跑飞/幻觉,主 agent 直接丢整个 trace,只记"这条路走不通",不被污染 |

## 链路拓扑:扁平扇出 > 深度嵌套

每层 agent 间通信都是"压缩-解压",**通信损耗累积**。5 级嵌套几乎不存在,现实常见:

- **扁平扇出**:1 主 agent 并行派 3-5 子 agent,独立上下文,收齐后主 agent synthesize
- **两级为主**:planner → N executor → reviewer,很少再嵌
- **循环而非深度**:主与同一子 agent 反复交互,而不是"子派孙"

> Anthropic 自己的 multi-agent research 系统观察:**agent 数量与 token 使用线性相关,但质量提升非线性**,过某点就是烧钱。

## 并行 vs 串行

- **独立任务**(如"查 A 主题" + "查 B 主题"):主 agent 在**同一条消息里**同时发起多个 Agent 调用,并行跑
- **依赖任务**(B 需要 A 的结果):只能串行

## 派不派的决策

| 情况 | 决策 |
|---|---|
| 要读很多文件 / 大量搜索 | **派**(脏活隔离) |
| 需要独立审视视角 | **派**(避免自我锚定) |
| 简单一两步能搞定 | **自己干**(派出去反而更慢更失真) |

## Claudian 的实际例子

> 用户:"帮我在 vault 里找所有跟'注意力机制'相关的笔记,整理成索引"

Claudian 决策:读很多文件、中间噪音大 → 派 Explore agent,prompt 写得很具体:"搜索提到 attention/注意力/self-attention 的 .md,按主题归类,返回文件路径+一句话摘要清单,**不要返回原文摘录**"。它跑完给主 agent 一张表,主 agent 只增加了这张表的上下文,没被 50 个文件原文淹没。

## 相关

- [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] — 单 agent 循环;本概念是其在"多 agent"方向上的延伸
- [[Wiki/Concepts/AIFoundations/Context-window|上下文窗口 & Context Rot]] — 多 agent 存在的根本动机
- [[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]] — 多 agent 是 Harness "上下文管理"支柱的最强工具
- [[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] — 同样以对抗 Context Rot 为动机(渐进式披露)
- [[Wiki/Entities/Claudian|Claudian]] — 拥有 `Agent` 工具的自证案例

## 引用来源

- 主题读本(推荐通读):[[Readers/AIFoundations/为什么上下文有限反而必须切多 Agent]]
- 原子 source:[[Wiki/Sources/AIFoundations/Multi-agent-conversation]] (raw: [[Raw/Notes/Multi-agent 对话]])

## 开放问题

- **压缩返回的信息损失**:子 agent "200 字汇报"不可避免丢信息,什么场景下主 agent 需要能按需追问子 agent 的原始上下文?(当前模型无此能力,一次返回即销毁)
- **主 agent 自己的上下文预算**:多少并发子 agent 收齐后就会让主 agent 自己撑爆?Anthropic 观察到的"agent 数非线性质量"拐点具体在哪?
- **跨子 agent 共享上下文**:两个子 agent 都需要相同的 10 份文件,各自独立读一遍是浪费。是否需要"共享只读上下文"机制?(Anthropic 当前没有原生支持)
