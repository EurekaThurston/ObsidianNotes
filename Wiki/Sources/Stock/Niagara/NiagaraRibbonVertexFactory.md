---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, vertex-factory, ribbon, gpu]
sources: 1
aliases: [NiagaraRibbonVertexFactory.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraRibbonVertexFactory.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRibbonVertexFactory.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraRibbonVertexFactory.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 239 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.11

## 职责

Ribbon 渲染的 VF——比 Sprite 多了 **tangent/distance buffer** 和 **multi-ribbon indexing**。

## `FNiagaraRibbonVertexDynamicParameter`(L19)

```cpp
struct FNiagaraRibbonVertexDynamicParameter { float DynamicValue[4]; };
```

Ribbon 的 dynamic material param 每条带独立值,4 通道。

## `FNiagaraRibbonUniformParameters`(L28)

稳定 uniform buffer,22 个字段:CameraRight/Up/ScreenAlignment + 大量 DataOffset(Position/Velocity/Width/Twist/Color/Facing/NormalizedAge/MaterialRandom/MaterialParam × 4/U0Override/V0RangeOverride/U1Override/V1RangeOverride)+ InterpCount / OneOverInterpCount / U0DistributionMode / U1DistributionMode / PackedVData + `bLocalSpace` + DeltaSeconds。

## `FNiagaraRibbonVFLooseParameters`(L60)

**Ribbon 专有 SRV**:

```cpp
SHADER_PARAMETER_SRV(Buffer<int>, SortedIndices)              // 排序后索引
SHADER_PARAMETER_SRV(Buffer<float4>, TangentsAndDistances)    // 每粒子的切线 + 沿带距离
SHADER_PARAMETER_SRV(Buffer<uint>, MultiRibbonIndices)        // 多带情况:粒子→ribbon id
SHADER_PARAMETER_SRV(Buffer<float>, PackedPerRibbonDataByIndex)  // 每 ribbon 的统计数据
SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataFloat)
SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataHalf)
```

**TangentsAndDistances** 是 Ribbon 独有——在 GT 生成 dynamic data 时算好每粒子的切线方向和沿带累积距离,传给 shader 做 tessellation。

**MultiRibbonIndices + PackedPerRibbonDataByIndex** 支持一个 emitter 多条独立 ribbon。

## 主类(L76)

```cpp
class FNiagaraRibbonVertexFactory : public FNiagaraVertexFactoryBase
{
    DECLARE_VERTEX_FACTORY_TYPE(FNiagaraRibbonVertexFactory);

    void SetRibbonUniformBuffer(FNiagaraRibbonUniformBufferRef);
    void SetVertexBuffer(...) / SetDynamicParameterBuffer(...);

    void SetSortedIndices(VB + SRV + offset);
    void SetTangentAndDistances(VB + SRV);
    void SetMultiRibbonIndicesSRV(VB + SRV);
    void SetPackedPerRibbonDataByIndexSRV(VB + SRV);

    void SetFacingMode(uint32);

    // Index buffer 状态(用于跨帧复用)
    FIndexBuffer* IndexBuffer;
    uint32 FirstIndex;
    int32 OutTriangleCount;

    const FNiagaraDataSet* DataSet;
};
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraVertexFactory]]
- Phase 6 `FNiagaraRendererRibbons`
