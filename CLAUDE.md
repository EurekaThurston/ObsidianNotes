# CLAUDE.md — Wiki Schema & Operating Instructions

> 本文档是 LLM(Claude/Claudian)维护本仓库的"作业规程"。
> 灵感来自 Karpathy 的 LLM Wiki 方法论(源文件见 [[Raw/Notes/Karpathy Wiki 方法论]],概念见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论]])。
> 人类负责:提供原始资料、提问、指方向。LLM 负责:读、写、归档、交叉引用、维护。

---

## 1. 三层架构

| 层 | 路径 | 所有者 | 可变性 |
|---|---|---|---|
| **Raw sources(原始资料)** | `Raw/` | 人类 | 只读(LLM 绝不修改) |
| **Wiki(知识库)** | `Wiki/` | LLM | LLM 完全拥有,持续更新 |
| **Schema(本文件)** | `CLAUDE.md` | 人 + LLM 共同演进 | 谨慎修改 |

特殊文件:
- `index.md` — 内容目录(按类别列出所有 wiki 页面)
- `log.md` — 时间线日志(追加式)
- `Wiki/Overview.md` — 当前最高层级的综合视图

---

## 2. 目录说明

```
Raw/
  Articles/   网页剪藏、博客文章(Obsidian Web Clipper 输出)
  Papers/     论文(PDF 或 markdown)
  Books/      书籍章节笔记
  Notes/      个人手记、播客/视频笔记、会议记录
  Assets/     图片等附件(Obsidian 附件目录指向这里)

Wiki/
  Overview.md     顶层综合:当前对整个知识领域的"最佳理解"
  Concepts/       概念页:想法、理论、方法、术语
    Methodology/    - 知识管理方法论相关概念
    UE4/            - UE4 引擎基础概念
    Niagara/        - Niagara 粒子系统概念
    (新主题时在此增加子目录)
  Entities/       实体页:人、组织、地点、产品、项目
    Methodology/    - 方法论相关人物/组织
    Stock/          - 代码实体:UE 公版引擎(见 §9 Code Roots)
    Project/        - 代码实体:Aki 项目魔改引擎
  Sources/        源文件摘要:每个 raw 文件或代码源对应一页
    Methodology/    - 方法论相关源摘要
    Stock/          - 代码源摘要:UE 公版
    Project/        - 代码源摘要:Aki 项目
  Syntheses/      跨源综合:对比、分析、专题报告、你问出来的好答案
    Niagara/        - Niagara 源码学习相关综合
    (新主题时在此增加子目录)
```

- 所有目录均按**主题**建子文件夹（大写开头）；新增主题时在对应父目录下加一个子目录。
- `Concepts/` 子目录名代表主题领域；同一概念跨仓有不同实现时，页内分节讨论，不复制成两份。
- `Entities/` 与 `Sources/` 的 `Stock/` / `Project/` 子目录专用于代码实体/摘要（见 §9）；非代码内容按主题放入 `Methodology/` 等目录。

---

## 3. 核心操作(Operations)

### 3.1 Ingest(摄入新资料)

当人类把一个文件放进 `Raw/` 并说"帮我吸收这个"时:

1. **读源**:通读 `Raw/xxx` 全文。
2. **讨论要点**:与人类对话,确认关键信息、你的理解、人类想强调什么。
3. **建立源页**:在 `Wiki/Sources/<topic>/` 写一页摘要,文件名与源文件对应(例如 `Raw/Articles/xxx.md` → `Wiki/Sources/Topic/xxx.md`)。包含:
   - 源路径链接、作者、日期、来源 URL(如有)
   - 3-10 句话核心摘要
   - 关键要点列表
   - 引用出的实体/概念(wikilink 形式)
4. **更新受影响页**:源里提到的每个实体/概念,去 `Wiki/Entities/<topic>/` 或 `Wiki/Concepts/<topic>/` 找对应页面:
   - 已有 → 增补、修订、标注矛盾
   - 没有但值得 → 新建
5. **更新 index.md**:新增页面登记;已更新页面可选更新 one-liner。
6. **更新 Overview.md**(如有重大影响):综合视图更新。
7. **追加 log.md**:一条 ingest 记录,格式见 §5。

一次 ingest 通常会触及 5-15 个 wiki 页面。**这是特性,不是 bug**——这就是为什么 wiki 比 RAG 强。

### 3.2 Query(提问)

当人类问问题时:

1. 先读 `index.md` 找相关页。
2. 深入相关页面,必要时回溯到 `Raw/` 查原文。
3. 给出带引用的答案:引用格式用 wikilink `[[Wiki/xxx]]`,原文用 `[[Raw/xxx]]`。
4. **好答案要能沉淀**:如果这个答案本身有长期价值(对比、分析、新发现),主动提议归档为 `Wiki/Syntheses/<topic>/xxx.md`。
5. 追加 log.md 一条 query 记录(可选,视是否重要)。

### 3.3 Lint(体检)

人类说"帮我 lint 一下 wiki"时:

- 找出相互矛盾的陈述(列出位置)
- 找出被新源推翻的旧陈述
- 找出孤儿页(没有任何页面链入)
- 找出多次被提到但还没有独立页面的概念
- 找出缺失的交叉引用
- 建议:下一步可以补哪些数据、问哪些问题、找哪些资料

追加 log.md 一条 lint 记录。

### 3.4 主题读本(Topic 级线性读物)

**每个议题(topic / issue)都必须配套产出一份"主题读本"**,归档到 `Wiki/Syntheses/<topic>/`。议题的粒度由产出量决定:结构化学习路径的每个 Phase、一份新入驻的重量级扫盲源、一组围绕同一主题的多个 raw source、用户提出的一个深度问题且答案跨多页——都算一个议题。

**读本不是摘要**。读本是一篇"**详细、精确、满满当当、一次读完即完整掌握该主题**"的长文。用户/人类读者**不需要跳来跳去**就能拿走此议题的全部知识。

#### 读本 vs 原子页:分工

原子 Source/Entity/Concept 页与主题读本并非取舍,而是**互补的两层**:

| 维度 | 原子页(Source/Entity/Concept) | 主题读本(Synthesis) |
|---|---|---|
| 主要服务对象 | LLM 检索、字段级查询、溯源 | 人类线性阅读、从零完整掌握 |
| 行文结构 | 结构化表/列表 | 叙事段落 + 关键代码片段 inline |
| 长度约束 | 短(20-250 行) | **不限,内容决定** |
| 完整度要求 | 局部精确 | **整体贯通,不遗漏** |
| 被替代性 | 可被读本末尾"深入阅读"章节指向 | 不可替代,人类阅读首选入口 |

**原则**:每次 ingest 同时产出原子页(spec 层)+ 读本(人类阅读层)。不要只产出原子页,那会让人类读者被迫跳转拼接。

#### 触发条件(必须产出读本)

1. **结构化学习路径的每个 Phase 收尾**(如 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 每完成一 Phase)
2. **新主题首次入驻**且本轮产出 ≥3 个原子页(如 AIApps / AIArt / Niagara 首次入驻时的场景;这类情形之前没强制产出读本,是历史债)
3. **多源交叉综合**——同一议题在多个 raw source 里都有重要内容,需要编织统一视图
4. **用户显式对某议题要求**"给我一份能一次读完的"

#### 写作要求(硬性)

- **线性叙事**:从头读到尾完成该议题的完整掌握,不强制跳转
- **准确、不遗漏 > 简短**:长度服从内容,不为压缩而省略关键细节
- **关键代码/原文片段 inline**(`>` 引用块),重要 aha 点直接引原文 + 行号/路径
- **问题驱动**的结构,不是字段罗列("为什么要 X?" 优于 "X 的字段有...")
- **重要陷阱 / 易错点用 ⚠️ 高亮**,并提前埋雷到读者下一步会踩的地方
- 末尾必有 **"深入阅读"** 章节,分层列出该议题的全部原子页 + 前置/后续议题的读本链接
- 末尾必有 **下一步预告**(下一 Phase、下一议题、或后续可深入方向)+ **本议题遗留的 open question 清单**(明确标注哪个 Phase/议题回答)
- 末尾一行签名:`*本读本由 [[Claudian]] 基于 <议题>的 X 个原子页综合生成,YYYY-MM-DD。*`

#### 文件命名

- 结构化学习路径阶段:`PhaseN-<主题片段>-读本.md`
- 其他议题:`<议题>-读本.md`(如 `Lora-深度指南-读本.md`、`AI-primer-v2-读本.md`)

#### 执行清单(产出读本时按序)

1. 原子页 ingest 完成(如属 ingest 场景)
2. 产出读本页
3. 追加 log.md 一条 `## [YYYY-MM-DD] synthesis | <议题> 读本`
4. 在 index.md 的 Syntheses 分区登记
5. 在该议题的总图/学习路径/总入口(若有)里添加读本链接
6. 视影响更新 [[Wiki/Overview]]

#### Entity 页的协作规则

当 Entity 页从**单一 Source** 派生(典型:一个 header 定义一个主类),Entity 页保持**紧凑**(~30-50 行):一句话角色 + 核心字段表 + 常用方法 + 相关链接 + 指回 Source 和读本。详细字段清单和代码片段放 Source 页,叙事和心智模型放读本。

Entity 页后续被多个 Source 引用时,可扩展为"跨 source 汇总入口",但仍保持精炼,细节下沉。

#### 历史债(待回补)

以下主题 ingest 时尚未建立读本规范,当时未产出:
- **AI 美术(LoRA 深度指南)**:5 Concepts + 5 Entities + 1 Source + Raw 源,产出量足,应有读本
- **AI 应用生态(AI Primer v2)**:8 Concepts + 1 Entity + 1 Synthesis + 1 Source,产出量足,应有读本
- **方法论(Karpathy LLM Wiki)**:产出 3 Concepts + 1 Entity + 1 Source,边界可选

用户提出时按此清单回补。

---

## 4. 页面格式约定

### 4.1 YAML Frontmatter(所有 wiki 页面)

```yaml
---
type: entity | concept | source | synthesis | overview
created: 2026-04-17
updated: 2026-04-17
tags: [tag1, tag2]
sources: 3   # 被多少个 raw source 支持
aliases: [别名1, 别名2]   # 可选

# ——以下字段仅代码相关页面(Entities/Stock|Project, Sources/Stock|Project)使用——
repo: stock | project            # 必填:属于哪个 code root(见 §9)
source_root: UE-4.26-Stock       # code root 的 name,可选(便于阅读)
source_path: Engine/Source/...   # 相对 code root 的路径
source_ref: 4.26 | Eureka        # 分支/tag
source_commit: b6ab0dee9         # git HEAD(公版)或 p4 CL(项目),ingest 当时的快照
twin: [[Entities/Project/XXX]]   # 另一个仓里的孪生页,可选
---
```

### 4.2 实体/概念页结构

```markdown
# 名称

> 一句话定义。

## 一句话角色 / 概览
段落总述(1-2 段,≤5 句)。

## 核心事实 / 字段速查
- 表格或列表,尽量紧凑
- 重要陷阱用 ⚠️ 标

## 相关
- [[实体/概念 A]] — 关系
- [[实体/概念 B]] — 关系

## 深入阅读
- 源摘要(含全字段 + 代码引用):[[Wiki/Sources/Topic/xxx]]
- 主题读本(若有):[[Wiki/Syntheses/Topic/<议题>-读本]] § N

## 开放问题 / 矛盾
- …
```

**Code Entity 页的紧凑约定**:当一个 Entity 由单一 Source 派生(典型:一个 header 定义一个主类),该 Entity 页应控制在 **30-50 行**。完整字段清单放 Source 页,叙事/心智模型放对应主题读本(见 §3.4 / §4.5),Entity 页只作为**稳定跨 source 入口**。若后续被多 Source 引用,可扩展为汇总枢纽,但仍保持精炼,细节下沉。

### 4.3 源摘要页结构

```markdown
# 源标题

- **原文**:[[Raw/Articles/xxx]]
- **作者**:
- **日期**:
- **URL**:
- **类型**:article / paper / book / note

## 摘要
3-10 句。

## 关键要点
- …

## 涉及实体 / 概念
- [[xxx]]、[[yyy]]

## 与既有 wiki 的关系
- 印证了:[[…]]
- 矛盾:[[…]]
- 新增:[[…]]
```

### 4.4 代码源摘要页结构(`Sources/Stock/*`、`Sources/Project/*`)

代码源不在 `Raw/` 下——源真相是 code root(见 §9)里的真实文件。摘要页记录"这一次 ingest 读了哪些代码、看出了什么"。

```markdown
# 文件名或模块名

- **Repo**:stock(或 project)
- **路径**:`Engine/Source/Runtime/.../Foo.h` [(GitHub)](可选链接)
- **快照**:commit `b6ab0dee9` / CL `12345`
- **同仓协作文件**:`Foo.cpp`、`FooInstance.h` …
- **Ingest 日期**:2026-04-17

## 职责 / 这个文件干什么的
2-5 句。

## 关键类型 / 函数 / 宏
- `UFoo`:…
- `FFooProxy`:…

## 依赖与被依赖
- 上游(此文件 include / 使用):`Bar.h`、…
- 下游(谁用它):`Baz.cpp`、…

## 关键代码片段
> `Foo.h:L120-L135` @ `b6ab0dee9`
> ```cpp
> // 只贴真正值得记的片段,不复制整文件
> ```

## 涉及实体 / 概念
- [[Entities/Stock/UFoo]]、[[Concepts/Topic/xxx]]

## 与另一仓差异(如已 ingest 孪生)
- 对比见 [[Syntheses/Topic/foo-stock-vs-project]]
```

### 4.5 主题读本页结构(见 §3.4)

通用骨架(适用所有议题,结构化学习路径/新主题入驻/综合专题均按此):

```markdown
# <议题> 读本 — <副标题>

> 本页是 <议题> 的**主题读本**——详细、精确、满满当当,一次读完即完整掌握<议题>的全部知识,不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

## 0. <议题> 要回答的问题
- 问题陈述
- 与前置议题的衔接(若有)
- 本议题的叙事主线(可给一张 ASCII 链条图)

## 1..N. 按叙事顺序展开
每节建议结构:
- 该节回答的问题 / 该原子(概念/类/方法)的角色
- 关键代码/原文片段 inline(`>` 引用块 + 行号/路径)
- 重要陷阱用 ⚠️,必要时提前埋雷
- 节末小结

## 全景回看
- 把所有原子拼成一张图
- 回顾整体数据流 / 控制流 / 概念依赖链

## N 条关键洞察
- 读者带走的核心结论

## <议题> 留下的问题
- 等后续议题/Phase 回答的 open question,明确标注"→ Phase X"或"→ <议题 Y>"

## 下一步预告
- 下一 Phase / 下一议题 / 可深入方向

## 深入阅读
- 原子 Source 页:[[...]] × N
- Entity 页:[[...]] × N
- 前置 Concept 页:[[...]] × N
- 导航/总图:[[...]]

---

*本读本由 [[Claudian]] 基于 <议题> 的 X 个原子页综合生成,YYYY-MM-DD。*
```

---

## 5. log.md 条目格式

**每条必须以 `## [YYYY-MM-DD] <op> | <title>` 开头**,这样可以用 grep 快速回溯:

```
grep "^## \[" log.md | tail -10
```

格式示例:

```markdown
## [2026-04-17] ingest | 某文章标题
- source: [[Raw/Articles/xxx]]
- 新建:[[Wiki/Sources/Topic/xxx]]、[[Wiki/Entities/Topic/某某]]
- 更新:[[Wiki/Concepts/Topic/某某]](+3 句)、[[Wiki/Overview]]
- 要点:…

## [2026-04-17] query | "X 和 Y 的区别"
- 归档为:[[Wiki/Syntheses/Topic/x-vs-y]]

## [2026-04-17] lint
- 矛盾:[[A]] 与 [[B]] 关于 Z 的陈述
- 孤儿:[[C]]
- 建议补源:…
```

---

## 6. 链接 & 命名约定

- **内部链接**一律用 wikilink:`[[Wiki/Concepts/Topic/abc]]` 或 `[[abc]]`(当无歧义时)。
- **文件名**用小写+连字符:`rag-vs-wiki.md`,避免空格(除非必须)。中文名可用,但尽量配英文 alias。
- **指回原始资料**必须标注:`[[Raw/Articles/xxx]]`,让人类能一键跳源。
- **图片**用 `![[xxx.png]]` 嵌入,文件放 `Raw/Assets/`。

---

## 7. 重要原则(Don'ts)

1. **绝不修改 `Raw/` 下任何文件**。raw 是 source of truth,只读。
2. **不要凭空编造事实**。wiki 里每一条陈述都应可追溯到某个 raw source(或明确标注为"推断")。
3. **发现矛盾时不要偷偷覆盖**——并列记录,在页面底部的"矛盾"区注明来源冲突,交给人类裁决。
4. **不要批量 ingest 而不更新 index 和 log**。bookkeeping 是这套方法的灵魂。
5. **不要把 `CLAUDE.md` 当成不可变的**——当工作流演进,回来更新本文件。

---

## 8. 工作流提示(给人类)

- 新资料进来:扔进 `Raw/对应子目录`,告诉 Claudian "帮我 ingest xxx"。
- 提问:直接问,Claudian 会先查 index 再答。
- 定期(每周 / 每 N 次 ingest 后)跑一次 lint。
- 查看 Obsidian 图谱视图 → 看 wiki 的形状(枢纽页、孤儿页)。
- 想要新输出形式(幻灯片、图表、canvas)→ 告诉 Claudian,它会生成并(如果有价值)归档到 `Syntheses/<topic>/`。

---

## 9. Code Roots(代码源)

本 wiki 纳管多个代码仓,代码本身**不复制进 `Raw/`**——直接从本机磁盘读。LLM 根据对话里的关键词路由到正确的仓。

> **多机协作**:本 vault 可能在多台机器间同步(例如公司电脑 + 家里电脑),各机器上的代码仓绝对路径不同,某些仓甚至不存在。因此:
> - **本文件(`CLAUDE.md`,同步)只定义逻辑**:有哪些 root id、它们的 aliases、VCS 类型、路由规则。
> - **绝对路径写在 `code-roots.local.md`(不同步)**——该文件被 `.gitignore`,每台机器各自维护。

### 9.1 登记的 Code Roots(逻辑定义)

| id | 中文名 | 分支/版本 | VCS | 触发关键词(aliases) |
|---|---|---|---|---|
| `stock` | UE 公版(个人学习版) | branch `Eureka`(基于 4.26) | git | 公版、UE 官方的、官方的、stock |
| `project-engine` | Aki 项目魔改引擎 | UE 4.26.2(`++UE4+Release-4.26`) | perforce | 项目引擎、魔改引擎、aki 引擎、project-engine |
| `project-game` | Aki 游戏项目(含 `Plugins/`) | — | perforce | 项目、项目里的、aki、插件、业务代码、Client、project-game |

**说明**:
- `stock` 的 root 是 UE 仓根(含 `Engine/` `Templates/` `UE4.sln` 等)。
- `project-engine` 的 root 选在 UE 仓根(`Engine/` 的父目录),使其与 `stock` 的相对路径形如 `Engine/Source/Runtime/...`,便于 diff。
- `project-game` 的 root 是含 `Client.uproject` 的游戏项目根;业务 C++ 在 `Source/`,项目私有插件在 `Plugins/`。
- 三仓都可能不在本机存在(见 §9.2)。

### 9.2 本机路径与缺失处理

- **路径表**:所有绝对路径只写在 vault 根的 `code-roots.local.md`。LLM 每次要读代码**前**必须先 Read 这个文件拿到路径。
- **文件不存在**:如果 `code-roots.local.md` 不存在,停下来提示用户创建(给一份模板),不瞎猜路径。
- **缺失的 root**:如果 `code-roots.local.md` 里**没有登记**某个 id(例如家里电脑没有 `project-engine` / `project-game`),用户问到时直接回答:"本机未登记 `<id>`,当前可用仓:...",不报错、不尝试替代路径。
- 新加一个 code root 的流程:在 §9.1 加一行逻辑定义 → 在各台机器的 `code-roots.local.md` 里补 path。

### 9.3 路由规则(LLM 按此顺序决定读哪个仓)

1. **显式关键词**:用户说话命中 §9.1 的 aliases → 直接定位到对应仓。
2. **当前页上下文**:`<current_note>` 指向的 wiki 页 frontmatter 有 `repo:` 字段 → 沿用。
3. **对比意图**:用户说"这段在项目里改了什么 / 两边区别 / diff"等 → 同时读相关的两个仓(通常 `stock` + `project-engine`),产出或更新 `Syntheses/Topic/xxx-stock-vs-project.md`。
4. **歧义兜底**:以上都没命中,或同一 alias 可能命中多仓时 → **停下来问用户**选哪个,不猜。
5. **命中了但本机没有**:按 §9.2 缺失处理,告诉用户并停下。

### 9.4 VCS 使用约定

**stock(git)**:
- ingest 时记录 `git -C <stock-path> rev-parse --short HEAD` 作为 `source_commit`。
- 追溯历史:`git -C <stock-path> log --follow <path>`、`git -C <stock-path> blame <path>`。

**project-engine / project-game(perforce)**:
- P4 CLI 通常在 `C:\Program Files\Perforce\p4.exe`(Windows 下装了 P4V 即有)。没有就降级。
- P4 配置文件 `p4config.txt` 位于项目树内,`P4CONFIG=p4config.txt` 环境变量 + cwd 进入项目树时自动生效。具体位置见 `code-roots.local.md`。
- 历史:`p4 filelog <path>`、`p4 annotate -c <path>`、`p4 describe <CL>`。首次调用可能需要用户在终端 `p4 login`。
- 若 `p4` 不可用或未登录,**降级为纯文件系统模式**(只 Read/Grep/Glob),`source_commit` 字段填 `"p4: unknown"`。

### 9.5 Ingest 代码的流程补丁(在 §3.1 基础上)

1. Read `code-roots.local.md`,确认相关 id 在本机可用。
2. 按 §9.3 路由决定 `repo`。
3. 记录快照:`source_commit` / `source_ref`。
4. Source 摘要页写到 `Wiki/Sources/<repo>/<文件名>.md`（如 `Sources/Stock/`），按 §4.4 模板。
5. Entity 页写到 `Wiki/Entities/<repo>/<类名或模块名>.md`（如 `Entities/Stock/`）。
6. 若另一仓已有同名 entity,**双向加 `twin:` 链**,并考虑生成/更新 `Syntheses/Topic/xxx-stock-vs-project.md`。
7. Concept 页写到 `Wiki/Concepts/<topic>/` 对应主题子目录;若一个概念在两仓实现差异大,页内开 `## Stock 实现` 和 `## Project 实现` 两节,分别链回各自 entity。
