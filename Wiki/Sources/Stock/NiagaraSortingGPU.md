---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, gpu, sort, compute-shader]
sources: 1
aliases: [NiagaraSortingGPU.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraSortingGPU.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSortingGPU.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraSortingGPU.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 81 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.13

## 职责

定义 **`FNiagaraSortKeyGenCS`**——生成粒子排序 key 的 Global Compute Shader。被 Phase 5 `NiagaraEmitterInstanceBatcher::GenerateSortKeys` 调。

## 常量

```cpp
#define NIAGARA_KEY_GEN_THREAD_COUNT 64
#define NIAGARA_COPY_BUFFER_THREAD_COUNT 64
#define NIAGARA_COPY_BUFFER_BUFFER_COUNT 3
#define NIAGARA_KEY_GEN_MAX_CULL_PLANES 10
```

## 全局 CVars

```cpp
extern int32 GNiagaraGPUCulling;                    // 是否做 GPU culling
extern int32 GNiagaraGPUSortingUseMaxPrecision;     // 强制最高精度 key
extern int32 GNiagaraGPUSortingCPUToGPUThreshold;   // 粒子数阈值,超过用 GPU sort
```

## `FNiagaraSortKeyGenCS`(L28)

```cpp
class FNiagaraSortKeyGenCS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FNiagaraSortKeyGenCS);
    SHADER_USE_PARAMETER_STRUCT(FNiagaraSortKeyGenCS, FGlobalShader);

    // Permutation
    class FEnableCulling : SHADER_PERMUTATION_BOOL("ENABLE_CULLING");
    class FSortUsingMaxPrecision : SHADER_PERMUTATION_BOOL("SORT_MAX_PRECISION");
    using FPermutationDomain = TShaderPermutationDomain<FEnableCulling, FSortUsingMaxPrecision>;

    BEGIN_SHADER_PARAMETER_STRUCT(FParameters, ...)
        // 输入 SRV:粒子数据
        SHADER_PARAMETER_SRV(Buffer, NiagaraParticleDataFloat / Half / Int)
        SHADER_PARAMETER_SRV(Buffer, GPUParticleCountBuffer)
        SHADER_PARAMETER(uint32, FloatDataStride / HalfDataStride / IntDataStride)
        SHADER_PARAMETER(uint32, ParticleCount)
        SHADER_PARAMETER(uint32, GPUParticleCountOffset / CulledGPUParticleCountOffset)
        SHADER_PARAMETER(uint32, EmitterKey)        // 多 emitter 共享 sort batch,用这个标记
        SHADER_PARAMETER(uint32, OutputOffset)

        // 排序参数
        SHADER_PARAMETER(FVector, CameraPosition / CameraDirection)
        SHADER_PARAMETER(uint32, SortMode)
        SHADER_PARAMETER(uint32, SortAttributeOffset)
        SHADER_PARAMETER(uint32, SortKeyMask / SortKeyShift / SortKeySignBit)

        // Culling(permutation FEnableCulling)
        SHADER_PARAMETER(uint32, CullPositionAttributeOffset / CullOrientationAttributeOffset / CullScaleAttributeOffset)
        SHADER_PARAMETER(int32, NumCullPlanes)
        SHADER_PARAMETER_ARRAY(FVector4, CullPlanes, [NIAGARA_KEY_GEN_MAX_CULL_PLANES])
        SHADER_PARAMETER(int32, RendererVisibility)
        SHADER_PARAMETER(uint32, RendererVisTagAttributeOffset)
        SHADER_PARAMETER(FVector2D, CullDistanceRangeSquared)
        SHADER_PARAMETER(FVector4, LocalBoundingSphere)
        SHADER_PARAMETER(FVector, CullingWorldSpaceOffset)

        // 输出 UAV
        SHADER_PARAMETER_UAV(Buffer, OutKeys)
        SHADER_PARAMETER_UAV(Buffer, OutParticleIndices)
        SHADER_PARAMETER_UAV(Buffer, OutCulledParticleCounts)
    END_SHADER_PARAMETER_STRUCT()
};
```

## 4 permutation

```
ENABLE_CULLING × SORT_MAX_PRECISION = 4 种 shader 变体
```

CVar `GNiagaraGPUCulling / GNiagaraGPUSortingUseMaxPrecision` 决定用哪个。

## 工作流

```
Phase 5 Batcher::AddSortedGPUSimulation(SortInfo) →
FGPUSortManager 注册 sort task →
PreRender 阶段:GPUSortManager 回调 Batcher::GenerateSortKeys →
Batcher 为每个 SortInfo 发起一次 FNiagaraSortKeyGenCS dispatch →
CS 读粒子数据 + view 参数 → 写 (key, particleIndex) 对 →
Culling permutation 还会写 OutCulledParticleCounts →
GPUSortManager 做后续 radix sort →
结果 SRV 给 Renderer 使用(Phase 6 VF 的 SortedIndices SRV)
```

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraGPUSort]](合并页)
- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]]
