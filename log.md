# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

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

