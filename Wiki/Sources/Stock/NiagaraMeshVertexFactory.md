---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, vertex-factory, mesh, gpu, instancing]
sources: 1
aliases: [NiagaraMeshVertexFactory.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraMeshVertexFactory.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraMeshVertexFactory.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraVertexFactories/Public/NiagaraMeshVertexFactory.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 198 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.12

## 职责

`FNiagaraMeshVertexFactory`——Mesh 实例化粒子的 VF。**唯一一个需要 `FStaticMeshDataType` Mesh 顶点数据**的 Niagara VF。

## `FNiagaraMeshUniformParameters`(L30)

```cpp
BEGIN_GLOBAL_SHADER_PARAMETER_STRUCT(FNiagaraMeshUniformParameters, ...)
    SHADER_PARAMETER(uint32, bLocalSpace)
    SHADER_PARAMETER(FVector, PivotOffset)
    SHADER_PARAMETER(int, bPivotOffsetIsWorldSpace)
    SHADER_PARAMETER(FVector4, SubImageSize)
    SHADER_PARAMETER(uint32, TexCoordWeightA / TexCoordWeightB)
    SHADER_PARAMETER(uint32, PrevTransformAvailable)
    SHADER_PARAMETER(float, DeltaSeconds)

    // Particle data offsets
    SHADER_PARAMETER(int, PositionDataOffset / VelocityDataOffset / ColorDataOffset)
    SHADER_PARAMETER(int, TransformDataOffset)   // ← Mesh 专有:完整 transform
    SHADER_PARAMETER(int, ScaleDataOffset / SizeDataOffset / SubImageDataOffset)
    SHADER_PARAMETER(int, MaterialParamDataOffset × 4)
    SHADER_PARAMETER(int, NormalizedAgeDataOffset / MaterialRandomDataOffset / CameraOffsetDataOffset)
    SHADER_PARAMETER(uint32, MaterialParamValidMask)
    SHADER_PARAMETER(FVector4, DefaultPos)
    SHADER_PARAMETER(int, SubImageBlendMode)

    // Facing
    SHADER_PARAMETER(uint32, FacingMode)
    SHADER_PARAMETER(uint32, bLockedAxisEnable)
    SHADER_PARAMETER(FVector, LockedAxis)
    SHADER_PARAMETER(uint32, LockedAxisSpace)

    SHADER_PARAMETER(uint32, NiagaraFloatDataStride)
    SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataFloat)
    SHADER_PARAMETER_SRV(Buffer<float>, NiagaraParticleDataHalf)
END_GLOBAL_SHADER_PARAMETER_STRUCT()
```

所有参数都塞一个 UBO 里(不像 Sprite/Ribbon 分 Stable + Loose)。

## 主类(L73)

```cpp
class FNiagaraMeshVertexFactory : public FNiagaraVertexFactoryBase
{
    DECLARE_VERTEX_FACTORY_TYPE(FNiagaraMeshVertexFactory);

    static void ModifyCompilationEnvironment(...) {
        OutEnvironment.SetDefine(TEXT("NIAGARA_MESH_FACTORY"), TEXT("1"));
        OutEnvironment.SetDefine(TEXT("NIAGARA_MESH_INSTANCED"), TEXT("1"));
        OutEnvironment.SetDefine(TEXT("NiagaraVFLooseParameters"), TEXT("NiagaraMeshVF"));
    }

    void SetSortedIndices(SRV, offset);
    void SetData(const FStaticMeshDataType&);                    // ★ Mesh 顶点数据
    void SetUniformBuffer(FNiagaraMeshUniformBufferRef);

    void Copy(const FNiagaraMeshVertexFactory& Other);           // 复用 VF 时

    static bool SupportsTessellationShaders() { return true; }   // ← 唯一支持 tessellation 的 Niagara VF

    int32 GetLODIndex() / SetLODIndex(int32);

protected:
    FStaticMeshDataType Data;                // ← Mesh 顶点数据(位置/法线/UV 等)
    int32 LODIndex;

    FRHIUniformBuffer* MeshParticleUniformBuffer;
    FNiagaraMeshInstanceVertices* InstanceVerticesCPU;

    FShaderResourceViewRHIRef ParticleDataFloatSRV / ParticleDataHalfSRV;
    uint32 FloatDataStride / HalfDataStride;

    FShaderResourceViewRHIRef SortedIndicesSRV;
    uint32 SortedIndicesOffset;
};

inline FNiagaraMeshVertexFactory* ConstructNiagaraMeshVertexFactory();   // 工厂
inline FNiagaraMeshVertexFactory* ConstructNiagaraMeshVertexFactory(ENiagaraVertexFactoryType, ERHIFeatureLevel::Type);
```

## Instancing 机制

UE GPU instancing:mesh 顶点(Position/Normal/UV)在 `Data`(StaticMesh render data),per-instance 数据(Position/Transform/Color)从 SRV 读。shader 里每个 `SV_InstanceID` 索引到粒子数组,变换 mesh 顶点。

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraVertexFactory]]
- Phase 6 `FNiagaraRendererMeshes`
