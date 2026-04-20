---
type: overview
created: 2026-04-17
updated: 2026-04-20
tags: [overview]
sources: 9
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新：2026-04-20

---

## 当前主题（4 个）

### 1. 知识库方法论（meta / bootstrap）

自举阶段的产出：基于 [[Raw/Notes/Karpathy Wiki 方法论]] 建的三层架构（Raw / Wiki / Schema）；详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]。

📖 **主题读本（推荐初读）**：[[Readers/Methodology/Llm-wiki-方法论-读本]] — 把本仓库的起点、历史源流、RAG 对比、Karpathy 三里程碑、本仓库如何具体化编成一篇线性读物。

**核心主张**：持续整合 > 查询时检索。RAG 每次查询都在从零发现知识，LLM Wiki 则在 ingest 时就完成跨源整合、交叉引用、矛盾标注。

### 2. Niagara 源码学习（UE 4.26）

面向 C++ 零基础、UE 源码零基础的学员，通过 AI 辅助学习 Niagara 插件 749 个文件中的运行时部分（约 286 个文件）。

- 路线：[[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 10 阶段路径
- **Phase 0 ✅**：心智模型建立 —— 📖 **主题读本**：[[Readers/Niagara/Phase0-心智模型-读本]] / 原子概念页:[[Wiki/Concepts/UE4/UE4-uobject-系统]]、[[Wiki/Concepts/UE4/UE4-资产与实例]]、[[Wiki/Concepts/Niagara/Niagara-vs-cascade]]、[[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]
- **Phase 1 ✅**（2026-04-19）：Asset 层三件套 —— 📖 **主题读本**：[[Readers/Niagara/Phase1-asset-layer-读本]] / 原子页见 [[Wiki/Entities/Stock/UNiagaraSystem]]、[[Wiki/Entities/Stock/UNiagaraEmitter]]、[[Wiki/Entities/Stock/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/UNiagaraScript]]、[[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]
- **Phase 2 ✅**（2026-04-20）:Component 层 —— 📖 **主题读本**:[[Readers/Niagara/Phase2-component-layer-读本]] / 原子页见 [[Wiki/Entities/Stock/UNiagaraComponent]]、[[Wiki/Entities/Stock/ANiagaraActor]]、[[Wiki/Entities/Stock/UNiagaraFunctionLibrary]]
- **Phase 3 ✅**（2026-04-20）:运行时实例层(心脏)—— 📖 **主题读本**:[[Readers/Niagara/Phase3-runtime-instance-读本]] / 原子页见 [[Wiki/Entities/Stock/FNiagaraSystemInstance]]、[[Wiki/Entities/Stock/FNiagaraEmitterInstance]]、[[Wiki/Entities/Stock/FNiagaraSystemSimulation]]
- **Phase 4 ✅**（2026-04-20）:数据模型(类型系统 + SoA + 参数存储)—— 📖 **主题读本**:[[Readers/Niagara/Phase4-data-model-读本]] / 原子页见 [[Wiki/Entities/Stock/FNiagaraTypeDefinition]]、[[Wiki/Entities/Stock/FNiagaraVariable]]、[[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo]]、[[Wiki/Entities/Stock/FNiagaraConstants]]、[[Wiki/Entities/Stock/FNiagaraDataSet]]、[[Wiki/Entities/Stock/FNiagaraDataSetAccessor]]、[[Wiki/Entities/Stock/FNiagaraParameterStore]]
- **Phase 5 ✅**（2026-04-20）:CPU 脚本执行(VM + ExecContext + Padding)—— 📖 **主题读本**:[[Readers/Niagara/Phase5-cpu-script-execution-读本]] / 原子页见 [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]]、[[Wiki/Entities/Stock/FNiagaraScriptExecutionContext]]、[[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]]、[[Wiki/Entities/Stock/FNiagaraGPUSystemTick]]、[[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]]
- **Phase 6 ✅**（2026-04-20）:渲染系统(Sprite/Ribbon/Mesh/Light 5 对)—— 📖 **主题读本**:[[Readers/Niagara/Phase6-rendering-读本]] / 原子页见 [[Wiki/Entities/Stock/UNiagaraRendererProperties]]、[[Wiki/Entities/Stock/FNiagaraRenderer]]、[[Wiki/Entities/Stock/UNiagaraSpriteRendererProperties]]、[[Wiki/Entities/Stock/UNiagaraRibbonRendererProperties]]、[[Wiki/Entities/Stock/UNiagaraMeshRendererProperties]]、[[Wiki/Entities/Stock/UNiagaraLightRendererProperties]]
- Phase 7-10 等待逐文件 ingest

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

### 3. AI 美术生成管线（LoRA / ComfyUI）

面向鸣潮美术向 TA 的落地方案，从 MidJourney + tag 库逐步迁移到 ComfyUI + 自训 LoRA。

📖 **主题读本（推荐初读）**：[[Readers/AIArt/Lora-深度指南-读本]] — 从战略（离开 MJ 的三个结构性动因）到技术（LoRA 原理 + 基座选型 + caption 反常识 + 多 LoRA 组合）到工具（kohya_ss + ComfyUI）到工程落地（6 个月路线图 + 合规），一次读完掌握全链路。

- 核心源：[[Wiki/Sources/AIArt/Lora-deep-dive]]（2026-04，Eureka × Claude 撰写）
- 技术路径：[[Wiki/Concepts/AIArt/Lora|LoRA]] + [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious/NoobAI]] 基座 + [[Wiki/Entities/AIArt/Kohya-ss|kohya_ss]] 训练 + [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] 部署
- 关键洞察：[[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略反常识]]
- **当前阶段：准备期**（评估/选型中，未开始 MVP 训练）

### 4. AI 应用生态（2026-04 新增）

**面向**：对 AI 完全没概念的非开发角色（美术、设计、策划、管理者）+ 开发者的主线脉络梳理。

📖 **主题读本（推荐初读）**：[[Readers/AIApps/AI-primer-v2-读本]] — 从 LLM 本质到 2026 技术栈三层全景的完整读物,覆盖三个怪癖/推理模型/Agent/MCP/RAG/三段论/Harness 四柱/Skills/OpenClaw/Vibe-Spec-Harness Coding,一次读完。

- 核心源：[[Wiki/Sources/AIApps/AI-primer-v2]]（2026-04-19，Eureka × Claude 撰写，v2）
- 专题综合：[[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]](方法论演进专题)
- 基础概念：[[Wiki/Concepts/AIApps/Llm|LLM]]、[[Wiki/Concepts/AIApps/Hallucination|幻觉]]、[[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]]、[[Wiki/Concepts/AIApps/Reasoning-model|推理模型]]
- Agent 时代：[[Wiki/Concepts/AIApps/Ai-agent|AI Agent]]、[[Wiki/Concepts/AIApps/Mcp|MCP]]、[[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]]、[[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]]
- 落地产品：[[Wiki/Entities/AIApps/OpenClaw|OpenClaw（小龙虾）]]

**核心洞察**：2026 年 AI 技术栈三层结构（Model / Harness / Skills）中，模型层正在商品化，真正差异化竞争在 Harness 和 Skills 两层。用户第一纪律：AI 给的具体事实必须验证。

---

## 跨主题联系

- **方法论 → AI 应用生态**：本 wiki 正是"LLM Wiki 方法论"本身；AI Primer 则是我们用这套方法论 ingest 的第一份广域综合型源材料。
- **AI 应用 → AI 美术**：AI Primer 为 AI 美术管线提供了术语和心智模型的全局坐标——LoRA/基座模型都是 LLM/Transformer 生态的应用分支。
- **方法论 → AI 美术**：同一 wiki 自证了"LLM 可以管理跨领域知识库"的假设。
- **Niagara ↔ AI**：目前无实质交集，未来可能涉及 Niagara GPU 模拟路径对 AI 推理的借鉴。

---

## 主题知识图

```
methodology (meta, 自举)
    ├── Karpathy ── 也出现在 AI 应用生态（Context Engineering / Vibe Coding）
    ├── LLM Wiki 方法论
    ├── RAG ── 补充了 Embedding 交叉引用
    └── Memex

Niagara 源码学习 (UE 4.26)
    ├── UE4 基础
    ├── Niagara 基础
    ├── stock 代码实体 (Asset 层)
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
    └── Niagara 学习路径 (10 阶段, Phase 0-6 ✅)

AI 美术 (LoRA/ComfyUI)
    ├── 概念：LoRA / 基座选型 / Caption / Trigger Word / Multi-LoRA
    └── 实体：Illustrious / NoobAI / Flux / Kohya-ss / ComfyUI

AI 应用生态 (2026-04 新增)
    ├── 基础
    │   ├── LLM
    │   ├── 幻觉
    │   ├── 上下文窗口 & Context Rot
    │   └── 推理模型
    ├── Agent 时代
    │   ├── AI Agent
    │   ├── MCP
    │   ├── Harness Engineering
    │   └── Agent Skills
    ├── 综合：Prompt → Context → Harness 三段论
    └── 实体：OpenClaw
```

---

## 开放问题 / 待观察

### 关于 Wiki 本身

- **规模边界**：当前约 37 页（AIApps 一次性新增 11 页），index.md 还够用；200 页时怎么办？
- **Concept 页跨仓共享**：`stock` 仓 ingest 时 UE4 概念页能复用多少？
- **跨主题链接**：AIApps 与 Methodology 已经开始交叉（Karpathy、RAG），而 AIArt / Niagara 仍相对独立——是特性还是可以挖深？

### 关于 Niagara 路径

- **Phase 7 启动时机**:用户要求一口气推完 Phase 3-10,正在按序推进。下一步是 Phase 7 数据接口系统(10 文件)
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

### 关于 AI 美术

- **基座选型实时性**：每 3-6 月要重查一次当前 state
- **许可变动追溯**：Illustrious 条款历史变动是否影响已训 LoRA？
- ~~**鸣潮团队落地阶段**~~：已确认 = **准备期**（2026-04-19）

### 关于 AI 应用生态

- **MCP 治理细节**：是否已捐赠基金会/哪个基金会——v2 采取了模糊措辞，待官方公告核实。
- **`agentskills.io` 域名真伪**：v1 提到，v2 已改为"Anthropic 工程博客/GitHub 规范仓库"——如需精确引用需核实。
- **OpenClaw 技术细节**：扫盲源未给 GitHub 仓库、许可证、硬件需求等细节，后续如用户关心可单独 ingest。
- **已补建(2026-04-20)**:[[Wiki/Concepts/AIApps/Embedding]]、[[Wiki/Concepts/Methodology/Vibe-coding]]、[[Wiki/Concepts/UE4/UE4-ddc]] (见 2026-04-20 lint 后的 🟡 批)
- **仍待建页**：Transformer、Function Calling、Subagent、HITL、Spec Coding、Mitchell Hashimoto、Martin Fowler 等 v2 中提及但未建页，按实际引用需要再建。

---

## 入口

- 所有页面目录：[[index]]
- 操作规程：[[CLAUDE]]
- 时间线日志：[[log]]
- **方法论原文**：[[Raw/Notes/Karpathy Wiki 方法论]]
- **AI 美术路线源**：[[Raw/Notes/Lora_Deep_Dive]]
- **AI 扫盲手册 v2**：[[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]

---

## 下一步建议

- **Niagara Phase 7**:数据接口系统(10 文件,DI 基类 + 典型 DI:Curve/Camera/CollisionQuery/StaticMesh/SkeletalMesh/Texture/RenderTarget2D)
- **AI 美术**：验证本机 kohya_ss 环境，跑第一个 MVP LoRA
- **AI 应用**：按需补建 Transformer / Function Calling / Spec Coding 等剩余待建页(Embedding / Vibe Coding 已在 2026-04-20 补建)
- **lint**：运行 "帮我 lint 一下 wiki"，检查 4 个主题的内部一致性、特别是新建 AIApps 主题的交叉引用闭环、Phase 1 新增 10 页的 back-link 情况
