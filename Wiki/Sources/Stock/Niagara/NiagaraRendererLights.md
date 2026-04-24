---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, light, runtime]
sources: 1
aliases: [NiagaraRendererLights.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererLights.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraRendererLights.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererLights.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: **31 行**(Phase 6 最小)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.10

## 职责

Light renderer 运行时。**不产生 mesh batch**——覆盖 `GatherSimpleLights` 把每粒子变成 `FSimpleLightEntry + FSimpleLightPerViewEntry` 提交给场景光源系统。

## 类声明

```cpp
class NIAGARA_API FNiagaraRendererLights : public FNiagaraRenderer
{
    struct SimpleLightData {
        FSimpleLightEntry LightEntry;              // 光源数据(颜色、强度、衰减)
        FSimpleLightPerViewEntry PerViewEntry;     // per-view 位置、半径
    };

    FNiagaraRendererLights(ERHIFeatureLevel::Type, const UNiagaraRendererProperties*, const FNiagaraEmitterInstance*);

    // 覆盖:GetViewRelevance 标记这个 renderer 没有 mesh 但有 light
    virtual FPrimitiveViewRelevance GetViewRelevance(const FSceneView*, const FNiagaraSceneProxy*) const override;

    // 覆盖:生成动态 light 数据
    virtual FNiagaraDynamicDataBase* GenerateDynamicData(const FNiagaraSceneProxy*, const UNiagaraRendererProperties*, const FNiagaraEmitterInstance*) const override;

    // 覆盖:把 SimpleLights 提交到场景
    virtual void GatherSimpleLights(FSimpleLightArray& OutParticleLights) const override;
};
```

**对比其他 Renderer**:
- Sprite/Ribbon/Mesh 都覆盖 `GetDynamicMeshElements`(走 mesh batch pipeline)
- Light **不覆盖** `GetDynamicMeshElements`(基类默认空实现),只覆盖 `GatherSimpleLights`
- Component 持有 `bHasLights` flag,SceneProxy 根据这个决定调 `GatherSimpleLights`

## 渲染流程

```
GT: GenerateDynamicData
    → 遍历 CPU DataSet 粒子
    → 读 Position / Color / Radius / Exponent / Enabled 绑定
    → 填 TArray<SimpleLightData>
    → 返回 FNiagaraDynamicDataLights(派生自 FNiagaraDynamicDataBase)

RT: SetDynamicData_RenderThread (基类方法)
    → 把 DynamicData 存到 this

RT 每 view: GatherSimpleLights (被 SceneProxy 调用)
    → 从 DynamicData 读 SimpleLightData 数组
    → 填入 OutParticleLights(FSimpleLightArray)
    → 场景渲染器最终把这些 simple lights 合进光照流程
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraLightRendererProperties]](合并)
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 基类
- Phase 2 `FNiagaraSceneProxy::GatherSimpleLights`
