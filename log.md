# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

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

## [2026-04-20] refactor | 读本独立顶层目录 Wiki/Readers/
- 触发:用户指出读本散在 `Syntheses/Methodology/`、`Syntheses/AIArt/`、`Syntheses/AIApps/`、`Syntheses/Niagara/` 四个子目录里,和普通专题(三段论演进、How-to-prompt、学习路径总图)混在一起"有点乱,到处都是"
- 方案:新增 `Wiki/Readers/` 顶层目录,5 份读本全部迁过去;`Syntheses/` 回归原定位——非读本类的专题综合
- 迁移(全部用 `git mv` 保留历史):
  - `Wiki/Syntheses/Methodology/Llm-wiki-方法论-读本.md` → `Wiki/Readers/Methodology/Llm-wiki-方法论-读本.md`
  - `Wiki/Syntheses/AIArt/Lora-深度指南-读本.md` → `Wiki/Readers/AIArt/Lora-深度指南-读本.md`(`Wiki/Syntheses/AIArt/` 本来只有读本,搬完即删空目录)
  - `Wiki/Syntheses/AIApps/AI-primer-v2-读本.md` → `Wiki/Readers/AIApps/AI-primer-v2-读本.md`
  - `Wiki/Syntheses/Niagara/Phase0-心智模型-读本.md` → `Wiki/Readers/Niagara/Phase0-心智模型-读本.md`
  - `Wiki/Syntheses/Niagara/Phase1-asset-layer-读本.md` → `Wiki/Readers/Niagara/Phase1-asset-layer-读本.md`
- CLAUDE.md 更新:
  - §2 目录说明:新增 `Readers/` 顶层目录的完整描述 + 明确 "`Syntheses/` vs `Readers/` 的分工"小节
  - §3.4:归档路径从 `Wiki/Syntheses/<topic>/` 改为 `Wiki/Readers/<topic>/`,新增"文件命名与路径"具体示例
  - §4.5:模板标题注明路径 `Wiki/Readers/<topic>/`
  - 历史债清单(§3.4 末尾)3 条 ✅ 条目的链接同步更新到新路径
- 连带同步改链接(用 sed 跨 12 个文件批量替换):
  - 5 份读本之间的互相引用
  - 5 个 Phase 1 Entity 页的"深入阅读"链接
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 0/1 顶部读本引流
  - [[index]]、[[Wiki/Overview]]、以及历史 log 条目里的 wikilink
- [[index]] 分区重构:`Readers(主题读本)` 提升为**顶层分区**,与 `Entities / Concepts / Sources / Syntheses` 平级(不再作为 Syntheses 的子分区);Syntheses 分区回到"非读本类专题综合"的精简列表
- 本条 log 描述时的路径均为**迁移后新路径**(旧路径仅在本条内提到一次)
- 下一步:读本独立目录生效,路径稳定,之后 Phase 2 读本直接入驻 `Wiki/Readers/Niagara/`

## [2026-04-20] synthesis | 历史债清算 — 三份主题读本一次性补齐
- 触发:4-19 CLAUDE.md §3.4 引入"主题读本"规则时,历史上三个主题(方法论/AIArt/AIApps)因 ingest 时规则尚未存在而未产出读本,已在当时钉为"历史债清单"。今日用户命令清算
- 新建 3 份读本,每份都按 CLAUDE.md §4.5 通用骨架(问题驱动叙事 + 代码/原文 inline + 陷阱 ⚠️ 高亮 + 深入阅读指回原子 + 开放问题):
  - [[Wiki/Readers/Methodology/Llm-wiki-方法论-读本]] — 方法论议题读本,~500 行,叙事主线"时间线(Memex 1945 → RAG → LLM Wiki 2026)+ 哲学线(关联文档 → 查询时拼碎片 → 持续编译)"交织,五节 Memex/RAG/方法论骨架/Karpathy/本仓库具体化,末尾举 2026-04-17 第一次 ingest 作为完整例子
  - [[Wiki/Readers/AIArt/Lora-深度指南-读本]] — AI 美术议题读本,~900 行,叙事主线"战略(为什么离开 MJ,三个结构性短板)→ 技术(LoRA 原理 → 基座选型 → Caption 反常识 → Trigger Word → Multi-LoRA)→ 工具(kohya_ss + ComfyUI)→ 工程(6 个月路线图 + 合规 + 硬件 + 评估准则)"
  - [[Wiki/Readers/AIApps/AI-primer-v2-读本]] — AI 应用生态议题读本,~1000 行,叙事主线"LLM 本质 → 三个怪癖(幻觉/Context Rot/不稳定)→ 新面孔(推理/多模态/Agent)→ 连接外部(MCP/RAG/记忆)→ 三段论演进 → Harness 四柱 → Agent Skills → 2026 技术栈三层 → Karpathy 三节点 → Vibe/Spec/Harness Coding"
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
- 新建:[[Wiki/Readers/Niagara/Phase0-心智模型-读本|Phase0-心智模型-导读]] — Phase 0 的线性读物,把四个概念页(UObject / Asset-Instance / Niagara-vs-Cascade / CPU-vs-GPU)编成自下而上一条叙事链  *(注:4-19 晚些时候重命名为"读本",详见本条上方的 refactor 条目)*
- 叙事结构:四层脑内地图(Layer 1 UObject → Layer 2 Asset/Instance → Layer 3 Niagara 哲学 → Layer 4 CPU/GPU 分叉),每层末尾小结 + 最后一节"四层地图回看"贯通
- 埋雷:第 2.8 节专门提前钉死"`FNiagaraEmitterHandle::Instance` 不是运行时 Instance"这个 Phase 1 必踩的命名陷阱,让读者到 Phase 1 时有预期
- 更新:[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 0 节顶加导读链接)、[[index]]、[[Wiki/Overview]]
- 动机:补齐方法论升级后 Phase 0 的导读缺位;今后任何 Phase 完成都要有配套线性读物

## [2026-04-19] synthesis | Niagara Phase 1 导读 + 方法论升级
- 触发:用户指出原子化 Source/Entity 页不符合人类线性阅读习惯(频繁跳转、碎片化)
- 核心产出:[[Wiki/Readers/Niagara/Phase1-asset-layer-读本|Phase1-asset-layer-导读]] — Phase 1 的**教科书章节**,500+ 行线性叙事,从 Content Browser 切入讲到图源抽象基类,关键代码片段 inline,不强制跳转  *(注:4-19 晚些时候重命名为"读本")*
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
  - [[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat-cheatsheet]] — 一页纸精简版（检查清单 + 8 技巧 + 反模式）,设计为可直接转发群里
- 更新:[[index]](Syntheses 新增"方法论"分区)
- 核心洞察:糟糕的 prompt 来自错误的心智模型（把 AI 当搜索引擎/全知神谕）——校正心智模型后,技巧都是推论

## [2026-04-19] ingest | LoRA 深度指南 — AI 美术生成管线首次入驻
- source: [[Raw/Notes/Lora_Deep_Dive]]（鸣潮美术向 TA 的 LoRA 落地方案）
- 新建主题目录:`Wiki/{Concepts,Entities,Sources}/AIArt/`
- 新建:
  - Source: [[Wiki/Sources/AIArt/Lora-deep-dive]]
  - Concepts (5): [[Wiki/Concepts/AIArt/Lora]]、[[Wiki/Concepts/AIArt/Base-model-selection]]、[[Wiki/Concepts/AIArt/Caption-strategy]]、[[Wiki/Concepts/AIArt/Trigger-word]]、[[Wiki/Concepts/AIArt/Multi-lora-composition]]
  - Entities (5): [[Wiki/Entities/AIArt/Illustrious-XL]]、[[Wiki/Entities/AIArt/NoobAI-XL]]、[[Wiki/Entities/AIArt/Flux]]、[[Wiki/Entities/AIArt/Kohya-ss]]、[[Wiki/Entities/AIArt/ComfyUI]]
- 更新:[[index]](新增 AI 美术 Concepts + Entities + Sources 三个分区)、[[Wiki/Overview]](从 1 主题扩至 3 主题综合,重写知识图)
- 要点:wiki 首次覆盖 AI 美术生成领域,和既有方法论/Niagara 主题完全独立；核心洞察是 "LoRA 让游戏团队把美术 DNA 固化成可版本化的文件"、Caption 反常识策略、基座选型的许可陷阱（Flux Dev 非商用）
- 下一步:用户可能要验证本机 kohya_ss 环境,或询问具体章节的实践细节

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

