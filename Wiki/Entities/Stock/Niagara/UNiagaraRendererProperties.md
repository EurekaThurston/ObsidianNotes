---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, renderer, properties, uclass]
sources: 1
aliases: [UNiagaraRendererProperties, Niagara Renderer Properties 基类]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraRendererProperties

> Niagara 渲染器的 **Asset 侧抽象基类**。每个具体 Renderer(Sprite/Ribbon/Mesh/Light)的 Properties 类都继承本类。`UNiagaraEmitter::RendererProperties` 数组装的就是本类子类的实例。

## 一句话角色

`UCLASS(ABSTRACT) UNiagaraRendererProperties : public UNiagaraMergeable`。配置一种粒子渲染方式——**什么材质、什么 facing、绑哪些粒子属性、哪些平台启用、多大排序 hint**。对偶运行时类是 `FNiagaraRenderer`(子类一对一)。

## 关键字段

| 字段 | 作用 |
|---|---|
| `Platforms` | `FNiagaraPlatformSet` 哪些平台启用(Phase 9) |
| `SortOrderHint` | 同 Emitter 内 renderer 排序 |
| `bIsEnabled` | 运行时可切 |
| `bMotionBlurEnabled` | 配合材质 |
| `AttributeBindings` | `TArray<const FNiagaraVariableAttributeBinding*>` 绑定哪些粒子属性 |

## 纯虚方法

子类必须实现:
- `CreateEmitterRenderer` — 创建对偶 `FNiagaraRenderer` 子类实例
- `CreateBoundsCalculator` — 为渲染器生成 bounds 计算器(Light 返回 nullptr)
- `GetUsedMaterials` — 返回用的 materials
- `GetRendererWidgets / GetRendererTooltipWidgets`(editor)

## 子类家族

| 子类 | SimTarget 支持 |
|---|---|
| `UNiagaraSpriteRendererProperties` | CPU + GPU |
| `UNiagaraRibbonRendererProperties` | CPU only |
| `UNiagaraMeshRendererProperties` | CPU + GPU |
| `UNiagaraLightRendererProperties` | CPU only |

## 相关辅助

- `FNiagaraRendererVariableInfo` — dataset offset → GPU buffer offset 映射
- `FNiagaraRendererLayout` — GT+RT 双 copy VF 变量布局
- `FNiagaraRendererFeedback` — editor 错误 + 自动 fix delegate

## 相关

- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]] — 对偶运行时基类
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — 持有 `RendererProperties` 数组
- 4 个具体类型 Entity(本 Phase)

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/Niagara/NiagaraRendererProperties]]
- 主题读本:[[Readers/UE/Niagara/Phase 6 - Niagara 粒子如何变成屏幕像素]] § 1-2
