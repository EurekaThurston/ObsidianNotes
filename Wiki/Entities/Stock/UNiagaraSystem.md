---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, class, asset, uclass]
sources: 1
aliases: [UNiagaraSystem, NiagaraSystem, Niagara System 资产]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraSystem.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraSystem

> Niagara 特效的**顶层资产类**。一个 `.uasset` 文件 = 一个 UNiagaraSystem = 若干 Emitter + System 脚本 + 编译产物。

## 一句话角色

`UNiagaraSystem : public UFXSystemAsset`。Niagara 三层架构里 Asset 层的顶点,通过 `TArray<FNiagaraEmitterHandle>` 间接持有一组 Emitter,再加两个 System 级脚本、对外暴露参数、编译产物。运行时由 `UNiagaraComponent` 引用,实例化为 `FNiagaraSystemInstance`(Phase 3)。

## 核心字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `EmitterHandles` | `TArray<FNiagaraEmitterHandle>` | 持有的 Emitter 列表 |
| `SystemSpawnScript / SystemUpdateScript` | `UNiagaraScript*` | System 级脚本(Emitter 级脚本也会合并进来) |
| `ExposedParameters` | `FNiagaraUserRedirectionParameterStore` | 对外暴露的 `User.*` 参数 |
| `EffectType` | `UNiagaraEffectType*` | scalability 归属(Phase 9) |
| `EmitterCompiledData / SystemCompiledData` | 编译产物 | 参数→DataSet 偏移映射,运行时高性能读取的基础 |
| `EmitterExecutionOrder` | `TArray<FNiagaraEmitterExecutionIndex>` | 依赖求解后的 tick 顺序 |
| `FixedBounds` + `bFixedBounds` | 固定包围盒,跳过每帧重算 |
| `WarmupTime / WarmupTickCount / WarmupTickDelta` | 预热参数 |

常用方法:`GetEmitterHandles()` / `GetSystemSpawnScript()` / `ForEachScript(TAction)` / `IsValid()` / `ComputeEmittersExecutionOrder()` / `CacheFromCompiledData()`。

## 相关

- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — `EmitterHandles` 的元素类型
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 被 handle 引用的 Emitter
- [[Wiki/Entities/Stock/UNiagaraScript]] — System 脚本类型
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 本类是典型 Asset
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — `UFXSystemAsset` 基类与 Cascade 同源

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/NiagaraSystem]]
- 主题读本(推荐初读):[[Readers/Niagara/Phase1-asset-layer-读本]] § 1

## 开放问题

- `bHasDIsWithPostSimulateTick` / `bHasAnyGPUEmitters` / `bNeedsSortedSignificanceCull` 的优化决策点?→ Phase 3/7/9
- `ActiveInstances` / `ActiveInstancesTemp` 的维护点?→ Phase 3
