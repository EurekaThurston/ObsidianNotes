---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, dataset, data-model, soa]
sources: 1
aliases: [FNiagaraDataSet, NiagaraDataSet]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSet.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraDataSet

> Niagara 的**粒子数据存储核心**。管理一组 `FNiagaraDataBuffer`(每 buffer = 一帧数据),以 SoA 布局存所有活跃粒子的所有属性。

## 一句话角色

`class NIAGARA_API FNiagaraDataSet`。本身是管理器——真正数据在 `FNiagaraDataBuffer`(基于 `FNiagaraSharedObject` 的并发保护基类)。典型用途:每个 `FNiagaraEmitterInstance::ParticleDataSet` 就是一个 DataSet,`SystemSimulation` 还有若干 per-pipeline DataSet。

**CPU/GPU 双存储**:
- CPU:`FloatData / Int32Data / HalfData`(`TArray<uint8>`)
- GPU:`GPUBufferFloat / Int / Half`(`FRWBuffer`)

## 核心字段

| 字段 | 作用 |
|---|---|
| `CompiledData` | 布局(变量列表 + layout,`FNiagaraDataSetCompiledData`) |
| `CurrentData` | 当前读的 DataBuffer |
| `DestinationData` | 模拟目标 DataBuffer(BeginSimulate/EndSimulate 之间) |
| `Data` | `TArray<FNiagaraDataBuffer*, TInlineAllocator<2>>` buffer 池 |
| `FreeIDsTable / SpawnedIDsTable / IDAcquireTag` | Persistent ID 系统 CPU 侧 |
| `GPUFreeIDs / GPUNumAllocatedIDs` | ID GPU 侧 |
| `MaxInstanceCount` | 分配上限 |

## 主流程(CPU/GPU 分叉)

```cpp
FNiagaraDataBuffer& BeginSimulate(bool bResetDestinationData=true);  // 从池里取一个 free buffer
void Allocate(int32 NumInstances, bool bMaintainExisting=false);     // 给 Destination 分空间
// ... VM 写 Destination ...
void EndSimulate(bool SetCurrentData=true);                          // Current ← Destination
```

线程约定:
- CPU sim **不在** RT(除非 readonly 读)
- GPU sim **必须在** RT

## SoA 布局

每个 component 自己一条连续 `uint8` 数组,`FloatStride = NumInstancesAllocated × 4`。SIMD 可一次读 4/8 粒子的同属性。

## 陷阱

- ⚠️ `CurrentData` 可能为 null(未 BeginSimulate)
- ⚠️ `FNiagaraDataBuffer` 被 RT 持有引用时不能改,要走 `FNiagaraSharedObject::Destroy()` 进延迟删除队列
- ⚠️ `FNiagaraDataVariableIterator` 慢!只能调试用

## 相关

- [[Wiki/Entities/Stock/FNiagaraDataBuffer]] — 一帧数据的实际持有者
- [[Wiki/Entities/Stock/FNiagaraEmitterInstance]] — 每 Emitter 一个 DataSet
- [[Wiki/Entities/Stock/FNiagaraDataSetAccessor]] — 类型安全访问模板
- [[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo]] — 布局依据
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraDataSet]]
- 主题读本:[[Readers/Niagara/Phase4-data-model-读本]] § 4

## 开放问题

- Buffer 池 `TInlineAllocator<2>` 够不够?→ § 源摘要 / Phase 8
- `GPUInstanceCountBufferOffset` 到 Phase 8 管理器 → Phase 8
