---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, sprite, runtime]
sources: 1
aliases: [NiagaraRendererSprites.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererSprites.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRendererSprites.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererSprites.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 102 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.4

## 职责

定义 `FNiagaraRendererSprites`——`UNiagaraSpriteRendererProperties` 的**运行时对偶**,继承 `FNiagaraRenderer`。每 Emitter Instance 一个。

## 类声明

```cpp
class NIAGARA_API FNiagaraRendererSprites : public FNiagaraRenderer
{
    // FNiagaraRenderer 虚方法覆盖
    virtual void CreateRenderThreadResources(NiagaraEmitterInstanceBatcher*) override;
    virtual void ReleaseRenderThreadResources() override;
    virtual int32 GetMaxIndirectArgs() const override;

    virtual void GetDynamicMeshElements(...) override;       // ★ 核心渲染
    virtual FNiagaraDynamicDataBase* GenerateDynamicData(...) override;  // ★ GT 快照
    virtual int GetDynamicDataSize() const override;
    virtual bool IsMaterialValid(const UMaterialInterface*) const override;

    #if RHI_RAYTRACING
    virtual void GetDynamicRayTracingInstances(...) override;
    #endif
};
```

## 内部方法(私有)

```cpp
struct FCPUSimParticleDataAllocation
{
    FGlobalDynamicReadBuffer& DynamicReadBuffer;
    FParticleRenderData ParticleData;           // Float + Half
    FGlobalDynamicReadBuffer::FAllocation IntData;
};

FCPUSimParticleDataAllocation ConditionalAllocateCPUSimParticleData(...);
TUniformBufferRef<FNiagaraSpriteUniformParameters> CreatePerViewUniformBuffer(...);
void SetVertexFactoryParticleData(FNiagaraSpriteVertexFactory&, ...);
void CreateMeshBatchForView(FSceneView*, ...);
```

每 view 的流程:
1. `ConditionalAllocateCPUSimParticleData` — CPU sim 情况,把 DataBuffer 数据上传成 global dynamic read buffer
2. `CreatePerViewUniformBuffer` — 组装 shader 的 per-view cbuffer(含 FacingMode、Camera 参数等)
3. `SetVertexFactoryParticleData` — 绑 VF 的 SRV 槽(Position/Color/...)
4. `CreateMeshBatchForView` — 构 `FMeshBatch` 提交给 `FMeshElementCollector`

## 缓存字段(从 Properties 拷贝)

```cpp
ENiagaraRendererSourceDataMode SourceMode;
ENiagaraSpriteAlignment Alignment;
ENiagaraSpriteFacingMode FacingMode;
FVector2D PivotInUVSpace;
ENiagaraSortMode SortMode;
FVector2D SubImageSize;

uint32 bSubImageBlend : 1;
uint32 bRemoveHMDRollInVR : 1;
uint32 bSortOnlyWhenTranslucent : 1;
uint32 bGpuLowLatencyTranslucency : 1;
uint32 bEnableCulling : 1;
uint32 bEnableDistanceCulling : 1;
uint32 bSetAnyBoundVars : 1;
uint32 bVisTagInParamStore : 1;

float MinFacingCameraBlendDistance / MaxFacingCameraBlendDistance;
FVector2D DistanceCullRange;
FNiagaraCutoutVertexBuffer CutoutVertexBuffer;
int32 NumCutoutVertexPerSubImage = 0;
uint32 MaterialParamValidMask = 0;

int32 RendererVisTagOffset;                     // VisibilityTag 在 DataSet 里的 offset
int32 RendererVisibility;

int32 VFBoundOffsetsInParamStore[ENiagaraSpriteVFLayout::Type::Num];
                                                // 每个 VF slot 对应的 ParameterStore offset

const FNiagaraRendererLayout* RendererLayoutWithCustomSort;
const FNiagaraRendererLayout* RendererLayoutWithoutCustomSort;
```

**设计要点**:**Properties 的数据在 Renderer 构造时被拷贝**,运行时不再反向查 Properties。这让 RT 侧能独立工作,避免跨线程竞争。

## 依赖

**上游**:`NiagaraRenderer.h` / `NiagaraSpriteRendererProperties.h` / 私有依赖 `NiagaraSpriteVertexFactory`(Phase 8)、`NiagaraCutoutVertexBuffer`

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraSpriteRendererProperties]](合并了 Renderer)
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 基类
