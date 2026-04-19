---
type: overview
created: 2026-04-17
updated: 2026-04-19
tags: [overview]
sources: 8
---

# Overview

> Wiki 的顶层综合视图。每次 ingest 有重大影响时,LLM 会在这里更新"当前的最佳理解"。

最后更新：2026-04-19

---

## 当前主题（4 个）

### 1. 知识库方法论（meta / bootstrap）

自举阶段的产出：基于 [[Raw/Notes/Karpathy Wiki 方法论]] 建的三层架构（Raw / Wiki / Schema）；详见 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]。

**核心主张**：持续整合 > 查询时检索。RAG 每次查询都在从零发现知识，LLM Wiki 则在 ingest 时就完成跨源整合、交叉引用、矛盾标注。

### 2. Niagara 源码学习（UE 4.26）

面向 C++ 零基础、UE 源码零基础的学员，通过 AI 辅助学习 Niagara 插件 749 个文件中的运行时部分（约 286 个文件）。

- 路线：[[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 10 阶段路径
- **Phase 0 ✅**：心智模型建立（[[Wiki/Concepts/UE4/UE4-uobject-系统]]、[[Wiki/Concepts/UE4/UE4-资产与实例]]、[[Wiki/Concepts/Niagara/Niagara-vs-cascade]]、[[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]）
- **Phase 1 ✅**（2026-04-19）：Asset 层三件套 —— [[Wiki/Entities/Stock/UNiagaraSystem]] / [[Wiki/Entities/Stock/UNiagaraEmitter]] / [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] / [[Wiki/Entities/Stock/UNiagaraScript]] / [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]，对应 5 个源摘要页在 `Wiki/Sources/Stock/`
- Phase 2-10 等待逐文件 ingest

**Phase 1 的关键收获**（从 5 个 header 里提炼）：
- **Asset 链路定型**：System →（`TArray<FNiagaraEmitterHandle>`）→ Handle(Id+Name+enabled+Instance) → Emitter →（脚本 / 渲染器 / SimStages）→ Script（字节码 + GPU shader）
- **命名陷阱**：`FNiagaraEmitterHandle::Instance` 仍在 Asset 层（Emitter 资产副本），与 Phase 3 的 `FNiagaraEmitterInstance`（运行时粒子模拟）**完全不是一回事**
- **EmitterSpawn/EmitterUpdate 脚本不独立编译**，被合并进 System 脚本（`UNiagaraScript::IsCompilable()` 对这两者返 false）
- **editor/runtime 解耦点**：`UNiagaraScriptSourceBase` 抽象基类在 runtime 模块，真正的图实现 `UNiagaraScriptSource + UNiagaraGraph` 在 NiagaraEditor 模块，使 runtime 能持图指针但不 link editor

### 3. AI 美术生成管线（LoRA / ComfyUI）

面向鸣潮美术向 TA 的落地方案，从 MidJourney + tag 库逐步迁移到 ComfyUI + 自训 LoRA。

- 核心源：[[Wiki/Sources/AIArt/Lora-deep-dive]]（2026-04，Eureka × Claude 撰写）
- 技术路径：[[Wiki/Concepts/AIArt/Lora|LoRA]] + [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious/NoobAI]] 基座 + [[Wiki/Entities/AIArt/Kohya-ss|kohya_ss]] 训练 + [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] 部署
- 关键洞察：[[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略反常识]]
- **当前阶段：准备期**（评估/选型中，未开始 MVP 训练）

### 4. AI 应用生态（2026-04 新增）

**面向**：对 AI 完全没概念的非开发角色（美术、设计、策划、管理者）+ 开发者的主线脉络梳理。

- 核心源：[[Wiki/Sources/AIApps/AI-primer-v2]]（2026-04-19，Eureka × Claude 撰写，v2）
- 主线叙事：[[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]]
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
    └── Niagara 学习路径 (10 阶段, Phase 0 ✅ / Phase 1 ✅)

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

- **Phase 2 启动时机**：等待用户指令，下一步是 `NiagaraComponent.h / NiagaraActor.h / NiagaraFunctionLibrary.h`（从场景入口视角把 Asset 连到 World）
- **Code root 本机可用性**：已登记 `stock`（D:\UE\...），`project-*` 本机未登记
- **Phase 1 遗留的 open question**（等 Phase 3+ 回答）：
  - `EmitterExecutionOrder.kStartNewOverlapGroupBit` 的 parallel tick 消费点
  - `RapidIterationParameters` vs `ExposedParameters` vs `User.*` 命名空间的协作关系
  - `FNiagaraVMExecutableData::DIParamInfo` 的技术债（GPU 信息不该在 VM 数据里）
  - `UNiagaraEmitter::Parent / ParentAtLastMerge` 的 merge 语义能传播什么

### 关于 AI 美术

- **基座选型实时性**：每 3-6 月要重查一次当前 state
- **许可变动追溯**：Illustrious 条款历史变动是否影响已训 LoRA？
- ~~**鸣潮团队落地阶段**~~：已确认 = **准备期**（2026-04-19）

### 关于 AI 应用生态

- **MCP 治理细节**：是否已捐赠基金会/哪个基金会——v2 采取了模糊措辞，待官方公告核实。
- **`agentskills.io` 域名真伪**：v1 提到，v2 已改为"Anthropic 工程博客/GitHub 规范仓库"——如需精确引用需核实。
- **OpenClaw 技术细节**：扫盲源未给 GitHub 仓库、许可证、硬件需求等细节，后续如用户关心可单独 ingest。
- **待建页**：Transformer、Embedding、Function Calling、Subagent、HITL、Vibe/Spec Coding、Mitchell Hashimoto、Martin Fowler 等 v2 中提及但未建页，按实际引用需要再建。

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

- **Niagara Phase 2**：读 `NiagaraComponent.h` / `NiagaraActor.h` / `NiagaraFunctionLibrary.h`（3 文件，Component 层，把 Asset 连到场景 / BP）
- **AI 美术**：验证本机 kohya_ss 环境，跑第一个 MVP LoRA
- **AI 应用**：按需补建 Transformer / Embedding / Vibe Coding 等提及但未建页面
- **lint**：运行 "帮我 lint 一下 wiki"，检查 4 个主题的内部一致性、特别是新建 AIApps 主题的交叉引用闭环、Phase 1 新增 10 页的 back-link 情况
