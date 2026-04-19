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

> 一个 Emitter 的**静态描述**:跑哪些脚本、如何渲染、CPU/GPU、模拟参数,以及 Niagara 独有的 **Emitter 继承/merge** 机制。

## 一句话角色

`UNiagaraEmitter : public UObject`,`UCLASS(MinimalAPI)`。Niagara 里"粒子行为的描述单元",被 `UNiagaraSystem` 通过 `FNiagaraEmitterHandle` 引用。头文件自述:"**存储 `FNiagaraEmitterInstance` 需要序列化的那部分属性,用于 Instance 初始化**"——纯粹的 Asset-describes-Instance 模型。

声明 `friend struct FNiagaraEmitterHandle`,允许 handle 访问内部(merge/rename 等场景)。

## 核心字段速查

**模拟模式与空间**

| 字段 | 类型 | 说明 |
|---|---|---|
| `SimTarget` | `ENiagaraSimTarget` | **CPUSim / GPUComputeSim** — 决定走 VectorVM 还是 Compute Shader |
| `bLocalSpace` | `bool` | 粒子坐标相对 Emitter 还是世界 |
| `bDeterminism` + `RandomSeed` | 确定性随机 |
| `bInterpolatedSpawning` | 插值 spawn(防帧率抖动) |
| `bRequiresPersistentIDs` | 粒子持久 ID |
| `bLimitDeltaTime` + `MaxDeltaTimePerTick` | dt 钳制 |

**脚本集合**

| 字段 | 类型 | 角色 |
|---|---|---|
| `SpawnScriptProps / UpdateScriptProps` | `FNiagaraEmitterScriptProperties` | ParticleSpawn / ParticleUpdate |
| `GPUComputeScript` | `UNiagaraScript*` | GPU 合并脚本 |
| `EventHandlerScriptProps` | `TArray<FNiagaraEventScriptProperties>` | N 个事件处理器 |
| `EmitterSpawnScriptProps / EmitterUpdateScriptProps`(editor-only)| Emitter 级,**不单独编译**,合并进 System 脚本 |

**其他**

- 渲染:`RendererProperties`(Phase 6)
- SimStage:`SimulationStages`(Phase 10)
- Scalability:`Platforms` + `ScalabilityOverrides`
- 继承(editor-only):`Parent` + `ParentAtLastMerge` + `MergeChangesFromParent()`
- 图源(editor-only):`GraphSource : UNiagaraScriptSourceBase*`
- 分配:`AllocationMode / PreAllocationCount / MemoryRuntimeEstimation`

常用方法:`GetScripts / GetScript(Usage, UsageId)` / `ForEachScript / ForEachEnabledRenderer` / `AddRenderer / RemoveRenderer` / `AddEventHandler` / `IsValid / IsReadyToRun / IsAllowedByScalability` / `GetMaxParticleCountEstimate`。

## 相关

- [[Wiki/Entities/Stock/UNiagaraSystem]] — 容器
- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — 引用方式(`friend` 关系)
- [[Wiki/Entities/Stock/UNiagaraScript]] — 脚本字段类型
- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] — `GraphSource` 类型
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — `SimTarget` 是分叉源头
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Merge 是 Niagara 架构升级

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/NiagaraEmitter]]
- 主题读本(推荐初读):[[Wiki/Syntheses/Niagara/Phase1-asset-layer-读本]] § 3

## 开放问题

- EmitterSpawn/Update 合并进 System 的具体编译机制?→ Phase 4/5
- `bSimulationStagesEnabled` vs `bDeprecatedShaderStagesEnabled` 为何并存?→ Phase 10
- `MergeChangesFromParent` 能传播什么?→ 编辑器模块加餐
- `SharedEventGeneratorIds` 的 "shared" 语义?→ Phase 4
- `AttributesToPreserve` × System 的 `bTrimAttributes` 协作机制?→ Phase 4/5
