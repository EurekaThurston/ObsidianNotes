---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, runtime, render-thread]
sources: 1
aliases: [NiagaraRenderer.h, FNiagaraRenderer 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRenderer.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRenderer.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRenderer.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 144 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.2(**运行时基类**)

## 职责

定义 Niagara 渲染器的**运行时侧基类** `FNiagaraRenderer`,以及 GT→RT 数据传递的基础结构 `FNiagaraDynamicDataBase / FParticleRenderData`。

与 `UNiagaraRendererProperties`(Asset 侧,Phase 6.1)配对——Properties 是"配置",Renderer 是"执行"。每个 Emitter Instance 持有一组 Renderer(对应 Emitter Asset 的 RendererProperties 数组,一对一创建)。

## 关键类型

### `FNiagaraDynamicDataBase`(L27)— GT→RT 数据传递基类

```cpp
struct FNiagaraDynamicDataBase
{
    explicit FNiagaraDynamicDataBase(const FNiagaraEmitterInstance* InEmitter);
    virtual ~FNiagaraDynamicDataBase();

    FNiagaraDataBuffer* GetParticleDataToRender(bool bIsLowLatencyTranslucent = false) const;
    ENiagaraSimTarget GetSimTarget() const;
    FMaterialRelevance GetMaterialRelevance() const;

protected:
    FMaterialRelevance MaterialRelevance;
    ENiagaraSimTarget SimTarget;

    union {
        FNiagaraDataBuffer* CPUParticleData;           // CPU sim 情况
        FNiagaraComputeExecutionContext* GPUExecContext;  // GPU sim 情况
    } Data;
};
```

**核心设计**:用 `union` 存 CPU DataBuffer 指针或 GPU ExecContext 指针,根据 `SimTarget` 选择。`GetParticleDataToRender` 读正确的路径。

子类(如 `FNiagaraDynamicDataSprites / DynamicDataRibbon / DynamicDataMesh`)扩展这个基类加 renderer-specific 数据(如 cut-out 顶点、ribbon 排序 key)。

### `FParticleRenderData`(L55)

```cpp
struct FParticleRenderData
{
    FGlobalDynamicReadBuffer::FAllocation FloatData;
    FGlobalDynamicReadBuffer::FAllocation HalfData;
};
```

CPU sim 情况下,`TransferDataToGPU` 把 DataSet 的 Float/Half 数据**一次性上传**成 `FGlobalDynamicReadBuffer` 分配——之后 VF 用 SRV 读。

### `FNiagaraRenderer`(L64)— 主类

```cpp
class FNiagaraRenderer
{
public:
    FNiagaraRenderer(ERHIFeatureLevel::Type, const UNiagaraRendererProperties*, const FNiagaraEmitterInstance*);
    virtual ~FNiagaraRenderer();

    virtual void Initialize(const UNiagaraRendererProperties*, const FNiagaraEmitterInstance*, const UNiagaraComponent*);
    virtual void CreateRenderThreadResources(NiagaraEmitterInstanceBatcher* Batcher);
    virtual void ReleaseRenderThreadResources();

    virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView*, const FNiagaraSceneProxy*) const;
    virtual void GetDynamicMeshElements(...);                                    // ★ 核心渲染钩子
    virtual FNiagaraDynamicDataBase* GenerateDynamicData(...);                    // ★ GT 生产动态数据
    virtual void GatherSimpleLights(FSimpleLightArray&) const;                    // Light renderer 覆盖
    virtual int32 GetDynamicDataSize() const;
    virtual bool IsMaterialValid(const UMaterialInterface*) const;

    #if RHI_RAYTRACING
    virtual void GetDynamicRayTracingInstances(...);
    #endif
};
```

### 关键虚方法语义

- **`GenerateDynamicData`**(GT 调):从 EmitterInstance 的 DataSet + RendererBindings **快照**出 renderer 本帧要用的数据。分配一个 `FNiagaraDynamicData*` 子类对象返回,GT 把它传给 RT。
- **`GetDynamicMeshElements`**(RT 调):拿到 DynamicData,为每个 View 构建 `FMeshBatch` 提交给 `FMeshElementCollector`。这是**真正的"画粒子到屏幕"**。
- **`SetDynamicData_RenderThread`**:GT 调 `ENQUEUE_RENDER_COMMAND` 跨线程设置 `DynamicDataRender` 字段。

### 静态方法

- `SortIndices(const FNiagaraGPUSortInfo&, const FNiagaraRendererVariableInfo& SortVariable, const FNiagaraDataBuffer&, FGlobalDynamicReadBuffer::FAllocation& OutIndices)`:给 CPU sim 情况做 sort key 生成
- `TransferDataToGPU(FGlobalDynamicReadBuffer&, const FNiagaraRendererLayout*, FNiagaraDataBuffer*)`:CPU→GPU 数据传输
- `GetDummyFloatBuffer / Float2Buffer / Float4Buffer / WhiteColorBuffer / IntBuffer / UIntBuffer / UInt4Buffer / TextureReadBuffer2D / HalfBuffer`:**空 SRV 池**——shader 需要某 SRV slot 但本次不用,绑这些

### 字段

```cpp
FNiagaraDynamicDataBase* DynamicDataRender;     // 当前帧要画的数据(RT 改)

#if RHI_RAYTRACING
FRWBuffer RayTracingDynamicVertexBuffer;
FRayTracingGeometry RayTracingGeometry;
#endif

uint32 bLocalSpace : 1;                         // Emitter 是 local space?
uint32 bHasLights : 1;                          // 有 light renderer(影响 SceneProxy 路径)
uint32 bMotionBlurEnabled : 1;
const ENiagaraSimTarget SimTarget;
uint32 NumIndicesPerInstance;
ERHIFeatureLevel::Type FeatureLevel;

TArray<UMaterialInterface*> BaseMaterials_GT;   // 缓存从 Properties 拿的材质
FMaterialRelevance BaseMaterialRelevance_GT;

TRefCountPtr<FNiagaraGPURendererCount> NumRegisteredGPURenderers;
```

## 与 `FNiagaraSceneProxy`(Phase 2 Component)的关系

Phase 2 的 `FNiagaraSceneProxy`(定义在 `NiagaraComponent.h` L650)持有 `TArray<FNiagaraRenderer*> EmitterRenderers`——每个 Emitter Instance 的 renderer 列表。SceneProxy 的 `GetDynamicMeshElements` 会遍历这个数组,为每个 renderer 调用其 `GetDynamicMeshElements`。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 本文件主类
- [[Wiki/Entities/Stock/Niagara/UNiagaraRendererProperties]] — 对偶 Asset 基类
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — SceneProxy 持有本类实例

## 开放问题

- `GenerateDynamicData` 与 Emitter `PostTick` 的时序(谁先)→ Phase 3/6 接缝
- `SetDynamicData_RenderThread` 如何避免中间帧丢失 → cpp
