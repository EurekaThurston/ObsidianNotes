---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, vertex-factory, sprite, gpu]
sources: 1
aliases: [NiagaraSpriteVertexFactory.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraSpriteVertexFactory.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSpriteVertexFactory.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraSpriteVertexFactory.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 223 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.10

## 职责

`FNiagaraSpriteVertexFactory : FNiagaraVertexFactoryBase`——Sprite 粒子渲染用的 VF。定义两个 uniform buffer struct(稳定参数 + loose 参数)+ 大量 per-draw state 字段。

## 两个 Uniform Buffer 结构

### `FNiagaraSpriteUniformParameters`(L21)

"稳定"参数:

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FNiagaraSpriteUniformParameters, NIAGARAVERTEXFACTORIES_API)
    SHADER_PARAMETER(uint32, bLocalSpace)
    SHADER_PARAMETER_EX(FVector4, TangentSelector, EShaderPrecisionModifier::Half)
    // ... NormalsSphereCenter / NormalsCylinderUnitDirection / SubImageSize /
    //     CameraFacingBlend / RemoveHMDRoll / MacroUVParameters / RotationScale /
    //     RotationBias / NormalsType / DeltaSeconds / PivotOffset

    // 粒子属性 offset(Phase 4 DataSet 里的 component offset,编码 half 在 MSB)
    SHADER_PARAMETER(int, PositionDataOffset)
    SHADER_PARAMETER(int, VelocityDataOffset)
    SHADER_PARAMETER(int, RotationDataOffset)
    // ... SizeDataOffset / SubimageDataOffset / ColorDataOffset /
    //     MaterialParamDataOffset × 4 / FacingDataOffset / AlignmentDataOffset /
    //     CameraOffsetDataOffset / UVScaleDataOffset / NormalizedAgeDataOffset /
    //     MaterialRandomDataOffset
    SHADER_PARAMETER(uint32, MaterialParamValidMask)
    SHADER_PARAMETER(int, SubImageBlendMode)

    // 默认值(粒子没绑定该属性时用)
    SHADER_PARAMETER(FVector4, DefaultPos)
    // ... DefaultSize / DefaultUVScale / DefaultVelocity / DefaultRotation /
    //     DefaultColor / DefaultMatRandom / DefaultCamOffset / DefaultNormAge /
    //     DefaultSubImage / DefaultFacing / DefaultAlignment /
    //     DefaultDynamicMaterialParameter × 4
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

### `FNiagaraSpriteVFLooseParameters`(L73)

"loose"参数(per-draw 变):

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FNiagaraSpriteVFLooseParameters, ...)
    SHADER_PARAMETER(uint32, NumCutoutVerticesPerFrame)
    SHADER_PARAMETER(uint32, NiagaraFloatDataStride)
    SHADER_PARAMETER(uint32, ParticleAlignmentMode)
    SHADER_PARAMETER(uint32, ParticleFacingMode)
    SHADER_PARAMETER(uint32, SortedIndicesOffset)
    SHADER_PARAMETER(uint32, IndirectArgsOffset)

    SHADER_PARAMETER_SRV(Buffer<float2>, CutoutGeometry)           // ← Cutout 顶点(Phase 6)
    SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataFloat)  // ← CPU sim 粒子数据
    SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataHalf)
    SHADER_PARAMETER_SRV(Buffer<int>, SortedIndices)
    SHADER_PARAMETER_SRV(Buffer<uint>, IndirectArgsBuffer)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

## `FNiagaraSpriteVertexFactory` 主类(L92)

```cpp
class FNiagaraSpriteVertexFactory : public FNiagaraVertexFactoryBase
{
    DECLARE_VERTEX_FACTORY_TYPE(FNiagaraSpriteVertexFactory);

    virtual bool RendersPrimitivesAsCameraFacingSprites() const override { return true; }

    void SetSpriteUniformBuffer(const FNiagaraSpriteUniformBufferRef&);
    FRHIUniformBuffer* GetSpriteUniformBuffer();

    void SetCutoutParameters(int32 NumVerticesPerFrame, FRHIShaderResourceView* GeometrySRV);
    int32 GetNumCutoutVerticesPerFrame() const;
    FRHIShaderResourceView* GetCutoutGeometrySRV() const;

    void SetSortedIndices(const FShaderResourceViewRHIRef& SRV, uint32 Offset);
    FRHIShaderResourceView* GetSortedIndicesSRV();
    int32 GetSortedIndicesOffset();

    void SetFacingMode(uint32) / GetFacingMode();
    void SetAlignmentMode(uint32) / GetAlignmentMode();

    void SetVertexBufferOverride(const FVertexBuffer*);

    FUniformBufferRHIRef LooseParameterUniformBuffer;
};
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraVertexFactory]]
- Phase 6 `FNiagaraRendererSprites`
