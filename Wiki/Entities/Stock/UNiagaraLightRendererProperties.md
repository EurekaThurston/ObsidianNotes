---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, light]
sources: 1
aliases: [UNiagaraLightRendererProperties, FNiagaraRendererLights, Light Renderer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraLightRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraLightRendererProperties / FNiagaraRendererLights

> **每粒子一个 SimpleLight**。不产生 mesh batch,走 `GatherSimpleLights` 提交场景光源系统。**CPU only**。

## 一句话角色

- Asset:`UNiagaraLightRendererProperties`(98 行,Phase 6 Properties 最小)
- 运行时:`FNiagaraRendererLights`(31 行,Phase 6 最小)

## 陷阱

- ⚠️ **CPU only** — GPU 粒子无法直接提交光源
- ⚠️ **`bAffectsTranslucency=true`** 配大量粒子 = 严重开销(代码注释明确警告)
- ⚠️ 不产生 mesh batch — 和其他 renderer 的代码路径不同

## 核心字段

```cpp
bUseInverseSquaredFalloff : 1;  // 物理正确 vs 自定义 Exponent
bAffectsTranslucency : 1;       // 半透明受光(开销大)
float RadiusScale;
FVector ColorAdd;
```

## 6 个 AttributeBinding + 6 个 DataSetAccessor 缓存

```cpp
Position / Color / Radius / Exponent / Scattering / Enabled (FNiagaraBool)
```

`LightRenderingEnabledBinding` 允许 per-particle 开关光源。

## 关键数据类

```cpp
struct SimpleLightData {
    FSimpleLightEntry LightEntry;          // 颜色、强度、衰减
    FSimpleLightPerViewEntry PerViewEntry; // 位置、半径
};
```

## 渲染流程(对比其他 Renderer)

```
GT: GenerateDynamicData → 遍历 CPU DataSet → 填 TArray<SimpleLightData>
RT: GatherSimpleLights(OutParticleLights) → 提交 FSimpleLightArray
```

**不调用** `GetDynamicMeshElements`。Component 级 `bHasLights=true` 触发 SceneProxy 的 `GatherSimpleLights` 路径。

## 相关

- [[Wiki/Entities/Stock/UNiagaraRendererProperties]] / [[Wiki/Entities/Stock/FNiagaraRenderer]]
- Phase 2 `FNiagaraSceneProxy::GatherSimpleLights`

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraLightRendererProperties]] / [[Wiki/Sources/Stock/NiagaraRendererLights]]
- 读本:[[Readers/Niagara/Phase6-rendering-读本]] § 7
