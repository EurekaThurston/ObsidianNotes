---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, gpu, sort, sort-info]
sources: 1
aliases: [NiagaraGPUSortInfo.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUSortInfo.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraGPUSortInfo.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraGPUSortInfo.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 76 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.7

## 职责

定义 **GPU 粒子排序的任务描述** `FNiagaraGPUSortInfo`,给 UE 的 `FGPUSortManager` 注册一次排序任务所需全部信息。

## `ENiagaraSortMode`(L14)

```cpp
enum class ENiagaraSortMode : uint8 {
    None,                // 不排序
    ViewDepth,           // 按相机近平面的距离(常用于半透明)
    ViewDistance,        // 按相机原点距离
    CustomAscending,     // 按自定义属性(默认 Particles.NormalizedAge)升序
    CustomDecending,     // 降序
};
```

## `FNiagaraGPUSortInfo`(L29)

```cpp
struct FNiagaraGPUSortInfo {
    static constexpr uint32 MaxCullPlanes = 10;

    // 粒子数据
    int32 ParticleCount;
    ENiagaraSortMode SortMode;
    int32 SortAttributeOffset;

    FShaderResourceViewRHIRef ParticleDataFloatSRV;
    FShaderResourceViewRHIRef ParticleDataHalfSRV;
    FShaderResourceViewRHIRef ParticleDataIntSRV;
    uint32 FloatDataStride / HalfDataStride / IntDataStride;

    // GPU count
    FShaderResourceViewRHIRef GPUParticleCountSRV;
    uint32 GPUParticleCountOffset;
    uint32 CulledGPUParticleCountOffset;

    // View 数据
    FVector ViewOrigin;
    FVector ViewDirection;

    // Culling
    bool bEnableCulling;
    int32 CullPositionAttributeOffset / CullOrientationAttributeOffset / CullScaleAttributeOffset;
    int32 RendererVisTagAttributeOffset;
    int32 RendererVisibility;
    FSphere LocalBSphere;
    FVector CullingWorldSpaceOffset;
    FVector2D DistanceCullRange;
    TArray<FPlane, TFixedAllocator<10>> CullPlanes;   // 最多 10 个 cull 平面

    // GPUSortManager 绑定
    FGPUSortManager::FAllocationInfo AllocationInfo;
    EGPUSortFlags SortFlags;

    FORCEINLINE void SetSortFlags(bool bHighPrecisionKeys, bool bTranslucentMaterial);
};
```

## `SetSortFlags` 的三轴

```cpp
SortFlags = EGPUSortFlags::KeyGenAfterPreRender | EGPUSortFlags::ValuesAsInt32 |
    (bHighPrecisionKeys ? HighPrecisionKeys : LowPrecisionKeys) |
    (bTranslucentMaterial ? AnySortLocation : SortAfterPreRender);
```

- **KeyGenAfterPreRender**:key 生成在 PreRender 后(渲染器能读场景 view 数据)
- **ValuesAsInt32**:排序结果是粒子索引(int32)
- **HighPrecisionKeys vs LowPrecisionKeys**:32-bit vs 16-bit 排序 key
- **AnySortLocation vs SortAfterPreRender**:半透明可以在任何阶段取结果;不透明必须在 PreRender 后

## 配合 Phase 5 Batcher

```
FNiagaraRendererSprites / Ribbons / Meshes::CreateRenderThreadResources
    → 构 FNiagaraGPUSortInfo
    → NiagaraEmitterInstanceBatcher::AddSortedGPUSimulation(SortInfo)
    → 注册到 FGPUSortManager
    → GPUSortManager 在 PreRender 回调 Batcher::GenerateSortKeys
    → Batcher 用 FNiagaraSortKeyGenCS(Phase 8.13)跑 compute shader 生成 keys
    → GPUSortManager 做实际 radix sort
    → 结果 UAV 传回 Renderer
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraGPUSort]](合并 SortInfo + SortingGPU + SortKeyGenCS)
- [[Wiki/Entities/Stock/Niagara/NiagaraEmitterInstanceBatcher]]
