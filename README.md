# ObsidianNotes — Eureka 的 LLM Wiki

> 一个用 **Andrej Karpathy 的 [LLM Wiki 方法论](https://karpathy.ai/)** 运行的个人知识库。
> 人类负责提供原始资料、提问、指方向；LLM（主力是 Claude）负责读、写、归档、交叉引用、维护一致性。

---
<p align="center">
  <img src="Raw/Assets/ReadmeHeader.jpeg" alt="秧秧镇楼" />
  <br/>
  秧秧镇楼
</p>

---

## 这是什么

一个 [Obsidian](https://obsidian.md/) vault，同时也是一个有 schema 的、由 LLM 持续维护的 markdown 知识库。和常规"笔记仓库"不同的是：**原始资料、结构化知识、维护规则严格三层分离**，LLM 负责在三者之间搬运和整合。

### 顶层架构

```
┌─────────────────────────────────────────────────────────┐
│  Schema     CLAUDE.md（每轮对话常驻的核心规则）         │
│             + Wiki/_templates/（按需读取的 schema 参考: │
│               页面模板 + 操作 checklist）               │
│             人 + LLM 共同维护,分层设计防 context rot    │
├─────────────────────────────────────────────────────────┤
│  Readers/   ⭐ 主题读本（人类阅读首选入口,LLM 拥有）    │
│             每个议题一份"一次读完即掌握"的长文           │
│             与 Wiki 同级独立目录,服务人类线性阅读        │
├─────────────────────────────────────────────────────────┤
│  Wiki/      LLM 完全拥有的原子 markdown 知识库          │
│   ├─ Concepts/    概念页 (原子)                         │
│   ├─ Entities/    实体页 (原子)                         │
│   ├─ Sources/     源摘要 (原子)                         │
│   ├─ Syntheses/   非读本类的跨源专题                    │
│   └─ _templates/  schema 参考(新页模板 + lint checklist) │
├─────────────────────────────────────────────────────────┤
│  Raw/       人类单向投喂的原始资料（LLM 只读）          │
│             文章、论文、笔记、书摘、资产                │
└─────────────────────────────────────────────────────────┘
```

核心主张：**持续整合 > 查询时检索**。
普通 RAG 每次查询都在从零拼碎片；这里的 LLM 在 ingest 那一刻就完成交叉引用、矛盾标注、专题综合——查询时读的是已经被 bookkeeping 过的结果。

**双重服务对象**：

- **LLM 检索 / 字段级查询**走原子页(`Concepts/` `Entities/` `Sources/`)——结构化、精确、LLM 友好
- **人类线性阅读**走主题读本(`Readers/`)——详细、精确、满满当当,打开一页从头读到尾即掌握某议题的全部知识,不用跳来跳去

**Schema 分层设计**:`CLAUDE.md` 只装**每轮对话都要遵守的核心规则**(心智模型、硬约束、操作触发条件、路由);页面模板、操作 checklist 等**条件性使用**的详细内容外放到 `Wiki/_templates/`,对应操作触发时才 Read——降低每轮对话的基础上下文噪声,保持长会话中对硬约束的注意力权重。每次给 schema 加新流程前都做一次"always-apply vs conditional"分层判断(见 `CLAUDE.md` §7 原则 6)。

---

## 入口文件

**如果你是第一次来的人类读者,推荐先挑一份主题读本直接读**:

| 主题 | 读本 |
|---|---|
| 本仓库方法论(想知道"这是什么、为什么存在") | [`Readers/Methodology/从 Memex 到 LLM Wiki.md`](Readers/Methodology/从 Memex 到 LLM Wiki.md) |
| AI 应用生态(LLM / Agent / MCP / Harness / Skills 全貌) | [`Readers/AIApps/AI 应用生态全景 2026.md`](Readers/AIApps/AI 应用生态全景 2026.md) |
| Niagara 源码学习 — 心智模型前置 | [`Readers/Niagara/Phase 0 - 上阵前的四层脑内地图.md`](Readers/Niagara/Phase 0 - 上阵前的四层脑内地图.md) |
| Niagara 源码学习 — Phase 1 资产层 | [`Readers/Niagara/Phase 1 - 从 System 到图源抽象基类.md`](Readers/Niagara/Phase 1 - 从 System 到图源抽象基类.md) |

**如果你是 LLM / 想索引 / 想查具体字段**,从下面这些文件进:

| 文件 | 作用 |
|---|---|
| [**`index.md`**](index.md) | 全量导航目录——所有 wiki 页面按类别列出 |
| [**`Wiki/Overview.md`**](Wiki/Overview.md) | 当前知识综合视图——"此刻对整个仓的最佳理解" |
| [**`CLAUDE.md`**](CLAUDE.md) | 作业规程/schema——LLM 维护本仓的完整操作手册 |
| [**`log.md`**](log.md) | 追加式时间线——每次 ingest / query / lint / synthesis / refactor 的流水账 |

---

## 当前主题（截至 2026-04-21）

本仓目前覆盖 3 个互相独立的主题,每个主题都配套一份主题读本。具体最新状态以 [`Wiki/Overview.md`](Wiki/Overview.md) 为准。

1. **知识库方法论（meta / bootstrap）** — 本仓自身的方法论基础，源自 Karpathy 的 LLM Wiki idea file。📖 [方法论读本](Readers/Methodology/从 Memex 到 LLM Wiki.md)。
2. **Niagara 源码学习（UE 4.26）** — 面向 C++ 零基础学员的 Niagara 插件 11 阶段(Phase 0-10)学习路径,~90 个运行时文件 + 11 份主题读本。**Phase 0-10 ✅ 全部完成 🎉**,并已通过 Phase 5-10 分组 targeted lint 质量基线确认。📖 推荐入口:[Phase 0 心智模型读本](Readers/Niagara/Phase 0 - 上阵前的四层脑内地图.md) / [学习路径总图](Wiki/Syntheses/Niagara/Niagara-learning-path.md)。
3. **AI 应用生态（2026-04 新增）** — Prompt → Context → Harness 三段论为主线的 AI 应用扫盲,面向美术 / 策划 / 管理者等非开发角色。📖 [AI 应用生态读本](Readers/AIApps/AI 应用生态全景 2026.md)。

---

## 四种核心操作

详见 [`CLAUDE.md`](CLAUDE.md) §3。

- **Ingest**（摄入）：把 `Raw/` 下新资料整合进 `Wiki/`，一次通常触及 5-15 个 wiki 页。
- **Query**（提问）：基于 wiki 回答，有长期价值的答案回流为 `Wiki/Syntheses/` 页面。
- **Lint**（体检）：定期扫描矛盾、孤儿页、缺失交叉引用，产出改进清单。
- **Synthesis / Reader**（主题读本）：任何有深度的议题产出一份"一次读完即完整掌握"的长文,归档到 `Readers/<topic>/`。这是本仓区别于普通 LLM Wiki 的关键扩展——见 [`CLAUDE.md`](CLAUDE.md) §3.4。

---

## 代码源（Code Roots）

本仓还纳管多个代码仓库（UE 公版、项目魔改引擎、游戏项目），代码本身**不复制进 `Raw/`**——LLM 按关键词路由到本机磁盘直接读。详见 [`CLAUDE.md`](CLAUDE.md) §9。

> 📌 **多机协作注意**：本仓可能在多台机器间同步，各机器绝对路径不同。路径表写在未同步的 `code-roots.local.md`（已 gitignore），每台机器各自维护。仓库里只有 `code-roots.local.md.example` 或等价模板——拉下来后需要自己照着建一份。

---

## 这**不是**什么

- ❌ 不是一份"即插即用的 Obsidian 模板"。仓内混有个人笔记、他人文章剪藏、UE 引擎源码讨论等，第三方不能直接复用。
- ❌ 不是"公开知识库"。此仓的价值在于**我个人的关注点 + LLM 的持续整合**——结构可借鉴，内容不普适。
- ❌ 不是 Karpathy LLM Wiki 方法论的官方实现。本仓是基于他 idea file 的个人演绎；方法论本身请看 `Raw/Notes/Karpathy Wiki 方法论.md` 或他的原帖。

## 如果你对方法论本身感兴趣

推荐阅读顺序:

1. 📖 [`Readers/Methodology/从 Memex 到 LLM Wiki.md`](Readers/Methodology/从 Memex 到 LLM Wiki.md)——**一次读完即完整掌握**本仓方法论的前因后果(Memex → RAG → LLM Wiki 演进脉络 + 本仓库的具体化实现),推荐作为第一个入口
2. [`CLAUDE.md`](CLAUDE.md)——读本落地为可执行规则的 schema(维护者视角)
3. [`Wiki/Concepts/Methodology/Llm-wiki-方法论.md`](Wiki/Concepts/Methodology/Llm-wiki-方法论.md)——方法论的一页原子总结(适合快速回查)
4. Karpathy 的原文(本仓 `Raw/Notes/Karpathy Wiki 方法论.md`)或社交媒体原始帖子——一手资料

或者直接 fork 本仓作为起手式,清空 `Raw/` 和 `Wiki/` 下的内容,保留 `CLAUDE.md` 改成自己的规则——就可以开始运转。

---

## 杂项

- **编辑器**：Obsidian（vault 根即 repo 根）。也能在 VSCode / 任何 markdown 编辑器里读写。
- **图谱视图**：建议用 Obsidian 图谱看本仓的"形状"——枢纽页和孤儿页一目了然。
- **仓库布局约定**：详见 [`CLAUDE.md`](CLAUDE.md) §2。
- **Licensing**：内容中的第三方材料（剪藏文章、论文、视频转述）版权归原作者；我自己写的内容与 wiki 结构采用 CC BY-NC 4.0。UE 引擎相关讨论仅限学习交流。
