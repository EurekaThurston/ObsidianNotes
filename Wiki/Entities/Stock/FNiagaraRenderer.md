---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, runtime, render-thread]
sources: 1
aliases: [FNiagaraRenderer, Niagara Renderer 运行时基类]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRenderer.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraRenderer

> Niagara 渲染器的 **运行时侧基类**(非 UObject)。与 `UNiagaraRendererProperties`(Asset 侧,Phase 6.1)配对。每 Emitter Instance 一个 Renderer 对象。

## 一句话角色

`class FNiagaraRenderer`(非 UObject)。Phase 2 `FNiagaraSceneProxy::EmitterRenderers` 数组里的元素。负责把 Emitter 的粒子数据(DataSet / GPU ExecContext)转化为 mesh batch / simple lights 提交给 UE 场景渲染器。

## 核心渲染流程

```cpp
GT: GenerateDynamicData(SceneProxy, Properties, Emitter)
    → 从 DataSet / RendererBindings 快照出本帧数据
    → 返回 FNiagaraDynamicData* 子类

GT→RT: SetDynamicData_RenderThread(DynamicData)

RT: GetDynamicMeshElements(Views, ViewFamily, VisMap, Collector, SceneProxy)
    → 从 DynamicData 构建 FMeshBatch
    → 提交给 FMeshElementCollector

RT: GatherSimpleLights(OutLights)  // 仅 Light renderer 覆盖
```

## 核心字段

```cpp
FNiagaraDynamicDataBase* DynamicDataRender;     // 当前帧数据
uint32 bLocalSpace : 1;                          // emitter 是 local space
uint32 bHasLights : 1;                           // 有 light renderer
uint32 bMotionBlurEnabled : 1;
const ENiagaraSimTarget SimTarget;
uint32 NumIndicesPerInstance;
ERHIFeatureLevel::Type FeatureLevel;
TArray<UMaterialInterface*> BaseMaterials_GT;
FMaterialRelevance BaseMaterialRelevance_GT;
```

## 配套数据类

- **`FNiagaraDynamicDataBase`** — GT→RT 数据传递基类。含 union { `FNiagaraDataBuffer* CPUParticleData; FNiagaraComputeExecutionContext* GPUExecContext; }`,按 SimTarget 选。
- **`FParticleRenderData`** — CPU sim 情况的 dynamic read buffer 分配(Float + Half)

## 静态辅助

- `SortIndices` — GPU 排序 key 生成
- `TransferDataToGPU` — CPU DataSet → GPU read buffer
- `GetDummy{Float,Float2,Float4,WhiteColor,Int,UInt,UInt4,TextureRead2D,Half}Buffer` — 空 SRV 池(shader slot 要绑但本次不用时)

## 子类家族

| 子类 | 职责 |
|---|---|
| `FNiagaraRendererSprites` | Billboard quad |
| `FNiagaraRendererRibbons` | 粒子连成带状 |
| `FNiagaraRendererMeshes` | 每粒子实例化 StaticMesh |
| `FNiagaraRendererLights` | 每粒子 simple light(不产 mesh batch) |

## 相关

- [[Wiki/Entities/Stock/UNiagaraRendererProperties]] — Asset 对偶
- [[Wiki/Entities/Stock/UNiagaraComponent]] — `FNiagaraSceneProxy` 持有本类数组

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraRenderer]]
- 主题读本:[[Readers/Niagara/Phase6-rendering-读本]] § 3
