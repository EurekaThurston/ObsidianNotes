---
type: overview
created: 2026-04-17
updated: 2026-04-20
tags: [overview]
sources: 9
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新:2026-04-25 今日 4 轮 refactor:(1) AIApps → AIFoundations 改名;(2) Niagara 下沉为 UE 子主题(`Niagara/` → `UE/Niagara/`);(3) `UE4/` → `UE/` 为 future UE5 留空间;(4) `Entities/Stock/*.md` + `Sources/Stock/*.md` 下沉 `Stock/Niagara/` 子命名空间(120 文件),CLAUDE.md §9.5 路径模板升级为 `<repo>/<module>/<文件名>`

---

## 当前主题（4 个）

### 1. 知识库方法论（meta / bootstrap）

自举阶段的产出：基于 [[Raw/Notes/Karpathy Wiki 方法论]] 建的三层架构（Raw / Wiki / Schema）；详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]。

📖 **主题读本（推荐初读）**：[[Readers/Methodology/从 Memex 到 LLM Wiki]] — 把本仓库的起点、历史源流、RAG 对比、Karpathy 三里程碑、本仓库如何具体化编成一篇线性读物。

**核心主张**：持续整合 > 查询时检索。RAG 每次查询都在从零发现知识，LLM Wiki 则在 ingest 时就完成跨源整合、交叉引用、矛盾标注。

### 2. UE / Niagara 源码学习（UE 4.26）

面向 C++ 零基础、UE 源码零基础的学员，通过 AI 辅助学习 Niagara 插件 749 个文件中的运行时部分（约 286 个文件）。

- 路线：[[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]] — 10 阶段路径
- **Phase 0 ✅**：心智模型建立 —— 📖 **主题读本**：[[Readers/UE/Niagara/Phase 0 - 上阵前的四层脑内地图]] / 原子概念页:[[Wiki/Concepts/UE/UE4-uobject-系统]]、[[Wiki/Concepts/UE/UE4-资产与实例]]、[[Wiki/Concepts/UE/Niagara/Niagara-vs-cascade]]、[[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]]
- **Phase 1 ✅**（2026-04-19）：Asset 层三件套 —— 📖 **主题读本**：[[Readers/UE/Niagara/Phase 1 - 从 System 到图源抽象基类]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraSystem]]、[[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]]、[[Wiki/Entities/Stock/Niagara/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/Niagara/UNiagaraScript]]、[[Wiki/Entities/Stock/Niagara/UNiagaraScriptSourceBase]]
- **Phase 2 ✅**（2026-04-20）:Component 层 —— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 2 - Component 层的五职责]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]]、[[Wiki/Entities/Stock/Niagara/ANiagaraActor]]、[[Wiki/Entities/Stock/Niagara/UNiagaraFunctionLibrary]]
- **Phase 3 ✅**（2026-04-20）:运行时实例层(心脏)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 3 - Niagara 的心脏]] / 原子页见 [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]]、[[Wiki/Entities/Stock/Niagara/FNiagaraEmitterInstance]]、[[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]]
- **Phase 4 ✅**（2026-04-20）:数据模型(类型系统 + SoA + 参数存储)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 4 - Niagara 的数据语言]] / 原子页见 [[Wiki/Entities/Stock/Niagara/FNiagaraTypeDefinition]]、[[Wiki/Entities/Stock/Niagara/FNiagaraVariable]]、[[Wiki/Entities/Stock/Niagara/FNiagaraTypeLayoutInfo]]、[[Wiki/Entities/Stock/Niagara/FNiagaraConstants]]、[[Wiki/Entities/Stock/Niagara/FNiagaraDataSet]]、[[Wiki/Entities/Stock/Niagara/FNiagaraDataSetAccessor]]、[[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]]
- **Phase 5 ✅**（2026-04-20）:CPU 脚本执行(VM + ExecContext + Padding)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 5 - Niagara 脚本如何跑起来]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceBase]]、[[Wiki/Entities/Stock/Niagara/FNiagaraScriptExecutionContext]]、[[Wiki/Entities/Stock/Niagara/FNiagaraComputeExecutionContext]]、[[Wiki/Entities/Stock/Niagara/FNiagaraGPUSystemTick]]、[[Wiki/Entities/Stock/Niagara/NiagaraEmitterInstanceBatcher]]
- **Phase 6 ✅**（2026-04-20）:渲染系统(Sprite/Ribbon/Mesh/Light 5 对)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 6 - Niagara 粒子如何变成屏幕像素]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraRendererProperties]]、[[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]]、[[Wiki/Entities/Stock/Niagara/UNiagaraSpriteRendererProperties]]、[[Wiki/Entities/Stock/Niagara/UNiagaraRibbonRendererProperties]]、[[Wiki/Entities/Stock/Niagara/UNiagaraMeshRendererProperties]]、[[Wiki/Entities/Stock/Niagara/UNiagaraLightRendererProperties]]
- **Phase 7 ✅**（2026-04-20）:数据接口系统(DI 生态 + 7 种典型 DI)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 7 - 最强扩展点 Data Interface]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCurve]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCamera]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCollisionQuery]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceStaticMesh]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceSkeletalMesh]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceTexture]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRenderTarget2D]]
- **Phase 8 ✅**（2026-04-20）:GPU 模拟(Shader 编译 + Sort + Count + VF + DrawIndirect)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 8 - Niagara 的 GPU 模拟管线]] / 原子页见 [[Wiki/Entities/Stock/Niagara/FNiagaraShader]]、[[Wiki/Entities/Stock/Niagara/FNiagaraGPUInstanceCountManager]]、[[Wiki/Entities/Stock/Niagara/FNiagaraGPUSort]]、[[Wiki/Entities/Stock/Niagara/FNiagaraVertexFactory]]、[[Wiki/Entities/Stock/Niagara/FNiagaraDrawIndirect]]
- **Phase 9 ✅**（2026-04-20）:世界管理(WorldMgr + Scalability + Pool + Settings + EffectType + PlatformSet)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 9 - Niagara 的世界管理与可扩展性]] / 原子页见 [[Wiki/Entities/Stock/Niagara/FNiagaraWorldManager]]、[[Wiki/Entities/Stock/Niagara/FNiagaraScalabilityManager]]、[[Wiki/Entities/Stock/Niagara/UNiagaraComponentPool]]、[[Wiki/Entities/Stock/Niagara/UNiagaraSettings]]、[[Wiki/Entities/Stock/Niagara/UNiagaraEffectType]]、[[Wiki/Entities/Stock/Niagara/FNiagaraPlatformSet]]
- **Phase 10 ✅**（2026-04-20）:**高级特性**(SimStages + Grid 2D/3D + NeighborGrid)—— 📖 **主题读本**:[[Readers/UE/Niagara/Phase 10 - Niagara 的高级特性]] / 原子页见 [[Wiki/Entities/Stock/Niagara/UNiagaraSimulationStage]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRWBase]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceGrid2DCollection]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceGrid3DCollection]]、[[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceNeighborGrid3D]]
- 🎉 **Niagara 学习路径 10 Phases 全部完成!**(2026-04-20 一日内从 Phase 3 推到 Phase 10)

**Phase 1 的关键收获**（从 5 个 header 里提炼）：
- **Asset 链路定型**：System →（`TArray<FNiagaraEmitterHandle>`）→ Handle(Id+Name+enabled+Instance) → Emitter →（脚本 / 渲染器 / SimStages）→ Script（字节码 + GPU shader）
- **命名陷阱**：`FNiagaraEmitterHandle::Instance` 仍在 Asset 层（Emitter 资产副本），与 Phase 3 的 `FNiagaraEmitterInstance`（运行时粒子模拟）**完全不是一回事**
- **EmitterSpawn/EmitterUpdate 脚本不独立编译**，被合并进 System 脚本（`UNiagaraScript::IsCompilable()` 对这两者返 false）
- **editor/runtime 解耦点**：`UNiagaraScriptSourceBase` 抽象基类在 runtime 模块，真正的图实现 `UNiagaraScriptSource + UNiagaraGraph` 在 NiagaraEditor 模块,使 runtime 能持图指针但不 link editor

**Phase 2 的关键收获**(从 3 个 header 里提炼):
- **`UNiagaraComponent` 是唯一主角**:Actor/FunctionLibrary 加起来不到 Component 的 1/5(741 行 vs 66+93),三种入口路径最终全部汇于一个 Component 实例
- **Asset ↔ Instance 是一对多**:`TUniquePtr<FNiagaraSystemInstance> SystemInstance` 独占,N 个 Component 引用同 Asset 就有 N 份独立实例
- **参数覆盖分层**:标量走 Component 的 18 个 `SetVariableXxx`,对象型(Mesh/Texture)走 `UNiagaraFunctionLibrary::Override*` —— DI 关注点分离
- **生命周期四源汇合**:UActorComponent / UFXSystemComponent / 用户语义 / Scalability 四条生命周期源全部在 `TickComponent` 和 `OnSystemComplete` 收口
- **`bForceSolo` 性能陷阱**:绕开 `FNiagaraSystemSimulation` 批量 Tick,"同 Asset 多实例" 典型场景性能退化可能数十倍
- **`ANiagaraActor` 是纯 observer 模式示例**:66 行的 ComponentWrapperClass,只订阅 Component 的 `OnSystemFinished` 一个 delegate 决定生死,不持任何运行时状态

### 3. AI 基础(AIFoundations)(2026-04 新增,2026-04-25 改名)

> 改名备忘:原名 "AIApps / AI 应用生态"。重构完成后,具体的"应用"(代码问答机器人、特效贴图工具、生成式智能体项目)都去了 [[#4. AI Agents（2026-04-25 新增)]] 主题,本节只剩基础概念 + 工程方法论 + 扫盲,改名为"AI 基础 / AIFoundations"更贴实。

**面向**:对 AI 完全没概念的非开发角色(美术、设计、策划、管理者)+ 开发者的主线脉络梳理。

📖 **主题读本（推荐初读）**：[[Readers/AIFoundations/AI 应用生态全景 2026]] — 从 LLM 本质到 2026 技术栈三层全景的完整读物,覆盖三个怪癖/推理模型/Agent/MCP/RAG/三段论/Harness 四柱/Skills/OpenClaw/Vibe-Spec-Harness Coding,一次读完。

- 核心源：[[Wiki/Sources/AIFoundations/AI-primer-v2]]（2026-04-19，Eureka × Claude 撰写，v2）
- 专题综合：[[Wiki/Syntheses/AIFoundations/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]](方法论演进专题)
- 基础概念：[[Wiki/Concepts/AIFoundations/Llm|LLM]]、[[Wiki/Concepts/AIFoundations/Hallucination|幻觉]]、[[Wiki/Concepts/AIFoundations/Context-window|上下文窗口 & Context Rot]]、[[Wiki/Concepts/AIFoundations/Reasoning-model|推理模型]]
- Agent 时代：[[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]]、[[Wiki/Concepts/AIFoundations/Mcp|MCP]]、[[Wiki/Concepts/AIFoundations/Harness-engineering|Harness Engineering]]、[[Wiki/Concepts/AIFoundations/Agent-skills|Agent Skills]]、[[Wiki/Concepts/AIFoundations/Agentic-grep|Agentic Grep]]
- 落地产品：[[Wiki/Entities/AIFoundations/OpenClaw|OpenClaw（小龙虾）]]

**核心洞察**：2026 年 AI 技术栈三层结构（Model / Harness / Skills）中，模型层正在商品化，真正差异化竞争在 Harness 和 Skills 两层。用户第一纪律：AI 给的具体事实必须验证。

> [!note] 项目级落地设计 & 多 agent 模拟已独立为主题 4
> 原本归在本节的"给美术做代码问答机器人"、"AI 特效贴图工具"两份项目级落地,以及新的"斯坦福小镇 / Park 生成式智能体"研究线,统一归 [[#4. AI Agents（2026-04-25 新增)]] 主题。

### 4. AI Agents（2026-04-25 新增)

**面向**:agent 作为独立主题需要单独的深度——既包括**生成式智能体模拟研究**(Stanford Smallville / Park 2023 → 2024 1000 人 / a16z AI Town),也包括**项目级 AI 应用落地**(美术代码问答 / 特效贴图工具)。两者共享 "agent" 这个名词但第一性原理不同:前者是**共生演化**,后者是**工具型派活**(对照 [[Wiki/Concepts/AIFoundations/Multi-agent]])。

📖 **主题读本(推荐初读)**:[[Readers/AIAgents/从斯坦福小镇到 1000 人数字社会]] — 生成式智能体 2023→2026 演进读本:三件套架构解四堵硬墙 + token 成本具体化 + Park 2024 评测转向 + a16z 选择性工程化 + 与工具型 multi-agent 的第一性原理之别 + 8 题综合自检。

- **生成式智能体模拟**
  - 核心 Source:[[Wiki/Sources/AIAgents/Park-2023-generative-agents]](arXiv 2304.03442 摘要级)、[[Wiki/Sources/AIAgents/Park-2024-1000-agents]](arXiv 2411.10109)、[[Wiki/Sources/AIAgents/Generative-agents-discussion]]
  - 核心 Entity:[[Wiki/Entities/AIAgents/Joon-sung-park]]、[[Wiki/Entities/AIAgents/Stanford-smallville]]、[[Wiki/Entities/AIAgents/A16z-ai-town]]
  - 核心 Concept:[[Wiki/Concepts/AIAgents/Generative-agents-architecture|生成式智能体架构(Memory+Reflection+Planning)]]、[[Wiki/Concepts/AIAgents/Agent-based-social-simulation|LLM 驱动的社会模拟]]
- **项目级 AI 应用落地**
  - [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot|给美术做代码问答机器人]] — agentic grep + wiki 复合记忆(2026-04-24)
    - 配套读本:[[Readers/AIAgents/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] 含 Dithered LOD Transition 贯穿案例
  - [[Wiki/Syntheses/AIAgents/Ai-texture-tool-design|AI 特效贴图工具设计]] — 三层架构 + VFX 数据语义 guardrail + GitHub 生态调研(2026-04-24)
    - 配套读本:[[Readers/AIAgents/让 AI 接特效贴图的长尾需求 - 架构与 GitHub 生态]]

**核心洞察**:"multi-agent" 一词其实指两种东西——工具型(主 agent 派活给子 agent,第一性原理是上下文隔离)和模拟型(多 agent 共生演化,目标是观察群体涌现)。看到这三个字先问动因。本 wiki 的 AIAgents 主题**同时容纳这两种形态**,读者要自觉保留"第一性原理不同"的警觉。

---

## 跨主题联系

- **方法论 → AI 应用生态**:本 wiki 正是"LLM Wiki 方法论"本身;AI Primer 则是我们用这套方法论 ingest 的第一份广域综合型源材料。
- **AI 应用生态 ↔ AI Agents**:两个主题在"agent"一词上相遇,但第一性原理不同。AIFoundations 的 [[Wiki/Concepts/AIFoundations/Multi-agent]] 是工具型上下文隔离派活,AIAgents 的 [[Wiki/Concepts/AIAgents/Agent-based-social-simulation]] 是共生演化观察涌现——同词异义,**第一性原理决定架构**。
- **AI Agents 内部的两条线**:学术模拟(Park 2023/2024、a16z AI Town)与项目级落地(代码问答机器人、特效贴图工具)共主题;共性是 agent 作为独立组织单元,差异在评测标准和服务对象(研究 / 工程)。
- **Niagara ↔ AI**:目前无实质交集,未来可能涉及 Niagara GPU 模拟路径对 AI 推理的借鉴。

---

## 主题知识图

```
methodology (meta, 自举)
    ├── Karpathy ── 也出现在 AI 应用生态（Context Engineering / Vibe Coding）
    ├── LLM Wiki 方法论
    ├── RAG ── 补充了 Embedding 交叉引用
    └── Memex

UE / Niagara 源码学习 (UE 4.26)
    ├── UE 基础概念 (topic: UE/)
    ├── Niagara 基础 (topic: UE/Niagara/,子主题)
    ├── stock 代码实体 (repo: Stock/,与 topic 正交)
    │   (按模块下沉 Stock/Niagara/)
    │   ├── UNiagaraSystem          ← 顶层资产
    │   ├── UNiagaraEmitter         ← 粒子行为单元
    │   ├── FNiagaraEmitterHandle   ← System→Emitter 引用包装
    │   ├── UNiagaraScript          ← 编译后脚本(字节码 + GPU shader)
    │   └── UNiagaraScriptSourceBase ← 图源抽象基类
    ├── stock 代码实体 (Component 层)
    │   ├── UNiagaraComponent       ← 承载层主角(五职责)
    │   ├── ANiagaraActor           ← ComponentWrapperClass
    │   └── UNiagaraFunctionLibrary ← BP 静态工具集
    ├── stock 代码实体 (运行时实例层 / 心脏)
    │   ├── FNiagaraSystemInstance    ← 单实例状态机 + 三阶段 Tick
    │   ├── FNiagaraEmitterInstance   ← Emitter 粒子数据 + Exec 上下文
    │   └── FNiagaraSystemSimulation  ← 同 Asset 多实例批量 Tick 调度
    └── Niagara 学习路径 (10 阶段, **Phase 0-10 全部完成 ✅**)

AI 应用生态 (2026-04 新增)
    ├── 基础
    │   ├── LLM
    │   ├── 幻觉
    │   ├── 上下文窗口 & Context Rot
    │   └── 推理模型
    ├── Agent 时代(基础概念)
    │   ├── AI Agent
    │   ├── Multi-agent / Subagent(工具型,派活)
    │   ├── MCP
    │   ├── Harness Engineering
    │   ├── Agent Skills
    │   └── Agentic Grep
    ├── 综合:Prompt → Context → Harness 三段论
    └── 实体:OpenClaw

AI Agents (2026-04-25 新增)
    ├── 生成式智能体模拟研究
    │   ├── 实体
    │   │   ├── Joon Sung Park(朴俊成)← 方向奠基人
    │   │   ├── Stanford Smallville   ← 谱系源头(Park 2023)
    │   │   └── a16z AI Town          ← TS + Convex 工业重写
    │   └── 概念
    │       ├── 生成式智能体架构 = Memory + Reflection + Planning
    │       └── LLM 驱动的社会模拟(共生演化,对照工具型 multi-agent)
    └── 项目级 AI 应用落地(Aki 项目组)
        ├── 给美术做代码问答机器人 — agentic grep + wiki 复合记忆
        └── AI 特效贴图工具设计 — 三层架构 + VFX guardrail + ComfyUI 基座
```

---

## 开放问题 / 待观察

### 关于 Wiki 本身

- **规模边界**：当前约 37 页（AIFoundations 一次性新增 11 页），index.md 还够用；200 页时怎么办？
- **Concept 页跨仓共享**：`stock` 仓 ingest 时 UE 概念页能复用多少？
- **跨主题链接**：AIFoundations 与 Methodology 已经开始交叉（Karpathy、RAG），而 Niagara 仍相对独立——是特性还是可以挖深？

### 关于 Niagara 路径

- **Niagara 学习路径已完结**(2026-04-20):10 Phase 全通,用户"一口气推完 Phase 3-10"目标达成。新增 ~160 原子页 + 10 篇读本。下一步可以 lint 一次 + 把 open questions 按需深入。
- **Code root 本机可用性**:已登记 `stock`(F:\UnrealEngine-4.26,2026-04-20 验证可用),`project-*` 本机未登记
- **Phase 1 遗留的 open question**(等 Phase 3+ 回答):
  - `EmitterExecutionOrder.kStartNewOverlapGroupBit` 的 parallel tick 消费点
  - `RapidIterationParameters` vs `ExposedParameters` vs `User.*` 命名空间的协作关系
  - `FNiagaraVMExecutableData::DIParamInfo` 的技术债(GPU 信息不该在 VM 数据里)
  - `UNiagaraEmitter::Parent / ParentAtLastMerge` 的 merge 语义能传播什么
- **Phase 2 遗留的 open question**(等 Phase 3/6/7/9 回答):
  - `FNiagaraSystemSimulation` 批量 Tick 具体做了什么 / 与 `bForceSolo` 对比 → Phase 3
  - `FNiagaraSystemInstance` 状态机、Init、Tick 流程 → Phase 3
  - `ENCPoolMethod` 五取值决策表 + `UNiagaraComponentPool` 实现 → Phase 9
  - `FNiagaraScalabilityManager` PreCull / 运行时 cull 决策 → Phase 9
  - `FNiagaraSceneProxy` 的 GT↔RT 数据流 → Phase 6
  - `OverrideSystemUserVariableStaticMesh`(给 `UStaticMesh*` 而非 Component)的 transform 采样处理 → Phase 7
  - `VectorVM FastPath` 注册了哪些算子 → Phase 5

### 关于 AI 应用生态

- **MCP 治理细节**：是否已捐赠基金会/哪个基金会——v2 采取了模糊措辞，待官方公告核实。
- **`agentskills.io` 域名真伪**：v1 提到，v2 已改为"Anthropic 工程博客/GitHub 规范仓库"——如需精确引用需核实。
- **OpenClaw 技术细节**：扫盲源未给 GitHub 仓库、许可证、硬件需求等细节，后续如用户关心可单独 ingest。
- **已补建(2026-04-20)**:[[Wiki/Concepts/AIFoundations/Embedding]]、[[Wiki/Concepts/Methodology/Vibe-coding]]、[[Wiki/Concepts/UE/UE4-ddc]] (见 2026-04-20 lint 后的 🟡 批)
- **仍待建页**：Transformer、Function Calling、Subagent、HITL、Spec Coding、Mitchell Hashimoto、Martin Fowler 等 v2 中提及但未建页，按实际引用需要再建。

---

## 入口

- 所有页面目录：[[index]]
- 操作规程：[[CLAUDE]]
- 时间线日志：[[log]]
- **方法论原文**：[[Raw/Notes/Karpathy Wiki 方法论]]
- **AI 扫盲手册 v2**：[[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]

---

## 下一步建议

- ~~**Niagara Phase 10**~~ 已完成。Niagara 学习路径 10 Phase 全部打通。
- **AI 应用**：按需补建 Transformer / Function Calling / Spec Coding 等剩余待建页(Embedding / Vibe Coding 已在 2026-04-20 补建)
- **lint**：运行 "帮我 lint 一下 wiki"，检查 3 个主题的内部一致性、特别是新建 AIFoundations 主题的交叉引用闭环、Phase 1 新增 10 页的 back-link 情况
