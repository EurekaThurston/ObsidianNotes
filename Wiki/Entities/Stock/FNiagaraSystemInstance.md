---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, instance, runtime, state-machine]
sources: 1
aliases: [FNiagaraSystemInstance, NiagaraSystemInstance, Niagara System 运行时实例]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSystemInstance.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraSystemInstance

> **单个 Niagara 特效实例的运行时"心脏"**。Phase 2 中 `UNiagaraComponent::SystemInstance` 这个 `TUniquePtr` 指向的就是本类。非 UObject,纯 C++。

## 一句话角色

`class FNiagaraSystemInstance`(非 UObject)。承担状态机、三阶段 Tick、Emitter 实例容器、per-instance DI 数据中枢四大职责。归属于某个 `FNiagaraSystemSimulation`(除非 Solo);持有一组 `TSharedRef<FNiagaraEmitterInstance>`。

## 四大职责与关键字段

| 职责 | 字段/类型 |
|---|---|
| 状态机(双状态) | `RequestedExecutionState` / `ActualExecutionState`(`ENiagaraExecutionState`) |
| 三阶段 Tick | `Tick_GameThread` / `Tick_Concurrent` / `FinalizeTick_GameThread` |
| Emitter 容器 | `TArray<TSharedRef<FNiagaraEmitterInstance>> Emitters` |
| DI 实例数据 | `TArray<uint8> DataInterfaceInstanceData` + `DataInterfaceInstanceDataOffsets` |
| 参数双缓冲 | `GlobalParameters[2]` / `SystemParameters[2]` / `OwnerParameters[2]` + `CurrentFrameIndex : 1` |
| 反向指针 | `TSharedPtr<FNiagaraSystemSimulation> SystemSimulation` |

## Reset 模式(EResetMode)

```cpp
enum class EResetMode { ResetAll, ResetSystem, ReInit, None };
```

- `ResetAll`:重置 Instance + 所有 Emitter Simulations
- `ResetSystem`:只重置 Instance(保留粒子)
- `ReInit`:彻底重建 Instance + Emitter(Asset 编辑后)
- `None`

## 三阶段 Tick 为什么是三阶段

- **GT 阶段**:收集输入(Transform / DeltaSeconds / InstanceParameters_GameThread)
- **Concurrent 阶段**:任意线程跑 CPU 模拟,可能几毫秒
- **Finalize GT 阶段**:DI 后 tick、bounds 更新、GPU Tick 入队、Complete 判断

## 陷阱

- ⚠️ 命名:`FNiagaraSystemInstance` vs `FNiagaraEmitterHandle::Instance`(Phase 1 Asset 层)**完全不是一回事**
- ⚠️ `bSolo / bForceSolo` 字段 —— Phase 2 `UNiagaraComponent::bForceSolo` 的运行时反映,Solo 模式下每实例独占一个 Simulation,批量优化失效
- ⚠️ `CachedEmitter` 在 `FNiagaraEmitterInstance` 是裸指针,由 System 级保证 lifetime;但 Asset 热重载时可能悬空(编辑器/开发场景)

## 相关

- [[Wiki/Entities/Stock/UNiagaraComponent]] — 拥有者(TUniquePtr)
- [[Wiki/Entities/Stock/FNiagaraEmitterInstance]] — 持有一组
- [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] — 批量调度器
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Asset
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 本类是 "Instance" 的典型

## 深入阅读

- 全字段清单 + 代码片段:[[Wiki/Sources/Stock/NiagaraSystemInstance]]
- 主题读本:[[Readers/Niagara/Phase 3 - Niagara 的心脏]] § 2-3

## 开放问题

- `DataInterfaceInstanceData` 的 16-byte 对齐为什么?→ Phase 7
- 双缓冲参数的读写竞态具体在哪一步规避?→ `.cpp`
- `bPooled` 导致 complete 时不 unbind 参数 —— Pool 实现细节?→ Phase 9
