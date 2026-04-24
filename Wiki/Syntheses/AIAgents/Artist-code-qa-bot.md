---
type: synthesis
created: 2026-04-24
updated: 2026-04-24
tags: [ai, agent, code-retrieval, artist-tooling, ue4, landing-design, aki]
sources: 1
aliases: [美术代码问答机器人, Code QA Bot for Artists, 美术 AI 问答]
---

# 给美术做代码问答机器人 — 项目级 AI 应用落地设计

> 一份针对 UE 项目组美术/TA 场景的具体 AI 落地方案:让不懂代码的同学能向机器人提问"这个材质宏开关是什么意思"、"Niagara 这个参数底层怎么工作"、"我这个开关没效果为什么",机器人基于项目源码给出**带证据的、用美术能听懂的语言**解释。
>
> 本方案是 [[Wiki/Concepts/AIFoundations/Agentic-grep|Agentic Grep]] + [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] 在项目代码场景的具体应用。

## 问题定义

**现状**:美术、TA、策划在日常使用引擎功能时,很多概念是经验主义:

- 材质参数、材质宏开关、材质静态开关、Niagara 模块参数 ...
- 开了和没开区别是什么?性能代价?什么时候该开?
- 项目自定义的功能/参数/开关,源码里才有权威答案

**瓶颈**:让美术自己读 C++/shader 源码不现实;问程序员又打断别人工作流且不 scale。

**目标**:一个 AI 机器人,美术用自然语言提问,机器人**基于项目源码**返回:
1. 用美术听得懂的语言解释
2. 附上原始代码段(带 `path:line`,可点开验证)
3. 必要时指出性能代价、使用建议、常见坑

## 核心机制

### 检索层:Agentic Grep 三件套

不走 RAG,也不预建向量索引。机器人临场用 `Grep` / `Glob` / `Read` 跨三仓搜索:

- `stock`(UE 公版引擎) — 引擎原生功能的权威答案
- `project-engine`(魔改引擎) — 项目侧对引擎的改动
- `project-game`(游戏项目 + 插件) — 项目自定义功能

机制详见 [[Wiki/Concepts/AIFoundations/Agentic-grep]]。

### 工作流

```
美术自然语言提问
  ↓
[0. 翻译] 自然语言 → 候选 symbol(查术语对照表 / UE 约定 / 追问)
  ↓
[1. grep] 在三仓跨文件搜索候选 symbol
  ↓
[2. Read] 命中后读 ±30 行上下文,把注释一起带出来
  ↓
[3. 交叉引用] 跟踪相关宏/CVar/UI 定义,补齐"从 UI 到 shader"全链路
  ↓
[4. 重写] 用"讲给美术听"的语气组织答案:概念 → UI 位置 → 底层机制 → 性能代价 → 使用建议
  ↓
[5. 证据] 每个陈述附 path:line,让程序员 review 答案时能快速判对错
  ↓
[6. 沉淀] 高价值问答 → 扔进 Raw/Notes/qa-log/ → 定期 ingest 成 wiki 页
```

### 两条腿并跑

| 路径 | 覆盖 | 优势 | 劣势 |
|---|---|---|---|
| **临场 grep** | 所有未 ingest 的长尾问题 | 覆盖全,零维护 | 首次查慢,质量看 prompt 和 LLM |
| **wiki 命中** | 高频 + 已 ingest 问题 | 答案精确,人类复核过,附读本 | 需要 wiki 先建设 |

初期全靠 grep;wiki 积累到一定量后,**绝大多数高频问题直接命中**,grep 退居长尾。这是 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] 的 compounding 效应在项目代码场景的具体兑现。

## 架构选型

| 组件 | 选型建议 |
|---|---|
| **接入层** | 飞书 / 钉钉机器人 → Claude Code headless 模式 / Anthropic Agent SDK |
| **代码访问** | 跑在能 mount UE 工程的机器上(git + p4 client),复用 `code-roots.local.md` 思路 |
| **Persona** | 做 2-3 个子命令 / [[Wiki/Concepts/AIFoundations/Agent-skills\|Skill]]:`/ask-artist`、`/ask-tech-artist`、`/ask-programmer`——同一检索底座,不同解释粒度 |
| **可追溯** | 强制每条答案附 `path:line` + 原始代码块 |
| **禁幻觉** | System prompt 明确:**未 grep 到证据不许作答**,说"没找到"比瞎编好 |
| **沉淀回路** | 高价值 Q&A 自动入 `Raw/Notes/qa-log/` → 定期人工 ingest 成 wiki 页 |
| **机密与审计** | 内网部署;不得发到公开群;保留访问日志 |

### Persona 差异示例

| Persona | 解释粒度 | 术语替换 | 代码展示 |
|---|---|---|---|
| `/ask-artist` | 类比日常经验、比喻优先 | "Early-Z"→"硬件提前偷懒丢像素"、术语首次出现加括号解释 | 代码块少、只留关键注释,重点给"勾这个 = 你看到什么 + 性能代价" |
| `/ask-tech-artist` | 概念级 + 数据流 | 保留 `discard` / `Early-Z` / `per-instance` 原词,加 1 句定义 | 关键代码块 inline,标 `path:line`,不省略 shader 宏展开 |
| `/ask-programmer` | 工程细节 + 性能模型 | 全部保留原词 | 全代码块 + 调用链 + 矛盾点分析 |

## 术语桥梁:美术词 ≠ 代码 symbol

这是 [[Wiki/Concepts/AIFoundations/Agentic-grep#破解关键词依赖:四路策略|Agentic Grep 破解关键词依赖]]的核心具体化。

**建议产物**:`Wiki/Concepts/Glossary/术语对照表.md`,结构:

| 美术 / 策划常说 | 代码 symbol | 所在模块 | 备注 |
|---|---|---|---|
| 半透明 | `bIsTranslucent` / `BLEND_Translucent` | MaterialShared | Blend mode 枚举 |
| 渐隐切 LOD | `DitheredLODTransition` / `USE_DITHERED_LOD_TRANSITION` | Material + Shaders | 有 mobile 单独开关 |
| 阴影不对 | `CastShadow` / `bCastShadowAsTwoSided` / `CastShadowAsMasked` | PrimitiveComponent + Material | 分 component / material 两层 |
| 项目 XX 切换 | 你们自定义的 `EAkiXXMode` | AkiGameplay/Plugins/XX | 跨插件 |

机器人第一跳**先查这张表**翻译词汇,再去 grep。每次 ingest 自定义功能就补一行。**半年后这张表 + wiki 就是项目 AI 的核心资产**,复利巨大。

### 优先 ingest 清单

对项目自定义代码(机器人天然不认识),优先 ingest:

1. 自定义 `UPROPERTY` 集中的头文件(UI 上所有看得到的开关源头)
2. 自定义 shader 的 `.usf` / `.ush`(里面的宏名就是未来要 grep 的关键词)
3. Plugin 的 `README` / `.uplugin` 描述
4. 项目自定义 CVar 的注册点(grep `IConsoleManager::Get\(\).RegisterConsoleVariable`)

## 部署风险与纪律

⚠️ **幻觉是头号敌人**。美术对代码没免疫力,一旦被误导很久纠不回来。System prompt 必须明确:

> 未 grep 到证据 → 说"我没找到相关代码",绝不作答。
> 有证据但不确定 → 列出证据 + 标注"以下推断仅供参考,建议找程序员复核"。

⚠️ **机密边界**。项目代码通常算机密,机器人回复**不能**截图发到公开群。部署要内网、审计日志要留。

⚠️ **p4 权限**。问到 `project-*` 仓的代码时,机器人得有 p4 读权限——要跟 IT 过一下 account。

⚠️ **术语漂移**。美术和程序员对同一功能叫法可能不同,机器人无法自发知道"美术说的 XX = 代码里的 YY"——需要术语对照表人工维护。

## POC 建议:最小可验证

**不要一上来就全项目铺开**。建议:

1. **选一个最常被问的子系统**(比如只覆盖材质 static switch)
2. **手动整理 1 份术语对照表**(10-20 行即可)
3. **只开 `/ask-material-artist` 这一个命令**
4. **找 3-5 个美术试用 2 周**
5. **跑通再扩**:加 Niagara、加 Lighting、加项目自定义模块...

评估标准:
- **命中率**:机器人回答被美术采纳的比例
- **幻觉率**:答错/瞎编被 caught 的比例(目标 < 2%)
- **时间节省**:美术少问程序员多少次
- **沉淀产出**:每周产生多少可 ingest 的 Q&A

## 与既有 wiki 的关系

- **第一性原理**:[[Wiki/Concepts/AIFoundations/Agentic-grep]] 是检索机制
- **长期记忆**:[[Wiki/Concepts/Methodology/Llm-wiki-方法论]] 是知识沉淀范式
- **架构框架**:[[Wiki/Syntheses/AIFoundations/Prompt-context-harness-evolution]] 三段论中 Harness 层的具体实例
- **技术栈**:[[Wiki/Concepts/AIFoundations/Agent-skills]] 是 persona 实现路径之一;[[Wiki/Concepts/AIFoundations/Multi-agent]] 若需更复杂检索(并行跨仓搜索)可升级
- **部署纪律**:[[Wiki/Concepts/AIFoundations/Hallucination]] 是首要防御对象
- **既有基础**:CLAUDE.md §9 三 code root 路由规则 + `Wiki/Sources/<repo>/` 源摘要已经是半成品弹药库

## 相关

- [[Wiki/Concepts/AIFoundations/Agentic-grep|Agentic Grep]] — 检索底层机制
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 长期记忆框架
- [[Wiki/Concepts/AIFoundations/Hallucination|幻觉]] — 首要防御对象
- [[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]] — Persona 实现路径
- [[Wiki/Entities/AIFoundations/OpenClaw|OpenClaw]] — Eureka 同事的类似栈,可参考
- [[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] — 本综合的线性读本版,含 Dithered LOD Transition 贯穿案例

## 引用来源

- [[Wiki/Sources/AIFoundations/Code-retrieval-conversation]] (raw: [[Raw/Notes/代码检索与美术问答机器人对话]])

## 开放问题

- **POC 评估指标**的基线怎么定?采纳率、幻觉率、节省时间——没有对照组怎么量化?
- **项目自定义代码**初期未 ingest 前,机器人第一次遇到时会不会反复"没找到",挫伤使用信心?是否需要冷启动时做一次批量 ingest?
- **蓝图/uasset 不可 grep**:纯蓝图逻辑和材质图无法走代码 grep 路径——是否需要"蓝图→文本 dump"工具链配合?
- **更新漂移**:项目代码持续变,wiki 里的答案会过时;lint 周期和触发机制需要定
- **多人协作**:多个美术同时问同一问题,机器人能不能识别"这个问题已经答过,指向某条 Q&A 历史"?
