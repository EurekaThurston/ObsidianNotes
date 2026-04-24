---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, sprite]
sources: 1
aliases: [UNiagaraSpriteRendererProperties, FNiagaraRendererSprites, Sprite Renderer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSpriteRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraSpriteRendererProperties / FNiagaraRendererSprites

> Niagara 最常用的渲染器——**每粒子一个 billboard quad**。CPU + GPU 都支持。本页合并 Asset 侧(Properties)+ 运行时(Renderer)。

## 一句话角色

- **Asset 侧**:`UNiagaraSpriteRendererProperties : UNiagaraRendererProperties`(316 行),配置 facing / alignment / subUV / cutout / 15+ 属性绑定
- **运行时**:`FNiagaraRendererSprites : FNiagaraRenderer`(102 行),构造时从 Properties 拷字段,RT 侧独立工作

## 关键能力

- **17 个 VF slot**:Position / Color / Velocity / Rotation / Size / Facing / Alignment / SubImage / MaterialParam0-3 / CameraOffset / UVScale / MaterialRandom / CustomSorting / NormalizedAge
- **5 种 FacingMode**:FaceCamera / FaceCameraPlane / CustomFacingVector / FaceCameraPosition / FaceCameraDistanceBlend
- **3 种 Alignment**:Unaligned / VelocityAligned / CustomAlignment
- **Cutout 优化**(editor):4 / 8 顶点贴合不透明区域,减少半透明 overdraw
- **SubUV blending**:子图间平滑混合
- **双 RendererLayout**:WithCustomSort / WithoutCustomSort

## 关键字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `Material` | `UMaterialInterface*` | "Use with Niagara Sprites" 要求 |
| `Alignment` / `FacingMode` | enums | 朝向控制 |
| `SortMode` + `bSortOnlyWhenTranslucent` | enums/bool | 排序 |
| `bGpuLowLatencyTranslucency` | bit | GPU 半透明用当前帧数据 |
| `PivotInUVSpace` | Vec2 | 旋转轴位置 |
| `SubImageSize` + `bSubImageBlend` | — | SubUV |
| `CutoutTexture` + `BoundingMode` + `AlphaThreshold` | editor | 半透明 overdraw 优化 |
| `MaterialParameterBindings` | array | 动态材质参数(会生成 MID) |

## Runtime 缓存(Renderer 侧)

```cpp
FVFBoundOffsetsInParamStore[ENiagaraSpriteVFLayout::Num];  // 17 个 offset 缓存
FNiagaraCutoutVertexBuffer CutoutVertexBuffer;
```

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraRendererProperties]] / [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 基类
- Phase 8 `FNiagaraSpriteVertexFactory`

## 深入阅读

- Properties 源:[[Wiki/Sources/Stock/Niagara/NiagaraSpriteRendererProperties]]
- Renderer 源:[[Wiki/Sources/Stock/Niagara/NiagaraRendererSprites]]
- 主题读本:[[Readers/UE/Niagara/Phase 6 - Niagara 粒子如何变成屏幕像素]] § 4
