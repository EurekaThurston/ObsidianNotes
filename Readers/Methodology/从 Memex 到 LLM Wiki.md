---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [methodology, llm, knowledge-management, reader, wiki]
sources: 5
aliases: [LLM Wiki 方法论读本, 方法论读本, 本仓库为什么存在]
---

# 从 Memex 到 LLM Wiki

> 本页是"LLM Wiki 方法论"这一议题的**主题读本**——详细、精确、满满当当,一次读完即完整掌握本仓库所依据的方法论、它的思想源流、与主流 RAG 范式的根本差异、提出者的背景与其方法论脉络,以及本仓库是如何把它具体实现的。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. 这个议题要回答的问题

本仓库是一个 LLM 主动维护的 markdown 知识库,存在于 Obsidian vault 里。它凭什么值得存在?跟主流的"把文件扔给 ChatGPT 问问题"有什么本质不同?

具体地,读完本文你应该能回答:

1. **为什么**要造这样一个 wiki,而不是直接用 RAG(把原始资料丢进 NotebookLM / ChatGPT 文件上传)?
2. **它的历史根源**是什么?它是新发明吗,还是某个 80 年前愿景的当代兑现?
3. **它的底层机制**是什么?三层架构、三操作、两个特殊文件分别指什么,为什么这么划分?
4. **它和 RAG 的根本区别**在哪里?不是技术细节,是哲学差异
5. **Karpathy 是谁**,为什么他提出的方法论值得采纳,在 AI 应用方法论演进里他占什么位置?
6. **本仓库**(`D:\Notes`)如何把这套抽象方法论**落地为具体规则**?`CLAUDE.md` 的角色?

叙事主线是**一条时间线 + 一条哲学线**交织:

```
时间线:  1945 Memex                  2022-2025 RAG                 2026-04 LLM Wiki
                                                                              │
                                                                              ▼
哲学线:  "关联文档 = 价值" ─→ "查询时拼碎片" ─→ "持续编译 + 复利"
         未能解决:"谁维护"                      LLM 解决了"谁维护"
```

下面按这条脉络,从最底层的历史愿景推进到本仓库的具体实现。

---

## 1. 1945 年的愿景:Memex 与"未能落地的问题"

### 1.1 Vannevar Bush 与 *As We May Think*

1945 年,时任美国战时科学研究办公室主任的 Vannevar Bush 在《大西洋月刊》发表 *As We May Think*,描绘了一台名为 **Memex** 的个人知识设备:

- 一台**私人**设备(不是公共网络)
- 使用者**主动策划**内容(决定收录什么、如何组织)
- 文档之间用 **"associative trails"**(关联小径)连接——**连接本身被视为一等公民的价值**,与文档内容同等重要

这是人类历史上对"外脑"最早的系统化设想。后来的 Web、Wiki、超文本系统,都或多或少受 Memex 启发。

### 1.2 Bush 未能解决的问题

Memex 愿景历经 80 年未能真正落地,Web 走向了完全不同的方向——公共性取代私人性、算法推送取代主动策划、链接被视为附属品而非核心价值。

为什么 Memex 路径失败?[[Wiki/Concepts/Methodology/Memex|Memex]] 页总结了 Bush 没解决的核心问题:

> **谁来建立和维护这些关联?**

人工建 link、更新摘要、保持条目一致——这部分 bookkeeping 工作量**会让任何人放弃**。一个人可以坚持做笔记一个月,但很少有人能持续数年维护一个几百页、互相交叉引用、随时修正的知识库。

### 1.3 LLM 解决了这个问题

[[Wiki/Sources/Methodology/Karpathy-llm-wiki|Karpathy 的一句话总结]]:

> **LLM 解决了这个问题——bookkeeping 成本趋近于零。**

LLM 不会厌倦、不会忘记更新链接、一次 ingest 可以触达 15 个文件、不会因为"要不要建一个新页面"而拖延。**Memex 愿景在 LLM 时代第一次具备可行性**。

这是本仓库存在的**最根本动机**。

### 1.4 小结

- Memex(1945)= 个人主动策划的外脑,关联本身即价值
- 愿景未落地的根本原因 = 人类无法持续承担 bookkeeping
- LLM 的出现 = 这个 80 年的瓶颈被打破
- **本仓库是 Memex 愿景的一次当代兑现**

---

## 2. 当前主流路径:RAG 与它的局限

在 LLM Wiki 方法论登场之前,"LLM + 文档"的主流范式是 RAG(Retrieval-Augmented Generation,检索增强生成)。要理解 Wiki 方法论的价值,必须先看清 RAG 的**结构性局限**。

### 2.1 RAG 怎么工作

比喻成"**开卷考试**":模型脑子里可能没答案,但把书摊在它面前它就能查。

底层机制是 **Embedding(向量嵌入)**:

- 把每段文字转成一个**几百到几千维的向量**
- **意思相近的文字在数字空间里距离也近**
- 检索 = 在向量空间里找"离问题最近的几段文字"
- 找到的片段塞进 prompt,让 LLM 基于片段生成答案

**关键优点**:它找的是**"语义相关"**内容,不是关键词匹配——"帮我找关于猫的段落"能同时找到 "猫"、"喵星人"、"tabby"、"家养宠物"。这是 RAG 比传统搜索强的核心。

产品例子:NotebookLM、ChatGPT 文件上传、大多数企业知识库。

### 2.2 RAG 的结构性局限

Karpathy 认为 RAG 有一个**根本缺陷**——不是工程问题,是范式问题:

> **LLM 每次查询都在从零发现知识,没有任何积累。**

具体表现([[Wiki/Concepts/Methodology/Rag|RAG 概念页]]列举):

| 缺陷 | 意味着什么 |
|---|---|
| **无状态** | 两次查询之间没有任何"理解"被持久化 |
| **合成靠当场** | 需要综合 5 篇材料的细微问题,每次都现场拼碎片 |
| **不处理矛盾** | 不同源之间的冲突不会被显式记录,只会在某次回答里碰巧浮现 |
| **查询延迟高** | 每次都要"检索 + 综合",没有预先编译好的答案 |

**打一个类比**:RAG 就像一个每次考试都现场查书的学生——考得多了不会"学会",只是越来越熟练地翻书。

### 2.3 RAG 对抗幻觉的价值

公平地说,RAG **不是一无是处**——它对付 [[Wiki/Concepts/AIFoundations/Hallucination|幻觉]]有奇效。AI 有"开卷"可抄,胡编空间就变小。这也是为什么 2024-2026 年绝大多数"可信 AI 应用"都建立在 RAG 上。

但这种"可信"是**一次性的**:回答完忘掉,下次同类问题再做一遍。对于需要**长期深耕、持续积累**的场景(研究、学习路径、持续项目),RAG 的复用率极低。

### 2.4 RAG vs LLM Wiki 完整对照

| 维度 | RAG | LLM Wiki |
|---|---|---|
| 知识积累 | **无** | **持续复利** |
| 交叉引用 | 查询时再找 | ingest 时已建立 |
| 矛盾标注 | 无 | 显式记录 |
| 合成 | 每次重做 | 一次编译,长期更新 |
| 维护成本 | 几乎为 0 | LLM 承担 bookkeeping |
| 查询延迟 | 较高 | 较低(答案已在 wiki) |
| 适合场景 | 一次性、广而浅 | 长期深耕、需要累积 |

### 2.5 两者并非互斥

要提前澄清一个常见误解——**LLM Wiki 不是在取代 RAG**,而是在其之上。

[[Wiki/Sources/Methodology/Karpathy-llm-wiki|Karpathy 原文]]明确提到:wiki 长到需要时,底层可以挂一个本地 markdown 搜索引擎(例如 `qmd`)作为快速检索层——**那一层本质上就是 RAG**。区别在于:**RAG 是检索基础设施,Wiki 是知识编译层**,后者站在前者之上,不是前者的敌人。

### 2.6 小结

- RAG = 查询时检索,开卷考试,无积累
- 它的结构性缺陷:无状态、合成靠当场、不处理矛盾、维护成本低(但知识复用也低)
- 它有真实价值(对抗幻觉、即开即用)但不适合长期项目
- LLM Wiki 与 RAG 可以共存,Wiki 在上,RAG 在下

---

## 3. LLM Wiki 方法论:Karpathy 的破局

### 3.1 一句话定义

[[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]的核心定义:

> 让 LLM 增量构建并持续维护一个持久化、互联的 markdown 知识库,而不是每次查询时从零检索——**知识是"编译一次,持续保鲜"**,而不是在每次提问时重新推导。

"编译一次,持续保鲜" 是它与 RAG 最直白的对比:
- RAG = 解释器(每次查询现场执行)
- Wiki = 编译器(ingest 时一次编译,查询时读结果)

### 3.2 三层架构(Three Layers)

方法论要求任何一个 LLM Wiki 仓库都有**清晰的三层结构**:

| 层 | 路径例子 | 所有者 | 可变性 |
|---|---|---|---|
| **Raw sources** | `Raw/` | 人类 | **只读**——LLM 绝不修改 |
| **Wiki** | `Wiki/` | LLM | LLM 完全拥有,持续更新 |
| **Schema** | `CLAUDE.md` / `AGENTS.md` | 人 + LLM 共同演进 | 谨慎修改 |

**为什么这么分**:

- **Raw 只读**保证真相可追溯——任何 wiki 页面都能回溯到未经 LLM 处理的原始文字,杜绝 AI 层层转写失真
- **Wiki 完全交给 LLM**,因为只有去掉"谁碰它"的权限冲突,LLM 才敢大规模改写、跨页更新
- **Schema 可演进**因为方法论本身会随着实践成熟——本仓库的 `CLAUDE.md` 已经迭代了多次(§3.4 "主题读本" 规则就是 2026-04-19 之后新增的)

### 3.3 三种核心操作

方法论要求 LLM 掌握三种操作,且**每种都有明确的 bookkeeping 责任**:

#### 3.3.1 Ingest(摄入新源)

当人类把一个文件放进 `Raw/` 并说"帮我吸收这个",LLM 做:

1. **读源**:通读原文
2. **讨论要点**:和人类对话,确认关键信息、人类想强调什么
3. **建立源页**:在 `Wiki/Sources/<topic>/` 写一页摘要
4. **更新受影响的实体/概念页**:源里提到的每个实体/概念,找对应页面
   - 已有 → 增补、修订、标注矛盾
   - 没有但值得 → 新建
5. **更新 `index.md`**:登记新页面
6. **更新 `Overview.md`**(如有重大影响)
7. **追加 `log.md`**:一条 ingest 记录

一次 ingest 通常触及 **5-15 个 wiki 页面**。原文明说:**这是特性,不是 bug**——这就是为什么 wiki 比 RAG 强。

#### 3.3.2 Query(提问)

当人类问问题:

1. 先读 `index.md` 找相关页
2. 深入相关页面,必要时回溯到 `Raw/` 查原文
3. 给出带引用的答案(wikilink 形式)
4. **好答案要能沉淀**:如果答案有长期价值(对比、分析、新发现),归档为 `Wiki/Syntheses/<topic>/xxx.md`
5. 视情况追加 `log.md` 一条 query 记录

**"好答案要沉淀"是复利的关键**——一次探索的结果被保留,下次同类问题已经有现成答案。这是 RAG 永远做不到的。

#### 3.3.3 Lint(体检)

人类说"帮我 lint 一下 wiki"时:

- 找出**相互矛盾的陈述**(列出位置)
- 找出**被新源推翻的旧陈述**
- 找出**孤儿页**(没有任何页面链入)
- 找出**多次被提到但还没有独立页面**的概念
- 找出**缺失的交叉引用**
- 建议:下一步可以补哪些数据、问哪些问题、找哪些资料

Lint 定期进行,保证知识库**健康度**,避免熵增。

### 3.4 两个特殊文件

方法论规定每个 LLM Wiki 仓库必有两个特殊文件:

#### `index.md` — 内容目录

- **Content-oriented**(按类别列)
- 每次 ingest 都由 LLM 更新
- 人类(和 LLM 自己)查询时**先读这里**,再深入相关页面

#### `log.md` — 时间线

- **Chronological**(追加式,不修改历史)
- 条目用一致前缀:`## [YYYY-MM-DD] op | title`(op ∈ ingest / query / lint / synthesis / refactor)
- 这样 **unix 工具**就能回溯:`grep "^## \[" log.md | tail -10`

两个文件互补:index 回答"**有什么**",log 回答"**发生过什么**"。

### 3.5 哲学:人与 LLM 的分工

方法论最核心的哲学是**分工明确**:

| 角色 | 职责 |
|---|---|
| **人类** | 策源、提问、指方向、思考意义 |
| **LLM** | 其他所有事——读、写、归档、交叉引用、保持一致 |

理由原文说得很直白:

> 知识库的瓶颈从来不是读或想,而是 **bookkeeping**。人会厌倦、会忘;LLM 不会。

一旦接受这个分工,之前"维护知识库是美好理想,终究会放弃"的宿命就被打破——**厌倦的那部分被外包给一个永不厌倦的实体**。

### 3.6 一个关键隐喻

[[Wiki/Sources/Methodology/Karpathy-llm-wiki|原文]]里有一个非常精准的隐喻:

> **Obsidian 是 IDE,LLM 是程序员,wiki 是 codebase。**

这个类比帮你理解一切:
- 你(人类)是产品经理,决定"我们要写什么"
- LLM 是工程师,负责"怎么写、维护一致性、跑测试(lint)"
- Obsidian 是工作环境,提供图谱视图、wiki 链接、搜索等"IDE 级"工具
- wiki 本身是被维护的产品,有版本(git),有架构(CLAUDE.md),有测试(lint)

### 3.7 重要澄清:刻意抽象

> [!warning] 方法论只规定骨架,不规定实现
> Karpathy 原文明确声明:
>
> > 此文件故意抽象,只描述 pattern,具体实现由你和你的 LLM 共同决定。

这意味着方法论**只规定骨架**——具体的:

- 目录如何分类(`Entities / Concepts / Sources / Syntheses` 是示范,不是强制)
- frontmatter 有哪些字段(示范,可增可减)
- log 条目格式(给了前缀约定,细节自定)
- 怎么命名文件(给了示例,自行选定)

都由每个 wiki 的拥有者和他的 LLM **共同演进**。这就是为什么本仓库的 `CLAUDE.md` 会越来越厚——它承载的是**本仓库特有的约定**。

### 3.8 适用场景

原文列举的可行场景极广:

- **个人成长**:学习某领域、建立"我自己的教科书"
- **研究专题**:论文综述、某方向长期跟踪
- **读书**:一套书的交叉对比与综合
- **团队内部 wiki**:代替 Confluence,由 AI 辅助维护
- **竞品分析**:持续跟踪竞品动向
- **旅行规划**:目的地信息聚合
- **课程笔记**:多门相关课程的交叉整合

共同特征:**需要长期、深度、跨源累积**的场景。不适合一次性问答。

### 3.9 小结

- 方法论 = 三层架构(Raw / Wiki / Schema)+ 三种操作(Ingest / Query / Lint)+ 两特殊文件(index / log)+ 人机分工哲学
- 关键创新:把 bookkeeping 外包给 LLM,让 Memex 愿景终于可行
- 刻意抽象:方法论只给骨架,具体规则由每个仓库自定

---

## 4. Karpathy — 方法论的提出者,以及更大的脉络

### 4.1 他是谁

[[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] 以几件事闻名:

- **AI 教育**:以神经网络 from-scratch 课程、LLM 实用观点著称(micrograd、nanoGPT 等)
- **履历**:OpenAI 创始团队成员;Tesla AI 前总监;2024 年离开 Tesla,独立做 AI 教育(Eureka Labs 等)
- **风格**:给 pattern 不给方案——"此文件故意抽象,具体实现由你和你的 LLM 共同决定" 是他一贯的写作风格

### 4.2 他在 AI 方法论演进里的三个贡献

Karpathy 不仅提出了 LLM Wiki 方法论,他还在更宏观的 **AI 应用方法论演进** 中留下了三个关键节点——这三件事互相呼应,反映一个共同的判断:**AI 时代的真正瓶颈不是模型能力,而是"怎么让 AI 帮你"的方法论**。

#### 4.2.1 LLM Wiki 方法论(2026-04)

本文讨论的主题。具体见 §3。

#### 4.2.2 Context Engineering(2025)

Karpathy 公开指出:

> **Context Engineering > Prompt Engineering**

意思是:你给 AI 的不该只是**一句话的指令**,而是一个**完整的信息环境**——相关资料、历史对话、工具、文件、约束。这个判断把从业者焦点从"怎么措辞"转到"怎么搭建 AI 的工作环境",直接催生了 Harness Engineering 这个大主题(参见 [[Wiki/Syntheses/AIFoundations/Prompt-context-harness-evolution|三段论]])。

**LLM Wiki 方法论可以看作 Context Engineering 的一个具体应用**——wiki 就是给 AI 提前编译好的 context,查询时直接用。

#### 4.2.3 Vibe Coding 命名(2025-02)

Karpathy 在推文中首次命名:

> There's a new kind of coding I call 'vibe coding'.

指的是:**你主导决策、AI 负责执行**的编程范式。你凭直觉("vibe")描述想要什么,让 AI 把它实现出来,必要时你指向纠偏。这个术语被 Y Combinator 等机构大力推广,成为 2025-2026 年 AI 编程生态的标签之一。

### 4.3 为什么他的方法论值得采纳

不只是因为"他是名人"。更重要的理由:

- **一致的底层判断**:三个贡献共享一个底层判断——AI 时代人的角色是"决策者/策源者",AI 的角色是"执行者/维护者"。LLM Wiki 是这个判断在"个人知识管理"这个场景的具体化
- **实用性 + 抽象性的平衡**:Karpathy 既给出可立即用的操作清单(ingest/query/lint 三件套),又保持方法论抽象到每人可适配——这是好方法论的标志
- **给 pattern 不给方案**:这让方法论**可演进**——本仓库的 `CLAUDE.md` 至今已迭代多次,没有被原文的具体结构卡住过

### 4.4 事实追溯提示

> [!warning] 已知的事实不确定点
> 关于 Karpathy 的具体引用,[[Wiki/Entities/Methodology/Karpathy|Karpathy entity 页]]诚实标注:
>
> - **Vibe Coding 推文原文**:v2 源中是间接转述,如需精确引用需另行追溯推特原帖
> - **Context Engineering 具体发言**:类似情况
> - **他对 Harness Engineering 的公开看法**:暂未查到
>
> 这些是已知的事实不确定点——需要精确引用时,回到推特/博客一手资料,不要依赖本 wiki 的二手转述。

### 4.5 小结

- Karpathy = 研究者 + 教育者 + 方法论倡导者
- 三个里程碑:LLM Wiki(2026)/ Context Engineering(2025)/ Vibe Coding 命名(2025-02)
- 共同底层判断:**AI 时代人的角色是决策者,AI 是执行者与维护者**
- LLM Wiki 是这个判断在知识管理领域的落地

---

## 5. 本仓库的具体化:从方法论到 `CLAUDE.md`

理论讲完了,现在看本仓库(`D:\Notes`)如何把这套**刻意抽象**的方法论**落地为可执行规则**。

### 5.1 本仓库的三层

按方法论 §3.2 的三层架构映射:

| 层 | 本仓库路径 | 说明 |
|---|---|---|
| **Raw** | `Raw/Articles/`、`Raw/Notes/`、`Raw/Papers/`、`Raw/Books/`、`Raw/Assets/` | 按源类型分子目录 |
| **Wiki** | `Wiki/Concepts/`、`Wiki/Entities/`、`Wiki/Sources/`、`Wiki/Syntheses/` | 按页面类型分子目录;每类下按**主题**建子文件夹(`Methodology/` `UE4/` `Niagara/` `AIFoundations/` 等) |
| **Schema** | `CLAUDE.md`、`code-roots.local.md`(不同步) | `CLAUDE.md` 是主 schema,本文件描述的方法论具体化 |

特殊文件:

- `index.md`(仓库根)— 内容目录
- `log.md`(仓库根)— 时间线

**关键特色**(本仓库独有的扩展):

1. **代码仓库纳管**(§9):本仓库纳管多个 UE 代码仓(stock / project-*),代码**不复制进 Raw**,而是从本机磁盘直接读,通过 `code-roots.local.md`(本机专用、不同步)维护绝对路径。这解决了"代码源太大不适合放 Raw,但仍想被 wiki 纳管"的问题
2. **主题子目录**:`Entities/` 和 `Sources/` 下的 `Stock/` / `Project/` 子目录专门放代码实体/代码源摘要;其他主题(方法论、AIFoundations 等)放 `Methodology/` 等子目录

### 5.2 `CLAUDE.md` 的角色:可执行的方法论

原方法论只定义了**三操作**的框架(ingest / query / lint 各做什么)。`CLAUDE.md` 把这些框架**翻译成具体步骤**:

- **§3.1 Ingest** 的 7 步清单(读源 → 讨论要点 → 建源页 → 更新实体/概念页 → 更新 index → 更新 Overview → 追加 log)
- **§3.2 Query** 的 5 步清单(查 index → 深入 → 回答 → 归档 synthesis → 视情况追加 log)
- **§3.3 Lint** 的检查项清单
- **§3.4 主题读本**(本仓库特有,4-19 加入)— 规定**任何议题都要产出一份"读本"**作为人类线性阅读的首选入口。这是本读本正在兑现的规则

其他章节:

- **§2 目录说明** — 具体的目录约定
- **§4 页面格式约定** — frontmatter、实体/概念/源/读本的模板
- **§5 log.md 条目格式** — `## [YYYY-MM-DD] op | title` 的具体形式
- **§6 链接 & 命名约定** — wikilink 格式、文件名风格
- **§7 Don'ts** — 绝不修改 Raw / 不凭空编造 / 不偷偷覆盖矛盾 / 不跳过 bookkeeping / 不把 CLAUDE.md 当死规则
- **§8 工作流提示** — 给人类的使用指南
- **§9 Code Roots** — 多代码仓纳管规则

### 5.3 演进历史(从 log.md 里可回溯)

`CLAUDE.md` 不是一次写死的。从 `log.md` 可以看到它的演进:

- **2026-04-17 bootstrap**:仓库初始化,第一版 CLAUDE.md
- **2026-04-18 refactor**:顶层 raw→Raw、wiki→Wiki 大写化 + 按主题分子目录
- **2026-04-19 synthesis | Phase 1 导读 + 方法论升级**:新增 §3.4 "Phase 导读" 规则
- **2026-04-19 refactor | 导读 → 主题读本,规则泛化**:把 §3.4 从"只限学习路径 Phase"泛化到所有议题

**每次 schema 演进都有一条 log 条目**——`CLAUDE.md` 本身就是这套方法论的**自举产物**。

### 5.4 `CLAUDE.md` 与原 idea file 的关系

[[Wiki/Sources/Methodology/Karpathy-llm-wiki|源页]]明确:

> 本源即第一源:这是整个 wiki 的"基础源文件",定义了本仓库所有后续操作的规则。

而 `CLAUDE.md` 是:

> 本仓库对此方法论的**具体化实现**。

时间线:

```
Raw/Notes/Karpathy Wiki 方法论.md   ← 从 Karpathy 发布的 idea file 拷贝
    │
    │ ingest(本仓库的第一次 ingest,见 [2026-04-17] log)
    ▼
Wiki/Sources/Methodology/Karpathy-llm-wiki.md  ← 源摘要
Wiki/Concepts/Methodology/Llm-wiki-方法论.md   ← 核心概念页
Wiki/Concepts/Methodology/Rag.md               ← 对比
Wiki/Concepts/Methodology/Memex.md             ← 思想源流
Wiki/Entities/Methodology/Karpathy.md          ← 提出者
    │
    │ 持续迭代
    ▼
CLAUDE.md  ← 具体化的 schema,持续演进
```

**自举式 ingest**:用这套方法论本身 ingest 这套方法论——这是 2026-04-17 log 条目里原话提到的一个关键点。

### 5.5 小结

- 本仓库三层 = Raw/Wiki/CLAUDE.md
- CLAUDE.md = 方法论抽象骨架 + 本仓库特有规则(代码仓纳管、主题读本、主题子目录等)
- CLAUDE.md 本身持续演进,每次变动有 log 可追溯
- **本仓库是方法论的自举产物** —— 用方法论 ingest 方法论本身

---

## 6. 全景回看:一条完整链路

把前 5 节串起来,用**一个完整的例子**演示方法论如何运转——以本仓库最早的一次 ingest 为例。

### 场景:2026-04-17,第一次 ingest

#### 输入

- 人类(Eureka)把 Karpathy 的 idea file 放进 `Raw/Notes/Karpathy Wiki 方法论.md`
- 说:"帮我 ingest 这个"

#### LLM(Claudian)执行

**1. 读源** — 读 `Raw/Notes/Karpathy Wiki 方法论.md` 全文

**2. 讨论要点** — 与 Eureka 确认:
- 这个 idea file 的核心是三层架构 + 三种操作
- 要特别强调"给 pattern 不给方案"的风格
- Eureka 希望本仓库严格遵循原文,同时允许扩展

**3. 建立源页** — 写 `Wiki/Sources/Methodology/Karpathy-llm-wiki.md`,按 §4.3 模板(源路径、作者、日期、摘要、关键要点、涉及实体/概念)

**4. 更新受影响的概念/实体** — 源里提到的:
- 新建 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]
- 新建 [[Wiki/Concepts/Methodology/Rag|RAG]](作为对比)
- 新建 [[Wiki/Concepts/Methodology/Memex|Memex]](思想源流)
- 新建 [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]](作者)

**5. 更新 index** — 在 `index.md` 的 Entities / Concepts / Sources 分区登记新建的 5 个页面

**6. 更新 Overview** — 这是第一次 ingest,Overview 首次有实质内容,确立"当前主题 = 方法论自举"

**7. 追加 log** — 写入一条 `[2026-04-17] ingest | Karpathy — LLM Wiki` 记录,列出新建的 5 个文件、要点、下一步

同时触发的 schema 演进:

- 修正 `CLAUDE.md` 里的 wikilink 指向 `Raw/Notes`(初版可能写错了)

#### 产出

**一次 ingest 触及 7 个文件 + 1 个 schema 修订**。这正是方法论 §3.1 里"一次 ingest 通常触及 5-15 个文件,这是特性不是 bug"的完美示例。

之后每次用 wiki,都可以从这 7 个文件的任何一个进入,交叉导航到其他相关页面——**知识网络从此开始复利**。

### 叙事要点回顾

| 节 | 主线 |
|---|---|
| §1 | **为什么需要**:Memex 愿景未落地 80 年,LLM 把 bookkeeping 瓶颈打破 |
| §2 | **与主流对比**:RAG 是解释器,每次重做;Wiki 是编译器,一次编译 |
| §3 | **方法论骨架**:三层 / 三操作 / 两文件 / 人机分工哲学 |
| §4 | **提出者背景**:Karpathy 在 AI 方法论里的三个节点,判断一致 |
| §5 | **本仓库实现**:CLAUDE.md 把抽象方法论落地为可执行规则 |
| §6 | **完整示例**:第一次 ingest 展示方法论如何运转 |

---

## 7. 八条关键洞察(带走)

1. **Memex(1945)vs LLM Wiki(2026)的唯一核心差别**:谁来维护。前者没解决,后者靠 LLM 解决。两者共享"私人、主动策划、关联优先"的哲学
2. **RAG 的根本缺陷不是技术的,是范式的**:它每次从零发现知识,没有积累。适合广而浅,不适合长期深耕
3. **LLM Wiki 不取代 RAG,而是站在其上**:RAG 是底层检索,Wiki 是知识编译层。两者可共存
4. **三层架构(Raw / Wiki / Schema)的真正威胁来自混淆**:改 Raw = 失去真相追溯;LLM 害怕碰 Wiki = bookkeeping 失灵;Schema 固化 = 方法论无法演进。边界必须清晰
5. **一次 ingest 触及 5-15 个文件是特性不是 bug**——这正是 wiki 比 RAG 强的根源。交叉引用、矛盾标注、跨页更新都在 ingest 时一次完成,查询时直接用
6. **好答案要沉淀回 `Syntheses/`**:让查询的产出也复利。本读本(以及所有其他读本)本身就是这条原则的产物——它们不是给 LLM 看的,是给人类**一次读完**的专题综合
7. **人机分工的底层判断**:人类做决策/策源/思考意义,LLM 做全部 bookkeeping。接受这个分工 = Memex 愿景终于可行
8. **方法论刻意抽象**:Karpathy 只给 pattern 不给方案,让每个仓库的 CLAUDE.md 自行演进。本仓库的 CLAUDE.md 从 2026-04-17 到今天已迭代多次,演进过程本身就证明这种"开放式 schema" 的价值

---

## 8. 议题遗留的问题(开放问题)

以下问题目前尚无定论,随着本仓库规模扩大或实践深入会逐步明晰:

### 8.1 规模边界

Karpathy 原文提到:index.md 在 **"~100 源、数百页"** 规模下够用,**超过后需要搜索引擎**(原文示范 `qmd`——本地 markdown 搜索引擎,本质上是个 RAG 层)。

本仓库当前规模(2026-04-20):
- Source 页:~10 篇
- Concept 页:~20 篇
- Entity 页:~15 篇
- Synthesis(含本读本):~7 篇
- 合计:**~50 页**

**距离需要引入 RAG 底层还有余量**。当 index.md 读起来开始吃力时(目测 150 页左右),就该考虑 qmd 或类似方案。

### 8.2 多人协作

原文提到"团队内部 wiki"可行,但**冲突解决、review 流程**未展开。若本仓库未来要多人协作,需要回答:

- 两人同时 ingest 同一批 raw 时,如何避免撞车
- 对 schema(`CLAUDE.md`)的修改是否需要 review
- 有不同 LLM(Claudian / GPT / Gemini)维护不同子树时,如何保证风格一致

### 8.3 跨仓库引用

不同主题(方法论 / Niagara / AIFoundations)是否该**分开建独立 wiki**,还是**一个大 wiki 用 tag 区分**?

本仓库当前走"一个大 wiki + 主题子目录"路线。优点:跨主题联系(如 Karpathy 同时出现在 Methodology 和 AIFoundations)自然落地。缺点:单个 wiki 规模触顶更快。

### 8.4 Embedding 模型选型

若未来引入 RAG 作为底层,**Embedding 模型**对效果影响很大(OpenAI text-embedding-3、BGE、Jina 等)。本 wiki 暂未展开,待真正需要时再选型。

### 8.5 Bush 原文 *As We May Think* 入驻

[[Wiki/Concepts/Methodology/Memex]] 目前仅以二手转述形式存在(转述自 Karpathy 原文对 Memex 的一句话总结)。若深入研究知识管理史,应把 Bush 1945 年原文 ingest 为 Raw source 并建独立摘要页。

### 8.6 Karpathy 的一手材料

[[Wiki/Entities/Methodology/Karpathy]] 目前依赖的**Vibe Coding 推文原文、Context Engineering 具体发言**都是二手转述。精确引用时需追溯推特原帖。他对 **Harness Engineering** 的公开看法暂未查到,可持续关注。

---

## 9. 下一步预告

本读本是方法论主题的**综合终点**——除非:

1. **新的方法论源**(例如 Karpathy 后续写了 LLM Wiki 2.0)入驻,此时本读本需要增补一节
2. **本仓库规模越过 100 页门槛**,触发"引入 RAG 底层"决策,此时可能需要写一篇《LLM Wiki × RAG 混合架构》的综合读本
3. **发现新的方法论流派**(竞争或补充 LLM Wiki 的)进入视野,可 ingest + 对比

主动可推进的方向:

- **Bush 原文 ingest**(§8.5)
- **Karpathy 推文一手材料 ingest**(§8.6)
- **本仓库 1 年后复盘**:回过头看方法论在实践中哪些规则好用、哪些经常打补丁,产出 《LLM Wiki 方法论实践一年复盘》

---

## 深入阅读

### 方法论的原子页(构成本读本的五篇原子)

- [[Wiki/Concepts/Methodology/Llm-wiki-方法论]] — LLM Wiki 方法论核心定义
- [[Wiki/Concepts/Methodology/Rag]] — RAG 对比参照 + Embedding 机制
- [[Wiki/Concepts/Methodology/Memex]] — 思想源流(1945 愿景)
- [[Wiki/Entities/Methodology/Karpathy]] — 提出者背景 + 三个方法论里程碑
- [[Wiki/Sources/Methodology/Karpathy-llm-wiki]] — 本仓库第一份源

### 原始资料

- [[Raw/Notes/Karpathy Wiki 方法论]] — Karpathy 的 idea file(第一份 raw source)

### 本仓库的具体实现

- [[CLAUDE]] — 本仓库的 schema,方法论的具体化规则集
- [[index]] — 本仓库内容目录
- [[log]] — 本仓库时间线

### 跨主题联系

- [[Wiki/Syntheses/AIFoundations/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]] — Karpathy 在 AI 应用方法论演进中的位置,参见该文第 2 阶段
- [[Wiki/Concepts/AIFoundations/Hallucination|幻觉]] — RAG 对抗幻觉的价值背景
- [[Wiki/Concepts/AIFoundations/Mcp|MCP]] — 另一条"让 AI 接触外部数据"路径,和 RAG 形成对照

---

*本读本由 [[Claudian]] 基于方法论议题的 5 个原子页 + Karpathy 原始 idea file 综合生成,2026-04-20。*
