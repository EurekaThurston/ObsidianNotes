# CLAUDE.md — Wiki Schema & Operating Instructions

> 本文档是 LLM(Claude/Claudian)维护本仓库的"作业规程"。
> 灵感来自 Karpathy 的 LLM Wiki 方法论(源见 [[Raw/Notes/Karpathy Wiki 方法论]]、概念见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论]]、完整演进脉络见 [[Wiki/Readers/Methodology/Llm-wiki-方法论-读本]])。
> 人类负责:提供原始资料、提问、指方向。LLM 负责:读、写、归档、交叉引用、维护。
>
> **本文件只含每轮对话都要遵守的核心规则。大块页面模板另存于 `Wiki/_templates/`,仅在真的要创建对应页面时 Read(降低每轮上下文噪声,防 context rot)。**

---

## 1. 三层架构

| 层 | 路径 | 所有者 | 可变性 |
|---|---|---|---|
| **Raw sources(原始资料)** | `Raw/` | 人类 | 只读(LLM 绝不修改) |
| **Wiki(知识库)** | `Wiki/` | LLM | LLM 完全拥有,持续更新 |
| **Schema(本文件)** | `CLAUDE.md` | 人 + LLM 共同演进 | 谨慎修改 |

特殊文件:
- `index.md` — 内容目录(按类别列所有 wiki 页)
- `log.md` — 时间线日志(追加式)
- `Wiki/Overview.md` — 顶层综合视图

---

## 2. 目录结构

```
Raw/
  Articles/   网页剪藏、博客文章
  Papers/     论文(PDF 或 markdown)
  Books/      书籍章节笔记
  Notes/      个人手记、播客/视频笔记、会议记录
  Assets/     图片等附件(Obsidian 附件目录)

Wiki/
  Overview.md         顶层综合
  Concepts/<主题>/    概念页:想法、理论、术语
  Entities/<主题>/    实体页:人、组织、产品、代码类
  Sources/<主题>/     源摘要:每个 raw 文件 / 代码源一页
  Syntheses/<主题>/   非读本类的跨源专题(对比、指南、学习路径总图)
  Readers/<主题>/     主题读本:每议题"一次读完即掌握"线性读物(§3.4)
  _templates/         ⭐ 页面模板,创建新页时 Read 对应文件
```

**目录约定**:

- 按**主题**建子目录(大写开头);新主题时新增
- `Entities/` 和 `Sources/` 的 `Stock/` / `Project/` 子目录专用于代码(§9)
- **`Syntheses/` vs `Readers/`**:Readers 是每议题必有的"人类首选入口";Syntheses 是非读本类专题(跨主题对比、指南、总图)。一议题通常一份读本 + 零到多份专题

---

## 3. 核心操作

### 3.1 Ingest(摄入新资料)

人类把文件放进 `Raw/` 并说"帮我 ingest"时:

1. 通读 `Raw/xxx` 全文
2. 与人类讨论要点,确认关键信息
3. 写源摘要:`Wiki/Sources/<topic>/xxx.md`(模板 `Wiki/_templates/Source.md` 或 `Code-Source.md`)
4. 更新受影响的实体/概念页(模板 `Entity-Concept.md`);缺就新建
5. 更新 `index.md`(登记新页)
6. 如有重大影响,更新 `Wiki/Overview.md`
7. 追加 `log.md`(格式见 §5)
8. **若本轮产出 ≥3 原子页、属新主题首次入驻、或结构化学习路径的一个 Phase 完成 → 同步产出读本(§3.4)**

一次 ingest 通常触及 5-15 个 wiki 页。**这是特性,不是 bug**——wiki 强于 RAG 的根源。

### 3.2 Query(提问)

1. 先读 `index.md` 找相关页
2. 深入相关页,必要时回溯 `Raw/` 查原文
3. 用 wikilink 引用,原文用 `[[Raw/...]]`
4. **好答案要沉淀**:有长期价值的答案主动归档为 `Wiki/Syntheses/<topic>/xxx.md`
5. 视情况追加 `log.md`

### 3.3 Lint(体检)

人类说"帮我 lint 一下 wiki"时,**先 Read [[Wiki/_templates/Lint-checklist]]**,按其中 0-10 节 checklist 执行。核心输出:

- 结构化 lint 报告(§9 的 8 类检查 + 建议处置顺序 🔴🟡🟢)
- log.md 追加 `## [YYYY-MM-DD] lint | ...`

**lint 是诊断不是治疗**——修复前要用户批准。

### 3.4 读本(主题线性读物)

**每个有深度的议题都必须配套产出一份主题读本**,归档 `Wiki/Readers/<topic>/`。模板 `Wiki/_templates/Reader.md` 含完整骨架 + 硬性要求清单。

**定义**:读本是"**详细、精确、满满当当、一次读完即完整掌握该议题**"的长文。**不是摘要**。人类不跳转就能拿走全部知识。

**触发条件**(任一):

1. 结构化学习路径的每个 Phase 收尾(如 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 每完成一 Phase)
2. 新主题首次入驻且本轮产出 ≥3 原子页
3. 多源交叉综合(同议题跨多 raw source)
4. 用户显式要求"给我一份能一次读完的"

**硬性要求**(见模板 header):线性叙事 / 不遗漏 > 简短 / 关键代码 inline / 问题驱动 / ⚠️ 陷阱埋雷 / 末尾深入阅读 + 下一步 + open questions + 签名行。

**原子 vs 读本 分工**:

| | 原子页(Concept/Entity/Source) | 读本 |
|---|---|---|
| 服务对象 | LLM 检索、字段级查询 | 人类线性阅读、完整掌握 |
| 结构 | 结构化表/列表 | 叙事段落 + 代码 inline |
| 长度 | 20-250 行 | **不限,内容决定** |
| 完整度 | 局部精确 | 整体贯通不遗漏 |

每次 ingest **同时**产出原子页 + 读本(若触发),**不要只产出原子页**——会让人类被迫跳转拼接。

**产出读本后的清单**:

1. 追加 `log.md`:`## [YYYY-MM-DD] synthesis | <议题> 读本`
2. 在 `index.md` 的 Readers 分区登记
3. 若议题有总图/学习路径,对应位置加读本链接
4. 视影响更新 `Wiki/Overview.md`

---

## 4. 页面格式

所有 wiki 页头部 YAML frontmatter:

```yaml
---
type: entity | concept | source | synthesis | overview
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [...]
sources: N       # 被多少个 raw source 支持
aliases: [...]   # 可选

# ——仅代码页面(Entities/Stock|Project, Sources/Stock|Project)——
repo: stock | project
source_root: UE-4.26-Stock
source_path: Engine/Source/...
source_ref: "4.26"
source_commit: b6ab0dee9
twin: [[Entities/Project/XXX]]     # 跨仓孪生页,可选
---
```

**`Wiki/_templates/`**:LLM 在做特定操作时按需 Read 的 schema 参考。**页面模板**(创建新页用)和**操作 checklist**(执行操作用)都放这里:

| 用途 | 触发 | 文件 |
|---|---|---|
| Entity / Concept 页骨架 | 新建 Entity/Concept | `Wiki/_templates/Entity-Concept.md`(单 source 派生的 Code Entity 控制 30-50 行) |
| Source(非代码)骨架 | 新建 Source,raw 在 `Raw/` 下 | `Wiki/_templates/Source.md` |
| Code Source 骨架 | 新建 Source,源真相在 code root | `Wiki/_templates/Code-Source.md` |
| Reader(主题读本)骨架 | 新建读本,详见 §3.4 | `Wiki/_templates/Reader.md` |
| Lint 作业 checklist | 执行 §3.3 lint | `Wiki/_templates/Lint-checklist.md` |

**Code Entity 紧凑约定**:单 source 派生时 30-50 行。完整字段清单放 Source,叙事放读本,Entity 只作稳定入口。多 source 引用后可扩展为汇总枢纽,仍精炼,细节下沉。

---

## 5. log.md 格式

**每条以 `## [YYYY-MM-DD] <op> | <title>` 开头**,方便 grep 回溯:

```
grep "^## \[" log.md | tail -10
```

op 取值:`ingest / query / lint / synthesis / refactor / bootstrap`。

示例:

```markdown
## [YYYY-MM-DD] ingest | 某文章标题
- source: [[Raw/Articles/xxx]]
- 新建:[[Wiki/Sources/Topic/xxx]]、[[Wiki/Entities/Topic/某某]]
- 更新:[[Wiki/Concepts/Topic/某某]](+3 句)、[[Wiki/Overview]]
- 要点:…
- 下一步:…
```

---

## 6. 链接 & 命名

- **内部链接**用 wikilink:`[[Wiki/Concepts/Topic/abc]]` 或 `[[abc]]`(无歧义时)
- **文件名**小写+连字符:`rag-vs-wiki.md`,避免空格(必要除外)。中文名可,但尽量配英文 alias
- **指回原始资料**必须标注 `[[Raw/Articles/xxx]]`,让人类一键跳源
- **图片**用 `![[xxx.png]]`,文件放 `Raw/Assets/`

---

## 7. 重要原则(Don'ts)

1. **绝不修改 `Raw/` 下任何文件**。raw 是 source of truth,只读
2. **不凭空编造**。wiki 每条陈述应可追溯到某 raw source(或明确标注"推断")
3. **矛盾不偷偷覆盖**——并列记录,在"矛盾"区注明来源冲突,交给人类裁决
4. **不批量 ingest 而不更新 index 和 log**。bookkeeping 是这套方法的灵魂
5. **不把 `CLAUDE.md` 当作不可变**——工作流演进时回来更新本文件
6. **新流程/规则落地前先做分层判断**:问一句"这是**每轮对话都要遵守**的,还是**只在做 X 时才需要**的?"
   - 每轮都要守(心智模型、硬约束、路由规则)→ 进 CLAUDE.md 某一节,尽量简短
   - 只在做 X 时要(页面模板、详细执行手册、一次性决策)→ 外移到独立文件(`Wiki/_templates/`、议题的 wiki 页、或专门 playbook)
   - **CLAUDE.md 只放 always-apply 的核心,防 context rot**。本条本身也按此规则落地——它是元规则,故进 §7

---

## 8. 工作流提示(给人类)

- 新资料进来:扔进 `Raw/对应子目录`,告诉 Claudian "帮我 ingest xxx"
- 提问:直接问,Claudian 会先查 index 再答
- 定期(每周 / 每 N 次 ingest 后)跑一次 lint
- 看 Obsidian 图谱视图 → 看 wiki 的形状(枢纽页、孤儿页)
- 要新输出形式(幻灯片、图表、canvas)→ 告诉 Claudian,它会生成并(如有价值)归档到 `Syntheses/<topic>/`

---

## 9. Code Roots(代码源)

本 wiki 纳管多个代码仓,代码**不复制进 `Raw/`**——从本机磁盘直接读。LLM 按关键词路由到正确的仓。

> **多机协作**:本 vault 可能同步到多台机器,代码仓绝对路径不同,某些仓可能不存在。因此:
> - **本文件(同步)只定义逻辑**:root id、aliases、VCS 类型、路由规则
> - **绝对路径写在 `code-roots.local.md`(不同步)**——已 `.gitignore`,每台机器各自维护

### 9.1 登记的 Code Roots

| id | 中文名 | 分支/版本 | VCS | 触发关键词 |
|---|---|---|---|---|
| `stock` | UE 公版(个人学习版) | branch `Eureka`(基于 4.26) | git | 公版、UE 官方的、stock |
| `project-engine` | Aki 项目魔改引擎 | UE 4.26.2 | perforce | 项目引擎、魔改引擎、aki 引擎 |
| `project-game` | Aki 游戏项目(含 `Plugins/`) | — | perforce | 项目、aki、插件、业务代码、Client |

**说明**:

- `stock` root = UE 仓根(含 `Engine/` `Templates/` `UE4.sln`)
- `project-engine` root = UE 仓根(`Engine/` 父目录),相对路径与 stock 对齐便于 diff
- `project-game` root = 含 `Client.uproject` 的游戏项目根

### 9.2 本机路径

- 绝对路径在 `code-roots.local.md`。**读代码前必须先 Read 此文件拿路径**
- `code-roots.local.md` 不存在:停下提示用户创建模板
- 某 id 未登记(如家里电脑没有 `project-*`):直接说"本机未登记 `<id>`",不猜
- 新加 code root:§9.1 加逻辑定义 + 各机器 `code-roots.local.md` 补 path

### 9.3 路由规则(按顺序)

1. **显式关键词**:用户说话命中 §9.1 aliases → 定位该仓
2. **当前页上下文**:`<current_note>` 的 frontmatter `repo:` 字段 → 沿用
3. **对比意图**:"这段在项目里改了什么 / 两边区别 / diff" → 同时读两仓,产出或更新 `Syntheses/Topic/xxx-stock-vs-project.md`
4. **歧义兜底**:未命中 / 同 alias 多仓 → **停下来问用户**,不猜
5. **命中但本机没有**:按 §9.2 处理,停下告诉用户

### 9.4 VCS 约定

**stock(git)**:

- ingest 时记 `git -C <path> rev-parse --short HEAD` 为 `source_commit`
- 历史:`git log --follow <file>` / `git blame`

**project-engine / project-game(perforce)**:

- `p4` 通常在 `C:\Program Files\Perforce\p4.exe`。没有就降级
- `p4config.txt` 在项目树内,`P4CONFIG=p4config.txt` + cwd 进入项目树即自动生效。路径见 `code-roots.local.md`
- 历史:`p4 filelog <path>` / `p4 annotate -c <path>` / `p4 describe <CL>`。首次需 `p4 login`
- `p4` 不可用:降级为纯文件系统模式(Read/Grep/Glob),`source_commit` 填 `"p4: unknown"`

### 9.5 Ingest 代码的流程(§3.1 基础上)

1. Read `code-roots.local.md` 确认 id 可用
2. 按 §9.3 路由决定 `repo`
3. 记录 `source_commit` / `source_ref` 到 frontmatter
4. 源摘要写到 `Wiki/Sources/<repo>/<文件名>.md`(模板 `Code-Source.md`)
5. Entity 写到 `Wiki/Entities/<repo>/<类名或模块名>.md`
6. 另一仓已有同名 entity:**双向加 `twin:` 链**,考虑产出 `Syntheses/Topic/xxx-stock-vs-project.md`
7. Concept 写到 `Wiki/Concepts/<topic>/`;若一概念两仓差异大,页内开 `## Stock 实现` / `## Project 实现` 两节
