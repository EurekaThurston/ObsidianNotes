---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, simulation, runtime, batched-tick]
sources: 1
aliases: [FNiagaraSystemSimulation, NiagaraSystemSimulation, 批量 Tick 调度器]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemSimulation.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraSystemSimulation

> **同一 `UNiagaraSystem` 资产 × 同一 `UWorld` × 同一 `ETickingGroup` 下所有实例的批量 Tick 调度器**。Niagara 性能优势的核心。

## 一句话角色

`class FNiagaraSystemSimulation : public TSharedFromThis<...>, FGCObject`(非 UObject,但注册为 GCObject 以保有 UObject 引用)。身份 = `(Asset, World, TickGroup)`。由 `FNiagaraWorldManager`(Phase 9)按此三元组索引持有。

Solo 实例是退化情形——每个 Solo Instance 各自一个 1 成员的 Simulation,批量优势失效(Phase 2 `bForceSolo` 陷阱的根源)。

## 核心字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `SystemInstances` | `TArray<FNiagaraSystemInstance*>` | 正在模拟的所有 instance |
| `PendingSystemInstances` | `TArray<...>` | 等待 spawn |
| `SpawningInstances` | `TArray<...>` | 非常规 spawn(level streaming) |
| `PausedSystemInstances` | `TArray<...>` | 暂停中 |
| `MainDataSet` / `SpawningDataSet` / `PausedInstanceData` | `FNiagaraDataSet` × 3 | 不同用途的粒子数据 |
| `SpawnInstanceParameterDataSet` / `UpdateInstanceParameterDataSet` | `FNiagaraDataSet` × 2 | System 参数聚合 DataSet |
| `SpawnExecContext` / `UpdateExecContext` | `TUniquePtr<FNiagaraScriptExecutionContextBase>` | System 级脚本上下文 |
| `SystemTickGroup` | `ETickingGroup` | 身份三元组之一 |
| `WeakSystem` | `TWeakObjectPtr<UNiagaraSystem>` | Asset(weak 防 GC 阻塞) |
| `TickBatch` | `FNiagaraSystemTickBatch` | **4-instance 滚动批** |
| `bIsSolo` | `uint32 : 1` | 是否为 Solo 实例退化 |

## Tick Batch = 4

```cpp
#define NiagaraSystemTickBatchSize 4
typedef TArray<FNiagaraSystemInstance*, TInlineAllocator<4>> FNiagaraSystemTickBatch;
```

每 4 个 instance 打一批,提交一个 TaskGraph 任务。这是 "50 个爆炸只走 12 个并行任务" 的根源。

## 两阶段 Tick + 内部 pipeline

```cpp
Tick_GameThread(DeltaSeconds, CompletionEvent)
    → SetupParameters_GameThread
    → PrepareForSystemSimulate
    → SpawnSystemInstances   // 跑 System Spawn 脚本
    → UpdateSystemInstances  // 跑 System Update 脚本
    → TransferSystemSimResults  // 派发到 Emitter
Tick_Concurrent(Context)  // 可在任意线程
```

## 多种 Binding(参数 → DataSet)

- `FNiagaraParameterStoreToDataSetBinding` 本类核心辅助结构(预计算 offset)
- 6 组 binding 把 System attribute 派发到 Emitter:`SpawnParameters / UpdateParameters / EventParameters / GPUParameters / RendererParameters`

## GPU Tick 5 模式

```cpp
enum class ENiagaraGPUTickHandlingMode {
    None, GameThread, Concurrent, GameThreadBatched, ConcurrentBatched
};
```

## 陷阱

- ⚠️ Solo 模式下退化为 1-instance simulation,批量 tick 失效(Phase 2 性能陷阱根源)
- ⚠️ `WeakSystem` 被 GC 后 `EffectType` 缓存仍然提供 scalability 计数——避免"GC 期间 count 归零"的尴尬
- ⚠️ Legacy ExecContexts(`bUseLegacyExecContexts` CVar)强制全部 solo——老版本向后兼容,性能差

## 相关

- [[Wiki/Entities/Stock/FNiagaraSystemInstance]] — 被调度的对象
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Asset,身份一部分
- [[Wiki/Entities/Stock/UNiagaraComponent]] — `bForceSolo` 开关的上游

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/NiagaraSystemSimulation]]
- 主题读本:[[Readers/Niagara/Phase 3 - Niagara 的心脏]] § 5-6

## 开放问题

- Batch size = 4 的依据?→ cpp profiling 注释
- `bInSpawnPhase` 区分 spawn 与 update?同 tick 内两次调用?→ `.cpp`
- `DataSetToEmitterEventParameters` 的嵌套 TArray 层级?→ `.cpp`
