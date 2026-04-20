---
type: synthesis
created: 2026-04-18
updated: 2026-04-20
tags: [niagara, UE4, learning-path, source-code, roadmap]
sources: 0
aliases: [Niagara 学习路径, niagara 源码路径]
repo: stock
source_root: UE-4.26-Stock
---

# Niagara 源码学习路径

> 本文档是针对 UE 4.26 Niagara 插件源码的**系统性学习路线图**。
> 面向对象：C++ 零基础、UE 源码零基础，依靠 AI 辅助学习。
> 用法：按阶段顺序推进，每个文件单独生成一篇 `Wiki/Sources/Stock/` 下的说明文档，本页作为导航索引。

---

## 插件全局概览

### 模块地图

Niagara 插件位于 `Engine/Plugins/FX/Niagara/`，共 7 个模块：

| 模块 | 文件数 | 类型 | 一句话职责 |
|---|---|---|---|
| `NiagaraCore` | 10 | 运行时 | 最底层基类：编译哈希、版本号、DataInterface 基类 |
| `NiagaraShader` | 17 | 运行时 | GPU 着色器编译与管理 |
| `NiagaraVertexFactories` | 17 | 运行时 | GPU 渲染所需的顶点工厂（sprite/ribbon/mesh） |
| `Niagara` | 242 | 运行时 ⭐ | **核心**：Asset、运行时、渲染、数据接口全部在这里 |
| `NiagaraAnimNotifies` | 6 | 运行时 | 动画通知触发 Niagara 特效 |
| `NiagaraEditor` | 391 | 编辑器 | 编辑器 UI、视口、编译流程 |
| `NiagaraEditorWidgets` | 66 | 编辑器 | 编辑器 Stack 面板 Widget |

**学习阶段只覆盖运行时模块**（NiagaraCore + NiagaraShader + NiagaraVertexFactories + Niagara），合计约 286 个文件。
编辑器模块（NiagaraEditor/NiagaraEditorWidgets）作为可选扩展，不在主路径内。

### 三层理解模型

在开始读代码之前，牢记 Niagara 的三层结构：

```
┌─────────────────────────────────────────┐
│  Layer 1: Asset（资产层，存在磁盘）       │
│  NiagaraSystem → NiagaraEmitter → NiagaraScript
│  "描述特效长什么样"（静态数据）            │
└─────────────────────────────────────────┘
           │  在场景中被 NiagaraComponent 引用
           ▼
┌─────────────────────────────────────────┐
│  Layer 2: Instance（实例层，运行时）      │
│  NiagaraSystemInstance → NiagaraEmitterInstance
│  "特效正在运行的状态"（动态对象）          │
└─────────────────────────────────────────┘
           │  每帧计算结果送往
           ▼
┌─────────────────────────────────────────┐
│  Layer 3: Render（渲染层，GPU）          │
│  NiagaraRenderer* + VertexFactory*       │
│  "粒子如何画到屏幕上"                    │
└─────────────────────────────────────────┘
```

---

## 学习路线图

### Phase 0 — 上阵前：基础心智模型 ✅
> **无需读代码。** 这一阶段通过概念建立正确的"脑内地图"，避免后面读代码时迷失。
>
> 📖 **主题读本(推荐初读)**：[[Readers/Niagara/Phase0-心智模型-读本]] — 把四个概念自下而上(UObject → Asset/Instance → Niagara vs Cascade → CPU/GPU)编成一条叙事链,一次读完掌握 Phase 1+ 所需全部前置。

- [[Wiki/Concepts/UE4/UE4-uobject-系统]] *(已完成)* — UCLASS / USTRUCT / UPROPERTY 宏是什么，UObject 为什么重要
- [[Wiki/Concepts/UE4/UE4-资产与实例]] *(已完成)* — Asset（Content Browser 里的文件）vs Instance（运行时对象）的本质区别
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] *(已完成)* — 为什么 Niagara 取代 Cascade，核心设计哲学差异
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] *(已完成)* — CPU Simulation 与 GPU Simulation 的分工，何时用哪种

**完成标志**：你能用自己的话解释"一个 NiagaraSystem 资产和一个正在运行的 NiagaraSystemInstance 有什么区别"。

---

### Phase 1 — 资产层：三件套 Asset ✅
> **目标**：理解 Niagara 特效的"静态定义"——存在磁盘上的数据结构长什么样。
>
> 📖 **主题读本(推荐初读)**：[[Readers/Niagara/Phase1-asset-layer-读本]] — 把下面 5 个文件讲成一个连贯故事,从 `UNiagaraSystem` 到图源抽象基类一气读完。

| # | 文件 | 模块 | 路径 | 源摘要 → 主实体 |
|---|---|---|---|---|
| 1.1 | `NiagaraSystem.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraSystem]] → [[Wiki/Entities/Stock/UNiagaraSystem]] |
| 1.2 | `NiagaraEmitter.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraEmitter]] → [[Wiki/Entities/Stock/UNiagaraEmitter]] |
| 1.3 | `NiagaraEmitterHandle.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraEmitterHandle]] → [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] |
| 1.4 | `NiagaraScript.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraScript]] → [[Wiki/Entities/Stock/UNiagaraScript]] |
| 1.5 | `NiagaraScriptSourceBase.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraScriptSourceBase]] → [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] |

**学习要点：**
- `UNiagaraSystem` 持有一个 `TArray<FNiagaraEmitterHandle>`，即"一个特效由多个 Emitter 组成"
- `UNiagaraEmitter` 持有渲染器列表（`TArray<UNiagaraRendererProperties*>`）和脚本列表（`NiagaraScript`）
- `UNiagaraScript` 是编译后的字节码容器，区分 `Spawn / Update / Event` 等脚本类型
- `FNiagaraEmitterHandle` 是"System 对 Emitter 的引用包装"，含启用/禁用状态

**难度**：⭐⭐☆☆☆（数据结构为主，逻辑少）

---

### Phase 2 — 场景入口：Component 层 ✅
> **目标**：理解特效如何被放入游戏世界，以及 Blueprint 如何调用。
>
> 📖 **主题读本(推荐初读)**：[[Readers/Niagara/Phase2-component-layer-读本]] — 把 Component/Actor/FunctionLibrary 三文件讲成一个完整故事,从三种入口路径到 Component 的 5 大职责、生命周期四源、Pool/Scalability/AutoDestroy 三方决策,一次读完掌握"特效如何从资产变成场景里跑着的东西"。

| # | 文件 | 模块 | 路径 | 源摘要 → 主实体 |
|---|---|---|---|---|
| 2.1 | `NiagaraComponent.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraComponent]] → [[Wiki/Entities/Stock/UNiagaraComponent]] |
| 2.2 | `NiagaraActor.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraActor]] → [[Wiki/Entities/Stock/ANiagaraActor]] |
| 2.3 | `NiagaraFunctionLibrary.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraFunctionLibrary]] → [[Wiki/Entities/Stock/UNiagaraFunctionLibrary]] |

**学习要点：**
- `UNiagaraComponent` 继承自 `UFXSystemComponent`，持有一个 `NiagaraSystemInstance`
- 关键生命周期：`Activate()` → 创建 SystemInstance → `DeactivateSystem()` → 销毁或池化
- `NiagaraFunctionLibrary` 提供 Blueprint 可调用的静态函数：`SpawnSystemAtLocation`、`SpawnSystemAttached` 等
- `NiagaraActor` 是极简封装，只是在 Actor 上挂一个 `NiagaraComponent`

**难度**：⭐⭐☆☆☆（接口清晰，重点是生命周期流程）

---

### Phase 3 — 运行时实例层 ✅
> **目标**：理解特效"活着"时的状态机与 Tick 流程。这是 Niagara 的心脏。
>
> 📖 **主题读本(推荐初读)**:[[Readers/Niagara/Phase3-runtime-instance-读本]] — 三个类由内到外讲清楚:EmitterInstance(最内粒子数据 + ExecContext)→ SystemInstance(中间状态机 + 三阶段 Tick)→ SystemSimulation(最外批量调度 + 4-instance Tick Batch)。读完能算出 `bForceSolo` 的定量退化。

| # | 文件 | 模块 | 路径 | 源摘要 → 主实体 |
|---|---|---|---|---|
| 3.1 | `NiagaraSystemInstance.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraSystemInstance]] → [[Wiki/Entities/Stock/FNiagaraSystemInstance]] |
| 3.2 | `NiagaraEmitterInstance.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraEmitterInstance]] → [[Wiki/Entities/Stock/FNiagaraEmitterInstance]] |
| 3.3 | `NiagaraSystemSimulation.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraSystemSimulation]] → [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] |

**学习要点：**
- `FNiagaraSystemInstance` 持有 `TArray<TSharedRef<FNiagaraEmitterInstance>>`，管理所有 Emitter 实例
- 状态机：`Uninitialized → Initialized → Running → Complete → Disabled`
- `FNiagaraSystemSimulation` 管理同一 World 下相同 System 的多个 Instance 的**批量 Tick**（性能优化关键）
- 理解 `TickGroup`（哪一帧阶段执行）与异步 Tick 的基本概念

**难度**：⭐⭐⭐☆☆（状态机复杂，先看状态枚举和主 Tick 函数即可）

---

### Phase 4 — 数据基础：类型与数据集 ✅
> **目标**：理解 Niagara 的"数据语言"——粒子属性如何定义、存储、访问。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase4-data-model-读本]] — 从共享常量到类型系统、SoA 布局、参数存储绑定图,一次读完掌握 Niagara 的数据语言全貌。

| # | 文件 | 模块 | 路径 | 源摘要 |
|---|---|---|---|---|
| 4.1 | `NiagaraTypes.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraTypes]] |
| 4.2 | `NiagaraCommon.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraCommon]] |
| 4.3 | `NiagaraConstants.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraConstants]] |
| 4.4 | `NiagaraDataSet.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraDataSet]] |
| 4.5 | `NiagaraDataSetAccessor.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraDataSetAccessor]] |
| 4.6 | `NiagaraParameters.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraParameters]] |
| 4.7 | `NiagaraParameterStore.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraParameterStore]] |

**学习要点：**
- `FNiagaraTypeDefinition`：Niagara 的类型系统（Float、Vector3、Int32 等），是所有属性的"身份证"
- `FNiagaraVariable`：`TypeDef + Name`，一个命名的变量，如 `Particles.Position`
- `FNiagaraDataSet`：一大块连续内存，存储所有活跃粒子的所有属性（SoA 布局：结构体数组→数组的结构体）
- `FNiagaraDataSetAccessor`：类型安全的读写粒子数据的辅助类
- `FNiagaraParameterStore`：存储 System/Emitter 级别的参数（非粒子属性），如用户暴露的浮点参数

**难度**：⭐⭐⭐☆☆（数据结构重要，SoA 布局是关键概念）

---

### Phase 5 — 脚本执行：CPU 虚拟机 ✅
> **目标**：理解 Niagara 脚本（Spawn/Update/Event）如何在 CPU 上被逐粒子执行。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase5-cpu-script-execution-读本]] — CPU VM 主路径清晰,GPU 部分作 Phase 8 预告。含 System 脚本的 PerInstanceFunctionHook 机制、CPU/GPU 参数布局 padding 映射、5 种 UniformBuffer 的用法。

| # | 文件 | 模块 | 路径 | 源摘要 |
|---|---|---|---|---|
| 5.1 | `NiagaraCore.h` | NiagaraCore | `Public/` | [[Wiki/Sources/Stock/NiagaraCore]] |
| 5.2 | `NiagaraDataInterfaceBase.h` | NiagaraCore | `Public/` | [[Wiki/Sources/Stock/NiagaraDataInterfaceBase]] |
| 5.3 | `NiagaraScriptExecutionContext.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] |
| 5.4 | `NiagaraScriptExecutionParameterStore.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraScriptExecutionParameterStore]] |
| 5.5 | `NiagaraEmitterInstanceBatcher.h` *(实际是 GPU Batcher,Phase 8 详)* | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]] |

**学习要点：**
- Niagara CPU 脚本最终跑在 **VectorVM**（`Engine/Source/Runtime/VectorVM`，引擎内置，不在插件）上——一个 SIMD 批量计算虚拟机
- `FNiagaraScriptExecutionContext` 是单次脚本执行的"执行上下文"：字节码 + 数据绑定 + DataInterface 绑定
- 理解"绑定"的概念：脚本里的变量名 → DataSet 里的内存偏移（编译期完成）
- `FNiagaraEmitterInstanceBatcher` 是 CPU/GPU 任务的调度器；Phase 8 再深入其 GPU 侧

**难度**：⭐⭐⭐⭐☆（执行机制较深，先理解"脚本是怎么知道读写哪块内存"即可）

---

### Phase 6 — 渲染系统 ✅
> **目标**：理解粒子数据如何变成屏幕上的像素。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase6-rendering-读本]] — Properties/Renderer 对偶 + 4 种类型能力边界 + GT→RT 数据流 + Cutout/Tessellation/SimpleLight 各自的特殊路径。

| # | 文件 | 模块 | 路径 | 源摘要 |
|---|---|---|---|---|
| 6.1 | `NiagaraRendererProperties.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRendererProperties]] |
| 6.2 | `NiagaraRenderer.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRenderer]] |
| 6.3 | `NiagaraSpriteRendererProperties.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraSpriteRendererProperties]] |
| 6.4 | `NiagaraRendererSprites.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRendererSprites]] |
| 6.5 | `NiagaraRibbonRendererProperties.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRibbonRendererProperties]] |
| 6.6 | `NiagaraRendererRibbons.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRendererRibbons]] |
| 6.7 | `NiagaraMeshRendererProperties.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraMeshRendererProperties]] |
| 6.8 | `NiagaraRendererMeshes.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRendererMeshes]] |
| 6.9 | `NiagaraLightRendererProperties.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraLightRendererProperties]] |
| 6.10 | `NiagaraRendererLights.h` | Niagara | `Public/` | [[Wiki/Sources/Stock/NiagaraRendererLights]] |

**学习要点：**
- `UNiagaraRendererProperties`（Asset 侧）vs `NiagaraRenderer`（运行时侧）：同样的 Asset/Instance 对偶
- 每种渲染器负责从 `FNiagaraDataSet` 读粒子数据 → 填充 GPU Buffer → 提交渲染命令
- `FNiagaraRendererSprites`：最常用，每粒子一个 Billboard quad
- `FNiagaraRendererRibbons`：需要粒子有序，生成连续带状几何体
- `FNiagaraRendererMeshes`：每粒子一个 Instanced Mesh，依赖 GPU Instancing
- `FNiagaraRendererLights`：每粒子创建一个动态点光源（开销大，慎用）

**难度**：⭐⭐⭐☆☆（理解 UE 渲染线程分离概念后会容易很多）

---

### Phase 7 — 数据接口系统 ✅
> **目标**：理解 Niagara 脚本如何访问"外部数据"（相机、骨骼网格、曲线等）。DataInterface 是 Niagara 最强大也最复杂的扩展点。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase7-data-interface-读本]] — DI 三路代码生成(VM/C++ lambda/HLSL)+ Per-Instance Data 生命周期 + 7 种典型 DI 能力矩阵 + 共享 skinning 缓存。

**第一轮：基类**

| # | 文件 | 模块 | 路径 | 源摘要 |
|---|---|---|---|---|
| 7.1 | `NiagaraDataInterfaceBase.h` | NiagaraCore | `Public/` | [[Wiki/Sources/Stock/NiagaraDataInterfaceBase]] (Phase 5.2 已覆盖) |
| 7.2 | `NiagaraDataInterface.h` | Niagara | `Classes/` | [[Wiki/Sources/Stock/NiagaraDataInterface]] |

**第二轮：典型 DI（由易到难）**

| # | 文件 | 模块 | 说明 | 源摘要 |
|---|---|---|---|---|
| 7.3 | `NiagaraDataInterfaceCurve.h` | Niagara | 最简单：读一条浮点曲线 | [[Wiki/Sources/Stock/NiagaraDataInterfaceCurve]] |
| 7.4 | `NiagaraDataInterfaceCurveBase.h` | Niagara | Curve DI 的公共基类 | [[Wiki/Sources/Stock/NiagaraDataInterfaceCurveBase]] |
| 7.5 | `NiagaraDataInterfaceCamera.h` | Niagara | 提供相机位置/方向给脚本 | [[Wiki/Sources/Stock/NiagaraDataInterfaceCamera]] |
| 7.6 | `NiagaraDataInterfaceCollisionQuery.h` | Niagara | 粒子物理碰撞查询 | [[Wiki/Sources/Stock/NiagaraDataInterfaceCollisionQuery]] |
| 7.7 | `NiagaraDataInterfaceStaticMesh.h` | Niagara | 从静态网格采样位置/法线 | [[Wiki/Sources/Stock/NiagaraDataInterfaceStaticMesh]] |
| 7.8 | `NiagaraDataInterfaceSkeletalMesh.h` | Niagara | 从骨骼网格采样（最复杂之一） | [[Wiki/Sources/Stock/NiagaraDataInterfaceSkeletalMesh]] |
| 7.9 | `NiagaraDataInterfaceTexture.h` | Niagara | 采样 2D 纹理 | [[Wiki/Sources/Stock/NiagaraDataInterfaceTexture]] |
| 7.10 | `NiagaraDataInterfaceRenderTarget2D.h` | Niagara | 读写 RenderTarget（GPU DI） | [[Wiki/Sources/Stock/NiagaraDataInterfaceRenderTarget2D]] |

**学习要点：**
- DI 的核心机制：在编辑器中"注册函数名" → 编译时绑定 → 运行时通过函数指针/shader binding 执行
- CPU DI vs GPU DI 的实现差异（CPU：C++ 函数；GPU：生成 HLSL 代码片段）
- `GetFunctions()` 方法：每个 DI 必须实现，声明它提供哪些函数给脚本
- `BindCPUSimulationFunction()` vs `GetParameterDefinitionHLSL() / GetFunctionHLSL()`

**难度**：⭐⭐⭐⭐☆（CPU DI 尚可；GPU DI 涉及代码生成，难度高）

---

### Phase 8 — GPU 模拟 ✅
> **目标**：理解 GPU Simulation 的完整管线——脚本如何变成 Compute Shader，粒子数据如何在 GPU 上流转。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase8-gpu-simulation-读本]] — Shader 编译 + GPU Instance Count + Sort + VertexFactory + Draw Indirect 完整链路。

| # | 文件 | 模块 | 路径 |
|---|---|---|---|
| 8.1 | `NiagaraShared.h` | NiagaraShader | `Public/` |
| 8.2 | `NiagaraShader.h` | NiagaraShader | `Public/` |
| 8.3 | `NiagaraShaderType.h` | NiagaraShader | `Public/` |
| 8.4 | `NiagaraShaderMap.h` | NiagaraShader | `Public/` |
| 8.5 | `NiagaraScriptBase.h` | NiagaraShader | `Public/` |
| 8.6 | `NiagaraGPUInstanceCountManager.h` | Niagara | `Classes/` |
| 8.7 | `NiagaraGPUSortInfo.h` | Niagara | `Classes/` |
| 8.8 | `NiagaraEmitterInstanceBatcher.h` *(GPU 侧)* | Niagara | `Classes/` |
| 8.9 | `NiagaraVertexFactory.h` | NiagaraVertexFactories | `Public/` |
| 8.10 | `NiagaraSpriteVertexFactory.h` | NiagaraVertexFactories | `Public/` |
| 8.11 | `NiagaraRibbonVertexFactory.h` | NiagaraVertexFactories | `Public/` |
| 8.12 | `NiagaraMeshVertexFactory.h` | NiagaraVertexFactories | `Public/` |
| 8.13 | `NiagaraSortingGPU.h` | NiagaraVertexFactories | `Public/` |
| 8.14 | `NiagaraDrawIndirect.h` | NiagaraVertexFactories | `Public/` |

**学习要点：**
- `FNiagaraShader`：每个 GPU Emitter 对应一个 Compute Shader 变体
- `FNiagaraShaderMap`：管理 Shader 变体缓存（类比材质的 FMaterialShaderMap）
- `FNiagaraEmitterInstanceBatcher`（GPU 侧）：每帧将所有 GPU Emitter 的 Dispatch 命令排队到渲染线程
- `NiagaraGPUInstanceCountManager`：GPU 上的粒子计数 Buffer，避免 CPU Readback
- `NiagaraDrawIndirect`：用 Indirect Draw 绕过 CPU 不知道粒子数量的问题
- Vertex Factory：连接粒子 Buffer 与材质 Shader 的"桥"

**难度**：⭐⭐⭐⭐⭐（需要对 UE 渲染管线和 RDG 有基础了解）

---

### Phase 9 — 世界管理与可扩展性 ✅
> **目标**：理解 Niagara 如何管理全局资源、LOD、对象池。
>
> 📖 **主题读本**:[[Readers/Niagara/Phase9-world-management-读本]] — 全局状态四层模型 + Scalability 决策引擎 + Pool 5 方法取舍 + PlatformSet 三态配置。

| # | 文件 | 模块 | 路径 |
|---|---|---|---|
| 9.1 | `NiagaraWorldManager.h` | Niagara | `Public/` |
| 9.2 | `NiagaraScalabilityManager.h` | Niagara | `Public/` |
| 9.3 | `NiagaraComponentPool.h` | Niagara | `Public/` |
| 9.4 | `NiagaraSettings.h` | Niagara | `Public/` |
| 9.5 | `NiagaraEffectType.h` | Niagara | `Classes/` |
| 9.6 | `NiagaraPlatformSet.h` | Niagara | `Classes/` |

**学习要点：**
- `FNiagaraWorldManager`：每个 UWorld 一个实例，持有所有 `FNiagaraSystemSimulation`，负责全局 Tick 调度
- `FNiagaraScalabilityManager`：基于距离/重要性/预算动态 kill 或降级特效
- `UNiagaraEffectType`：特效类型（如 "Blood"、"Fire"），定义可扩展性预算和 LOD 规则
- `UNiagaraComponentPool`：复用 Component 避免频繁 GC（`SpawnSystemAtLocation` 内部就用这个）
- `UNiagaraSettings`：项目级全局配置（默认渲染器、FixedBounds 等）

**难度**：⭐⭐⭐☆☆（逻辑清晰，偏系统架构）

---

### Phase 10 — 高级特性（选修）
> **目标**：了解 Niagara 的前沿能力——Simulation Stages 与 Grid 流体模拟。

| # | 文件 | 模块 | 路径 |
|---|---|---|---|
| 10.1 | `NiagaraSimulationStageBase.h` | Niagara | `Public/` |
| 10.2 | `NiagaraDataInterfaceRW.h` | Niagara | `Classes/` |
| 10.3 | `NiagaraDataInterfaceGrid2DCollection.h` | Niagara | `Classes/` |
| 10.4 | `NiagaraDataInterfaceGrid2DCollectionReader.h` | Niagara | `Classes/` |
| 10.5 | `NiagaraDataInterfaceGrid3DCollection.h` | Niagara | `Classes/` |
| 10.6 | `NiagaraDataInterfaceNeighborGrid3D.h` | Niagara | `Classes/` |

**学习要点：**
- `UNiagaraSimulationStageBase`：在一次 Emitter Tick 内，定义多次 Dispatch pass，实现 Iteration 式计算
- `Grid2DCollection` / `Grid3DCollection`：在 GPU 上维护一个 2D/3D 网格 Buffer，供粒子读写
- `NeighborGrid3D`：空间哈希，用于粒子间邻域查询（流体模拟的基础）
- 这些是实现 Fluids、Smoke、水面等高级效果的底层机制

**难度**：⭐⭐⭐⭐⭐（需要 Phase 7/8 完全掌握后才适合学习）

---

## 总览表

| 阶段 | 主题 | 文件数 | 难度 | 备注 |
|---|---|---|---|---|
| Phase 0 | 心智模型 | 0（概念） | ⭐ | 必读，无代码 |
| Phase 1 | Asset 三件套 | 5 | ⭐⭐ | 数据结构为主 |
| Phase 2 | Component 层 | 3 | ⭐⭐ | 接口/生命周期 |
| Phase 3 | 运行时实例 | 3 | ⭐⭐⭐ | 状态机 + Tick |
| Phase 4 | 数据模型 | 7 | ⭐⭐⭐ | SoA 布局关键 |
| Phase 5 | CPU 脚本执行 | 5 | ⭐⭐⭐⭐ | VectorVM 关联 |
| Phase 6 | 渲染系统 | 10 | ⭐⭐⭐ | 渲染线程概念 |
| Phase 7 | 数据接口 | 10 | ⭐⭐⭐⭐ | 最强扩展点 |
| Phase 8 | GPU 模拟 | 14 | ⭐⭐⭐⭐⭐ | 最难，需前置 |
| Phase 9 | 世界管理 | 6 | ⭐⭐⭐ | 架构/性能 |
| Phase 10 | 高级特性 | 6 | ⭐⭐⭐⭐⭐ | 选修 |
| **合计** | | **~69 文件** | | |

---

## 学习建议

1. **不要跳跃**：Phase 1→2→3 是地基，必须按序。Phase 8/10 跳进去会看不懂。
2. **先看 `.h` 再看 `.cpp`**：头文件是"说明书"，`.cpp` 是"实现细节"。让 AI 先给你讲头文件。
3. **结合 Editor 使用**：每读一个类，在 Niagara Editor 里找到对应的 UI 操作（如"添加 Emitter"就是在操作 `NiagaraSystem` 里的 `EmitterHandles`）。
4. **概念 > 细节**：每个文件重点理解"这个类是干什么的，有哪些主要字段，和哪些类协作"，不需要背函数实现。
5. **标注矛盾**：读到和预期不符的地方，记录到对应 wiki 页面的"开放问题"区。

---

## 进度追踪

### 已完成文档

**Phase 0 ✅ 完成**（2026-04-18）
**Phase 1 ✅ 完成**（2026-04-19）
**Phase 2 ✅ 完成**（2026-04-20）
**Phase 3 ✅ 完成**（2026-04-20）
**Phase 4 ✅ 完成**（2026-04-20）
**Phase 5 ✅ 完成**（2026-04-20）
**Phase 6 ✅ 完成**（2026-04-20）
**Phase 7 ✅ 完成**（2026-04-20）
**Phase 8 ✅ 完成**（2026-04-20）
**Phase 9 ✅ 完成**（2026-04-20）

### Phase 0（概念页）
- [x] [[Wiki/Concepts/UE4/UE4-uobject-系统]]
- [x] [[Wiki/Concepts/UE4/UE4-资产与实例]]
- [x] [[Wiki/Concepts/Niagara/Niagara-vs-cascade]]
- [x] [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]

### Phase 1（Asset 层）
- [x] [[Wiki/Sources/Stock/NiagaraSystem]] → [[Wiki/Entities/Stock/UNiagaraSystem]]
- [x] [[Wiki/Sources/Stock/NiagaraEmitter]] → [[Wiki/Entities/Stock/UNiagaraEmitter]]
- [x] [[Wiki/Sources/Stock/NiagaraEmitterHandle]] → [[Wiki/Entities/Stock/FNiagaraEmitterHandle]]
- [x] [[Wiki/Sources/Stock/NiagaraScript]] → [[Wiki/Entities/Stock/UNiagaraScript]]
- [x] [[Wiki/Sources/Stock/NiagaraScriptSourceBase]] → [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]

### Phase 2（Component 层）
- [x] [[Wiki/Sources/Stock/NiagaraComponent]] → [[Wiki/Entities/Stock/UNiagaraComponent]]
- [x] [[Wiki/Sources/Stock/NiagaraActor]] → [[Wiki/Entities/Stock/ANiagaraActor]]
- [x] [[Wiki/Sources/Stock/NiagaraFunctionLibrary]] → [[Wiki/Entities/Stock/UNiagaraFunctionLibrary]]

### Phase 3（运行时实例）
- [x] [[Wiki/Sources/Stock/NiagaraSystemInstance]] → [[Wiki/Entities/Stock/FNiagaraSystemInstance]]
- [x] [[Wiki/Sources/Stock/NiagaraEmitterInstance]] → [[Wiki/Entities/Stock/FNiagaraEmitterInstance]]
- [x] [[Wiki/Sources/Stock/NiagaraSystemSimulation]] → [[Wiki/Entities/Stock/FNiagaraSystemSimulation]]

### Phase 4（数据模型）
- [x] [[Wiki/Sources/Stock/NiagaraTypes]] → [[Wiki/Entities/Stock/FNiagaraTypeDefinition]] / [[Wiki/Entities/Stock/FNiagaraVariable]] / [[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo]]
- [x] [[Wiki/Sources/Stock/NiagaraCommon]]
- [x] [[Wiki/Sources/Stock/NiagaraConstants]] → [[Wiki/Entities/Stock/FNiagaraConstants]]
- [x] [[Wiki/Sources/Stock/NiagaraDataSet]] → [[Wiki/Entities/Stock/FNiagaraDataSet]]
- [x] [[Wiki/Sources/Stock/NiagaraDataSetAccessor]] → [[Wiki/Entities/Stock/FNiagaraDataSetAccessor]]
- [x] [[Wiki/Sources/Stock/NiagaraParameters]]
- [x] [[Wiki/Sources/Stock/NiagaraParameterStore]] → [[Wiki/Entities/Stock/FNiagaraParameterStore]]

### Phase 5（CPU VM）
- [x] [[Wiki/Sources/Stock/NiagaraCore]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceBase]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]]
- [x] [[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] → [[Wiki/Entities/Stock/FNiagaraScriptExecutionContext]] / [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]] / [[Wiki/Entities/Stock/FNiagaraGPUSystemTick]]
- [x] [[Wiki/Sources/Stock/NiagaraScriptExecutionParameterStore]]
- [x] [[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]] → [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]]

### Phase 6（渲染）
- [x] [[Wiki/Sources/Stock/NiagaraRendererProperties]] → [[Wiki/Entities/Stock/UNiagaraRendererProperties]]
- [x] [[Wiki/Sources/Stock/NiagaraRenderer]] → [[Wiki/Entities/Stock/FNiagaraRenderer]]
- [x] [[Wiki/Sources/Stock/NiagaraSpriteRendererProperties]] → [[Wiki/Entities/Stock/UNiagaraSpriteRendererProperties]]
- [x] [[Wiki/Sources/Stock/NiagaraRendererSprites]]
- [x] [[Wiki/Sources/Stock/NiagaraRibbonRendererProperties]] → [[Wiki/Entities/Stock/UNiagaraRibbonRendererProperties]]
- [x] [[Wiki/Sources/Stock/NiagaraRendererRibbons]]
- [x] [[Wiki/Sources/Stock/NiagaraMeshRendererProperties]] → [[Wiki/Entities/Stock/UNiagaraMeshRendererProperties]]
- [x] [[Wiki/Sources/Stock/NiagaraRendererMeshes]]
- [x] [[Wiki/Sources/Stock/NiagaraLightRendererProperties]] → [[Wiki/Entities/Stock/UNiagaraLightRendererProperties]]
- [x] [[Wiki/Sources/Stock/NiagaraRendererLights]]

### Phase 7（数据接口）
- [x] [[Wiki/Sources/Stock/NiagaraDataInterface]] → [[Wiki/Entities/Stock/UNiagaraDataInterface]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceCurve]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceCurve]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceCurveBase]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceCamera]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceCamera]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceCollisionQuery]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceCollisionQuery]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceStaticMesh]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceStaticMesh]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceSkeletalMesh]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceSkeletalMesh]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceTexture]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceTexture]]
- [x] [[Wiki/Sources/Stock/NiagaraDataInterfaceRenderTarget2D]] → [[Wiki/Entities/Stock/UNiagaraDataInterfaceRenderTarget2D]]

### Phase 8（GPU）
- [x] [[Wiki/Sources/Stock/NiagaraShared]]
- [x] [[Wiki/Sources/Stock/NiagaraShader]] → [[Wiki/Entities/Stock/FNiagaraShader]]
- [x] [[Wiki/Sources/Stock/NiagaraShaderType]]
- [x] [[Wiki/Sources/Stock/NiagaraShaderMap]] (stub)
- [x] [[Wiki/Sources/Stock/NiagaraScriptBase]]
- [x] [[Wiki/Sources/Stock/NiagaraGPUInstanceCountManager]] → [[Wiki/Entities/Stock/FNiagaraGPUInstanceCountManager]]
- [x] [[Wiki/Sources/Stock/NiagaraGPUSortInfo]] → [[Wiki/Entities/Stock/FNiagaraGPUSort]]
- [x] [[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]] (Phase 5.5 已覆盖 GPU 侧)
- [x] [[Wiki/Sources/Stock/NiagaraVertexFactory]] → [[Wiki/Entities/Stock/FNiagaraVertexFactory]]
- [x] [[Wiki/Sources/Stock/NiagaraSpriteVertexFactory]]
- [x] [[Wiki/Sources/Stock/NiagaraRibbonVertexFactory]]
- [x] [[Wiki/Sources/Stock/NiagaraMeshVertexFactory]]
- [x] [[Wiki/Sources/Stock/NiagaraSortingGPU]]
- [x] [[Wiki/Sources/Stock/NiagaraDrawIndirect]] → [[Wiki/Entities/Stock/FNiagaraDrawIndirect]]

### Phase 9（世界管理）
- [x] [[Wiki/Sources/Stock/NiagaraWorldManager]] → [[Wiki/Entities/Stock/FNiagaraWorldManager]]
- [x] [[Wiki/Sources/Stock/NiagaraScalabilityManager]] → [[Wiki/Entities/Stock/FNiagaraScalabilityManager]]
- [x] [[Wiki/Sources/Stock/NiagaraComponentPool]] → [[Wiki/Entities/Stock/UNiagaraComponentPool]]
- [x] [[Wiki/Sources/Stock/NiagaraSettings]] → [[Wiki/Entities/Stock/UNiagaraSettings]]
- [x] [[Wiki/Sources/Stock/NiagaraEffectType]] → [[Wiki/Entities/Stock/UNiagaraEffectType]]
- [x] [[Wiki/Sources/Stock/NiagaraPlatformSet]] → [[Wiki/Entities/Stock/FNiagaraPlatformSet]]

### Phase 10（高级特性，选修）
- [ ] [[Wiki/Sources/Stock/NiagaraSimulationStageBase]]
- [ ] [[Wiki/Sources/Stock/NiagaraDataInterfaceRW]]
- [ ] [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollection]]
- [ ] [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollectionReader]]
- [ ] [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid3DCollection]]
- [ ] [[Wiki/Sources/Stock/NiagaraDataInterfaceNeighborGrid3D]]

---

*路径由 [[Claudian]] 基于 UE 4.26 Niagara 插件结构分析生成，2026-04-18。*
