---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, mesh, runtime]
sources: 1
aliases: [NiagaraRendererMeshes.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererMeshes.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRendererMeshes.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererMeshes.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 74 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.8

## 职责

Mesh renderer 的运行时。核心:**为每个 LOD 的每个 section 准备 instanced draw 数据**。

## 类声明

```cpp
class NIAGARA_API FNiagaraRendererMeshes : public FNiagaraRenderer
{
    virtual void ReleaseRenderThreadResources() override;
    virtual void GetDynamicMeshElements(...) override;
    virtual FNiagaraDynamicDataBase* GenerateDynamicData(...) override;
    virtual int32 GetDynamicDataSize() const override;
    virtual bool IsMaterialValid(const UMaterialInterface*) const override;

    #if RHI_RAYTRACING
    virtual void GetDynamicRayTracingInstances(...) final override;
    #endif

    void SetupVertexFactory(FNiagaraMeshVertexFactory*, const FStaticMeshLODResources&) const;
};
```

## 关键字段

```cpp
FStaticMeshRenderData* MeshRenderData;              // StaticMesh 的 render data

// 每 LOD × 每 section 的 (Count, Offset) 对
TArray<TArray<TPair<int32, int32>>> IndexInfoPerSection;

ENiagaraSortMode SortMode;
ENiagaraMeshFacingMode FacingMode;

// flag
uint32 bOverrideMaterials : 1;
uint32 bSortOnlyWhenTranslucent : 1;
uint32 bLockedAxisEnable : 1;
uint32 bEnableCulling : 1;
uint32 bEnableFrustumCulling : 1;
uint32 bSubImageBlend : 1;

FVector2D SubImageSize;
FVector PivotOffset;
ENiagaraMeshPivotOffsetSpace PivotOffsetSpace;
FVector LockedAxis;
ENiagaraMeshLockedAxisSpace LockedAxisSpace;

// Culling
FSphere LocalCullingSphere;
FVector2D DistanceCullRange;
int32 RendererVisTagOffset;
int32 RendererVisibility;
uint32 MaterialParamValidMask;

int32 MeshMinimumLOD = 0;

const FNiagaraRendererLayout* RendererLayoutWithCustomSorting;
const FNiagaraRendererLayout* RendererLayoutWithoutCustomSorting;
```

## GetLODIndex

```cpp
int32 GetLODIndex() const;
```

根据相机距离 + `MeshMinimumLOD` 决定用哪个 LOD。所有 instance 共享同一 LOD(不支持 per-instance LOD 选择)。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraMeshRendererProperties]](合并)
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]]
- Phase 8 `FNiagaraMeshVertexFactory`
