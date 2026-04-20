---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, ribbon, runtime]
sources: 1
aliases: [NiagaraRendererRibbons.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererRibbons.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRendererRibbons.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererRibbons.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 103 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.6

## 职责

Ribbon 渲染的运行时侧。最难的部分:**每帧按 RibbonId + LinkOrder 排序粒子,生成 index buffer 连成多条 ribbon**。

## 类声明

```cpp
class NIAGARA_API FNiagaraRendererRibbons : public FNiagaraRenderer
{
    // FNiagaraRenderer 虚方法覆盖(与 Sprites 类似)
    virtual void CreateRenderThreadResources(NiagaraEmitterInstanceBatcher*) override;
    virtual void ReleaseRenderThreadResources() override;
    virtual void GetDynamicMeshElements(...) override;
    virtual FNiagaraDynamicDataBase* GenerateDynamicData(...) override;
    virtual int32 GetDynamicDataSize() const override;
    virtual bool IsMaterialValid(const UMaterialInterface*) const override;

    #if RHI_RAYTRACING
    virtual void GetDynamicRayTracingInstances(...) override;
    #endif
};
```

## 关键内部方法

```cpp
// 按排序后的 segment 数据生成索引
template <typename TValue>
static TValue* AppendToIndexBuffer(TValue* OutIndices, uint32& OutMaxUsedIndex,
                                    const TArrayView<int32>& SegmentData,
                                    int32 InterpCount, bool bInvertOrder);

// 主 index buffer 生成——按 ribbon 分组
template <typename TValue>
void GenerateIndexBuffer(FGlobalDynamicIndexBuffer::FAllocationEx&,
                          int32 InterpCount,
                          const FVector& ViewDirection,
                          const FVector& ViewOriginForDistanceCulling,
                          struct FNiagaraDynamicDataRibbon* DynamicData) const;
```

`InterpCount` 对应 tessellation factor——每段原始粒子间插入多少个细分点。

## 渲染流程

```cpp
void SetupMeshBatchAndCollectorResourceForView(FSceneView*, ..., FMeshBatch&, FNiagaraMeshCollectorResourcesRibbon&);
void CreatePerViewResources(FSceneView*, ..., FNiagaraRibbonUniformBufferRef&, FGlobalDynamicIndexBuffer::FAllocationEx&);
FCPUSimParticleDataAllocation AllocateParticleDataIfCPUSim(...);
```

## 缓存字段

```cpp
ENiagaraRibbonFacingMode FacingMode;
FNiagaraRibbonUVSettings UV0Settings / UV1Settings;
ENiagaraRibbonDrawDirection DrawDirection;
ENiagaraRibbonTessellationMode TessellationMode;
float CustomCurveTension;
int32 CustomTessellationFactor;
bool bCustomUseConstantFactor;
float CustomTessellationMinAngle;
bool bCustomUseScreenSpace;

uint32 MaterialParamValidMask;
const FNiagaraRendererLayout* RendererLayout;

// Mutable:动态根据 segment 统计自适应
mutable float TessellationAngle = 0;
mutable float TessellationCurvature = 0;
mutable float TessellationTwistAngle = 0;
mutable float TessellationTwistCurvature = 0;
mutable float TessellationTotalSegmentLength = 0;
```

**自适应 tessellation**:`mutable` 字段根据每帧统计粒子段的角度、twist、长度,动态调整下帧 tessellation 因子。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraRibbonRendererProperties]](合并)
- [[Wiki/Entities/Stock/FNiagaraRenderer]]
