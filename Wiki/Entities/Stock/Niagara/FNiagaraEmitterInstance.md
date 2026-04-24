---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, emitter-instance, runtime, particle-simulation]
sources: 1
aliases: [FNiagaraEmitterInstance, NiagaraEmitterInstance, Emitter 运行时实例]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterInstance.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraEmitterInstance

> **单个 Emitter 在一个 System Instance 下的运行时对象**。持有当前活跃粒子数据、Spawn/Update 脚本执行上下文、事件数据、bounds 缓存。非 UObject。

## 一句话角色

`class FNiagaraEmitterInstance`(非 UObject)。Asset 侧是 `UNiagaraEmitter`(Phase 1),Instance 侧就是本类。被 `FNiagaraSystemInstance::Emitters`(`TArray<TSharedRef<...>>`) 持有,通过 parent 指针反向访问 System 级数据。

每帧流程:`PreTick() → Tick(DeltaSeconds) → PostTick() → HandleCompletion()`。被 SystemInstance 的 `Tick_Concurrent` 触发。

## 核心字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `ParticleDataSet` | `FNiagaraDataSet*` | **核心状态**——粒子数据(Phase 4) |
| `SpawnExecContext` / `UpdateExecContext` | `FNiagaraScriptExecutionContext` | CPU 脚本执行(Phase 5) |
| `GPUExecContext` | `FNiagaraComputeExecutionContext*` | GPU 路径的替代(Phase 8) |
| `RendererBindings` | `FNiagaraParameterStore` | 给 Renderer 读的参数(Phase 6) |
| `SpawnInfos` | `TArray<FNiagaraSpawnInfo>` | 本帧 spawn 调度(rate/burst/event) |
| `EventInstanceData` | `TUniquePtr<FEventInstanceData>` | 事件系统(有事件才分配) |
| `ExecutionState` | `ENiagaraExecutionState` | **单一状态**(比 System 简单) |
| `CachedEmitter` | `UNiagaraEmitter*` **裸指针** | Asset 缓存;System 级保证 lifetime |
| `ParentSystemInstance` | `FNiagaraSystemInstance*` | 反向指针 |
| `InstanceSeed` | `int32` | 本实例专属随机种子 |

## CPU / GPU 分叉就在这里

```cpp
FORCEINLINE int32 GetNumParticles() const {
    if (GPUExecContext) { return GetNumParticlesGPUInternal(); }
    // CPU 走 ParticleDataSet
}
```

Emitter 资产里的 `SimTarget`(CPU/GPU)决定本实例走哪条路径;两条路径并不共存。

## 陷阱

- ⚠️ 与 `FNiagaraEmitterHandle::Instance`(Asset 副本,Phase 1)**完全不是一回事**
- ⚠️ `CachedEmitter` 裸指针 —— Asset 热重载时可能悬空
- ⚠️ GPU 路径下 `GetNumParticles` 有 latency(需要 fence 才精确)
- ⚠️ `CalculateFixedBounds()` editor-only,会引发 GPU readback stall —— 不要日常调用

## 相关

- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]] — 持有者
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — Asset(CachedEmitter)
- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterHandle]] — 关联的 Asset handle
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU/GPU 分叉在此体现

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/Niagara/NiagaraEmitterInstance]]
- 主题读本:[[Readers/UE/Niagara/Phase 3 - Niagara 的心脏]] § 4

## 开放问题

- `FEventInstanceData` 里的 `FNiagaraDataSet*` 裸指针 owner?→ Phase 4
- GPU `GPUExecContext` 的 owner?→ Phase 8
- `bCombineEventSpawn` 对 `ExecIndex()` 为什么不安全?→ Phase 5
