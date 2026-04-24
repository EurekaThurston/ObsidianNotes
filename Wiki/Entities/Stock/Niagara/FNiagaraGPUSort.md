---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, gpu, sort, sort-info, sort-key-gen]
sources: 1
aliases: [FNiagaraGPUSortInfo, FNiagaraSortKeyGenCS, Niagara GPU 排序]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUSortInfo.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraGPUSort 家族

> Niagara GPU 粒子排序机制。本页合并 `FNiagaraGPUSortInfo`(任务描述)+ `FNiagaraSortKeyGenCS`(key 生成 compute shader)+ `ENiagaraSortMode`。

## 一句话角色

- **`ENiagaraSortMode`**:5 种(None / ViewDepth / ViewDistance / CustomAscending / CustomDecending)
- **`FNiagaraGPUSortInfo`**:描述一次 GPU 排序任务所需全部信息(粒子 SRV + view 参数 + culling 数据 + GPUSortManager AllocationInfo)
- **`FNiagaraSortKeyGenCS`**:key 生成 compute shader(4 permutation:ENABLE_CULLING × SORT_MAX_PRECISION)

## SortInfo 关键字段

| 字段 | 作用 |
|---|---|
| `ParticleCount` | 粒子数 |
| `SortMode` / `SortAttributeOffset` | 排序模式 + 自定义属性 offset |
| `ParticleDataFloat/Half/Int SRV` + Stride | 粒子数据源 |
| `GPUParticleCountSRV / Offset` | 实际粒子数(非分配数) |
| `ViewOrigin / ViewDirection` | view 排序 |
| `bEnableCulling` | 是否 view cull |
| Culling data | Position/Orientation/Scale offset + 10 个 CullPlanes + Sphere + 距离范围 |
| `AllocationInfo` | `FGPUSortManager::FAllocationInfo`,batch 分配 |
| `SortFlags` | GPUSortManager 排序约束 |

## `SetSortFlags` 三轴

```cpp
SortFlags = KeyGenAfterPreRender | ValuesAsInt32 |
    (HighPrecisionKeys / LowPrecisionKeys) |
    (AnySortLocation / SortAfterPreRender);
```

`bTranslucentMaterial = true` 走 `AnySortLocation`(能在任何阶段读结果),不透明必须 `SortAfterPreRender`。

## 工作流

```
Phase 6 Renderer::CreateRenderThreadResources:
    → 构 FNiagaraGPUSortInfo
    → Phase 5 Batcher::AddSortedGPUSimulation(SortInfo)
    → 注册到 FGPUSortManager

PreRender 阶段:
    → GPUSortManager 回调 Batcher::GenerateSortKeys(BatchId, NumElements, Flags, KeysUAV, ValuesUAV)
    → Batcher 为每个 SortInfo dispatch 一次 FNiagaraSortKeyGenCS(4 种 permutation 按 CVar 选)
    → CS 并行读粒子数据 → 算 depth/distance/自定义 key → 写 (key, particleIndex) 到 UAV
    → 如启用 culling,同时写 OutCulledParticleCounts
    → GPUSortManager 做 radix sort

结果:
    → Renderer 从 GPUSortManager 拿到 sorted indices SRV
    → Phase 6 VF 的 SortedIndices SRV 绑这个
```

## `FNiagaraSortKeyGenCS` Parameters

6 个 SRV + 10 个 uniform + 11 个 culling 参数 + 3 个 UAV。`NIAGARA_KEY_GEN_THREAD_COUNT=64`。

## 相关

- [[Wiki/Entities/Stock/Niagara/NiagaraEmitterInstanceBatcher]] — `AddSortedGPUSimulation / GenerateSortKeys`
- [[Wiki/Entities/Stock/Niagara/FNiagaraShader]] — 主 compute shader(sort 是 global shader,不同体系)

## 深入阅读

- 源 × 2:[[Wiki/Sources/Stock/Niagara/NiagaraGPUSortInfo]] / [[Wiki/Sources/Stock/Niagara/NiagaraSortingGPU]]
- 读本:[[Readers/UE/Niagara/Phase 8 - Niagara 的 GPU 模拟管线]] § 5
