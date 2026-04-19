---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, class, asset, uclass, emitter]
sources: 1
aliases: [UNiagaraEmitter, NiagaraEmitter, Niagara Emitter 资产]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitter.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraEmitter

> 一个 Emitter 的**静态描述**:跑哪些脚本、如何渲染、CPU/GPU、是否 local space、是否确定性,以及 Niagara 独有的 **Emitter 继承/merge** 机制。

## 概览

`UNiagaraEmitter : public UObject`,`UCLASS(MinimalAPI)`。它是 Niagara 里"粒子行为的描述单元",被 `UNiagaraSystem` 通过 `FNiagaraEmitterHandle` 引用。一个 Emitter 持有:

- 若干 `UNiagaraScript*`(ParticleSpawn / ParticleUpdate / GPUCompute / N × EventHandler)
- 若干 `UNiagaraRendererProperties*`(渲染器配置,Phase 6)
- 若干 `UNiagaraSimulationStageBase*`(Simulation Stages,Phase 10)
- 一份 `UNiagaraScriptSourceBase* GraphSource`(editor-only,此 Emitter 内所有脚本的共享图源)
- 模拟参数、scalability 规则、平台过滤、继承关系等

> Emitter 与 Instance 的关系:"Emitter stores the attributes of an FNiagaraEmitterInstance that need to be serialized and are used for its initialization."(头文件自述)——明确的 Asset-describes-Instance 模型。

## 关键事实 / 属性

### 模拟模式与空间(`UPROPERTY`)

| 字段 | 类型 | 说明 |
|---|---|---|
| `SimTarget` | `ENiagaraSimTarget` | **CPUSim / GPUComputeSim**,决定走 VectorVM 还是 Compute Shader |
| `bLocalSpace` | `bool` | 粒子坐标相对 Emitter 还是世界 |
| `bDeterminism` + `RandomSeed` | 确定性随机(相同 dt 相同输出) |
| `bInterpolatedSpawning` | 插值 spawn,高 spawn rate / 快速移动下防止节奏感被帧率破坏 |
| `bRequiresPersistentIDs` | 给粒子分配跨帧唯一 ID |
| `bLimitDeltaTime` + `MaxDeltaTimePerTick` | 限制单 tick dt |

### 内存 / 分配

- `AllocationMode` (`EParticleAllocationMode`) + `PreAllocationCount` — 预分配策略
- `MemoryRuntimeEstimation` RuntimeEstimation — 运行时反馈的峰值用于下次估算
- `MaxInstanceCount`(后计算)— Emitter 允许的最大并发实例粒子数

### 脚本集合

| 字段 | 类型 | 角色 |
|---|---|---|
| `SpawnScriptProps` | `FNiagaraEmitterScriptProperties` | ParticleSpawnScript |
| `UpdateScriptProps` | `FNiagaraEmitterScriptProperties` | ParticleUpdateScript |
| `GPUComputeScript` | `UNiagaraScript*` | GPU 模式下统一的 compute shader 脚本 |
| `EventHandlerScriptProps` | `TArray<FNiagaraEventScriptProperties>` | N 个事件处理器 |
| `EmitterSpawnScriptProps / EmitterUpdateScriptProps` (editor-only) | Emitter 级,**不单独编译**,被合并进 System 脚本 |

### 渲染与阶段

- `RendererProperties`(`TArray<UNiagaraRendererProperties*>`)— Sprite/Mesh/Ribbon/Light 等(Phase 6)
- `SimulationStages`(`TArray<UNiagaraSimulationStageBase*>`)— GPU 多 pass 计算(Phase 10)

### Scalability

- `Platforms`(`FNiagaraPlatformSet`)— 允许的平台/quality 组合
- `ScalabilityOverrides`(`FNiagaraEmitterScalabilityOverrides`)— 覆盖 EffectType 规则
- `CurrentScalabilitySettings`(运行时 resolve 后的结果)

### Emitter 继承(editor-only)

这是 Niagara 相对 Cascade 的重要架构升级:

- `Parent`(`UNiagaraEmitter*`)— 此 Emitter 是从哪个模板 duplicate 出来的
- `ParentAtLastMerge` — 上次 merge 时父的快照(三方 merge 用)
- `MergeChangesFromParent()` — 把父 Emitter 的增量改动合并下来
- `CreateWithParentAndOwner / DuplicateWithoutMerging` — 构造入口

### 编辑器源

- `GraphSource`(`UNiagaraScriptSourceBase*`)— 此 Emitter 内部所有脚本共享的图;保存在 `.uasset` 里但运行时不需要

### 事件系统(依赖结构)

`FNiagaraEmitterScriptProperties` 包含 `EventReceivers` / `EventGenerators` 列表,这意味着**事件配置挂在脚本上**,不是挂在 Emitter 上。同一 Emitter 不同脚本可以有独立的事件收发定义。

### 常用方法

- `GetScripts(OutScripts, bCompilableOnly)` — 取所有脚本(过滤可编译性)
- `GetScript(Usage, UsageId)` — 按 usage 查
- `ForEachScript(TAction)` / `ForEachEnabledRenderer(TAction)` — 遍历(模板)
- `AddRenderer / RemoveRenderer / AddEventHandler / RemoveEventHandlerByUsageId / AddSimulationStage / ...`
- `IsValid / IsReadyToRun / IsAllowedByScalability`
- `GetMaxParticleCountEstimate()`

## 相关

- [[Wiki/Entities/Stock/UNiagaraSystem]] — 容器
- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — System 持有 Emitter 的方式;是此类的 `friend`
- [[Wiki/Entities/Stock/UNiagaraScript]] — 脚本字段类型
- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] — GraphSource 的类型
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — `SimTarget` 是此分叉的字段
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Emitter merge 机制是 Niagara 的关键架构改进

## 引用来源

- [[Wiki/Sources/Stock/NiagaraEmitter]](`NiagaraEmitter.h` @ `b6ab0dee9`)

## 开放问题 / 矛盾

- EmitterSpawn/Update 两个 editor-only 脚本的最终归宿:它们被"合并"进 SystemSpawn/UpdateScript 的具体机制,需要在 Phase 5(脚本执行上下文)或 Phase 4 读编译流水线时确认
- `bSimulationStagesEnabled` 与 `bDeprecatedShaderStagesEnabled` 并存的历史原因;后者按注释叫 "Experimental",实际上是前者的前身
- `SharedEventGeneratorIds` 的 "shared" 语义:Spawn/Update 都用同一个 generator,还是别的意思?
- `AttributesToPreserve` × `UNiagaraSystem::bTrimAttributes`:具体 trim 决策在哪个编译阶段发生?
