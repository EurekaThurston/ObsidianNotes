# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

## [2026-04-27] ingest | 费曼学徒冬瓜《Ralph + 多智能体协同,让 AI 长时高品质工作》bilibili 视频

- source: [[Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜]] (URL: https://www.bilibili.com/video/BV1t9oZBDENp/, 2026-04-26 发布)
- 触发:用户问"多 agent 怎么实操,有现成工具吗,刷到的 AI 工作流网站是干这个的吗",归档此视频以沉淀答案
- 抓取方式:Claudian WebFetch 仅拿到页面元数据(标题 + 简介),**未拿到完整字幕/逐字稿**——本轮所有 wiki 内容是元数据 + 既有概念外推,raw 与 source 两处均显式标注此局限
- 新建:
  - [[Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜]] — 视频元数据 + 抓取局限说明
  - [[Wiki/Sources/AIFoundations/Ralph-multi-agent-video]] — 源摘要(印证 Multi-agent + Harness,新增 Ralph)
  - [[Wiki/Concepts/AIFoundations/Ralph-pattern]] — 新概念页(while 循环 + 文件接力 / 与多 agent 的时空正交对比 / 4 个常见坑 / 经验库 ↔ Hashimoto 定义)
- 更新:
  - [[Wiki/Concepts/AIFoundations/Multi-agent]]:sources 1→2;补"实操工具栈"小节(A 编码型 / B 可视化 / C SDK 三类边界,显式回答"AI 工作流网站是干这个的吗"); 补"Ralph 模式:正交补充"小节;相关链补 Ralph;来源块补第二条
  - [[Wiki/Concepts/AIFoundations/Harness-engineering]]:sources 1→2;相关链补 Multi-agent + Ralph;来源块补第二条(标注 Ralph 经验库 = Hashimoto 定义最朴素落地)
  - [[index]]:Concepts/AIFoundations 加 Ralph-pattern 一行;Sources/文章笔记 加 Ralph-multi-agent-video 一行;header 更新
- 不动:
  - 既有读本 [[Readers/AIFoundations/为什么上下文有限反而必须切多 Agent]] 暂不更新——单 source 衍生量不够触发(§3.4),后续若用户整理逐字稿再补"实操工具篇"
  - [[Wiki/Overview]] 暂不动——本轮属"既有概念交叉补强 + 1 个新概念",未到顶层综合改动门槛
- 要点(给用户的答案):
  - 视频提的"主+计划+代码+测试"多智能体 = wiki 已有 [[Wiki/Concepts/AIFoundations/Multi-agent]] 的角色实例化
  - 视频实操平台 = Claude Code 命令行 + `.claude/agents/<role>.md`(Claudian 同体系,Agent 工具就是这一机制的入口)
  - 用户刷到的 AI 工作流网站(Dify/Coze/n8n/...)= B 类可视化编排,跟视频"AI 写代码"场景是不同物种,有重叠不等价
  - Ralph(while 循环 + 文件接力)是与多 agent 正交的另一条 harness 路线,实战常组合
- 自验:
  - Glob `Raw/Notes/Ralph + 多智能体协同 - 费曼学徒冬瓜.md` ✓
  - Glob `Wiki/Sources/AIFoundations/Ralph-multi-agent-video.md` ✓
  - Glob `Wiki/Concepts/AIFoundations/Ralph-pattern.md` ✓
- 下一步:用户若要落地实操,推荐起点是装命令行版 Claude Code 在小项目里建 `.claude/agents/{planner,coder,reviewer}.md` 跑一遍

---

## [2026-04-25] refactor | UE4 → UE 改名 + Stock/ 下沉 Niagara 子命名空间

- 触发:用户"把 ue4 改成 ue 吧,然后你刚刚提到的 stock/niagara 也改一下"——两个动作一起做,省一次批量 replace
  - **改名 UE4 → UE**:上一轮 Niagara 下沉时用了 `UE4/`,但该主题其实承载的是"UE 引擎(含 4.x / 未来 5.x)的基础概念与子模块",改 `UE/` 为 future UE5 留空间。文件名 `UE4-uobject-系统.md` 等保留(它们是"UE4 版本的 UObject 系统",名字准确)
  - **Stock/ 下沉 Niagara**:上一轮 log 记为"等第二个模块冒头再做"的 open,这次直接做。消除未来路径不一致的隐患
- 重命名(2 部分):
  - **UE4 → UE**:3 个 topic-keyed 目录整目录 git mv
    - `Wiki/Concepts/UE4/` → `Wiki/Concepts/UE/`(含子目录 `Niagara/` + 3 文件)
    - `Wiki/Syntheses/UE4/` → `Wiki/Syntheses/UE/`(含子目录 `Niagara/` + 1 文件)
    - `Readers/UE4/` → `Readers/UE/`(含子目录 `Niagara/` + 11 文件)
  - **Stock/ → Stock/Niagara/**:120 文件 git mv
    - `Wiki/Entities/Stock/*.md`(53 文件)→ `Wiki/Entities/Stock/Niagara/`
    - `Wiki/Sources/Stock/*.md`(67 文件)→ `Wiki/Sources/Stock/Niagara/`
- 路径批量替换(PowerShell 扫全 vault .md 排除 log.md),**5 个前缀**一次搞定:
  - `Wiki/Concepts/UE4/` → `Wiki/Concepts/UE/`
  - `Wiki/Syntheses/UE4/` → `Wiki/Syntheses/UE/`
  - `Readers/UE4/` → `Readers/UE/`
  - `Wiki/Entities/Stock/` → `Wiki/Entities/Stock/Niagara/`
  - `Wiki/Sources/Stock/` → `Wiki/Sources/Stock/Niagara/`
  - 共 **143 文件更新**(含所有读本相互引用、Phase N↔N+1 串接、stock 代码 entity 的 60+ 交叉引用、AIAgents 新页对 Niagara 读本/entity 的引用、CLAUDE.md §2.1 示例等)
  - 防御 double-subnamespace:末尾强制收敛 `Stock/Niagara/Niagara/` → `Stock/Niagara/`(未触发,仅保险)
- 节标题改名(不被路径 replace 覆盖的部分):
  - [[index]]:`### UE4 基础` → `### UE 基础`;`### UE4 / Niagara 基础` → `### UE / Niagara 基础`;`### Niagara 代码实体(stock / UE 4.26)` → `### UE / Niagara 代码实体(repo: stock / UE 4.26,module: Niagara)`;`### 代码源摘要 — stock(UE 4.26 Niagara)` → `### UE / Niagara 代码源摘要(repo: stock / UE 4.26,module: Niagara)`;2 处 `### UE4 / Niagara 源码学习` → `### UE / Niagara 源码学习`;header 更新
  - [[Wiki/Overview]]:`### 2. UE4 / Niagara 源码学习` → `### 2. UE / Niagara 源码学习`;知识图 UE4/ → UE/;header 更新;Stock 代码实体分支加"(按模块下沉 Stock/Niagara/)"注脚
  - [[CLAUDE]] §2.1 主题路由表:
    - `UE4/ 及子模块` 行 → `UE/ 及子模块`,示例 `UE/Niagara/`、`UE/Material/`(future)
    - `Entities/Stock/、Sources/Stock/` 行升级为 `Entities/<repo>/<module>/、Sources/<repo>/<module>/`,显式点明 **repo + module 两级**
    - 拆分先例末尾条改写为 `Niagara/ → UE/Niagara/`(2026-04-25),注明同期 `UE4→UE` + `Stock/→Stock/Niagara/` 配套操作
  - [[CLAUDE]] §9.5 ingest 流程:
    - 原 `Wiki/Sources/<repo>/<文件名>.md` → `Wiki/Sources/<repo>/<module>/<文件名>.md`(+module 子命名空间说明)
    - 原 `Wiki/Entities/<repo>/<类名或模块名>.md` → `Wiki/Entities/<repo>/<module>/<类名>.md`
- 不改的:
  - 文件名 `UE4-uobject-系统.md` 等保留("UE4 版本的 X" 命名准确)
  - 所有 `UE 4.26` 版本引用保留(这是具体引擎版本,不是主题名)
  - `UE4.sln` 文件名引用保留(Unreal 自带的解决方案文件名)
  - frontmatter 字段 `source_root: UE-4.26-Stock` / `repo: stock` 等保留
  - log.md 历史条目保留旧 `UE4/` / `Stock/` 叙述(append-only §3.1)
- 自验:
  - Grep `Wiki/(Concepts|Syntheses)/UE4/` 或 `Readers/UE4/` → 仅 log.md 历史命中 ✓
  - Grep `\[\[Wiki/Entities/Stock/(?!Niagara/)` 和 `\[\[Wiki/Sources/Stock/(?!Niagara/)` → 0 文件命中(所有引用都已带 Niagara 中段)✓
  - `find -maxdepth 5 -type d` 确认新结构:
    - `Wiki/Concepts/UE/Niagara/`、`Wiki/Syntheses/UE/Niagara/`、`Readers/UE/Niagara/` 存在
    - `Wiki/Entities/Stock/Niagara/`、`Wiki/Sources/Stock/Niagara/` 存在(120 文件各就各位)
    - 旧 `UE4/` 目录全部已清理
- 意义:至此 4 轮 refactor(AIApps→AIFoundations / Niagara 下沉 UE / UE4→UE / Stock 模块下沉)把"topic 两级(`UE/Niagara/`)+ repo 两级(`Stock/Niagara/`)"的路径规范全部打通,未来加 Material / 加 UE5 / 加 project-engine Niagara 魔改均有现成位置

---

## [2026-04-25] refactor | Niagara 下沉为 UE4 子主题(topic 维度)

- 触发:用户"把 Niagara 移到 UE4 下吧,UE4/Niagara/各种文档"——反映已有事实:Niagara 是 UE4 引擎的一个模块,放作顶层主题在概念上矮了一级;且未来若涉及其他 UE4 模块(Material / Animation 等),topic 分层不得不做
- 范围决定:**只移 topic-keyed 目录**(Concepts / Syntheses / Readers 下的 `Niagara/`);**不动 repo-keyed** 的 `Wiki/Entities/Stock/` 与 `Wiki/Sources/Stock/` ——按 [[CLAUDE]] §9 约定,repo(stock / project-engine / project-game)是代码实体的顶层 key,与 topic(UE4 / UE4/Niagara)是正交两轴。Niagara 源码 entity 依然在 `Entities/Stock/<Class>.md`,但对应读本在 `Readers/UE4/Niagara/<...>.md`——两条路径各自其是,不重叠
- 重命名(3 子目录 → 嵌入 UE4 下,14 文件 `git mv` 保留历史):
  - `Wiki/Concepts/Niagara/` → `Wiki/Concepts/UE4/Niagara/`(2 文件:Niagara-vs-cascade / Niagara-cpu-vs-gpu模拟)
  - `Wiki/Syntheses/Niagara/` → `Wiki/Syntheses/UE4/Niagara/`(1 文件:Niagara-learning-path)
  - `Readers/Niagara/` → `Readers/UE4/Niagara/`(11 文件:Phase 0-10 读本全系)
- 路径批量替换:PowerShell 扫全 vault `.md`(排除 log.md),3 个路径前缀替换:
  - `Wiki/Concepts/Niagara/` → `Wiki/Concepts/UE4/Niagara/`
  - `Wiki/Syntheses/Niagara/` → `Wiki/Syntheses/UE4/Niagara/`
  - `Readers/Niagara/` → `Readers/UE4/Niagara/`
  - 共 **86 文件更新**(含 CLAUDE.md §3.4 模板示例 / Overview.md 知识图 / index.md 登记 / 各 Phase 读本交叉引用 / AIAgents 新页引用 Niagara 读本 / Niagara 内部 Phase N+1 ← Phase N 串接链接 等)
- 删除空目录 3 个:`rmdir D:/Notes/Wiki/Concepts/Niagara/`、`Wiki/Syntheses/Niagara/`、`Readers/Niagara/`
- 节标题改名:
  - [[index]] `### Niagara 基础` → `### UE4 / Niagara 基础`;2 处 `### Niagara 源码学习` → `### UE4 / Niagara 源码学习`
  - [[Wiki/Overview]] `### 2. Niagara 源码学习（UE 4.26）` → `### 2. UE4 / Niagara 源码学习（UE 4.26）`
  - [[Wiki/Overview]] 主题知识图 `Niagara 源码学习 (UE 4.26)` 子树改写,注明 topic 维度(UE4/ 与 UE4/Niagara/)与 repo 维度(Stock/)正交
- CLAUDE.md §2.1 主题路由表更新:
  - 原 Niagara 独立行删除,合并进 UE4 行:`| UE4/ 及子模块 | UE4 基础 + UE4/Niagara/(当前) + UE4/Material/ 等(未来) | 非 UE 引擎 |`
  - 新增一行专讲 `Entities/Stock/`、`Sources/Stock/` 的 repo 维度,并强调"与 topic 维度两个轴"
  - "已有的主题拆分先例"末尾加 2026-04-25 Niagara → UE4/Niagara 这条
- **Entities/Stock/ 和 Sources/Stock/ 未改动**(保留 §9 约定;53 + 67 = 120 文件原地保留,所有 wikilink 形如 `[[Wiki/Entities/Stock/UNiagaraSystem]]` 仍有效)
- **未来可选的进一步下沉**(记作 open):若 future 有项目引擎 Material 魔改或其他非 Niagara UE 模块要 ingest,Stock 下应该加模块子命名空间 `Entities/Stock/Niagara/` + `Entities/Stock/Material/` 等 ——本次未做是因为:当前 Stock/ 全部是 Niagara 文件,加 `Niagara/` 子目录改动 120 文件但零现实收益,等真有第二模块冒头再做
- 自验:
  - Grep `(Wiki/Concepts|Wiki/Syntheses|Readers)/Niagara/` 全仓 → 仅 log.md 命中(历史)✓
  - `find -maxdepth 4 -type d -name Niagara` → 只剩 `Readers/UE4/Niagara`、`Wiki/Concepts/UE4/Niagara`、`Wiki/Syntheses/UE4/Niagara` ✓
  - `[[Wiki/Entities/Stock/...]]`、`[[Wiki/Sources/Stock/...]]` 未改动,60+ 读本中的这类链接保持原样 ✓
- 不做的:
  - 未移 Entities/Stock/ 和 Sources/Stock/(§9 repo 维度保留)
  - 未 rename frontmatter 字段(`repo: stock` 等)
  - 未追溯修改 log.md 历史条目(append-only,保留 "Niagara/" 旧路径叙述)

---

## [2026-04-25] refactor | 主题 AIApps → AIFoundations 改名

- 触发:用户反馈"AI Apps 这个名字感觉有点不符合现在我们对它的定位"——重构把项目级落地(代码问答 / 特效贴图)和生成式智能体研究迁去 AIAgents 之后,AIApps 目录实际只剩基础概念(LLM / 幻觉 / Context / Agent / MCP / Harness / Skills)+ 方法论综合(三段论)+ 扫盲(Primer v2)+ 代表产品(OpenClaw)。"应用"已经名不副实
- 候选对比(给出 6 个候选 + 利弊分析):AIFoundations ⭐⭐⭐⭐⭐ / AIPrimer ⭐⭐⭐⭐ / AIStack ⭐⭐⭐ / AIEcosystem ⭐⭐⭐ / AIEngineering ⭐⭐ / AI(平级)⭐⭐
- 用户选定 **AIFoundations**(CamelCase 风格与现有 Methodology / Niagara / AIArt / AIAgents 一致)
- 重命名(5 子目录,18 文件):
  - `Wiki/Concepts/AIApps/` → `Wiki/Concepts/AIFoundations/`(11 文件:Agent-skills / Agentic-grep / Ai-agent / Context-window / Embedding / Hallucination / Harness-engineering / Llm / Mcp / Multi-agent / Reasoning-model)
  - `Wiki/Sources/AIApps/` → `Wiki/Sources/AIFoundations/`(3 文件:AI-primer-v2 / Code-retrieval-conversation / Multi-agent-conversation)
  - `Wiki/Entities/AIApps/` → `Wiki/Entities/AIFoundations/`(1 文件:OpenClaw)
  - `Wiki/Syntheses/AIApps/` → `Wiki/Syntheses/AIFoundations/`(1 文件:Prompt-context-harness-evolution)
  - `Readers/AIApps/` → `Readers/AIFoundations/`(2 文件:AI 应用生态全景 2026 / 为什么上下文有限反而必须切多 Agent)
- 路径批量替换:PowerShell 递归扫全 vault `.md` 文件(排除 log.md),`AIApps` → `AIFoundations` 整词替换 — 共 **40 文件更新**
- 删除空目录 5 个:原 5 个 AIApps 子目录已空,`rmdir` 清理
- 节标题改名:
  - [[index]] 4 处 `### AI 应用生态` → `### AI 基础(AIFoundations)` / `### AI 基础(AIFoundations — 产品 / 项目)`
  - [[Wiki/Overview]] `### 3. AI 应用生态(2026-04 新增)` → `### 3. AI 基础(AIFoundations)(2026-04 新增,2026-04-25 改名)`,加改名备忘 callout
  - [[index]] 头部 "AI 应用生态" → "AI 基础" in 主题并行描述
  - [[Wiki/Overview]] 最后更新行加改名说明
- 历史 log 条目按 §3.1 step 9 "本轮"规则不适用于已归档条目——**保留原 `AIApps` 路径叙述不追溯修改**。共 80 处 `AIApps` 出现仅在 log.md,分布在 2026-04-19 到 2026-04-25 各条 ingest / synthesis / refactor / lint 条目
- 未改的 CLAUDE.md:不含 AIApps 路径硬引用(只讲规则不举例),无需改
- 自验:
  - Grep `AIApps` 全仓 → 仅 log.md 命中(预期)✓
  - `find -type d -name AIApps` → 空 ✓
  - 新路径全部 Glob 通过 ✓
- 主题定位文档化(供后续 ingest 决策,写进本 log 方便回溯):
  - **AIFoundations**(AI 基础):LLM / 幻觉 / Context / Embedding / Reasoning / Agent / Multi-agent / MCP / Harness / Skills / Agentic-grep 等**基础概念 + 方法论**;Primer v2 扫盲源;OpenClaw 作地基产品实例;Prompt → Context → Harness 三段论综合。**不放"具体应用"**
  - **AIAgents**(具体 agent):生成式智能体学术研究线(Smallville / Park 2024 / a16z AI Town)+ 项目级 AI 应用落地(美术代码问答 / 特效贴图工具 / Niagara-Material AI 理解管道)
  - **AIArt**:已有独立主题(LoRA 等视觉生成议题)
  - 未来可能新增:AICoding(Vibe / Spec / Harness coding 系列,若展开)
- 不做的事:
  - **未 rename 内部变量** / frontmatter 字段(只是目录重命名)
  - **未改 tag**:tag `ai`、`agent` 等保持(tag 不绑目录名)
  - **未回收 log 历史**(append-only)
- **元规则升级(同一轮追加)**:用户指出"主题分工写在 log 里每次 ingest LLM 不会主动读",按 [[CLAUDE]] §7.6 元规则判断——主题路由属**每轮要守的路由规则**,应进 CLAUDE.md。新增 [[CLAUDE]] §2.1 "主题路由":5 主题决策表(Methodology / AIFoundations / AIAgents / AIArt / Niagara / UE4)+ 开新主题 vs 归现有的判断规则 + 已有主题拆分先例 + 改名时机指引。简短 5-10 行决策导向,细节清单仍归 index.md

---

## [2026-04-25] synthesis | 从斯坦福小镇到 1000 人数字社会 读本

- 触发:用户显式要求"记得写读本"(§3.4 触发条件 4);本轮同期 ingest 产出 ≥3 原子页 + 新主题 AIAgents 首次入驻,也满足触发条件 2+3
- 新建(1 页):
  - [[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] — 线性读本,8 节叙事(议题问题→2023 前→三件套具体工作→复现经济学→Park 2024 跃迁→a16z 工程化→2026 生态→与工具型 multi-agent 本质区别)+ 全景回看 + 10 条关键洞察 + 8 题自检(综合/反推/取舍/假设,非检索浅题)+ 留下的问题 + 下一步 + 深入阅读 + 签名
- 更新:
  - [[index]] Readers 新增 "### AI Agents" 分区 + header 加本读本登记
  - [[Wiki/Overview]] 主题 4 节 📖 主题读本指向本篇
- 关键叙事选择:
  - 开题不写"Smallville 是什么",而是**"LLM 直接套 agent 会撞哪四堵硬墙"**——让三件套的每一件对应一堵墙,避免"架构罗列"的干涩
  - §3 把"数千美元"成本具体化到**每 agent 每 tick LLM 调用类目 + 每天 22500 次 + 45000 次 × 0.85 美分 × 摊销系数**——把抽象成本拆成可以反推优化的结构,这是读本价值所在
  - §4 强调**评测标准从 believability → 真人 GSS 对照**,比架构变化更重要——方法论转向是 2024 跃迁的本质
  - §5 的 a16z "选择性工程化" 提炼:砍 reflection + 砍规模 / 加多人 + 加持久化——这不是忠实复现,理解这点才能选型
  - §7 专门警戒"multi-agent" 同词异义,和 AIApps 下的 [[Wiki/Concepts/AIApps/Multi-agent]] 做清晰边界——这是本议题在 wiki 内部最重要的一处 guardrail
  - 自检 8 题全部综合/反推/取舍/假设题;特别是第 3 题("$50 预算怎么削减")、第 5 题("哪类政策敢信哪类不敢")、第 8 题("5 句话向投资人讲清 2023→2024 重要性")都是需要串多节的难题
- open questions(读本 §议题留下的问题):
  - 跨文化样本泛化
  - 成本下降下一跳板
  - 数字孪生伦理/法律
  - 25 agent 规模是不是最优
  - 与传统游戏 AI(行为树/GOAP)取代还是共存
  - 反向用 agent 研究 LLM 偏见
- 自验:Glob [[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] ✓

---

## [2026-04-25] ingest | 斯坦福小镇与生成式智能体对话 — Park 2023→2024 三年脉络 + AIAgents 新主题首次入驻

- source: [[Raw/Notes/斯坦福 AI 小镇与生成式智能体对话]](对话归纳,非逐字)
- 触发:用户发来知乎 p/656007815 链接(抓取 403),询问"这篇讲什么 / 斯坦福'筱桢'是谁 / 复现难度 / 最新进展";"筱桢"应为"小镇"的视觉近形字误。定位议题:**斯坦福小镇 / Generative Agents**,跨 2023-2026 脉络梳理
- 决定新开主题目录 `AIAgents/` — 本轮产出 ≥3 原子页且需要和既有 [[Wiki/Concepts/AIApps/Multi-agent]](工具型)做清晰边界,单独主题胜过挤 AIApps
- 新建(10 页):
  - [[Raw/Notes/斯坦福 AI 小镇与生成式智能体对话]] — 原始对话归纳(含 web 搜索整合)
  - [[Wiki/Sources/AIAgents/Generative-agents-discussion]] — 本对话源摘要
  - [[Wiki/Sources/AIAgents/Park-2023-generative-agents]] — arXiv 2304.03442 摘要级源(未逐字读 PDF,基于 abstract + HAI 报道 + 开源 README)
  - [[Wiki/Sources/AIAgents/Park-2024-1000-agents]] — arXiv 2411.10109 摘要级源(基于 HAI 新闻 + 项目主页)
  - [[Wiki/Entities/AIAgents/Joon-sung-park]] — 方向奠基人 entity
  - [[Wiki/Entities/AIAgents/Stanford-smallville]] — 25 agent 沙盘项目 entity
  - [[Wiki/Entities/AIAgents/A16z-ai-town]] — TS + Convex 工业重写 entity
  - [[Wiki/Concepts/AIAgents/Generative-agents-architecture]] — Memory Stream + Reflection + Planning 三件套概念
  - [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]] — 研究范式,对照工具型 multi-agent
- 同步产出读本(见下一条 log):[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]]
- 更新:
  - [[Wiki/Concepts/AIApps/Multi-agent]]、[[Wiki/Concepts/AIApps/Context-window]]——交叉引用 AIAgents 新概念(相关区会在后续 lint 补;本轮不改,避免 2026-04-25 一口气改太多文件)
  - [[Wiki/Overview]] — 从 3 主题升 4 主题,新增 "### 4. AI Agents(2026-04-25 新增)" 节,主题知识图同步加 AIAgents 分枝
  - [[index]] — Entities/Concepts/Sources/Readers/Syntheses 5 分区各加 "### AI Agents" 子分区 + header 更新
- 关键要点(议题核心叙事,不是复述):
  - **三件套 = 解 agent 长期运行四堵硬墙**(遗忘 / 无人格 / 短视 / 记忆污染),不是可选扩展
  - **复现瓶颈在 token 不在代码**:论文级 25 agent × 2 天 ≈ $1000-$5000;国人流传"100 秒 ≈ 6 元 RMB"
  - **Park 2024 跃迁核心是评测标准质变**:从主观 believability → 真人 GSS 对照(85% 匹配 ≈ 真人两周后 test-retest)
  - **a16z AI Town 是选择性工程化**,砍 reflection/规模,加多人/持久化/本地 LLM 默认
  - **"multi-agent" 同词异义**:工具型(上下文隔离派活)vs 模拟型(共生演化观察涌现)—— 本 wiki AIAgents 主题容纳两者,读者要保留警觉
- 关键诚实:
  - Park 2023/2024 两篇论文 PDF **本 wiki 未逐字读过**,源页内容基于 arXiv 摘要 + Stanford HAI 报道 + 开源代码 README + 二级文章整合。如需字段级引用,需后续下载 PDF 单独 ingest
  - 知乎 p/656007815 本体抓取 403,基于 anchor 文字 + 同系列文章(如 p/649244166、p/672157210)的检索还原
- open questions(记在 Source 页 + 读本):
  - 跨文化样本泛化(Park 2024 是美国人数据)
  - 成本下降下一跳板
  - 伦理 / 法律 / 知情同意边界
  - 和传统游戏 AI 的融合 / 取代
  - Park 2023 附录 prompt 模板待单独 ingest
- 自验:Glob 以下 9 新路径,全部存在 ✓
  - Raw/Notes/斯坦福 AI 小镇与生成式智能体对话.md
  - Wiki/Sources/AIAgents/Generative-agents-discussion.md
  - Wiki/Sources/AIAgents/Park-2023-generative-agents.md
  - Wiki/Sources/AIAgents/Park-2024-1000-agents.md
  - Wiki/Entities/AIAgents/Joon-sung-park.md
  - Wiki/Entities/AIAgents/Stanford-smallville.md
  - Wiki/Entities/AIAgents/A16z-ai-town.md
  - Wiki/Concepts/AIAgents/Generative-agents-architecture.md
  - Wiki/Concepts/AIAgents/Agent-based-social-simulation.md

---

## [2026-04-25] refactor | AIApps/ 下 2 份项目级落地 → AIAgents/ 归并

- 触发:用户决定"给美术做代码问答机器人"与"让 AI 接特效贴图的长尾需求"两份"项目级 AI 应用落地"迁至新开的 AIAgents 主题(与斯坦福小镇 / 生成式智能体模拟同主题,共享 agent 第一性原理的警觉讨论);AIApps 保留基础概念定位
- 移位(4 页,`git mv` 保留历史):
  - `Wiki/Syntheses/AIApps/Artist-code-qa-bot.md` → [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]]
  - `Wiki/Syntheses/AIApps/Ai-texture-tool-design.md` → [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]]
  - `Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆.md` → [[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]]
  - `Readers/AIApps/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态.md` → [[Readers/AIAgents/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]]
- 更新链接(PowerShell 一次性脚本,跨 8 文件的 4 路径前缀替换):
  - [[index]] 4 处
  - [[Wiki/Overview]] 2 处(并重构 AIApps 节)
  - [[Wiki/Concepts/AIApps/Agentic-grep]] 2 处
  - [[Wiki/Sources/AIApps/Code-retrieval-conversation]] 3 处
  - [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]] 1 处自引
  - [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design]] 2 处(自引 + 跨引)
  - [[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] 1 处
  - [[Readers/AIAgents/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]] 9 处
- 删除空目录:原 `Readers/AIApps/` 只留"AI 应用生态全景 2026"和"为什么上下文有限反而必须切多 Agent"2 份真属 AIApps 基础的读本 — **不删,保留 AIApps 目录**
- 历史 log 条目(§3.1 step 9 的"本轮"规则不适用于已归档条目)保留原路径叙述,不追溯修改——以下条目的路径在当时属实:
  - `[2026-04-24] refactor | AI 特效贴图工具 TextureTool/ → AIApps/ 归并`
  - `[2026-04-24] synthesis | AI 特效贴图工具主题读本`
  - `[2026-04-24] synthesis | AI 特效贴图工具设计 + GitHub 现成项目调研`
  - `[2026-04-24] ingest | 代码检索与美术问答机器人对话`
- 自验:Grep `Wiki/Syntheses/AIApps/(Artist|Ai-texture)|Readers/AIApps/(给美术|让 AI)` → 仅在 log.md 命中(历史条目,预期) ✓;新路径 4 文件 Glob 全部存在 ✓
- 主题定位备忘(供后续 ingest 决策):
  - **AIApps**:基础概念(LLM / 幻觉 / Context / Agent / MCP / RAG / Harness / Skills / Agentic Grep)+ 产品(OpenClaw)+ 方法论综合(Prompt → Context → Harness 三段论)
  - **AIAgents**:具体的 agent 研究与落地 —— 生成式智能体学术线(Smallville / Park 2024 / a16z AI Town)+ 项目级 AI 应用落地(代码问答 / 特效贴图)
  - 概念页归 AIApps,具体 agent 项目归 AIAgents——分工清晰后,AIApps 下的 Agentic-grep、Multi-agent 等仍作基础概念留守

---

## [2026-04-24] refactor | AI 特效贴图工具 TextureTool/ → AIApps/ 归并

- 触发:用户决定本议题作为"项目级 AI 应用落地"的又一实例,与 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] 同族,不单开一级主题目录
- 移位(2 页):
  - `Wiki/Syntheses/TextureTool/design.md` → [[Wiki/Syntheses/AIApps/Ai-texture-tool-design]](同时按 §6 改名为小写连字符)
  - `Readers/TextureTool/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态.md` → [[Readers/AIApps/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]](文件名不变)
- 删除空目录:`Wiki/Syntheses/TextureTool/`、`Readers/TextureTool/`
- 更新链接:
  - 读本内 2 处 `[[Wiki/Syntheses/TextureTool/design]]` → 新路径
  - 综合页 1 处 dead link `[[Raw/Notes/Ai-texture-tool-design-conversation|本次对话]]`(该 Raw 源从未创建)趁重构修为引用 log 条目
- 更新 [[index]]:
  - Readers "### 特效贴图 AI 工具" 分区合并进 "### AI 应用生态"
  - Syntheses 同上
  - header 更新
- 历史 log 条目(§3.1 step 9 的"本轮"规则不适用于已归档条目)保留原路径叙述,不追溯修改——以下条目的路径在当时属实:
  - `[2026-04-24] synthesis | AI 特效贴图工具主题读本` 内 [[Readers/TextureTool/...]]
  - `[2026-04-24] synthesis | AI 特效贴图工具设计 + GitHub 现成项目调研` 内 [[Wiki/Syntheses/TextureTool/design]]
- 建议:今后"项目级 AI 应用落地"议题(美术/策划/TA 向 AI 工具)继续归 AIApps/,不新开主题目录,保持与 Concepts/Readers/Syntheses 的 AIApps 分区一致
- 自验:Grep `TextureTool` 仅剩 log 历史条目;Glob 新路径 2 文件确认存在

---

## [2026-04-24] synthesis | AI 特效贴图工具主题读本

- 触发:用户显式要求"出个读本"(§3.4 触发条件 4)
- 新建(1 页):
  - [[Readers/TextureTool/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]] — 线性读本,9 节叙事链(问题→AI 解→三层架构→LLM 做路由→VFX 四重数据语义→GitHub 生态分类→POC→风险→全景与同构)+ 8 题自检(综合/反推/取舍/假设,非检索浅题)+ 留下的问题 + 深入阅读索引
- 更新:
  - [[index]] — Readers 新增"特效贴图 AI 工具"分区 + header 更新
- 关键叙事选择:
  - 开题用"美术需求长尾 + 你不擅长 DCC 开发"切入,立刻定问题
  - 反直觉点:"组件齐全,你要写的不到 20%"——避免美术 AI 工具"从零搭"的幻觉
  - 最致命陷阱单独成节(§4):distortion map 喂 Real-ESRGAN 的完整事故链,作为 data texture guardrail 的情感锚点
  - 色彩空间错误命名"最隐蔽"——不报错、只"亮度怪怪的"、半年后才爆雷——这是读本级别需要讲清的长周期坑
  - 全景回看对照 [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]],抽象出"项目级 AI 应用落地通用套路"5 条,提升读本级别的归纳价值
  - 自检 8 题全部综合/反推/取舍/假设题,典型:事故链反推、基座选型取舍、通用套路抽象、5 句话向非技术解释
- open questions(读本 §议题留下的问题):
  - data-aware 超分的学术方案
  - 与 Niagara/Material 集成
  - 批处理规划器
  - ComfyUI vs 自建 UI 的长期取舍
  - 冷启动信任塑造

---

## [2026-04-24] synthesis | AI 特效贴图工具设计 + GitHub 现成项目调研

- 触发:用户提出"想做 AI 特效贴图处理工具,美术需求千奇百怪(缩放/去模糊/四方连续/...),不想每个新需求写代码",显式要求归档设计 + 调研 GitHub 现成项目
- 新建(1 页):
  - [[Wiki/Syntheses/TextureTool/design]] — 综合:三层工具架构(L1 原子/L2 模型/L3 代码沙箱)+ VFX 数据语义 guardrail + GitHub 项目调研(按 agent 框架/Adobe MCP/ComfyUI 生态/四方连续专项分类)+ POC 路径 + 与 Artist-code-qa-bot 对照
- 更新:
  - [[index]] — Syntheses 新增"特效贴图 AI 工具"分区 + header 更新
- 关键发现(GitHub 生态 2026-04):
  - **架构范本**:GenArtist(NeurIPS 2024)用 MLLM 分解子任务树 + 自验证;AgentLego 提供 LLM 视觉工具 API 库
  - **ComfyUI 很可能是正解**:twwch/comfyui-workflow-skill 已经支持"自然语言 → ComfyUI workflow JSON",且作为 Claude Code skill 工作 —— 跟本仓 Claude 生态最合拍
  - **四方连续已成熟**:sagieppel 提供经典 + SD inpaint 双方法;camenduru/seamless、brick2face、dream-textures、Tiled Diffusion(CVPR 2025)各有覆盖
  - **Adobe MCP** 可行但不推荐独占后端:adb-mcp 已验证 Claude Desktop,但 PS 不擅长 VFX 数据贴图
- 关键设计选择:
  - LLM 做路由,不碰像素——与 Artist-code-qa-bot 同构
  - 高频下沉 L1,长尾保持 L3 代码兜底——避免"所有需求都做成工具"的工程灾难
  - VFX 特有 guardrail 写进 system prompt:通道语义 / 位深 / alpha 预乘 / 色彩空间 / tile 验证
  - data texture(flow map / distortion / packed mask) 禁用 AI 超分模型
- 留下的问题(见综合文末):VFX 数据贴图的 data-aware 超分、和 Niagara/Material 工作流衔接、批处理 vs 交互、ComfyUI 作基座 vs 自建 UI 长期取舍
- 未产出读本:本轮仅 1 新原子页,未触发 §3.4 读本产出条件

---

## [2026-04-24] ingest | 代码检索与美术问答机器人对话 — Claudian 的 agentic grep 方法论 + 项目级落地设计

- source: [[Raw/Notes/代码检索与美术问答机器人对话]](对话归纳,非逐字)
- 触发:用户了解到同事用"grep"概念 + Claude Code + OpenClaw 检索 UE 项目源码成功,引出"给美术做代码问答机器人"的项目级 AI 落地方向;用户显式要求"记录到知识库里,放 AIApps 下"
- 新建(4 页):
  - [[Raw/Notes/代码检索与美术问答机器人对话]] — 原始对话归纳
  - [[Wiki/Sources/AIApps/Code-retrieval-conversation]] — 源摘要
  - [[Wiki/Concepts/AIApps/Agentic-grep]] — 新概念:agentic grep 方法论 + 三件套 + vs RAG + 未知 symbol 四破解策略 + 训练数据先验"作弊成分"
  - [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] — 项目级落地设计综合:架构 / 三 persona / 沉淀回路 / 术语桥梁 / 部署风险 / POC
- 同步产出读本(§3.4 触发条件 2 精神满足:本轮 4 原子页,跨多源综合):
  - [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] — 线性读本,Dithered LOD Transition 作贯穿 running example,8 题自检(综合/反推/取舍/假设题,非检索浅题)
- 更新:
  - [[Wiki/Concepts/Methodology/Rag]] — 相关区加 Agentic-grep 对照链接
  - [[Wiki/Overview]] — AI 应用生态区加 Agentic-grep 概念 + Artist-code-qa-bot 项目级落地设计条目
  - [[index]] — Concepts/AI 应用生态 + Sources + Syntheses + Readers 四分区各登记 + header 更新
- 关键叙事选择:
  - 反直觉开题:"我同事说 grep"——引出"AI 其实是自己写 rg 命令,不是 RAG"
  - 范本展示:Dithered LOD Transition demo 全链路追踪(UI → shader 宏 → discard → stencil 优化 → mobile 性能警告),6 文件 path:line 证据
  - 诚实的弱点:训练数据先验对 UE 原生 symbol "作弊",换项目自定义 symbol 完全失效——这一点用户追问得很准,读本 §4 专门讲
  - 长期出路:wiki 作为术语桥梁 + 长期记忆,术语对照表是"复利巨大"的核心资产
  - 部署纪律:禁幻觉放第一位("美术对代码无免疫力,一次瞎说污染半年信任")
- 要点:
  - Agentic grep = Grep + Glob + Read 三件套,字面匹配,无 embedding 无索引,零维护
  - vs RAG:代码域(符号化系统)agentic grep 更准,自然语言文档 RAG 仍优
  - 未知 symbol 四策略:锚点反推(LOCTEXT / DisplayName) > 命名约定穷举 > 种子爬行 > 停下追问
  - Wiki 的 compounding 效应:alias / tag 本身就是词汇桥;术语对照表是核心产物
  - Persona 分层:`/ask-artist` / `/ask-tech-artist` / `/ask-programmer` 同底座不同粒度
  - POC 最小路径:一子系统(材质 static switch)+ 3-5 美术 + 2 周 + 四评估指标(采纳率/幻觉率/节省时间/沉淀产出)
- open questions(留在读本 §议题留下的问题):
  - POC 评估基线如何量化(无对照组)
  - 冷启动(自定义代码未 ingest 前挫伤信心)
  - 蓝图/uasset 不可 grep(显著盲区)
  - 更新漂移 + lint 周期
  - multi-agent 架构升级拐点
- 自验:
  - Glob [[Raw/Notes/代码检索与美术问答机器人对话]] ✓
  - Glob [[Wiki/Sources/AIApps/Code-retrieval-conversation]] ✓
  - Glob [[Wiki/Concepts/AIApps/Agentic-grep]] ✓
  - Glob [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] ✓
  - Glob [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] ✓(见下步)

---

## [2026-04-21] refactor | 读本全量改名 — 去 `-读本` 后缀 + 文章标题式带空格文件名

- 触发:用户反馈原文件名格式"看着难受",要求改成文章标题式(如 `Phase 7 - 最强扩展点 Data Interface`、`从 Memex 到 LLM Wiki`)
- 规则变更(写入 [[CLAUDE]] §6):
  - **Wiki 原子页**(Concepts/Entities/Sources)仍是小写+连字符(不变)
  - **Readers 主题读本**:文章标题式,带空格,不加 `读本` 后缀;H1 与文件名(去 `.md`)完全一致
  - Niagara 学习路径保留 `Phase N - 标题` 序列前缀
- 重命名(15 文件,git mv 保留历史):
  - `Readers/Methodology/Llm-wiki-方法论-读本.md` → `从 Memex 到 LLM Wiki.md`
  - `Readers/AIArt/Lora-深度指南-读本.md` → `LoRA 深度指南.md`
  - `Readers/AIApps/AI-primer-v2-读本.md` → `AI 应用生态全景 2026.md`
  - `Readers/AIApps/Multi-agent-读本.md` → `为什么上下文有限反而必须切多 Agent.md`
  - `Readers/Niagara/Phase0-心智模型-读本.md` → `Phase 0 - 上阵前的四层脑内地图.md`
  - `Readers/Niagara/Phase1-asset-layer-读本.md` → `Phase 1 - 从 System 到图源抽象基类.md`
  - `Readers/Niagara/Phase2-component-layer-读本.md` → `Phase 2 - Component 层的五职责.md`
  - `Readers/Niagara/Phase3-runtime-instance-读本.md` → `Phase 3 - Niagara 的心脏.md`
  - `Readers/Niagara/Phase4-data-model-读本.md` → `Phase 4 - Niagara 的数据语言.md`
  - `Readers/Niagara/Phase5-cpu-script-execution-读本.md` → `Phase 5 - Niagara 脚本如何跑起来.md`
  - `Readers/Niagara/Phase6-rendering-读本.md` → `Phase 6 - Niagara 粒子如何变成屏幕像素.md`
  - `Readers/Niagara/Phase7-data-interface-读本.md` → `Phase 7 - 最强扩展点 Data Interface.md`
  - `Readers/Niagara/Phase8-gpu-simulation-读本.md` → `Phase 8 - Niagara 的 GPU 模拟管线.md`
  - `Readers/Niagara/Phase9-world-management-读本.md` → `Phase 9 - Niagara 的世界管理与可扩展性.md`
  - `Readers/Niagara/Phase10-advanced-features-读本.md` → `Phase 10 - Niagara 的高级特性.md`
- 批量更新(PowerShell 脚本一次性处理):
  - 每个读本 H1 替换(去"读本"+换成新单标题)
  - 全仓 .md 文件(除 log.md)的旧 stem wikilink 替换 → 新 stem
  - 总计修改 91 文件:CLAUDE.md / README.md / index.md / Wiki/Overview.md / 所有 15 读本 / Niagara-learning-path synthesis / 所有 Niagara Entity + Concept + Source 页 / 所有 AIApps + AIArt + Methodology Concept 页 / 引用 AI 应用生态的 UE4-ddc 等
- 模板同步更新:[[Wiki/_templates/Reader.md]] 路径示例 + H1 示例 + alias 模板
- 不处理:[[log.md]] 所有历史条目保留旧 stem(append-only 历史,grep 回溯用)
- 自验:
  - Glob `Readers/**/*.md` → 15 新路径全部存在 ✓
  - Grep 旧 stem 模式 → 只在 log.md 命中(预期) ✓

---

## [2026-04-21] synthesis | Multi-agent / Subagent 架构读本

- 触发:用户显式要求"还是写一个读本吧"(命中 [[CLAUDE]] §3.4 触发条件 4)
- 新建:[[Readers/AIApps/Multi-agent-读本]] — 7 节主体(三种病 / 第一性原理 / 子 agent 本质 / 四个次级收益 / 链路拓扑 / 并行 vs 串行 / 决策与实例)+ 全景回看 + 10 条关键洞察 + 8 题自检(均为综合/反推/取舍/假设题,非检索题)+ 留下的问题 + 下一步预告 + 深入阅读 + 签名
- 更新:
  - [[Wiki/Concepts/AIApps/Multi-agent]] — 引用来源区顶部加读本链接
  - [[Wiki/Concepts/AIApps/Ai-agent]] — 引用来源区加读本链接
  - [[Wiki/Sources/AIApps/Multi-agent-conversation]] — "与既有 wiki 的关系"加配套读本条目
  - [[index]] — Readers 分区登记 + footer 更新(+1 读本)
- 关键叙事选择:
  - 开题用"反直觉 Q":上下文有限=切多 agent 的理由(反着)
  - 主线 7 节按"病 → 原理 → 机制 → 收益 → 拓扑 → 调度 → 决策"递进,不是按概念罗列
  - 代码/prompt 示例 inline(派活 prompt 骨架、模型分工表、拓扑 ASCII 图)
  - 陷阱用 `> [!warning]`,本质归纳用 `> [!abstract]`,Anthropic 原话用 `> [!quote]`
  - 自检 8 题每题都需要跨 2-3 节串联才能答,特别是第 6 题("5 句话给文科朋友解释"——真懂才讲得出)和第 8 题("给一个多 agent 反而更糟的场景"——反向测试)
- 不处理的:
  - "agent 数量 vs 质量非线性"拐点的定量研究仍空白,读本记为 open question
  - "子 agent 上下文快照 + 事后追问"机制 Anthropic API 暂无,读本记为 open question
- 自验:Glob [[Readers/AIApps/Multi-agent-读本]] 通过

## [2026-04-21] ingest | Multi-agent 对话 — Claudian 自解多 agent 工作机制

- source: [[Raw/Notes/Multi-agent 对话]](对话归纳,非逐字)
- 新建:
  - [[Raw/Notes/Multi-agent 对话]] — 原始对话归纳
  - [[Wiki/Sources/AIApps/Multi-agent-conversation]] — 源摘要
  - [[Wiki/Concepts/AIApps/Multi-agent]] — 独立概念页,从 [[Wiki/Concepts/AIApps/Ai-agent]] 里只有 4 行概述的 Subagent 小节升格为完整概念
- 更新:
  - [[Wiki/Concepts/AIApps/Ai-agent]] — Subagent 小节精简并下链新概念页;sources 1→2;相关区新增 Multi-agent 链接
  - [[Wiki/Concepts/AIApps/Context-window]] — "subagent 是应对 Context Rot 的手段"升级为"多 agent 存在的第一性原理",相关区新增独立链接
  - [[index]] — 新增 Multi-agent 概念登记 + Multi-agent-conversation source 登记 + 头部 footer 更新
- 要点:
  - 反直觉核心:"上下文有限,切多 agent 才有意义"——反过来说才对。多 agent 的第一性原理是**上下文隔离**,不是分工并行
  - 三个病机动:Context Rot / Prompt Cache 5 分钟 TTL / 指令漂移
  - 有效上下文 = N × 200K,代价是 agent 间通信必须压缩
  - 四个次级收益:角色专用 system prompt+工具集 / 模型分工(Opus+Haiku) / 独立审视视角 / 失败隔离
  - 链路拓扑衰减:扁平扇出 > 深度嵌套;Anthropic 观察 agent 数量 vs 质量非线性
  - 子 agent 字面=一次全新 Claude 对话,派 prompt 要像对刚入职同事,强制压缩汇报防信息倾倒
- 不产读本:本轮只 2 原子页,未命中 [[CLAUDE]] §3.4 触发(<3 原子页、非新主题首次入驻、用户未显式要求)
- 自验:对本条 log 声称的 3 个新建路径各跑 Glob,已全部存在(见下一步之前的验证)

## [2026-04-21] lint | Niagara Phase 5-10 分组 targeted 全量检查 + 3 处微修复

- 触发:学习路径 Phase 0-10 全部完成后的质量基线确认(Phase 0-4 先前已稳定,本轮重点 5-10)
- 方法:按 Phase 分组套用 [[Wiki/_templates/Lint-checklist]] 的思路,**不做全 vault 扫描**,只扫每 Phase 相关原子页 + 读本 + 学习路径对应章节 + log 对账 + index 登记。每 Phase 派一个 Explore agent 并行检查 6 类(frontmatter / broken wikilinks / fs-log 对账 / 交叉引用闭合 / 读本签名与模板痕迹 / Phase 特有一致性)
- 覆盖规模:
  - Phase 5(CPU 脚本执行):11 文件(5 Source + 5 Entity + 1 Reader)
  - Phase 6(渲染系统):17 文件(10 Source + 6 Entity + 1 Reader)
  - Phase 7(数据接口):18 文件(9 Source + 8 Entity + 1 Reader)
  - Phase 8(GPU 模拟):19 文件(13 Source + 5 Entity + 1 Reader)
  - Phase 9(世界管理):13 文件(6 Source + 6 Entity + 1 Reader)
  - Phase 10(高级特性):12 文件(6 Source + 5 Entity + 1 Reader)
  - **合计:90 文件全扫**,全部绿色通过(0 broken / 0 模板残留 / 0 log 对账差异 / commit 统一 `b6ab0dee9`)
- 微修复(3 处,均为 🟢 低优化:Source→Entity 方向的头部显式链接,提升导航可发现性):
  - 修改:[[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] L23-26 — 头部新增"派生 Entity"小节,列出块 B/C/D 对应的 3 个 Entity wikilink
  - 修改:[[Wiki/Sources/Stock/NiagaraDataInterface]] L23-32 — 头部新增"主实体 + Phase 7 直接子类 Entity(×7)+ Phase 10 分支"三节,显式闭合 DI 主基类到 7 子类的 Source→Entity 链接,并标注 RW 基类分叉
  - 修改:[[Wiki/Entities/Stock/UNiagaraDataInterfaceCurve]] L17-19 — 标题导语升级为带 ⚠️ 的"合并页说明",明确 Curve Entity 覆盖 `UNiagaraDataInterfaceCurveBase`(无独立 Entity)+ `UNiagaraDataInterfaceCurve` 两个类,并显式下链两个对应 Source
- 关键发现:
  - Phase 6、Phase 8 满分通过(零建议),质量基线最稳
  - Phase 5/7 各有一个 Source→Entity 头部显式链的缺口(上游信息可推断但需跳转),本轮已补
  - 跨 Phase 继承链完整:DI 链 Phase 5 Base → Phase 7 UNiagaraDataInterface → Phase 7 七子类 / Phase 10 RWBase → Phase 10 三 Grid/NeighborGrid,全部双向闭合
  - 读本签名 6/6 合格(含 commit hash + Claudian 签名 + 日期)
  - Phase 8.8 NiagaraEmitterInstanceBatcher 的跨 Phase 引用(Phase 5 ↔ Phase 8)措辞一致,无重复摄入
- 不处理的:
  - 源码原注释的 TODO(ComponentPool FIFO / EffectType ScreenFraction):已在 Phase 9 读本"留下的问题"段落显式 backlog,不算残留
  - 4.26 未涉 RDG,技术栈一致,无须标注
- 下一步:本次 lint 确认学习路径 Phase 5-10 质量基线稳固,可作为长期检索 / 人类线性阅读的可信基础。后续若对 Phase 0-4 也做同样 targeted 检查,可进一步补全全路径 lint 记录

---

## [2026-04-20] ingest | Niagara Phase 10 · 高级特性(6 头文件,**学习路径终点**)

- 源(code,stock @ `b6ab0dee9`):合计 944 行
  - SimulationStageBase(78)/ DataInterfaceRW(246,RW DI 基础)
  - Grid2DCollection(268)/ Grid2DCollectionReader(95)
  - Grid3DCollection(158)/ NeighborGrid3D(99)
- 新建(12):6 Source + 5 Entity + 1 读本
- 要点:
  - **SimStages 多 pass 架构**:每 emitter 多 stage,每 stage 独立 script + IterationSource(Particles | DataInterface)+ Iterations 次数
  - **`Iterations` × `MinStage/MaxStage` 收合**:Jacobi 等迭代式用一条 stage + Iterations=N 代替 N 条(Phase 8.5 shader 侧元数据)
  - **RW DI 四钩子**:`PreStage / PostStage / PostSimulate / ResetData` —— 每 SimStage 四时点回调
  - **`AsIterationProxy = this`** 让 SimStage 识别 DI 可作 iteration source;`GetElementCount` 告诉 dispatch 多少 thread
  - **Grid2D vs Grid3D 存储策略差异**:2D 用 Texture2DArray(每 slice 一个 attribute),3D 用 RWTexture3D + tile 打包(RHI 没 Texture3DArray)
  - **Grid 共享 HLSL 函数族**:NumCells / CellSize / UnitToIndex 等,所有 Grid DI 共用 API
  - **Grid2DCollectionReader** 跨 emitter 数据交互;`GetEmitterDependencies` 声明 tick 依赖
  - **NeighborGrid3D ≠ Grid3DCollection**:前者存对象列表(粒子索引,SPH/boids),后者存场(烟雾密度)
  - **`MaxNeighborsPerCell` lossy 设计**:超容 drop 不 warn,SPH 精度风险
- **Niagara 学习路径 10/10 全通**!一日内从 Phase 3 推到 Phase 10(8 Phase,~120 原子页 + 8 读本)
- 下一步:lint 扫一次 wiki,按需深入 open questions

---

## [2026-04-20] ingest | Niagara Phase 9 · 世界管理(6 头文件)

- 源(code,stock @ `b6ab0dee9`):合计 1365 行
  - WorldManager(286)/ScalabilityManager(140)/ComponentPool(149)
  - Settings(75)/EffectType(337)/PlatformSet(378,扒前 200)
- 新建(13):6 Source + 6 Entity + 1 读本
- 要点:
  - **`FNiagaraWorldManager` 每 World 一个** + FGCObject,集中 SystemSimulations[TickGroup] + ScalabilityManagers[EffectType] + ComponentPool + SkinningData + ViewLocations
  - **Scalability 四层叠加**:Settings → EffectType → System Override → Platforms 匹配
  - **cull 与 OnSystemFinished 解耦**:`bIsScalabilityCull=true` 走 Internal 路径,不触发用户 delegate
  - **4 种 cull**:Distance / VisibleTimeout / EffectTypeInstanceCount / PerSystemInstanceCount
  - **Significance Handler 扩展点**:默认 Distance/Age,可自定义
  - **5 种 CullReaction**:Deactivate / Immediate / Resume / ImmediateResume(Kill vs Asleep)
  - **UpdateFrequency 5 档**:SpawnOnly / Low / Medium / High / Continuous,每 EffectType 独立
  - **Pool 5 方法**:None / AutoRelease / ManualRelease / ManualRelease_OnComplete / FreeInPool,高频小特效 + AutoRelease 是核心
  - **PrimePool 预热**:关卡加载时预分配,避免战斗首次 spawn 卡顿
  - **PlatformSet 三态双 mask**:QualityLevelMask + SetQualityLevelMask 两 bit 表达 Enabled/Disabled/Default 三态
  - **CVar 条件过滤**:除了 device profile,还可按 CVar 值范围过滤
- 下一步:Phase 10 高级特性(6 文件,SimStages + Grid DI,最后一个 phase)

---

## [2026-04-20] ingest | Niagara Phase 8 · GPU 模拟(13 头文件,8.8 已在 Phase 5.5)

- 源(code,stock @ `b6ab0dee9`):合计 2500 行
  - Shader 编译(5):NiagaraShared(902,扒前 400)/Shader(168)/ShaderType(160)/ShaderMap(8 stub)/ScriptBase(53)
  - GPU Count + Draw(2):GPUInstanceCountManager(127)/DrawIndirect(130)
  - GPU Sort(2):GPUSortInfo(76)/SortingGPU(81)
  - VertexFactory(4):VF base(135)/Sprite(223)/Ribbon(239)/Mesh(198)
- 新建(19):13 Source + 5 Entity + 1 读本
  - Entity 合并度高:Shader 家族合 4 → 1,Sort 合 2 → 1,VF 合 4 → 1,DrawIndirect 合 3 → 1,InstanceCountMgr 独立
  - 读本:[[Readers/Niagara/Phase8-gpu-simulation-读本]](8 节 + 9 条洞察)
- 要点:
  - **NiagaraShader 模块分离**:接口/实现,让 RHI/Shader 不拉 Niagara 主模块
  - **FNiagaraShader 30+ LAYOUT_FIELD**:粒子 Buffer(Float/Int/Half × SRV/UAV)+ 5 种 ConstantBuffer[2] + SimStage + Event × 4 + DI
  - **`FNiagaraDataInterfaceParamRef` non-virtual + binary memory image**:支持 shader cache 序列化
  - **共享 CountBuffer + DrawIndirect**:GPU count 不 readback,`RHIDrawIndexedPrimitiveIndirect` + GPU 自动生成 args
  - **`FNiagaraDrawIndirectArgGenTaskInfo` 双用**:`NumIndicesPerInstance = -1` 做 clear,同一 dispatch 合并
  - **GPU Sort 架构**:Niagara 只做 key 生成(`FNiagaraSortKeyGenCS`,4 permutation),radix 由 UE `FGPUSortManager`
  - **`FSimulationStageMetaData MinStage/MaxStage 区间**:一条元数据覆盖 N 次迭代(Jacobi 等)——Phase 10 预告
  - **3 种 VF 能力矩阵**:Mesh 独占 tessellation + 需 StaticMesh 顶点数据;Sprite/Ribbon 全 SRV-based
  - **Half offset MSB 编码** + Half/Float SRV 双路径:shader 按位选 SRV,零分支
  - **`CheckAndUpdateLastFrame`** 避免多 view 重复绑定
- 下一步:Phase 9 世界管理(6 文件,⭐⭐⭐)

---

## [2026-04-20] ingest | Niagara Phase 7 · 数据接口系统(9 头文件,7.1 已在 Phase 5)

- 源(code,stock @ `b6ab0dee9`):合计 2998 行(大文件扒头部)
  - 主基类:NiagaraDataInterface.h(890,扒前 400)
  - Curve:CurveBase(251)/ Curve(58)
  - Camera(107)/ CollisionQuery(93)/ Texture(63)/ RenderTarget2D(138)
  - StaticMesh(491,扒前 250)/ SkeletalMesh(976,扒前 300)
- 新建(18):9 Source + 8 Entity + 1 读本
  - Entity 压缩:Curve 把 Base+Float 合并为一个
- 要点:
  - **三路代码生成**:编译期 `GetFunctions` 注册签名,运行时 CPU `GetVMExternalFunction → lambda`,运行时 GPU `GetParameterDefinitionHLSL + GetFunctionHLSL → compute shader 拼接`
  - **Per-Instance Data blob**:SystemInstance 16-byte 对齐分配,DI 按 offset 读写
  - **VM 绑定模板族**:`TNDIBinder` 系列 + `DEFINE_NDI_FUNC_BINDER` 宏,声明式注册函数
  - **Curve LUT + ExposedTexture**:预计算 + 跨系统(脚本/材质)复用
  - **SkeletalMesh 共享 SkinningData**:世界级 `FNDI_SkeletalMesh_GeneratedData` TMap<MeshComp, TSharedPtr>,引用计数 + RWLock,多 DI 共享
  - **TickGroup Prereqs**:DI 可强制 System Instance 的 TickGroup(Camera 要晚,Physics 要早)
  - **RW DI**:`UNiagaraDataInterfaceRenderTarget2D : UNiagaraDataInterfaceRWBase`,继承链不同——Phase 10 基础
  - **CPU Access 陷阱**:Mesh 没开就只能 GPU 采样三角形/顶点
  - **GPU only DI**:Texture DI 明确只支持 GPUComputeSim
- 下一步:Phase 8 GPU 模拟(14 文件,最大 Phase)

---

## [2026-04-20] ingest | Niagara Phase 6 · 渲染系统(10 头文件)

- 源(code,stock @ `b6ab0dee9`):合计 1776 行,10 文件成对
  - 基类:NiagaraRendererProperties(259)/ NiagaraRenderer(144)
  - Sprite:SpriteRendererProperties(316)/ RendererSprites(102)
  - Ribbon:RibbonRendererProperties(361)/ RendererRibbons(103)
  - Mesh:MeshRendererProperties(288)/ RendererMeshes(74)
  - Light:LightRendererProperties(98)/ RendererLights(31)
- 新建(17):10 Source + 6 Entity + 1 读本
  - Entity 压缩为 6 个(Properties 和 Renderer 成对合并一个 Entity)
  - 读本:[[Readers/Niagara/Phase6-rendering-读本]](8 节 + 8 条洞察)
- 要点:
  - **Properties/Renderer 对偶**:Asset 侧 UObject,Runtime 侧非 UObject,通过 CreateEmitterRenderer 工厂连接
  - **一个 Emitter 可多 Renderer**(火焰 = Sprite + Light + Mesh 叠加)
  - **GT→RT 三阶段**:GenerateDynamicData(GT)→ SetDynamicData_RenderThread(桥)→ GetDynamicMeshElements(RT)
  - **`FNiagaraDynamicDataBase::Data` union**:CPU sim 存 `DataBuffer*`,GPU sim 存 `ComputeExecutionContext*`,按 SimTarget 选
  - **`FNiagaraRendererLayout` GT+RT 双 copy**,Finalize 时拷贝,RT 只读
  - **Half type 编码到 offset 最高位**(`Offset |= 1 << 31`)—— shader 按位判断类型
  - **`bGpuLowLatencyTranslucency`** 走 TranslucentDataToRender 低延迟通路(半透明 + 不透明同步)
  - **4 种类型能力矩阵**:
    - Sprite:通用,CPU+GPU,17 VF slot,5 种 facing,Cutout 优化,VR 稳定 facing
    - Ribbon:**CPU only**,多带 per emitter,RibbonId+LinkOrder,自适应 tessellation,双 UV channel
    - Mesh:CPU+GPU,StaticMesh 实例化,14 VF slot,全粒子共享 LOD(不支持 per-instance LOD)
    - Light:**CPU only**,不产 mesh batch,走 GatherSimpleLights,bAffectsTranslucency 严重开销警告
- 下一步:Phase 7 数据接口系统(10 文件:DI 基类 × 2 + 典型 DI × 8)

---

## [2026-04-20] ingest | Niagara Phase 5 · CPU 脚本执行(5 头文件)

- 源(code,stock @ `b6ab0dee9`):合计 1185 行
  - `NiagaraCore/Public/NiagaraCore.h`(6 行,仅 typedef)
  - `NiagaraCore/Public/NiagaraDataInterfaceBase.h`(136 行)
  - `Niagara/Classes/NiagaraScriptExecutionContext.h`(531 行,核心)
  - `Niagara/Public/NiagaraScriptExecutionParameterStore.h`(195 行)
  - `Niagara/Classes/NiagaraEmitterInstanceBatcher.h`(317 行,**实际是 GPU Batcher**)
- 新建(11):5 Source + 5 Entity + 1 读本
  - Source:NiagaraCore / NiagaraDataInterfaceBase / NiagaraScriptExecutionContext / NiagaraScriptExecutionParameterStore / NiagaraEmitterInstanceBatcher
  - Entity:UNiagaraDataInterfaceBase / FNiagaraScriptExecutionContext / FNiagaraComputeExecutionContext / FNiagaraGPUSystemTick / NiagaraEmitterInstanceBatcher
  - 读本:[[Readers/Niagara/Phase5-cpu-script-execution-读本]](7 节 + 7 条洞察)
- 要点:
  - `NiagaraCore` 模块只有 3 个文件,**接口/实现分离**:让 NiagaraShader / NiagaraVertexFactories 依赖 DI 接口不拖累主模块
  - **CPU VM Execute** 核心:把 `Parameters / FunctionTable / UserPtrTable / DataSetInfo / ConstantBufferTable` 喂给引擎内置 VectorVM 的 `Exec(Context, ByteCode, NumInstances)`
  - **System 脚本的 PerInstanceFunctionHook** 解决"一批 instance 的 User DI 不同"—— VM 遇到 DI 调用 → hook → 按当前 instance 索引查该 instance 的 DI 函数
  - **CPU 紧凑布局 vs GPU 对齐布局**的映射由 `FNiagaraScriptExecutionPaddingInfo` 编译期生成,4 个 uint16(SrcOffset/DestOffset/SrcSize/DestSize)
  - **`CopyCurrToPrev` vs Phase 3 `GlobalParameters[2]` 双缓冲是两件事**:前者服务脚本内读 `Previous.*` 命名空间,后者服务 GT/RT 跨线程
  - **`FNiagaraGPUSystemTick::InstanceData_ParamData_Packed`** 紧凑 byte 布局 + 16-byte 对齐,让 ParamData 能直接 upload 成 UniformBuffer
  - **`NiagaraEmitterInstanceBatcher` 不是 CPU batcher**,而是 RT 驻留的 GPU compute 总调度器,分 PreInitViews/PostInitViews/PostOpaqueRender 三 stage dispatch —— 文件名有误导,Phase 8 完整展开
  - 5 种 UniformBuffer(`UBT_Global / System / Owner / Emitter / External`)
- 下一步:Phase 6 渲染系统(10 文件,Sprite/Ribbon/Mesh/Light × Properties+Renderer)

---

## [2026-04-20] ingest | Niagara Phase 4 · 数据模型(7 头文件)

- 源(code,stock @ `b6ab0dee9`):合计 5663 行
  - `NiagaraTypes.h`(1739)、`NiagaraCommon.h`(1200)、`NiagaraConstants.h`(209)
  - `NiagaraDataSet.h`(554)、`NiagaraDataSetAccessor.h`(619)
  - `NiagaraParameters.h`(82)、`NiagaraParameterStore.h`(1260)
- 新建(15):7 Source + 7 Entity + 1 读本
  - Source:NiagaraTypes / NiagaraCommon / NiagaraConstants / NiagaraDataSet / NiagaraDataSetAccessor / NiagaraParameters / NiagaraParameterStore
  - Entity:FNiagaraTypeDefinition / FNiagaraVariable / FNiagaraTypeLayoutInfo / FNiagaraConstants / FNiagaraDataSet / FNiagaraDataSetAccessor / FNiagaraParameterStore
  - 读本:[[Readers/Niagara/Phase4-data-model-读本]](8 节 + 8 条洞察)
- 更新:learning-path / Overview / index / log
- 要点:
  - Niagara **自建类型系统**(`FNiagaraTypeDefinition`),不直接用 UStruct—— 为了把任何类型拍成 Float/Int32/Half 三路 component,适配 VM 和 GPU
  - **`FNiagaraBool` 是 int32**,True=-1 / False=0(VM compare+select 友好)
  - **SoA 内存图像**:`FNiagaraDataBuffer::FloatData` 按 component 为主序组织,`FloatStride = NumInstancesAllocated × 4`
  - **Double buffer + 池化**:2-3 个 DataBuffer,`CurrentData` 读 / `DestinationData` 写 / RT 读那个单独保留
  - **`FNiagaraSharedObject`** 原子 ReadRefCount(INDEX_NONE 作写锁)+ 延迟删除队列解决 GT/RT 并发
  - **Persistent ID 双整数**(Index + AcquireTag)—— Index 复用,AcquireTag 区别历史;只在需要跨帧追踪时才开
  - **参数命名空间**(User./Engine./Particles./Emitter./Module./Initial./Previous./Constants.)—— 命名决定存储位置
  - **ParameterStore 三路 dirty**(参数/DI/UObject)—— 改一个参数不重传 200 个 DI,增量推送
  - **`FNiagaraParameters` vs `FNiagaraParameterStore`** 遗留技术债 —— editor 用前者,运行时用后者
- 上下文管理:
  - 本次采用"小/中文件全读 + 大文件(Types/Common/ParameterStore)读前 400 行"的策略。大文件后半(约 1600 行)未扒,读本已标注"按需 offset 读"
  - 原子页规模压缩:Source 150-250 行,Entity 30-50 行(比 Phase 2/3 略紧凑)
- 下一步:Phase 5 CPU 脚本执行(5 文件:NiagaraCore / NiagaraDataInterfaceBase / NiagaraScriptExecutionContext / NiagaraScriptExecutionParameterStore / NiagaraEmitterInstanceBatcher CPU 侧)

---

## [2026-04-20] ingest | Niagara Phase 3 · 运行时实例层(3 头文件)

- 源(code,stock @ `b6ab0dee9`):
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemInstance.h`(574 行)
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstance.h`(239 行)
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemSimulation.h`(429 行)
- 新建(7):
  - Source × 3:[[Wiki/Sources/Stock/NiagaraSystemInstance]]、[[Wiki/Sources/Stock/NiagaraEmitterInstance]]、[[Wiki/Sources/Stock/NiagaraSystemSimulation]]
  - Entity × 3:[[Wiki/Entities/Stock/FNiagaraSystemInstance]]、[[Wiki/Entities/Stock/FNiagaraEmitterInstance]]、[[Wiki/Entities/Stock/FNiagaraSystemSimulation]]
  - 读本:[[Readers/Niagara/Phase3-runtime-instance-读本]](7 节 + 7 条洞察 + 完整 Tick 时序图)
- 更新:learning-path / Overview / index / log
- 要点:
  - 三类都非 UObject(性能选择);生命周期靠 `TUniquePtr`/`TSharedRef`,UObject 引用靠 `TWeakObjectPtr`/`FGCObject`
  - **双状态机** Requested vs Actual 解耦用户意图与实际状态
  - **三阶段 Tick** GT/Concurrent/Finalize,避免 GT 阻塞但保持 DI 后处理顺序
  - **Tick Batch = 4**,同 Asset 多实例批量 tick 的具体实现
  - **参数双缓冲** `[2]` 数组 + `CurrentFrameIndex:1` 位翻转,解决 RT 读上一帧与 GT 写当前帧的竞态
  - **`FNiagaraParameterStoreToDataSetBinding`** 是 Phase 1 编译产物的运行时消费者,实现"无字符串查表"memcpy
  - **`bForceSolo` 退化 5-20×** 给出定量估算(50 实例场景)
  - **Emitter event 通讯** 靠 SystemInstance::EmitterEventDataSetMap (`TMap<(EmitterName,EventName), DataSet*>`)
- 下一步:Phase 4 数据模型(7 文件,SoA 布局为核)

- 源(code,stock @ `b6ab0dee9`,branch `Eureka`):
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponent.h`(741 行)
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraActor.h`(66 行)
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraFunctionLibrary.h`(93 行)
- 新建(7):
  - Source 页 ×3:[[Wiki/Sources/Stock/NiagaraComponent]]、[[Wiki/Sources/Stock/NiagaraActor]]、[[Wiki/Sources/Stock/NiagaraFunctionLibrary]]
  - Entity 页 ×3(紧凑 30-50 行):[[Wiki/Entities/Stock/UNiagaraComponent]]、[[Wiki/Entities/Stock/ANiagaraActor]]、[[Wiki/Entities/Stock/UNiagaraFunctionLibrary]]
  - 读本 ×1:[[Readers/Niagara/Phase2-component-layer-读本]](8 节 + 9 条洞察 + open questions + 深入阅读)
- 更新(3):
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]]:Phase 2 区标 ✅,文件表补 Source/Entity 链接,进度追踪 3 个 checkbox 全勾
  - [[Wiki/Overview]]:Phase 2 条目 + "Phase 2 的关键收获" 6 条 + 知识图增 Component 层块 + open questions 补 Phase 2 遗留 + "下一步建议" 换为 Phase 3
  - [[index.md]]:最后更新注释 + Entities/Sources/Readers 三分区各登记新页
- 要点(Phase 2 的核心心智):
  - **`UNiagaraComponent` 是唯一主角**,Actor/FunctionLibrary 加起来不到 1/5(66+93 vs 741)
  - **Asset ↔ Instance 一对多**:`TUniquePtr<FNiagaraSystemInstance>` 独占,N Component 引用同 Asset = N 份独立 Instance
  - **Component 五职责**:Asset 持有 / Instance 管理 / 参数覆盖层 / 场景集成 / 生命周期调度
  - **参数覆盖分层**:标量走 Component 18 个 `SetVariableXxx`,对象型(Mesh/Texture)走 FunctionLibrary `Override*`(DI 关注点分离)
  - **生命周期四源汇于一点**:UActorComponent / UFXSystemComponent / 用户语义 / Scalability 四源全部在 `TickComponent` + `OnSystemComplete` 收口
  - **`bForceSolo` 性能陷阱**:绕开 `FNiagaraSystemSimulation` 批量 Tick,"同 Asset 多实例" 典型场景退化数十倍——读本 § 4 和 Entity 页 `[!warning]` 都强调了
  - **`ANiagaraActor` 是纯 observer 范式**:66 行的 ComponentWrapperClass,只订阅 `OnSystemFinished` 一个 delegate 决定生死
  - **`SpawnSystemAttached` 的 auto-attachment 子系统**:6 字段 + 3 保存槽,实现 "生成时挂载、结束时解绑并还原 transform" 的工程细节
  - **Phase 2 遗留 open questions**:`FNiagaraSystemSimulation`(→P3)、`FNiagaraSystemInstance` 状态机(→P3)、`ENCPoolMethod` 决策(→P9)、`FNiagaraScalabilityManager`(→P9)、`FNiagaraSceneProxy` GT↔RT(→P6)、`VectorVM FastPath`(→P5)
- 方法论验证:
  - 本次首次把 `FNiagaraSceneProxy`(定义在 Component.h 末尾)和 `VectorVM FastPath`(定义在 FunctionLibrary 末尾)两个"物理位置在本 Phase 但职责在其他 Phase"的内容,按约定"记存在 + 指向正确 Phase"处理——既不污染叙事,也不漏
  - Ingest 前先与用户对齐 4 个边界问题(Proxy/VM/Age/ForceSolo),用户 OK 后一口气产出所有原子页 + 读本 + bookkeeping,没有回头返工——说明"先对齐要点、再一次产出"的工作流是有效的
- 下一步:
  - Phase 3(运行时实例层,3 文件):`NiagaraSystemInstance.h` / `NiagaraEmitterInstance.h` / `NiagaraSystemSimulation.h`,首个 ⭐⭐⭐ 难度阶段,状态机复杂度显著提升。等用户指令启动

---

## [2026-04-20] refactor | Readers 提升为顶层目录(与 Raw/Wiki 同级)

- 触发:用户请求把 `Readers/` 从 `Wiki/` 里移出来,放到与 `Raw/` / `Wiki/` 同级
- 动机:Readers 服务"人类线性阅读"、Wiki 服务"LLM 原子检索",服务对象不同;物理分层到顶层可让首次来访者在仓库根直接看到"给人读的入口",降低发现成本
- 操作:
  - `git mv Wiki/Readers Readers`(5 个读本全部保持目录结构平移:AIApps / AIArt / Methodology / Niagara)
  - 全库批量替换 `Wiki/Readers` → `Readers`,共 33 个 markdown 文件链接更新(log / index / Overview / 概念页 / 实体页 / 读本内部引用 / 模板等)
  - 更新 [[CLAUDE.md]] §1 "三层架构"→"顶层架构",4 行表格(加 Readers 一行 + 服务对象列)
  - 更新 [[CLAUDE.md]] §2 目录结构树:Readers 单列顶层块;`Syntheses/` vs `Readers/` 对比说明加一句"独立于 Wiki 顶层"
  - 更新 [[README.md]] 顶层架构 ASCII 图:Readers 独立块,插在 Schema 和 Wiki 之间,强调"与 Wiki 同级"
- 不变的:
  - Readers 仍由 LLM 拥有(与 Wiki 同属 LLM 产物),只是物理分层
  - §3.4 读本产出流程、模板位置(`Wiki/_templates/Reader.md`)、触发条件全部不变
  - 现存读本 frontmatter / 内容无需改动
- 验证:`grep -r "Wiki/Readers"` 返回 0 匹配,`ls Readers/` 看到 4 个主题子目录 + 5 个读本文件完整
- 下一步:无后续任务,本次是纯结构调整

---

## [2026-04-20] refactor | 4 个读本批量刷 Callouts(收尾)

- 触发:昨天 Phase 1 读本 4 处示范看下来视觉满意,用户请求把其余 4 个读本也统一刷一遍
- 范围:[[Readers/Methodology/Llm-wiki-方法论-读本]]、[[Readers/Niagara/Phase0-心智模型-读本]]、[[Readers/AIApps/AI-primer-v2-读本]]、[[Readers/AIArt/Lora-深度指南-读本]]
- 策略:heading 级 ⚠️ 标题转 `[!warning]` callout(heading 去 emoji,配标题短语);inline ⚠️ 标记(bullet / table cell / numbered list)**保留不动**——精准点标记不适合整块 callout
- 统计:
  - Methodology 读本:2 个 heading ⚠️ → warning callout(§3.7 刻意抽象、§4.4 事实追溯)
  - Phase0 读本:2 个 heading ⚠️ → warning callout(§2.8 Handle.Instance 陷阱、§3.7 兼容性提示)
  - AIApps 读本:5 个 heading/段落 ⚠️ → warning callout(§1.2 没有元认知、§2.1.5 第一纪律、§3.1.7 推理幻觉、§9.6 信息缺口、Karpathy 事实追溯)
  - AIArt 读本:4 个 heading/段落 ⚠️ → warning callout(§4.7 角色 caption、§6.5 权重冲突、§7.2 Civitai 禁用、§9.3 LoRA 许可继承)
  - 共 13 处转换,保留 3 处 inline ⚠️(都在 AIArt 读本,分别是 rank 过大 bullet、Flux Dev 表格、Dev 非商用数字列表)
- 覆盖率:5 个读本(含 Phase 1 昨天先做的 4 处)合计 **17 个 callout**,分布:warning ×13 + question ×1 + abstract ×1 + quote ×1 + 其他
- 副产品:读本视觉统一达成——全库不再有裸 ⚠️ emoji 作为章节标题,所有警告都有统一视觉分层
- 不做的:
  - 为每个读本再硬加 [!question] / [!abstract] / [!quote] — 本次只做 ⚠️ 一类,保持批量修改的简单性;后续写新读本按 Reader.md 模板约定自然产出多样化 callout 即可
- 收工:今天共 7 个 commit(首次 lint + 三档修复 + Lint playbook + 5 项视觉/使用改进 + quote 配色两次调整 + 读本 callout 全库覆盖),从"读本概念页历史债"到"视觉统一"走通一个完整回路

---

## [2026-04-20] refactor | vault 使用 / 视觉 5 项改进(post-lint 打磨)

- 触发:用户问"这个 vault 使用和视觉上还能怎么改",我给了三档建议,用户挑了真值得改的 4 项 + 开放问题汇总机制,一口气都做掉
- 新建:
  - `.gitattributes`(覆盖原仅 1 行版本)— 强制 `eol=lf` + 给常见文本/二进制类型显式标注,消除 Windows 下 git 刷屏的 CRLF 警告
  - [[.obsidian/snippets/readers.css]] — reading mode 专用 CSS:820px 行宽 + 1.75 行距 + H1/H2 底线分隔 + H3 accent 色 + callout 圆角 + 表格斑马条。启用方式:Obsidian 设置 → Appearance → CSS snippets → 打开"readers"
- 更新:
  - [[CLAUDE]] §3.1 — ingest 流程补第 9 步"自验(强制)":对本轮 log 里所有"新建 [[...]]"路径跑一次 Glob,差异立即停下告诉用户。解决这次 lint 发现的 cheatsheet 幽灵文件类型问题(即便本次是用户主动删除,自验一样会触发复核)。属 always-apply,进 CLAUDE.md 内嵌而非外移
  - [[Wiki/_templates/Lint-checklist]] 新增 §11"开放问题汇总"— 周期性扩展动作(不进标准 0-10 强制项),用 grep 抓全仓 `## 开放问题` 节聚合为临时快照页,让用户一次过一遍决定"已解 / 保留 / 升级为下次 ingest 目标"。建议频率:每月 1 次或每 5 次 lint 一次。元规则记录:没做成独立 op,因为与 lint 逻辑连续;满足"能复用就不新开操作"
  - [[Wiki/_templates/Reader]] 模板 — 引入 Obsidian Callouts 约定作为读本标准写法:9 类 callout(warning/question/abstract/quote/tip/note/example/info/折叠语法)替代裸 emoji,从此生成读本时直接按此约定产出
  - [[Readers/Niagara/Phase1-asset-layer-读本]] — 做 4 处示范性 callout 转换:第 0 节 Phase 要回答的问题 → `[!question]`、§2.5 命名陷阱(原 ⚠️)→ `[!warning]`、§2.6 Handle 本质归纳 → `[!abstract]`、§3 UNiagaraEmitter 源码自述 → `[!quote]`。其余 4 个读本暂不批量改,下次 lint 时评估是否需要统一回溯
- 元规则应用记录(本轮共 3 次分层判断):
  - `.gitattributes` / CSS snippet → 文件系统级基建,不进 CLAUDE.md,原地落
  - ingest 自验 → 每次 ingest 都要做 = always-apply → 进 CLAUDE.md §3.1 作为步骤 9
  - 开放问题汇总 → 周期性使用 = 条件性 → 进 Lint-checklist.md 作为扩展节而非强制节;且选择"复用 lint" 而非"开新 op"
- 不做的:
  - 换 Obsidian 主题 / 装美化插件(边际收益低)
  - Dataview 自动生成 index(人工+LLM 维护的带描述索引质量优于自动列表)
  - 装 Templater(CLAUDE.md + _templates/ 已替代此角色)
  - Readers/ 批量改 callout(先看 Phase1 4 处示范的效果;若用户满意再批量)
- 下一步:
  - 用户 commit push 后,在 Obsidian 里打开 Phase1 读本,启用 readers.css snippet,对比视觉改善
  - 若示范满意,可批量把其余 4 个读本的 ⚠️ 和长引用块按新 callout 约定改写

---

## [2026-04-20] refactor | 新增 Lint checklist — 元规则第二次应用

- 触发:首次 lint 实践后,总结经验固化为可复用 checklist;用户提示"补吧,不过你要考虑好该放在哪"——明确点名让我做元规则分层判断
- 元规则分层判断(应用 [[CLAUDE]] §7 原则 6):
  - 每轮对话都要守? ❌ 只在 lint 时才用 → 外移,不进 CLAUDE.md
  - 候选位置:
    - (A) `Wiki/_templates/Lint-checklist.md` — 复用现有文件夹,但 `_templates/` 语义需稍扩
    - (B) 新建 `Wiki/_playbooks/Lint.md` — 语义最准,但为 1 个文件开文件夹过早抽象
    - (C) CLAUDE.md §3.3 inline — 违反分层,污染 always-load context
  - **选 A**:`_templates/` 本质是"LLM 在做特定操作时按需 Read 的 schema 参考",页面模板是它的第一子类,操作 checklist 是第二子类;开新文件夹是过早抽象,待累积 ≥3 份操作 playbook 再考虑拆分
- 新建:[[Wiki/_templates/Lint-checklist]] — 10 节完整作业规程,从前置 inventory → 8 类检查(孤儿 / broken / log 对账 / 缺失概念 / frontmatter / 交叉引用 / 矛盾 / 读本签名) → 报告格式 → 收尾 checklist;含 agent delegation 判断、人工验证硬要求、"诊断不是治疗"原则
- 更新:
  - [[CLAUDE]] §3.3 — 从"列 6 条检查项"改为"Read [[Wiki/_templates/Lint-checklist]] 按其执行",核心输出仍留主文件,细节外移
  - [[CLAUDE]] §4 — `_templates/` 表格从"页面结构模板"改为"schema 参考",增加 Lint-checklist 一行,显式承认了扩展语义
  - [[README]] — 三层架构图 / Wiki 树 / Schema 分层设计段落三处同步措辞("页面模板" → "schema 参考:页面模板 + 操作 checklist")
- 副产品/洞察:
  - 元规则本次应用**不只是"去哪"**,还催生了"要不要开新文件夹"的决策——分层判断扩展到结构层面,不只是内容层面
  - 首次 lint 的实操经验暴露了几条 checklist 里最重要的条款:§3 filesystem vs log 对账(本次发现 cheatsheet 幽灵文件就靠它)、§0 规模门槛决定是否 delegate agent、收尾"人工验证可疑点"(agent 会误报,本次就手动验证了 cheatsheet 丢失和 Claudian broken)
  - 未来若出现 Ingest playbook / Synthesis playbook,累积到 ≥3 份时触发 `_templates/` 分拆为 `_templates/` + `_playbooks/`
- 下一步:以当前 schema 再跑一次 lint(试验 checklist 自身),或直接进入 Niagara Phase 2 ingest

---

## [2026-04-20] synthesis | lint 🟡🟢 项清理 — 3 个新概念页 + 小问题修补

- 触发:🔴 项完成后用户"黄的也修一下",一口气把 🟡 和 🟢 全部扫清
- 新建 3 个高频缺失概念页(对应 lint 报告 🟡 中的三个)。每页标准 header + 概览 + 字段速查 + ⚠️ 陷阱 + 相关 + 深入阅读 + 开放问题;长度 70-95 行:
  - [[Wiki/Concepts/UE4/UE4-ddc]] — Derived Data Cache;UE4 机器/团队级编译产物缓存;Niagara 用 `FNiagaraVMExecutableDataId` 直接作 key,也记录 shader/texture/mesh 等其他典型用户
  - [[Wiki/Concepts/Methodology/Vibe-coding]] — Karpathy 2025-02 命名;定位为 Vibe/Spec/Harness 三阶段演进的起点;含对非开发角色的适用视角(美术/策划/管理者)
  - [[Wiki/Concepts/AIApps/Embedding]] — 把语义变成几何距离的数学底座;RAG 的检索层 + 多模态对齐 + 聚类/推荐/异常检测等延伸应用
- 交叉引用 wiring(让新页不成孤儿):
  - [[Wiki/Entities/Methodology/Karpathy]] 的"Vibe Coding 命名"条目改为 wikilink
  - [[Wiki/Concepts/Methodology/Rag]] 正文的"Embedding（向量嵌入）"改为 wikilink + 相关节增一条
  - [[Wiki/Entities/Stock/UNiagaraScript]] 相关节增一条 DDC 回链
  - [[Wiki/Sources/Stock/NiagaraScript]] 涉及实体节增一条 DDC 回链
  - [[Readers/Niagara/Phase1-asset-layer-读本]] 第 472 行首次提到 DDC 改为 wikilink
  - [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]] 相关节增一条 Vibe Coding 链
- 🟢 小问题修补:
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]] line 16 `[[Wiki/Sources/Stock/]]` 目录式死链改为普通 code 样式
  - [[Wiki/Concepts/AIArt/Multi-lora-composition]] 相关节补一条 Caption-strategy 链
  - [[Wiki/Concepts/AIApps/Harness-engineering]] 相关节补 Llm / Mcp / Reasoning-model / Vibe-coding 四条,解决交叉密度偏低
- 更新:
  - [[index]] — Concepts 的方法论/UE4/AI 应用生态三个分区各新增一条
  - [[Wiki/Overview]] — "待建页"列表去掉 Embedding / Vibe Coding,加一行"已补建(2026-04-20)"标注;"下一步建议"的 AI 应用一行同步更新
- 回查 lint 报告剩余项的状态:
  - ✓ `FNiagaraSystemInstance`(7 处) — 留到 Phase 3 自然 ingest,不急
  - ✓ [[Wiki/Sources/AIApps/AI-primer-v2]] 缺 `aliases` — 尚未补,可下一轮 lint 一并处理(属 cosmetic)
  - ✓ `Niagara-vs-cascade` vs `Niagara-cpu-vs-gpu模拟` 对 Cascade GPU 能力措辞强弱不一致 — 未改(不算矛盾,只是措辞),留待合适时机统一口径
- 本轮合计影响文件:16 个(3 新建 + 13 更新)

---

## [2026-04-20] refactor | lint 🔴 项清理 — Claudian 页 + 读本回链 × 13 + cheatsheet 去引

- 触发:2026-04-20 lint 后用户决策:(1) Claudian 选方案 a 建实体页;(2) cheatsheet 已人工删除,清理所有残留引用
- 新建: [[Wiki/Entities/Claudian]] — 本仓 LLM 作者身份实体页,固化签名约定和运行模型,同时消解读本模板 Reader.md 的 `[[Claudian]]` 占位问题(原 6 处读本签名 + 1 处 learning-path 的 broken link 自动解析)
- 更新(读本回链,13 页,在"引用来源"节新增"主题读本(推荐通读)"行):
  - AIApps 8 页: [[Wiki/Concepts/AIApps/Llm]]、[[Wiki/Concepts/AIApps/Hallucination]]、[[Wiki/Concepts/AIApps/Context-window]]、[[Wiki/Concepts/AIApps/Ai-agent]]、[[Wiki/Concepts/AIApps/Mcp]]、[[Wiki/Concepts/AIApps/Harness-engineering]]、[[Wiki/Concepts/AIApps/Agent-skills]]、[[Wiki/Concepts/AIApps/Reasoning-model]] → 全部回链 [[Readers/AIApps/AI-primer-v2-读本]]
  - AIArt 5 页: [[Wiki/Concepts/AIArt/Lora]]、[[Wiki/Concepts/AIArt/Base-model-selection]]、[[Wiki/Concepts/AIArt/Caption-strategy]]、[[Wiki/Concepts/AIArt/Trigger-word]]、[[Wiki/Concepts/AIArt/Multi-lora-composition]] → 全部回链 [[Readers/AIArt/Lora-深度指南-读本]]
  - 执行:用 sed 批量替换"## 引用来源"节下的 source 行,7+4 个页面(Llm/Lora 用 Edit 先行完成)
- 删除引用(cheatsheet 用户已手动删除源文件,清理残留):log.md:230 原条目"[[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat-cheatsheet]] — 一页纸精简版..." 已去除。其余 line 224 的"一页纸简化版"只是"下一步猜测",非实际引用,保留
- 更新:[[index]] — Entities 方法论分区新增 Claudian 条目
- lint 报告三个 🔴 项(读本孤立 / cheatsheet 丢失 / Claudian broken link)全部解决;🟡🟢 未动,留待下一批次
- 副产品:读本模板 [[Wiki/_templates/Reader.md]] 末尾签名 `*本读本由 [[Claudian]] 生成*` 从此不再是 broken link
- 下一步:🟡 建 DDC / Vibe-coding / Embedding 三个高频缺失概念页;🟢 修若干小问题(Niagara-learning-path:16 目录 link、Multi-lora 缺 Caption-strategy 链、Harness 交叉密度等)

---

## [2026-04-20] lint | 首次体检 — 读本回链缺口 + 4 个缺失概念页 + 1 个丢失文件

- 触发:用户首次使用 lint 功能
- 范围:Wiki/ 下 47 个非模板页 + 根目录 index/README/log/Overview,排除 `_templates/` 占位符
- 方法:并行 grep 全量 wikilink + frontmatter,派 Explore agent 交叉分析,手动验证 2 个可疑点

### 关键发现

1. **系统性:读本被孤立** — 13 个 AIApps/AIArt 概念页没有一个回链对应主题读本
   - AIApps 8 页(Llm/Hallucination/Context-window/Reasoning-model/Ai-agent/Mcp/Harness-engineering/Agent-skills)未回链 [[Readers/AIApps/AI-primer-v2-读本]]
   - AIArt 5 页(Lora/Base-model-selection/Caption-strategy/Trigger-word/Multi-lora-composition)未回链 [[Readers/AIArt/Lora-深度指南-读本]]
   - 结果:[[Readers/AIApps/AI-primer-v2-读本]] 全无非索引入链(接近孤儿)
   - **根因**:本仓的读本范式是在概念页之后才定型的(§3.4),历史概念页尚未按新模板回链
   - **建议**:批量补"深入阅读"节回链,可作为一次小型 refactor

2. **丢失文件:`How-to-prompt-ai-chat-cheatsheet`**
   - log.md:173 声称 2026-04-19 已创建此页("一页纸精简版"),但文件系统里不存在
   - 未在后续 commit 中看到删除记录
   - **建议**:确认是否需要补建,或从 log 移除该行(加"未落地"标注)

3. **损坏链接:`[[Claudian]]`** — 6 处读本签名 + 1 处 Niagara-learning-path 指向不存在的页面
   - 根因:读本模板 Reader.md 末尾签名约定 `*本读本由 [[Claudian]] 生成…*`,但 `Wiki/Entities/Claudian.md` 未建
   - **建议**两选一:(a) 建 `Wiki/Entities/Claudian.md` 作为本仓 LLM 作者实体;(b) 改签名为纯文本"由 Claudian 生成",避免占位式 broken link

4. **缺失概念页(高频术语但无独立页)**:
   - `FNiagaraSystemInstance`(7 处) — Phase 3 自然会建,暂不急
   - `DDC / Derived Data Cache`(6 处) — 建议建 [[Wiki/Concepts/UE4/UE4-ddc]],Niagara 编译身份证直接作 DDC key
   - `Vibe Coding`(6 处) — Overview 已列待建;建议建 [[Wiki/Concepts/Methodology/Vibe-coding]]
   - `Embedding`(5 处) — Overview 已列待建;建议建 [[Wiki/Concepts/AIApps/Embedding]]

5. **其他小问题**
   - `Niagara-learning-path.md:16` 指向目录 `[[Wiki/Sources/Stock/]]`(非页面),应删或改具体文件
   - `Multi-lora-composition` 不链 `Caption-strategy`(两者逻辑紧密),建议补
   - `Harness-engineering` 与其他 AIApps 概念页交叉密度偏低,建议补 `Llm / Reasoning-model / Mcp` 三条
   - `Wiki/Concepts/Niagara/Niagara-vs-cascade` 与 `Niagara-cpu-vs-gpu模拟` 对 Cascade GPU 能力描述措辞强弱不一致(轻微,不算矛盾)
   - `Sources/AIApps/AI-primer-v2.md` 缺 `aliases` 字段(其他 source 页均有)

### 不用处理的

- index.md 已覆盖全部 47 页 ✓
- 所有 Stock 代码页 frontmatter 完整(repo/source_root/source_path/source_ref/source_commit 齐备) ✓
- 所有读本 frontmatter 合规(type: synthesis + tags 含 reader) ✓
- Niagara 5 个实体页 ↔ Phase1 读本双向链完整 ✓
- 无"updated < created"时间异常 ✓

### 下一步建议(按优先级)

1. 🔴 **高**:查清 `How-to-prompt-ai-chat-cheatsheet` 丢失原因(补建 or log 标注)
2. 🟡 **中**:决定 `[[Claudian]]` 处理方式 → 批量修 7 处 broken link
3. 🟡 **中**:批量补 AIApps/AIArt 概念页的读本回链(13 页 × "深入阅读"节)
4. 🟢 **低**:建 DDC / Vibe-coding / Embedding 三个高频缺失概念页
5. 🟢 **低**:修 `Niagara-learning-path.md:16` 的目录 link

等待用户指示是否要在本轮就动手修,还是只记录等批次处理。

---

## [2026-04-20] refactor | 新增元规则:新流程/规则的归处分层判断
- 触发:用户指令"以后有涉及到新的流程的时候你都考虑一下是需要写到 CLAUDE.md 里还是单独开一个文档,包括这条也是"
- 元规则本身的归处分析(自洽应用):
  - 性质:每次考虑扩 schema 时的元判断
  - 频率:always-apply
  - 结论:进 CLAUDE.md §7,作为原则 6,和既有"不把 CLAUDE.md 当作不可变"(原则 5)配对
- CLAUDE.md §7 新增原则 6(全文):
  > 新流程/规则落地前先做分层判断:问一句"这是每轮对话都要遵守的,还是只在做 X 时才需要的?"
  > - 每轮都要守 → 进 CLAUDE.md 某一节,尽量简短
  > - 只在做 X 时要 → 外移到独立文件(Wiki/_templates/、议题的 wiki 页、或专门 playbook)
  > - CLAUDE.md 只放 always-apply 的核心,防 context rot
- 更新 [[README]]:
  - 三层架构图的 Schema 层注明"CLAUDE.md + Wiki/_templates/",Wiki/ 树加 `_templates/` 一栏
  - 新增一段 "Schema 分层设计" 解释"always-apply vs conditional"原则,指向 CLAUDE.md §7 原则 6
- 这条 refactor 是本轮 CLAUDE.md 瘦身的自然延续——瘦身是一次性动作,**分层判断原则**把这次的治理经验固化为未来的规则
- 下一步:任何后续 schema 扩展都应先问这一句;如果判断模糊就先在 log 里记录,待 2-3 次同类情形后回看规律再做选择

## [2026-04-20] refactor | CLAUDE.md 分层瘦身 — 模板外移,防 context rot
- 触发:用户关心的不是 token 成本,而是 CLAUDE.md 膨胀导致 context rot(长对话中 schema 内容占用注意力,挤占硬约束的遵守可靠性)
- 核心洞察:CLAUDE.md 里**每轮对话都要遵守的核心规则** vs **条件性使用的详细模板** 应当分层——前者 always-loaded,后者 on-demand Read
- 方案:
  - 新建 `Wiki/_templates/` 目录(4 份模板):
    - `Entity-Concept.md`(~50 行)— Entity/Concept 页结构
    - `Source.md`(~36 行)— 非代码 raw source 摘要
    - `Code-Source.md`(~54 行)— 代码源摘要,含 frontmatter + 代码片段引用
    - `Reader.md`(~82 行)— 主题读本完整骨架 + header 里嵌入 §3.4 硬性要求备查
  - CLAUDE.md 瘦身:468 行 / 21 KB → **274 行 / 11 KB**(-41%)
    - 删除纯考古:§3.4 历史债清单(8 行,事实永久在 log.md)
    - 删除冗余:§2 各子目录逐条自注释("Methodology/ - 方法论相关概念"等 ~15 行)、§5 log 三示例压到一示例、§3.4 "读本 vs 原子页"表与 §2 分工说明的重复表述
    - 外移:§4.2-4.5 四份完整 markdown 模板(原 ~130 行)压为 §4 一张"页面类型 → 模板文件"指针表
    - 压缩:§9.4 VCS 约定文字精简,§2 目录结构去冗余注释
- 预期收益:
  - **每轮对话 CLAUDE.md 占用从 ~7K tokens 降到 ~3.7K tokens**(-47%)
  - 更关键:剩下的 274 行几乎全是 always-apply 规则,噪声比显著下降
  - 长对话中对 §7 Don'ts、§3.4 读本规则等硬性约束的遵守可靠性应当上升
  - 代价:创建新页时多一次 `Read Wiki/_templates/<type>.md`(每次 50-80 行),但这是 task-local 聚焦,与"每轮全量加载"完全不同的 attention 经济学
- 设计原则(写进 CLAUDE.md 头部 preamble):"本文件只含每轮对话都要遵守的核心规则。大块页面模板另存 `Wiki/_templates/`,仅在真的要创建对应页面时 Read"
- 下一步:观察后续几次 ingest/读本创建的实际体验;如果模板外移后某个模板高频被 Read,考虑把核心一句话约束提回 CLAUDE.md

## [2026-04-20] refactor | 读本独立顶层目录 Readers/
- 触发:用户指出读本散在 `Syntheses/Methodology/`、`Syntheses/AIArt/`、`Syntheses/AIApps/`、`Syntheses/Niagara/` 四个子目录里,和普通专题(三段论演进、How-to-prompt、学习路径总图)混在一起"有点乱,到处都是"
- 方案:新增 `Readers/` 顶层目录,5 份读本全部迁过去;`Syntheses/` 回归原定位——非读本类的专题综合
- 迁移(全部用 `git mv` 保留历史):
  - `Wiki/Syntheses/Methodology/Llm-wiki-方法论-读本.md` → `Readers/Methodology/Llm-wiki-方法论-读本.md`
  - `Wiki/Syntheses/AIArt/Lora-深度指南-读本.md` → `Readers/AIArt/Lora-深度指南-读本.md`(`Wiki/Syntheses/AIArt/` 本来只有读本,搬完即删空目录)
  - `Wiki/Syntheses/AIApps/AI-primer-v2-读本.md` → `Readers/AIApps/AI-primer-v2-读本.md`
  - `Wiki/Syntheses/Niagara/Phase0-心智模型-读本.md` → `Readers/Niagara/Phase0-心智模型-读本.md`
  - `Wiki/Syntheses/Niagara/Phase1-asset-layer-读本.md` → `Readers/Niagara/Phase1-asset-layer-读本.md`
- CLAUDE.md 更新:
  - §2 目录说明:新增 `Readers/` 顶层目录的完整描述 + 明确 "`Syntheses/` vs `Readers/` 的分工"小节
  - §3.4:归档路径从 `Wiki/Syntheses/<topic>/` 改为 `Readers/<topic>/`,新增"文件命名与路径"具体示例
  - §4.5:模板标题注明路径 `Readers/<topic>/`
  - 历史债清单(§3.4 末尾)3 条 ✅ 条目的链接同步更新到新路径
- 连带同步改链接(用 sed 跨 12 个文件批量替换):
  - 5 份读本之间的互相引用
  - 5 个 Phase 1 Entity 页的"深入阅读"链接
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 0/1 顶部读本引流
  - [[index]]、[[Wiki/Overview]]、以及历史 log 条目里的 wikilink
- [[index]] 分区重构:`Readers(主题读本)` 提升为**顶层分区**,与 `Entities / Concepts / Sources / Syntheses` 平级(不再作为 Syntheses 的子分区);Syntheses 分区回到"非读本类专题综合"的精简列表
- 本条 log 描述时的路径均为**迁移后新路径**(旧路径仅在本条内提到一次)
- 下一步:读本独立目录生效,路径稳定,之后 Phase 2 读本直接入驻 `Readers/Niagara/`

## [2026-04-20] synthesis | 历史债清算 — 三份主题读本一次性补齐
- 触发:4-19 CLAUDE.md §3.4 引入"主题读本"规则时,历史上三个主题(方法论/AIArt/AIApps)因 ingest 时规则尚未存在而未产出读本,已在当时钉为"历史债清单"。今日用户命令清算
- 新建 3 份读本,每份都按 CLAUDE.md §4.5 通用骨架(问题驱动叙事 + 代码/原文 inline + 陷阱 ⚠️ 高亮 + 深入阅读指回原子 + 开放问题):
  - [[Readers/Methodology/Llm-wiki-方法论-读本]] — 方法论议题读本,~500 行,叙事主线"时间线(Memex 1945 → RAG → LLM Wiki 2026)+ 哲学线(关联文档 → 查询时拼碎片 → 持续编译)"交织,五节 Memex/RAG/方法论骨架/Karpathy/本仓库具体化,末尾举 2026-04-17 第一次 ingest 作为完整例子
  - [[Readers/AIArt/Lora-深度指南-读本]] — AI 美术议题读本,~900 行,叙事主线"战略(为什么离开 MJ,三个结构性短板)→ 技术(LoRA 原理 → 基座选型 → Caption 反常识 → Trigger Word → Multi-LoRA)→ 工具(kohya_ss + ComfyUI)→ 工程(6 个月路线图 + 合规 + 硬件 + 评估准则)"
  - [[Readers/AIApps/AI-primer-v2-读本]] — AI 应用生态议题读本,~1000 行,叙事主线"LLM 本质 → 三个怪癖(幻觉/Context Rot/不稳定)→ 新面孔(推理/多模态/Agent)→ 连接外部(MCP/RAG/记忆)→ 三段论演进 → Harness 四柱 → Agent Skills → 2026 技术栈三层 → Karpathy 三节点 → Vibe/Spec/Harness Coding"
- 更新:
  - [[CLAUDE]] §3.4 历史债清单全部标 ✅,明确"今后新主题 ingest 同步产出读本,不再累积"
  - [[index]] Syntheses 分区新增方法论/AIArt 两个分类,三份读本各自登记
  - [[Wiki/Overview]] 三个主题段落顶部各加"📖 主题读本(推荐初读)"引流
- 总产出:3 份读本,约 **2400 行**叙事内容
- 收获/洞察:
  - 三份读本互相交织——方法论读本引用 AIApps 的 Karpathy,AIArt 读本引用方法论读本作为工作范式,AIApps 读本引用 AIArt 作为 AI 应用的具体落地。读本之间形成**内聚但可独立消化**的知识网络
  - 写读本比写原子页**压力大很多**——必须做完整叙事,不能跳章节,不能省略 trap。但读完单篇就"拿走全部知识"的体验远超读多篇原子页
  - CLAUDE.md §4.5 的模板和"问题驱动 / 代码 inline / 陷阱 ⚠️"硬性要求实际写作中非常好用,保证了三份读本的一致风格
- 下一步:读本规则全面生效,Phase 2(Niagara Component 层)起按新规则直接产出读本。AIArt / AIApps 新增原子页时,对应读本相关章节同步更新

## [2026-04-19] refactor | 导读 → 主题读本,规则泛化到所有议题
- 触发:用户指出"导读"命名有歧义(听着像摘要),而实际要的是**详细、精确、满满当当、一次读完即掌握全部知识、不用跳来跳去**的长文
- 同时指出:这种产物不应只存在于结构化学习路径(如 Niagara 各 Phase),**应覆盖所有议题**——任何有深度的 topic(扫盲源、方法论专题、跨源综合)都该有配套的人类阅读入口
- 改名:`Phase0-心智模型-导读.md` → `Phase0-心智模型-读本.md`;`Phase1-asset-layer-导读.md` → `Phase1-asset-layer-读本.md`(用 `git mv` 保留历史)
- CLAUDE.md 升级:
  - §3.4 重写 "Phase 导读" → "**主题读本(Topic 级线性读物)**",**触发条件从"学习路径阶段收尾"泛化到四种**(结构化路径 Phase / 新主题首次入驻 ≥3 原子页 / 多源交叉综合 / 用户显式要求)
  - 明确界定:"读本不是摘要",四条硬性特征(详细/精确/满满当当/一次读完)
  - 新增"原子页 vs 读本"分工对照表,明确两者互补不替代
  - 新增**历史债清单**:AIArt、AIApps、Methodology 三个主题当时 ingest 时没建读本,用户需要时按清单回补
  - §4.5 模板从 "Phase 导读" 重命名为 "主题读本",改写为通用骨架(适用所有议题,非仅学习路径)
  - §4.2 "Code Entity 紧凑约定" 中的 "Phase 导读" 改称 "主题读本"
- 连带同步改链接:
  - 5 个 Entity 页(Phase 1):`导读 § N` → `主题读本 § N`
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]]:Phase 0/1 顶部"线性读物"提示统一改为"主题读本"
  - [[index]]、[[Wiki/Overview]]:所有 Phase 0/1 链接同步
- 历史 log 条目(4-19 Phase 0 导读补齐、Phase 1 导读 + 方法论升级)**保留原文**,如实记录当时称"导读"的事实
- 下一步:Phase 2 按新规则产出 "Phase 2 读本";用户需要时可回补 AIArt / AIApps 主题读本

## [2026-04-19] synthesis | Niagara Phase 0 导读(补齐)
- 新建:[[Readers/Niagara/Phase0-心智模型-读本|Phase0-心智模型-导读]] — Phase 0 的线性读物,把四个概念页(UObject / Asset-Instance / Niagara-vs-Cascade / CPU-vs-GPU)编成自下而上一条叙事链  *(注:4-19 晚些时候重命名为"读本",详见本条上方的 refactor 条目)*
- 叙事结构:四层脑内地图(Layer 1 UObject → Layer 2 Asset/Instance → Layer 3 Niagara 哲学 → Layer 4 CPU/GPU 分叉),每层末尾小结 + 最后一节"四层地图回看"贯通
- 埋雷:第 2.8 节专门提前钉死"`FNiagaraEmitterHandle::Instance` 不是运行时 Instance"这个 Phase 1 必踩的命名陷阱,让读者到 Phase 1 时有预期
- 更新:[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 0 节顶加导读链接)、[[index]]、[[Wiki/Overview]]
- 动机:补齐方法论升级后 Phase 0 的导读缺位;今后任何 Phase 完成都要有配套线性读物

## [2026-04-19] synthesis | Niagara Phase 1 导读 + 方法论升级
- 触发:用户指出原子化 Source/Entity 页不符合人类线性阅读习惯(频繁跳转、碎片化)
- 核心产出:[[Readers/Niagara/Phase1-asset-layer-读本|Phase1-asset-layer-导读]] — Phase 1 的**教科书章节**,500+ 行线性叙事,从 Content Browser 切入讲到图源抽象基类,关键代码片段 inline,不强制跳转  *(注:4-19 晚些时候重命名为"读本")*
- 方法论升级(写入 [[CLAUDE]]):
  - 新增 §3.4 "Phase 导读":结构化学习路径每阶段收尾强制产出线性读物,定位"教科书章节"与原子页 spec 角色互补;准确/不遗漏 > 简短,不为压缩而压缩
  - 新增 §4.5 "Phase 导读页结构"模板
  - 修订 §4.2 "Code Entity 页的紧凑约定":单 source 派生的 entity 页控制在 30-50 行,字段细节下沉到 Source,叙事下沉到导读,Entity 只作稳定跨 source 入口
- Entity 页瘦身(按新约定):[[Wiki/Entities/Stock/UNiagaraSystem]]、[[Wiki/Entities/Stock/UNiagaraEmitter]]、[[Wiki/Entities/Stock/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/UNiagaraScript]]、[[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] 全部重写,每页 35-55 行,保留"一句话角色 + 核心字段速查 + 常用方法 + 深入阅读指回 Source/导读 + 开放问题"
- 更新:[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 1 节顶部加导读链接)、[[index]](Syntheses/Niagara 分区登记导读)、[[Wiki/Overview]](Niagara Phase 1 行加导读引流)
- 收获:
  - 原子页(LLM 检索友好)+ 导读(人类阅读友好)的双层产出,今后作为结构化学习路径的标准动作
  - Entity 的合理边界:单 source 派生时瘦身;多 source 交叉时再扩为汇总枢纽
- 下一步:Phase 2(Component 层,3 文件)按新模板走,ingest 完收尾产一份简版导读(预计 250 行左右)

## [2026-04-19] ingest | Niagara Phase 1 — Asset 层三件套(5 文件)
- 代码源(stock,`b6ab0dee9`,UE 4.26)5 个文件,全部在 `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/`:
  - `NiagaraSystem.h`、`NiagaraEmitter.h`、`NiagaraEmitterHandle.h`、`NiagaraScript.h`、`NiagaraScriptSourceBase.h`
- 新建(10 页):
  - Sources (5): [[Wiki/Sources/Stock/NiagaraSystem]]、[[Wiki/Sources/Stock/NiagaraEmitter]]、[[Wiki/Sources/Stock/NiagaraEmitterHandle]]、[[Wiki/Sources/Stock/NiagaraScript]]、[[Wiki/Sources/Stock/NiagaraScriptSourceBase]]
  - Entities (5): [[Wiki/Entities/Stock/UNiagaraSystem]]、[[Wiki/Entities/Stock/UNiagaraEmitter]]、[[Wiki/Entities/Stock/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/UNiagaraScript]]、[[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]
- 更新:[[index]](新增 "Niagara 代码实体" 分区 + "代码源摘要 — stock" 分区)、[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 1 全部打勾 + 标注完成日期)
- 要点:
  - **Asset 三件套的关系网清晰了**:`UNiagaraSystem`(容器,持有 `TArray<FNiagaraEmitterHandle>`)→ `FNiagaraEmitterHandle`(USTRUCT 包装,含唯一 Id + Name + enabled + `UNiagaraEmitter*`)→ `UNiagaraEmitter`(脚本 + 渲染器 + 继承 merge)→ `UNiagaraScript`(编译后产物,字节码 + GPU shader 双形态)
  - **关键命名陷阱**:`FNiagaraEmitterHandle::Instance` 是 Emitter 资产副本(仍在 Asset 层),**不是**运行时 Instance(后者叫 `FNiagaraEmitterInstance`,Phase 3 再看)
  - **EmitterSpawn/EmitterUpdate 脚本不可单独编译**(`UNiagaraScript::IsCompilable` 对这两者返 false),它们被合并进 System 脚本
  - **editor/runtime 模块分离**:`UNiagaraScriptSourceBase` 是抽象基类在 `Niagara`,实现 `UNiagaraScriptSource + UNiagaraGraph` 在 `NiagaraEditor`,这是"runtime 能持图源指针但不 link editor 图实现"的解耦点
  - 编译产物三件套:`FNiagaraVMExecutableDataId`(身份证/DDC key)+ `FNiagaraVMExecutableData`(字节码 + 属性 + DI 绑定)+ `FNiagaraShaderScript`(GPU 侧,Phase 8)
- 下一步:Phase 2 — `NiagaraComponent.h / NiagaraActor.h / NiagaraFunctionLibrary.h`(Component 层,从场景入口视角把 Asset 连到 World)
- 待 Phase 3+ 回看的 open questions(分散记录在各页"开放问题"节):
  - `EmitterExecutionOrder.kStartNewOverlapGroupBit` 的 parallel tick 消费点
  - `RapidIterationParameters` vs `ExposedParameters` vs `User.*` 命名空间的协作关系
  - `FNiagaraVMExecutableData::DIParamInfo` 的技术债(GPU 信息不该在 VM 数据里)
  - `UNiagaraEmitter::Parent / ParentAtLastMerge` 的 merge 语义(能传播什么)

## [2026-04-19] ingest | AI 应用技术发展脉络与核心概念扫盲手册 v2 — 新主题 AIApps 入驻
- source: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]（Eureka × Claude，面向零基础读者的 AI 应用生态综合扫盲）
- 上游素材：B 站视频 BV1zSDMBUE5o + 配套飞书文档
- 处理流程：v1 由 Eureka 基于视频/飞书先让 Claude 起草 → Eureka 邀请本 Claudian 做事实审阅 → 审出 v1 若干瑕疵（Vibe Coding 归属、Fowler 三分类/四支柱自相矛盾、MCP 捐赠细节、agentskills.io 域名疑点等）+ 建议补缺（幻觉/推理模型/多模态/Embedding/Subagent/HITL 等）→ Claudian 重写 v2 → v2 ingest 入库 → v1 删除
- 新建主题目录：`Wiki/{Concepts,Entities,Sources,Syntheses}/AIApps/`
- 新建：
  - Source (1): [[Wiki/Sources/AIApps/AI-primer-v2]]
  - Concepts (8): [[Wiki/Concepts/AIApps/Llm]]、[[Wiki/Concepts/AIApps/Hallucination]]、[[Wiki/Concepts/AIApps/Context-window]]、[[Wiki/Concepts/AIApps/Reasoning-model]]、[[Wiki/Concepts/AIApps/Ai-agent]]、[[Wiki/Concepts/AIApps/Mcp]]、[[Wiki/Concepts/AIApps/Harness-engineering]]、[[Wiki/Concepts/AIApps/Agent-skills]]
  - Synthesis (1): [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]]
  - Entity (1): [[Wiki/Entities/AIApps/OpenClaw]]
- 更新：
  - [[Wiki/Entities/Methodology/Karpathy]]（追加 Context Engineering 倡导者 + Vibe Coding 命名者两个身份；sources 1→2）
  - [[Wiki/Concepts/Methodology/Rag]]（补 Embedding 向量嵌入机制解释 + 与 AIApps/Hallucination、Mcp 的交叉引用；sources 1→2）
  - [[index]]（新增 AIApps 在 Entities/Concepts/Syntheses/Sources 四个分区的条目）
  - [[Wiki/Overview]]（主题从 3 扩至 4，新增 AIApps 小节、重写主题知识图、扩充开放问题）
- 删除：Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册.md（v1，被 v2 取代）
- 要点：
  - wiki 首次覆盖"AI 应用生态"主题，与既有 Methodology / Niagara / AIArt 并列
  - 主线叙事：Prompt → Context → Harness 三段论；核心洞察"模型正在商品化，竞争在 Harness 和 Skills 两层"
  - 对非技术读者的第一纪律已固化：**AI 给的具体事实必核验（幻觉的结构性导致）**
  - 修正 v1 的几个事实错误，同时补全了幻觉、推理模型、多模态、Embedding、Subagent、HITL、Context Rot 等 v1 缺失的关键概念
- 下一步：用户可能后续会：(a) 让我按 v2 补建 Transformer / Embedding / Vibe Coding 等提及但未建页面；(b) 让我为 wiki 跑一次 lint 检查 AIApps 内部交叉引用是否闭环；(c) 拿这套文档给团队推广时要求生成一页纸简化版

## [2026-04-19] query | 如何向 AI 提问（chat 场景）— 团队推广用
- 触发:团队内推广 AI 使用,观察到"结果质量 ↔ 使用意愿"的正反馈循环,但大多数人不知如何有效提问
- 归档为:
  - [[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat]] — 详细版（心智模型 + 5 要素 + 8 技巧 + 反模式 + 好/差对照 + 团队推广经验）
- 更新:[[index]](Syntheses 新增"方法论"分区)
- 核心洞察:糟糕的 prompt 来自错误的心智模型（把 AI 当搜索引擎/全知神谕）——校正心智模型后,技巧都是推论

## [2026-04-18] refactor | raw/wiki 顶层+笔记首字母大写化、UE 全大写
- 变更:顶层目录 `raw/` → `Raw/`、`wiki/` → `Wiki/`；所有 raw 子目录 `Articles/Papers/Books/Notes/Assets/`
- 变更:笔记文件名英文首字母大写：`overview → Overview`、`rag → Rag`、`memex → Memex`、`karpathy → Karpathy`、`llm-wiki-方法论 → Llm-wiki-方法论`、`niagara-* → Niagara-*`、`ue4-* → UE4-*` 等
- 变更:内容里所有 "UE"（含 `ue4`、`ue 官方的`、`ue4对象系统`）全大写为 `UE4`/`UE`
- 更新:所有 .md 文件的 wikilink 路径同步（.obsidian/app.json 的 `attachmentFolderPath` 改为 `Raw/Assets`）
- 更新:[[CLAUDE]](§2 目录结构 Raw/Wiki 大写、§3 路径、§4 模板、§5 log 示例、§6 链接约定、§9.5 ingest 路径)
- 踩坑:PowerShell 的 `-ne` 对字符串**默认大小写不敏感**，首次 batch 替换误判 "无变化"。改用 `.Equals()` 后正常

## [2026-04-18] refactor | wiki 目录大写化 + 主题分类
- 变更:所有 wiki 子目录改为大写开头（`concepts/` → `Concepts/` 等）
- 变更:各目录内按主题建子文件夹：`Concepts/{Methodology,UE4,Niagara}/`、`Entities/{Methodology,Stock,Project}/`、`Sources/{Methodology,Stock,Project}/`、`Syntheses/{Niagara}/`
- 迁移:10 个 wiki 文件移至新路径，全部内部 wikilink 同步更新
- 更新:[[CLAUDE]](§2 目录结构、§3 路径引用、§4 模板示例、§5 log 格式、§6 链接约定、§9.5 代码 ingest 路径)、[[index]]、[[Wiki/Overview]]（批量 wikilink 替换）

## [2026-04-17] bootstrap | 仓库初始化
- 按 Karpathy LLM Wiki 方法论搭建三层架构
- 新建目录:`Raw/{articles,papers,books,notes,assets}`、`Wiki/{entities,concepts,sources,syntheses}`
- 新建:[[CLAUDE]](schema)、[[index]](目录)、[[log]](本文件)、[[Wiki/Overview]](占位)
- 源文档:`Karpathy Wiki 方法论.md`(暂放根目录)
- 下一步:等待第一个 raw source。

## [2026-04-17] ingest | Karpathy — LLM Wiki
- source: [[Raw/Notes/Karpathy Wiki 方法论]](从根目录迁入 Raw/Notes/)
- 新建:
  - [[Wiki/Sources/Methodology/Karpathy-llm-wiki]]
  - [[Wiki/Concepts/Methodology/Llm-wiki-方法论]]
  - [[Wiki/Concepts/Methodology/Rag]]
  - [[Wiki/Concepts/Methodology/Memex]]
  - [[Wiki/Entities/Methodology/Karpathy]]
- 更新:[[index]](登记 5 个新页)、[[Wiki/Overview]](首次有实质内容,确立"当前主题=方法论自举")、[[CLAUDE]](修正 wikilink 指向 Raw/notes)
- 要点:自举式 ingest——用这套方法论本身 ingest 这套方法论。wiki 现在可以正式运转。
- 下一步:扔进来任意一个真实 source,验证流程。

## [2026-04-18] ingest | Niagara Phase 0 — 基础心智模型（4 概念页）
- 新建:
  - [[Wiki/Concepts/UE4/UE4-uobject-系统]]（UObject/UCLASS/UPROPERTY 宏、GC、反射、命名前缀、智能指针速查）
  - [[Wiki/Concepts/UE4/UE4-资产与实例]]（Asset vs Instance 二元模型、Component 桥梁、为什么分离）
  - [[Wiki/Concepts/Niagara/Niagara-vs-cascade]]（设计哲学对比、SIMD vs 逐粒子、脚本阶段说明）
  - [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]（VectorVM、Compute Shader、选择依据、源码分叉点）
- 更新:[[index]](新增 UE4 基础 + Niagara 基础两个 Concepts 分类)、[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 0 全部打勾)
- 要点:Phase 0 建立心智模型，无需读代码；重点是资产/实例二元模型 + CPU/GPU 分叉
- 下一步:Phase 1 — 读 NiagaraSystem.h / NiagaraEmitter.h / NiagaraScript.h

## [2026-04-18] synthesis | Niagara 源码学习路径制定
- 扫描 `stock` 仓 `Engine/Plugins/FX/Niagara/` 全部 7 个模块，749 个文件
- 新建:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 更新:[[index]](新增 Niagara Syntheses 分类)
- 要点:10 阶段学习路径，Phase 0(概念)→ Phase 9(世界管理)+ Phase 10(高级选修)，共约 69 个文件；每阶段含文件清单、学习要点、难度评级、进度 checkbox
- 下一步:从 Phase 0 概念页开始，或直接从 Phase 1 `NiagaraSystem.h` 开始逐文件 ingest

## [2026-04-20] refactor | Niagara Readers Phase 3-10 复查修正
- 背景:Phase 3-10 读本在上次 ingest 时 context 顶到 90%+,需要二次核查
- 方法:4 个 sub-agent 并行(P3+4 / P5+6 / P7+8 / P9+10),按源码 cross-check,再由主线挑战 agent 误报、直接访问源码验证
- 修正 4 个读本:
  - [[Readers/Niagara/Phase3-runtime-instance-读本]] §0 + §深入阅读 + 末尾签名:文件行数 574/239/429 → 573/238/428(off-by-one 修正)
  - [[Readers/Niagara/Phase6-rendering-读本]] §5.3:**删除编造的 5 个 Ribbon `mutable` tessellation 字段**(TessellationCurvature 等在 4.26 头文件里不存在);只保留真实配置字段 + 警告注释
  - [[Readers/Niagara/Phase6-rendering-读本]] §5.4:`ScaledUsingSegmentLength` → `ScaledUsingRibbonSegmentLength`(枚举真名)
  - [[Readers/Niagara/Phase8-gpu-simulation-读本]] §2.4:注明 `FNiagaraShaderMapId` 物理定义在 `NiagaraShared.h`,本节归在 ShaderMap 叙事下只因它是 Map 的 key
  - [[Readers/Niagara/Phase10-advanced-features-读本]] §6.2:补 `NeighborGrid3DRWInstanceData::bool SetGridFromCellSize` 字段 + 双驱动模式说明
- 确认无需改:Phase 3 `ENiagaraGPUTickHandlingMode = 5 值`(agent 误报 4)、Phase 5 `PaddedParameterSize uint32`(agent 误报 uint16)——已在源码直接 grep 反证
- 链接完整性:4 个 agent 独立 Glob 验证,全部 `[[...]]` 可解析,**零断链**
- 要点:重要教训——context 窗口爆满时容易幻觉字段名;复查必须跑到源码一行一行核对。sub-agent 也会误报,主线必须挑战其结论
- 下一步:Phase 11+? 学习路径已标记完结;若要扩充,可考虑 Editor 模块(NiagaraEditor)或 Niagara 材质/渲染与 UE 渲染管线的集成专题

## [2026-04-20] refactor | Niagara Readers Phase 4-10 补链接(结构性漏链)
- 背景:用户指出"断链"只是浅层,更严重的是**该写 wikilink 却没写**(只写纯文本)。以 [[Readers/Niagara/Phase0-心智模型-读本]]、[[Readers/Niagara/Phase1-asset-layer-读本]] 为标准复检
- 发现:Phase 6 仅 2 链、Phase 7/8/9 仅 1 链(只签名行的 Claudian)、Phase 10 的 Source/Entity 列表也是纯文本。远低于 Phase 1 的 21 链基线
- 修正涉及 7 个读本:
  - [[Readers/Niagara/Phase4-data-model-读本]]:Entity×7 文本 → wikilink;前置段补 Phase 0/1/2/3 跨读本链接 + Concept 链接;加 [[#深入阅读]] 指针
  - [[Readers/Niagara/Phase5-cpu-script-execution-读本]]:前置段"Phase 3 读本"/"Phase 4 读本"文本 → wikilink;新增下一步 Phase 6/8 链接 + Overview;加 [[#深入阅读]] 指针 + Phase 8 内联链接
  - [[Readers/Niagara/Phase6-rendering-读本]]:深入阅读整块重写,Source×10 / Entity×6 全部 wikilink,增加 Phase 2/3/4/5 前置跨读本链接、Phase 7 下一步链接、Concept 链接、Overview;header 加学习路径链接 + Asset/Instance Concept 内联链接 + [[#深入阅读]] 指针
  - [[Readers/Niagara/Phase7-data-interface-读本]]:同样重写,Source×9 / Entity×8 全 wikilink;补 Phase 2/3/4/5 前置、Phase 8 下一步、Phase 10 后续深入、CPU/GPU Concept 链接;header 加学习路径链接 + CPU/GPU Concept 内联链接 + [[#深入阅读]] 指针
  - [[Readers/Niagara/Phase8-gpu-simulation-读本]]:Source×13 / Entity×5 全 wikilink;分组呈现(Shader 编译链/GPU 基础/VertexFactory/Sort&Indirect);补 Phase 4/5/6/7 前置、Phase 9/10 下一步;header 加学习路径链接 + CPU/GPU Concept 内联 + [[#深入阅读]] 指针
  - [[Readers/Niagara/Phase9-world-management-读本]]:Source×6 / Entity×6 全 wikilink;补 Phase 2/3/7 前置、Phase 10 终点链接、Asset/Instance Concept 链接;header 补全
  - [[Readers/Niagara/Phase10-advanced-features-读本]]:Source×6 / Entity×5 全 wikilink;重构前置段用真实读本 wikilink;header 补全学习路径链接 + [[#深入阅读]] 指针
- 指标:wikilink 占用数从 Phase 6=2 / 7=1 / 8=1 / 9=1 涨到 20/30/31/21;Phase 4=15→24, Phase 5=11→20, Phase 10=12→27
- 全量校验(Python 脚本跑 Phase 0-10,共 247 条 wikilink 实例):**零断链**(仅 `[[Claudian]]` 依赖 Obsidian shortest-path 解析,实际文件在 Wiki/Entities/Claudian.md)
- 要点:上一轮复查只检查"已有链接是否断",漏了"该写却没写"的结构性缺失。**Lint 心智应包含"链接密度是否符合读本模板"**。未来 ingest 时注意 Reader 模板 §7 的"深入阅读"必须含 Source×N / Entity×N / 前置议题 / 下一步导航 / Claudian 签名 5 块结构
- 下一步:考虑给 [[Wiki/_templates/Reader]] 补一条"深入阅读必须完整 wikilink 化"的硬性要求

## [2026-04-20] refactor | Niagara Phase 3 读本全量核查与补完
- 方法:不再抽查,读完读本全部 683 行 + 3 个源文件全部内容(NiagaraEmitterInstance.h 238 + NiagaraSystemInstance.h 573 + NiagaraSystemSimulation.h 428 = 1239 行),逐条对比
- 修正 [[Readers/Niagara/Phase3-runtime-instance-读本]]:
  - **硬错误 × 2**
    - §5.6 binding 计数 "6 组" 与展示的 7 字段矛盾 → 改为 "2 + 5 共 7 组",解释方向(Component→System vs System→Emitter)
    - §4.6 `FEventInstanceData` 漏列 `UpdateEventGeneratorIsSharedByIndex` / `SpawnEventGeneratorIsSharedByIndex` 2 字段 → 补全并注释用途
  - **关键漏点 × 5(crit)**
    - §4.8 新增 **Emitter 自身的 ExecutionState**(单态机)—— Phase 3 叫"状态机"却只讲了 System 双态机,Emitter 单态机完全漏掉。含 `IsDisabled/IsInactive/IsComplete` + `ShouldTick` + `bResetPending`
    - §5.2 新增 **bSolo + bForceSolo 双标志**——和 RequestedState/ActualState 完全平行的"用户意图 vs 系统实际"设计,Simulation 自己也有 `bIsSolo`,三者协同
    - §5.1 补全 Simulation **4 数组 + 1 升级队列**的实例成员管理(SystemInstances / SpawningInstances / PausedSystemInstances / PendingSystemInstances / PendingTickGroupPromotions),不是简化的单 TArray
    - §5.4 新增 **Tick Group Promotion 机制**——Phase 3 的核心话题之一,原读本零字提及。含 `UpdateTickGroups_GameThread / AddTickGroupPromotion / CalculateTickGroup / UpdatePrereqs / TickBehavior`,解释为什么 WorldManager 索引键包含 TickGroup
    - §2.5 新增 **DI per-instance data blob 登记**——虽然细节 Phase 7 讲,但 SystemInstance 上的 `DataInterfaceInstanceData` 16 字节对齐 byte blob + `PreTick/PostTick` 数组 + `PerInstanceDIFunctions[ScriptType]` 数组,Phase 3 必须登记存储形态
  - **次要补充 × 15(nice)**
    - §2.1 补 `Deactivate / Complete / OnPooledReuse` 三种结束 API 的区别 + `bPooled` 对 Unbind 流程的影响
    - §2.4 `ManualTick` vs `AdvanceSimulation` 区分(两个不同方法,读本原来措辞容易混淆),加 `RequiresDistanceFieldData/DepthBuffer/EarlyViewData/ViewUniformBuffer` + `FeatureLevel`
    - §3.1 展开 `FInstanceParameters` 结构(ComponentTrans/DeltaSeconds/TimeSeconds/RealTimeSeconds/EmitterCount/NumAlive/TransformMatchCount/RequestedExecutionState)+ 说明"Concurrent 只读快照不回读"约束
    - §3.2 补 **Simulation 层 Wait**(`WaitForSystemTickComplete / WaitForInstancesTickComplete`),解释 Instance 级 Wait 与 Simulation 级 Wait 的分工
    - §4.5 补 **Emitter DirectBinding 快捷通道**(`SpawnIntervalBinding` / `InterpSpawnStartBinding` / `SpawnGroupBinding` / `SpawnExecCountBinding` / `UpdateExecCountBinding`)—— "无字符串查表" 的另一形态
    - §4.7 补 `CachedEmitterCompiledData TSharedPtr`——Asset→Instance 的编译产物快照桥,Pool 场景必要
    - §4.9 新增 **Emitter 运行时统计与 bitfield**(EmitterAge/InstanceSeed/TickCount/TotalSpawnedParticles/CPUTimeCycles/MaxRuntimeAllocation 等)
    - §5.5 展开 **Tick / Spawn 两条并列 pipeline** + `FNiagaraSystemSimulationTickContext` 结构(Owner/System/Instances/DataSet/DeltaSeconds/SpawnNum/EffectsQuality/...)+ 双工厂 `MakeContextForTicking / MakeContextForSpawning`
    - §5.6 补 `FNiagaraConstantBufferToDataSetBinding`(与 `FNiagaraParameterStoreToDataSetBinding` 并列的另一套)+ **6 个 Engine DirectBinding**(`SpawnNumSystemInstancesParam / UpdateNumSystemInstancesParam / SpawnGlobalSpawnCountScaleParam / ...`)—— Scalability 全局 scale 直写底层 byte buffer 的路径
    - §5.7 修 `SpawningDataSet` 描述从"level streaming 情境"改为"any out-of-tick spawn"(源码注释原文就是 outside of tick,不限 streaming)
  - §0 "10 条关键洞察"(原 7 条)—— 新增 Emitter 单态机 / Tick Group Promotion / 2+5+6 绑定总览 / DI blob 不归 DI 管 4 条新洞察
- 节结构:§4 现有 9 小节(新增 4.8 Emitter 状态机 + 4.9 运行时统计);§5 现有 9 小节(新增 5.2 Solo 双标志 + 5.4 Tick Group Promotion)
- 校验:
  - 15 条 wikilink / 14 unique,**零断链**
  - Python 脚本跑 26 个 subsection 编号,**无重号**,连续排列 2.1-2.5 / 3.1-3.3 / 4.1-4.9 / 5.1-5.9
  - 抽检 25+ 条技术声明(类签名 / 字段 offset / 方法签名 / enum 值)与源码完全一致
- 行数:683 → 940(+257 行,+37%)
- 要点:上轮"抽查"错过的最大漏点**不是具体字段错,而是整段机制缺位**(Emitter 状态机、Tick Group Promotion)。全量读的工作量:主 agent 读 ~1900 行(读本 683 + 3 源 1239),对价是发现 2 硬错 + 5 crit 漏点 + 15 nice
- 下一步:用户指示 Phase 3 后继续 Phase 4-10 的全量核查;节奏建议每 Phase 单独开工(读本 + 对应源文件),避免单 agent 上下文撑爆

## [2026-04-21] refactor | Niagara 11 篇读本 + Reader 模板新增"自检问题"章节
- 用户需求:每篇读本末尾要有"读完即可自测的深刻题",不是回原文检索的浅题
- 触发分层判断(CLAUDE §7.6):
  - 每轮都要守(读本生产规约)→ 进 CLAUDE.md §3.4 + Reader.md 模板
  - 题目本身(per-reader 内容)→ 直接落到各 reader 的新章节
- 改动:
  - 11 篇 Niagara 读本(`Readers/Niagara/Phase0..Phase10`)各加一节 `## 自检问题(读完回答)`,放在"关键洞察"之后、"留下的问题"之前
  - 每篇 5-8 题,均为综合 / 反推 / 取舍 / 假设题,典型句式:"为什么 X 必须这样设计?反过来会撞到什么?"、"在 <场景> 下选 A 还是 B?判断标准?"、"<某优化> 的代价转移到了哪里?"、"用 5 句话向不熟悉本议题的人解释 <核心概念>"
  - [[Wiki/_templates/Reader.md]]:硬性要求清单加一条 + 骨架插入新章节(含设计要点 + 模板)
  - [[CLAUDE.md]] §3.4:硬性要求一句话 + 一段详细规约(目的 / 题型禁忌 / 题型必须 / 数量 / 不在文中给答案)
- 设计原则:
  - **不能在原文检索某段直接抄出来答**——这种问题查 wiki 即可,不需要读本
  - **必须把多节内容串起来 / 反推设计动机 / 假设场景预测后果** 才能答
  - 不在文中给答案,留给读者自测
- 校验:11 个 Edit 全部 success,各 reader 的"自检问题"章节插入位置一致(均在"留下的问题"前),格式统一
- 影响:本规约对未来所有读本生产都生效;Niagara 11 篇也补全了

## [2026-04-21] lint | 全仓体检(规模 160 wiki + 15 reader + 4 raw,~1688 wikilink)
- 触发:用户 `lint`;上次 lint 2026-04-21(Phase 5-10 targeted)。首次在 175 页规模下做全仓扫描
- 执行:按 [[Wiki/_templates/Lint-checklist]] §0-§8,delegate general-purpose agent 扫全仓(>100 页 per checklist §0)
- 关键发现:
  - 🔴 无严重问题(零 orphan、零 frontmatter 不合规、log ↔ fs 对账 0 差异)
  - 🟡 **11 条 broken wikilink**(全部指向未建 Entity 页,集中在 Niagara Sources):
    - `FNiagaraDataBuffer`×4(NiagaraDataSet/FNiagaraDataSet/FNiagaraGPUInstanceCountManager/FNiagaraVertexFactory)
    - `FNiagaraFunctionSignature`、`FNiagaraDataSetID`(NiagaraCommon)
    - `FNiagaraParameters`(NiagaraParameters)、`FNiagaraVariableBase`(NiagaraParameterStore)
    - `FSimulationStageMetaData`(NiagaraScriptBase)
    - `FNiagaraScriptExecutionParameterStore` + `FNiagaraScriptInstanceParameterStore`(NiagaraScriptExecutionParameterStore)
    - 成因:ingest 时作前瞻链接,后决定不单建页但未改 plain text
  - 🟡 目录式死链 1 条:`Wiki/Concepts/UE4/UE4-ddc.md:75` 的 `[[Wiki/Sources/Stock/]]`
  - 🟢 [[Wiki/Overview]] L182 "仍待建页"列表陈旧:**Subagent** 已补建为 [[Wiki/Concepts/AIApps/Multi-agent]],应从列表删除
  - 🟢 [[Wiki/Concepts/AIApps/Embedding]] 仅 1 条非索引反链(`Rag.md`),枢纽性偏薄
  - 🟢 `TickGroup` 14 页反复提及无独立概念页——候选建 `Wiki/Concepts/UE4/UE4-tickgroup.md`,非硬需求
- 人工验证:抽查 4 个声称缺失的 Entity(`FNiagaraDataBuffer`/`FNiagaraParameters`/`FNiagaraVariableBase`/`FSimulationStageMetaData`)全部 fs 确认不存在;Overview L182 确认仍列 Subagent
- 建议下一步(待用户批准):
  - 🟡 修 11 条 broken link:建议策略 A(改 plain text + 父 Source 内描述注脚),工作量小,保持原子页紧凑
  - 🟡 UE4-ddc.md:75 目录链改具体页面或 code 样式
  - 🟢 更新 Overview L182 删 Subagent
- 不处理(含理由):
  - Niagara-vs-cascade / CPU-vs-GPU 对 Cascade GPU 措辞差异(上轮已标"轻微不一致");非矛盾,留到主题 refactor 时一次处理
  - `_templates/` 内的占位符(预期)
  - log.md append-only 历史条目用旧 stem 指向已改名读本(有意保留,grep 回溯用)

## [2026-04-22] refactor | README 加头图
- 触发:用户想给 README 加头图(粘贴了一张鸣潮风格废墟角色图)
- 改动:
  - [[README.md]] H1 标题下、blockquote 上插入标准 markdown `![](Raw/Assets/ReadmeHeader.jpeg)`——用 `![]()` 而非 `![[]]` 以兼容 GitHub 渲染
  - Asset 文件 `Raw/Assets/ReadmeHeader.jpeg`(2.4 MB JPEG,用户命名 PascalCase),放 `Raw/Assets/` 符合 CLAUDE §6 附件约定
- 影响:`Raw/Assets/` 首次入驻一张图片(之前仅 `.gitkeep`)

## [2026-04-25] synthesis | Niagara / Material AI 理解与生成管道 —— 正反两向 uasset 议题
- 触发:Eureka 在读完"代码问答机器人"和"AI 贴图工具"两份读本后,想扩展其他落地方向。一轮头脑风暴后(8 个候选方向),用户聚焦到"Niagara / Material 理解器"单点;随后追问两件事——(a) 读不了 .uasset 的解法按"复用性/易用性/准确度"评估 (b) 是否可能逆向(自然语言→文本→uasset)。另一会话在讨论期间完成了 AIApps→AIAgents 主题拆分,本次产出直接落 AIAgents。
- 代码验证(stock 4.26):
  - `Engine/Source/Editor/MaterialEditor/Public/MaterialEditingLibrary.h` —— CreateMaterialExpression / ConnectMaterialExpressions / ConnectMaterialProperty / RecompileMaterial / LayoutMaterialExpressions 全部 BlueprintCallable,Material 编辑 API 完整
  - `Engine/Plugins/FX/Niagara/Source/NiagaraEditor/Public/NiagaraClipboard.h` —— Niagara T3D copy/paste 通道存在
  - `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceArrayFunctionLibrary.h` —— Niagara BlueprintCallable 全是 runtime(Set/Get Array),**没有**编辑器创建 API,证实 4.26 Niagara 编辑层不开放 Python
  - `Engine/Source/Runtime/Engine/Classes/Materials/Material.h` L833 —— `TArray<UMaterialExpression*> Expressions`
  - `Engine/Source/Runtime/Engine/Classes/Materials/MaterialExpression.h` L23 —— `struct FExpressionInput`
- 新建:
  - [[Wiki/Concepts/AIAgents/Uasset-textualization]] —— uasset→文本代理的概念抽象,含三种方案族 T3D / Python dumper / CUE4Parse
  - [[Wiki/Syntheses/AIAgents/Niagara-material-ai-comprehension]] —— 正反双向管道设计综合页(方案评估矩阵 / Material vs Niagara 4.26 不对称 / 双层输出 / 语义模型三张表 / 逆向三重 guardrail / POC 次序)
  - [[Readers/AIAgents/给 AI 看懂 Niagara 和材质 - 正反两条管道]] —— 主题读本,~600 行,含 UMaterialEditingLibrary Python 伪代码 / Niagara 模板化逆向流程图 / 10 条关键洞察 / 8 题自检 / 代码索引
- 更新:
  - [[index.md]]:AI Agents Concepts 加 Uasset-textualization;AI Agents Readers 加新读本;AI Agents Syntheses 加新综合页;头部"最后更新"改写为本轮
- 关键决策:
  - 方案选型按资产类型拆分 —— Material 正/逆皆走 Python + EditingLibrary;Niagara 正向走 T3D(Clipboard)+ 反射混合,逆向走 R3 模板化克制路径(不允许从零生成,只改允许参数)
  - 逆向三重 guardrail:路径隔离(强制 /Game/AI_Generated/ 前缀)+ 编译必过才保存(RecompileMaterial 为天然单元测试)+ 溯源元数据(prompt / 时间 / 模板参照)
  - POC 次序:Material 正向 → Material 逆向 → Niagara 正向 →(选做)Niagara 逆向;**绝对不要四格齐开**,Material 证明通路后才投 Niagara
  - Compounding 机制:正向 dumper 积累三张语义模型表(节点白名单 / 参数值域 / 命名模式),直接作为逆向 generator 的约束,两向共享同一块 wiki
- 要点:
  - .uasset 是二进制反射序列化流,不是符号化系统,agentic grep 基本失效 —— 这是代码问答机器人读本点名的盲区,本议题正面解
  - Material 和 Niagara 在 4.26 下编辑器 API 成熟度**差一档**,套同一方案会翻车
  - 逆向比正向危险一档:错了直接污染版本库,必须三重 guardrail 硬约束
- 下一步:
  - 等用户决定是否进 POC;若进,从 Material 正向 dumper 起步,1-2 周跑通
  - 未来 DataTable / Blueprint 资产理解时,复用 Uasset-textualization 概念页的方案族框架
- 自验证(Glob):4 个新路径(1 概念 + 1 综合 + 1 读本 + 本条 log 引用)全部确认存在

## [2026-04-26] synthesis | 桌宠作为 AI 应用入口议题首次入驻
- 触发:Eureka 想做桌宠作为日后所有 AI 应用的入口。多轮对话:方向/方案盘点 → AIRI 深度解析(GitHub README + plugins/ + apps/ + mcp-launcher 姊妹仓 WebFetch)→ 用户判断 AIRI 不够适配 → 评估"从零自做"可行性 → 用户选 (a)+(c) 落 wiki + MCP 手册
- 关键决策:议题归 AIAgents(具体 agent 实现 / 项目级落地)而非 AIFoundations,符合 CLAUDE §2.1 路由
- 新建:
  - [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] —— 议题概念,三层架构(形象/Hub/MCP host)+ L1/L2/L3 路线分级 + 反模式预警(MCP 后置 / Hub 绑死前端)
  - [[Wiki/Entities/AIAgents/AIRI]] —— Project AIRI(moeru-ai/airi v0.9.0)实体页;26+ provider / Live2D+VRM / Electron 桌面端 / plugins 五个 / mcp-launcher 姊妹仓 / 7 维评分(LLM 5⭐ 形象 5⭐ MCP 2⭐ 契合度 2⭐ 装配参考 5⭐)
  - [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] —— 选型矩阵综合;L1/L2/L3 + 现成项目横向(AIRI/Open-LLM-VTuber/Fay/VPet)+ 按用户偏好的决策矩阵 + 五条原则
  - [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] —— MCP host 实施手册;五职责 + 配置兼容 Claude Desktop + MCP Manager 实现骨架(命名空间 `<server>__<tool>`)+ Vercel AI SDK 桥接(JSON Schema 天然兼容)+ 安全(filesystem 限根/首次确认/审计/超时)+ 调试三件套(Inspector/sidecar log/devtools)+ AIRI 三路线对比(plugin/launcher daemon/prompt 模拟)+ L2 的 P2 子阶段(P2.1-P2.7)
  - [[Readers/AIAgents/桌宠 AI 入口的从零方案]] —— 主题读本;入口语义三分(A 启动器/B 单体助手/C AI 总线⭐)→ AIRI 评估(强项 + 三个错配)→ 三档路线分级 → L2 装配清单(Tauri/pixi-live2d/Vercel AI SDK/MCP SDK/SQLite-vec/Claude Desktop 配置格式)→ 三层架构 ASCII → MCP host 五职责浓缩 + 数据流时序 → P0-P5 分阶段 + 每段最小验收 + P3 候选(笔记搜索⭐/代码问答/贴图/Niagara)→ 10 条避坑 → 全景回看 → 10 条洞察 → 9 题自检(题型:错配返工 / L2-L3 取舍 / sidecar 第一性原理 / MCP-first 连锁 / Vercel SDK 可替代性 / 命名空间冲突场景 / P2 诊断顺序 / P3 选型取舍 / 5 句话讲第一性原理)
- 更新:
  - [[index.md]]:头部"最后更新"改写为本轮;AI Agents Entities 加 AIRI;Concepts 加 Desktop-pet-as-ai-hub;Syntheses 加 Desktop-pet-stack-comparison + Mcp-host-implementation;Readers 加桌宠从零方案
- 关键洞察(本读本沉淀):
  - 桌宠是壳,Hub 是脑,**MCP host 才是"入口"语义本身** —— 名字是桌宠,本质是私人 MCP host
  - AIRI 最有价值的不是它的躯壳而是它的依赖装配清单 —— 库选得对,但怎么拼是自己的事
  - **MCP-first 不可让步**:决定 Hub 整个数据流形状(tool 注册 / function calling 桥接 / result 回灌 / 设置 UI 数据模型),后置 = 推倒重来
  - **Vercel AI SDK + MCP SDK 的 JSON Schema 天然兼容** 是 L2 选型核心理由 —— 这条桥不用自写,选别的栈就要自搭
  - 配置格式照抄 Claude Desktop 是杠杆最大的兼容性投资 —— 一份配置三个 host 通用
  - P2 是验证整条路的最小回路 —— 跑通它就知道项目能不能走下去
  - 第一个自定义 MCP server 选"笔记搜索":本仓 wiki 直接成 LLM 扩展记忆,自我闭环最快
- 下一步:
  - 用户决定何时动手 P0;若动手,推荐另开仓库跟随读本 P0-P5
  - 可先做的预备:给本 wiki 写 MCP server(暴露 search_notes / read_note / list_recent),P3 阶段直接挂上桌宠
  - AIRI 持续观察:mcp-launcher 进展、@proj-airi/server-runtime 是否暴露稳定接口
- 自验证:Glob 5 个新路径全部确认存在(1 概念 + 1 实体 + 2 综合 + 1 读本)

## [2026-04-26] refactor | MCP 概念页教学化重写
- 触发:Eureka 在桌宠议题里跟着架构推进发现自己其实没真正理解 MCP——意识到"USB-C 类比"懂了不等于"为什么我要写 MCP server"懂了。请求把对话里"场景驱动 + 四种解法对比 + server 代码骨架"三块通用化后融进 MCP 概念页(明确不要桌宠相关内容)
- 改动:[[Wiki/Concepts/AIFoundations/Mcp]] 整体重写,从 64 行扩到 ~140 行
  - 新增"一个能立刻代入的场景"(ChatGPT 读本地文件做不到 → LLM 根本限制)
  - 新增"几种解法"对比表(A 复制粘贴 / B RAG / C Function Calling / D MCP),让 MCP 在替代方案矩阵里被理解,而非孤立的"USB-C 类比"
  - "三个角色"补具体 host 例子(Claude Desktop / Cursor / Windsurf / Cline)+ ASCII 图 + "杠杆从哪儿来"小结(server 写一次 N 用 / Host 实现一次挂全部)
  - **新增"MCP server 长什么样"整节**,~30 行 TypeScript 代码骨架(read_file 例子,tools/list + tools/call + stdio transport),配 jsonc 配置示例;让概念从"懂定义"变成"摸过代码"
  - "与相关概念的区别"表格加 RAG 行(原表只有 Skill / CLAUDE.md);加"简记四件套"
  - "开放问题"补"命名空间常用做法 `<server>__<tool>` 前缀"和"发现/分发机制 / 应用商店"
  - frontmatter `updated` → 2026-04-26
- 决策原则(用户明确要求):**纯 MCP 内容,不带桌宠**——避免概念页被某个具体应用绑死。桌宠侧 MCP 实施细节继续放 [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]],概念页只讲协议本身
- 不改:其他引用本概念页的地方([[Readers/AIFoundations/AI 应用生态全景 2026]] / [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] / [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]])保持不变,因本次只是把概念页讲深,反链/事实没动
- 自验证:Glob 确认 `Wiki/Concepts/AIFoundations/Mcp.md` 存在

## [2026-04-26] refactor | MCP 概念页加"发现机制"章节
- 触发:Eureka 在 MCP 概念清楚后追问"AI 工具怎么知道 MCP server 在哪"——这是 MCP 学完概念后第一个会冒出的实操疑问。先口述讲清后,用户要求把这部分(纯协议层发现机制)沉淀到概念页
- 改动:[[Wiki/Concepts/AIFoundations/Mcp]] 把原"配置方式"小 tip 块替换为整节 `## Host 怎么找到 server(发现机制)`,~70 行
  - **核心论断 callout**:MCP 协议本身不规定 server 怎么被发现 → 发现是 Host 的事 → "位置"就是配置里那行 command+args
  - **配置文件位置表**(Claude Desktop Win/Mac / Cursor / Windsurf / Cline / 自研)+ "为什么大家都抄 Claude Desktop 格式"tip(事实标准的形成)
  - **配置内容示例**(filesystem npx + git uvx + 本地 node 三种 entry)
  - **Host 启动时序**(spawn → stdio → initialize → tools/list → 注册给 LLM)
  - **三种 server 描述方式**表(本地 stdio / npx / uvx / 远程 SSE-HTTP)+ "npx -y 和 uvx 让用户不用预装"tip
  - **当前发现现状** "没有官方应用商店"warning + 5 来源表(官方仓库 / awesome list / 第三方 registry Smithery+mcp-get+Pulse / 厂商自营 / 集中式 launcher mcp-launcher)
- 同步收紧"开放问题"原"发现/分发机制"条 —— 正文已答,留下"官方应用商店是否出现 + 非技术用户门槛"作真正 open
- 不改:其他章节("场景驱动引入"/"三角色"/"server 长什么样"/"行业采纳"/"与相关概念区别"/"相关")保持上一轮重写后状态
- 决策:与上一轮重写规则一致——纯 MCP 协议内容,不带任何特定应用(桌宠/wiki)的语义
- 自验证:Glob 确认页面存在

## [2026-04-26] synthesis | wiki 首次被 AI 实时消费 —— notes-server 上线 + 方法论闭环
- 触发:本日早些时候完成桌宠议题落地后(见 [2026-04-26] synthesis | 桌宠作为 AI 应用入口议题首次入驻),Eureka 表示对 MCP 仍是"跟着推进点头",不真懂。请求把对话中"场景驱动 + 四解法对比 + server 代码骨架"沉淀进 [[Wiki/Concepts/AIFoundations/Mcp]](该页两次 refactor 见早些 log)。随后用户追问发现机制,再次扩页。再随后用户提议"自己写一个 server 跑一下"——实操进入。
- 第一阶段:实操起步(quote-server)
  - 检测环境:Node 22.15 + npm 11.8 ✅;Claude Desktop 是 MS Store 版(沙箱重定向至 `Local\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\`),本机已有 Claude Code(`LocalAppData\claude-cli-nodejs`)
  - 在 `D:\Notes\mcp-servers\quote-server\` 建首个 server(2 tool: `get_random_quote` 8 句硬编码 + `roll_dice` 带参数 demo);烟雾测试通过 stdio handshake + tools/list + 双 tool call 全过
  - 挂 Claude Desktop:配置首写到错路径(`%APPDATA%\Claude\` 标准路径,被 Store 沙箱忽略)→ 用户自行打开 Settings 触发真路径(`Local\Packages\...\LocalCache\Roaming\Claude\`)→ Edit 合并(保留 preferences 字段)
  - 用户验证:不点名 tool 自然语言"给我一句关于编程的名言激励一下我"成功触发 `get_random_quote`,返回 Hoare 那句(QUOTES 数组第 5 条字字一致)+ Claude 自加中文翻译和"加油"鼓励语 → 三件事一次验证:**自然语言触发 / 真实 server 调用 / LLM 在 server 之上的语境加工**
- 第二阶段:升级 search/read(把"玩具"变"工具")
  - 加 `search_notes`(case-insensitive 全文 walk + 命中行 ±1 上下文)+ `read_note`(防越界 `..`);两 tool description 都重写到"不点名也能触发"水平(明确说"用户的 wiki 是 curated second brain, 应优先于通用知识")
  - 烟雾测试 search_notes("DataInterface")在 ~50ms 内返回 3 文件真实命中,涵盖 index.md / log.md / Niagara 读本
  - 用户测试 prompt:"我之前是怎么理解 Niagara DataInterface 三路代码生成的?"——Claude 走完整 5 步链:[1] 承认记忆里没有 → [2] search 三路 失败 → [3] 改关键词 search 命中 → [4] read_note 拉 Entity 全文 → [5] 综合输出三段表 + 引用 `UNiagaraDataInterface.md:43` / `Niagara-learning-path.md:230`。**这是 wiki 第一次被 AI 实时使用、引用、回灌**
- 第三阶段:工程组织重构(quote-server → notes-server,vault 外迁)
  - 用户反馈:不想让 mcp-servers 工程目录 + node_modules 污染 vault;问"有没有全局 MCP 位置"
  - 答:**MCP 协议根本不规定 server 物理位置**——纯工程组织事;npm/uvx 自动管理,本地 server 自定。给 D:\(已有 Claude/code/ObsidianVault 等)和 C:\(AppData/Local 是 idiomatic Windows 用户级工具位)选项;用户选 A:`D:\Claude\mcp-servers\`(已有 D:\Claude\ 是 Claude Code CWD,语义自洽,桌宠未来也住这)
  - 重要发现(改正前轮误判):**MS Store 版 Claude Desktop 沙箱 NOT 是问题** —— 日志 `[LocalMcpServerManager] Connected to quote-server (4 tools)` 证明能 spawn node。真问题是 Claude Desktop 周期性把内部状态反写 config(用户没经 GUI 加的 server 被覆盖清空)。建议:(a)长期用 GUI 加 / (b)主战场放 Claude Code(实际用户用的是 Claude Desktop 的 Code 模式,共用配置)
  - 拆迁:新建 `D:\Claude\mcp-servers\notes-server\`(纯笔记工具)+ `VAULT_ROOT` 环境变量解耦(server 不再硬编码 vault 路径,通过 MCP 配置 env 字段注入,迈向"通用工具"第一步);更新 Claude Desktop config + `D:\Claude\.claude\settings.local.json` 权限白名单(预批 `mcp__notes-server__search_notes` / `mcp__notes-server__read_note` 免弹窗);删除 `D:\Notes\mcp-servers\` 整目录(21MB)
  - 终验:用户问"我的 wiki 里关于 MCP host 的论述要点是什么?",Claude 综合三个**今日才创建**的文件(`Mcp-host-implementation.md` + `Desktop-pet-as-ai-hub.md` + `Mcp.md` 重写版),输出"概念层 / 施工层"两线索分述,§0-§12 实施手册八点骨架还原,直接引用"桌宠是 MCP host 的人格化前端"原 callout 句。**通用训练绝无可能产生此类内容**,链路完全成立
- 关键决策与原则
  - **工具 ≠ 数据**:notes-server 服务 vault 但不该住在 vault 里;`D:\Notes\` 重回"纯知识"语义
  - **VAULT_ROOT env 注入**:server 与具体 vault 解耦,理论上可发布 npm 包供任何 Obsidian 用户使用
  - **单 server 单领域**:quote-server 时期 4 个混杂 tool 的反模式立即纠正,桌宠 P3 起步即遵循
  - **配置兼容 Claude Desktop 格式**:notes-server 配置在 Claude Desktop 与 (假设的) Claude Code 之间天然通用 —— [[Wiki/Concepts/AIFoundations/Mcp]] §"Host 怎么找到 server"原则的实证
- 落地资产
  - `D:\Claude\mcp-servers\notes-server\index.js` 与 `package.json`(91 deps;5953 字节代码;两 tool description ~200 字工程级)
  - `D:\Claude\mcp-servers\.gitignore`(node_modules + package-lock)
  - 配置文件:Claude Desktop config(MS Store 沙箱路径)+ `D:\Claude\.claude\settings.local.json`
- 方法论里程碑(本条最重要)
  - vault 首次完成 **"写 → 被 AI 读 → 被回灌"** 闭环。[[Wiki/Concepts/Methodology/Llm-wiki-方法论]] 从理论到亲手验证;[[Raw/Notes/Karpathy Wiki 方法论]] 提出的"为 LLM 维护 wiki 反过来 LLM 用 wiki 服务你"今日实证
  - 用户认知曲线一日完成:不懂 MCP → 三角色 + 发现机制 + 描述工程全清晰 → 写过四个 tool → 经历完整迁移重构 → 第一次让 AI 用自己的笔记
- 自验证:Glob 确认 `D:\Notes\mcp-servers\` 已不存在;Bash 确认 `D:\Claude\mcp-servers\notes-server\index.js` 存在 + 烟雾测试通过(VAULT_ROOT env 注入,搜索真返回 wiki 命中)
- 下一步:Eureka 转入桌宠 P0 议题(Tauri + Live2D 透明壳 + 系统托盘 + 全局快捷键)。本轮 MCP 工程闭环到此为止,后续 P0/P1/P2 会复用今日 notes-server 作为 P3 已成原型

## [2026-04-26] synthesis | 桌宠议题加"团队分发版"——VFX 团队 mascot 第二人设
- 触发:Eureka 在准备进 P0 前临时提出三个新需求——(a) 让桌宠播 Flipbook 特效以贴近 VFX 团队 / (b) 后续给整个特效组用,需要统一管理 agents 和设置 / (c) 希望有设置页面。
- 关键洞察:三件事在架构上汇聚到**配置层模型**。Flipbook 需配置 + 触发规则 + 资产管理;团队管理本质是 admin/user 配置分层;设置页是配置的 UI 表层。先把配置层搞对,后面三件事都顺。
- 三个架构决策(用户拍板):
  - **a. Team config source = B(git repo)** —— TA 团队天然爱 git,PR 走代码审,有 diff/blame/revert,出问题能 git revert
  - **b. Admin 模型 = 几个 lead/TA** —— 不是单 admin(bus factor=1),不是全员(易乱);桌宠端不区分 admin,身份完全由 git 写权限决定
  - **c. P0 包含 P0.5(配置层 + 简单设置页)** —— 不预留后期所有 setting 改起来都疼
- 新建:[[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]](团队分发架构专题,~280 行)。覆盖 8 节:
  - §0 个人/团队两人设本质差别表(用户数 / 配置 / MCP / 形象 / 维护)
  - §1 Flipbook(为什么是 VFX 团队杀手级 / pixi 自带 AnimatedSprite 0 新依赖 / 触发系统 4 类 + 引入时机 / 资产格式约定 / "个人项目 → 团队 IP"心理学杠杆)
  - §2 配置三层(layer 1-4 优先级 / 四 source 对比表 / B 方案 simpleGit clone+pull 实现 / team-config repo 长什么样)
  - §3 Admin/user 权限(per-key lock + reset to team default + admin 不在桌宠端区分,git 决定)
  - §4 设置页结构(6 tab + admin 模式开关)
  - §5 调整后阶段路线(P0.5/P1.5/P4.5/P5 distribution)
  - §6 6 个开放问题(Git LFS for sprite sheet / 离线 cache 兜底时长 / lock 后 user override 怎么处理 / 特效预览组件 / VAULT_ROOT 是否 user-local / ...)
  - §7 同代码库两种部署(配置启停团队特性,不分叉 repo)
- 更新:
  - [[Readers/AIAgents/桌宠 AI 入口的从零方案]]:
    - §6 阶段路线表加 4 个新阶段(P0.5🅂/P1.5/P4.5🅂/P5🅂),🅂 标记团队版独有
    - §6 加双人设 callout + 团队版必须颠倒"不要先做配置 UI"反 warning
    - §6.1.5 P0.5 最小验收(三层 store + 设置页骨架 + lock 机制)
    - §6.2.5 P1.5 最小验收(PixiJS AnimatedSprite + 资产目录约定 + 至少 2 示例特效)
    - §10 深入阅读加新综合页链接
  - [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]]:
    - 加 "两种部署形态" 表(个人 vs 团队)
    - 相关链接加 team-distribution 综合
  - [[index.md]]:头部"最后更新"改写;Syntheses 加 team-distribution 条
- 决策原则:
  - **同代码库 + 配置启停**:不分两套代码维护;不配 team repo URL 时退化为个人版,完全等价
  - **Live2D=身体 / Flipbook=特效环绕**:语义模型对齐 VFX 团队"角色 + 特效"分工(Niagara 式),团队成员秒懂
  - **桌宠端不区分 admin**:权限完全由 git 写权限决定,这是 git 自带的,我们什么都不做
  - **配置兼容 schema**:user-settings.json 与 team-defaults.json 同 schema,只 lock 字段差别
- 工期影响:个人版 1-2 月 v0.1 → 团队版 2-3 月 v0.1。值得,P0.5 不前置后期返工成本远大
- 下一步:Eureka 装 Rust → P0.0(Tauri scaffold + Vue + TS + 三层 config 骨架)+ P0.1(透明置顶窗口)开工
- 自验证:Glob 确认 5 处更新全部存在(1 新综合 + 3 修改文件 + 本 log 条)

## [2026-04-27] synthesis | Solaris 3 v0.1.0 完工 —— 桌宠 P0 阶段全部 ship
- 触发:昨日(2026-04-26)桌宠 AI 入口议题落地 wiki 后,Eureka 决定执行 L2 自做路线。当日完成 MCP 学习闭环 + notes-server 上线;今日(2026-04-27)从零执行 P0 全部 6 个主 phase + 2 个 polish,产出可分发 v0.1.0
- 项目仓库:`https://github.com/EurekaThurston/Solaris-3`(从昨晚起步;独立仓,不进 vault,符合"工具 ≠ 数据"原则)
- 项目 commit 序列(8 步,含 polish):
  - `b02055b` P0.0 + P0.1 — Tauri 2 + Vue 3 scaffold + 透明置顶无装饰窗口 + 磨砂卡片占位 + 拖动整窗
  - `2aa989e` P0.2 — 系统托盘 (tray-icon feature) + tauri-plugin-global-shortcut + Ctrl+Space toggle + 托盘菜单(显示/隐藏/退出)
  - `1261619` P0.3 — Live2D 角色(pixi.js@^6.5 + pixi-live2d-display@^0.4 + Cubism Core SDK)+ Hiyori Pro 测试(本地未入仓)+ 自动眨眼/idle/物理(Cubism 内置)
  - P0.4 — `pnpm tauri build` 验证产 NSIS 6.6MB / MSI 7.7MB / raw exe 15MB(无独立 commit,验证步骤)
  - `088538a` P0.5 — 三层 config 模型(defaults/team/user via tauri-plugin-store)+ Vue Router 多窗口架构 + macOS 风设置 UI(角色/快捷键 tab)+ 跨窗口 reactive(slider 实时联动主窗口 Live2D)
  - `ad6666a` P0.5+ — 窗口位置持久化(onMoved + 300ms debounce + setPosition 启动恢复)+ 运行时改快捷键(Tauri command set_summon_hotkey + HotkeysTab 录制 UI)+ Hiyori Pro 换 Hiyori Free + 模型入仓(Free Material License 允许)
- 产出资产
  - 8MB Solaris-3 项目仓(含 Hiyori Free + Cubism Core + 完整 src)
  - 6.6MB NSIS 安装器 / 7.7MB MSI / 15MB raw exe(单文件嵌入所有静态资产,运行时仅依赖系统 WebView2)
  - GitHub Release v0.1.0(供同事直接下安装器,SmartScreen 警告点"仍要运行"即可)
- 关键 7 个 bug + lessons learned(详见 [[D:/Solaris-3/DEVLOG.md]] 完整列表):
  1. **Tauri 2 capabilities 细粒度权限**:`start_dragging` 等 API 必须显式列在 default.json,`core:default` 不含 destructive。教训:报"not allowed"先查 capability 文件,不查网
  2. **Cargo lib.name 改了忘改 main.rs 引用**:scaffold `solaris_3_scaffold_lib` 改 `solaris_3_lib` 后 `main.rs::run()` 调用要同步改;教训:grep crate 名是改名标配步骤
  3. **MS Store 版 Claude Desktop 误判**(与本项目无关但有方法论价值):我先入为主认为"沙箱不能 spawn",看 main.log 才发现完全成功,真正问题是 Claude Desktop 反向覆盖 config;教训:**先看日志再下结论**
  4. **`WebviewUrl::App("index.html#/settings")` 全白屏**:Tauri 2 把 path 当字面文件路径,`#` 不是 fragment 而是文件名一部分;修法:URL 不带 hash,前端按 `window.label` 决定路由
  5. **Vue Router 4 `router.isReady()` 在 `app.use(router)` 前调用永远 hang**:initial navigation 在 `app.use(router)` 时启动,顺序错就死锁。修法:严格 `createApp → use(router) → await isReady → mount`
  6. **`tauri-plugin-global-shortcut` handler by-move 闭包**:捕获的初始 shortcut 不会跟着 unregister/register 变;修法:handler 不比较 shortcut,任何已注册组合按下都 toggle(因只注册一个)
  7. **Live2D 不同发行版文件名前缀差异**:Pro `hiyori_pro_t11.*` vs Free `hiyori_free_t08.*`;defaults.ts 的 modelPath 必须匹配实际文件名;未来 P1+ 加模型选择器应**自动扫描** `public/live2d/<dir>/*.model3.json`
- 关键架构决策(为什么这么设计)
  - **三层配置 P0.5 就上不 P4 后置**:团队分发场景必需,后期改架构成本极高;同时是跨窗口 reactive 链路的基础,P5+ Flipbook trigger 也吃这条管道
  - **设置 = 独立 Tauri 窗口而非 modal**:主窗口 400×500 + 透明 + 置顶装不下表单;独立窗口让用户能"边改边看 Live2D 实时变化"
  - **Vue Router hash mode + label-based routing**:Tauri 2 不支持 URL fragment(Bug 4);source of truth 是 Tauri 端 label,前端 follow
  - **Hiyori Free 入仓 vs Pro 不入仓**:Free Material License 明确允许"小规模商用 + 大型组织内部使用 + 项目内分发";Pro 限制更严走 P4.5 git sync 路线
  - **跨窗口 reactive = tauri-plugin-store + Tauri events**:`setUser()` 三件套(更新 reactive + 持久化 + emit event);其他窗口 listen 后替换本地 state → effective computed 重算 → watch 触发 UI 反应;**整链路完全声明式**,加新 setting 0 改动
- 方法论里程碑(本日真正的"为什么记 log")
  - 昨日完成 wiki 闭环(写 → AI 读 → 回灌),今日完成**议题 → 设计 → 落地**全链路:
    - 4 月 26 早:议题入驻 wiki + Mcp 概念页教学化 + notes-server 上线
    - 4 月 26 晚:议题暴露第二人设(团队版),架构升级为三层 config + Flipbook + git sync,沉淀团队分发综合页
    - 4 月 27:照着读本 P0 阶段表 8 步 ship 出可分发 v0.1.0;P0.5 三层配置完全照 [[Wiki/Syntheses/AIAgents/Desktop-pet-team-distribution]] §2 实现
  - **wiki 不是只用来读的,是用来"指导施工"的**——读本里 §6 P0-P5 阶段表今日逐字执行,综合页 §2 配置三层模型今日逐字落码。这是 [[Wiki/Concepts/Methodology/Llm-wiki-方法论]] 从"知识库"升级为"工程蓝图"的实证
  - **DEVLOG 与 wiki log 双层记录**:wiki log 记方法论级里程碑(本条目);项目 DEVLOG 记每个 bug 的具体症状/根因/修法/教训(7 条详细)。两者各自服务对象不同——wiki log 给未来读取议题脉络的我看,DEVLOG 给未来在 Solaris-3 上动手的人(我或同事)看
- 下一步:P1 启动(LLM 流式聊天)
  - 子阶段:P1.0 Vercel AI SDK + provider / P1.1 设置加 AI tab(API key)/ P1.2 聊天 UI(下方滑出 + 气泡)/ P1.3 流式渲染 / P1.4 多轮上下文 in-memory / P1.5 Hiyori 表情联动(可选)
  - 决策点:默认 provider(推 OpenAI 兼容 endpoint 覆盖国内厂商)/ 聊天 UI 形态(推下方滑出)/ 上下文持久化(P1 in-memory,P4 SQLite)
  - 不在 P1 做的:MCP 接入(P2)/ TTS+ASR(P5 黑洞)/ Flipbook(P1.5 可插)
- 自验证(Glob):
  - [[D:/Solaris-3/DEVLOG.md]] 存在(今日新建)
  - GitHub Release v0.1.0 已发(用户可访问 `https://github.com/EurekaThurston/Solaris-3/releases/tag/v0.1.0` 验证)
  - 项目 commits `b02055b`(P0.0+1)/ `2aa989e`(P0.2)/ `1261619`(P0.3)/ `088538a`(P0.5)/ `ad6666a`(P0.5+) 全部 push 到 origin/main
