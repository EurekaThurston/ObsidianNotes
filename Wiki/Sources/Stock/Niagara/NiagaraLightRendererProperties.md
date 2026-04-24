---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, renderer, light, properties]
sources: 1
aliases: [NiagaraLightRendererProperties.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraLightRendererProperties.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraLightRendererProperties.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraLightRendererProperties.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 98 行(Phase 6 Properties 最小)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 6 — 渲染系统 6.9

## 职责

把每粒子变成一个**简单场景光源**(`FSimpleLightEntry`)。不产生 mesh batch,走场景的 simple light 系统。

> [!warning] 性能警告(来自代码注释)
> - CPU only(GPU 情况 light 无法提交给场景光源系统)
> - `bAffectsTranslucency=true` + 多粒子 = 严重开销——注释说 "create only a few particle lights at most"

## 主类

```cpp
UCLASS(editinlinenew, MinimalAPI, meta = (DisplayName = "Light Renderer"))
class UNiagaraLightRendererProperties : public UNiagaraRendererProperties
{
    // CPU only
    virtual bool IsSimTargetSupported(ENiagaraSimTarget InSimTarget) const override
    { return InSimTarget == CPUSim; }

    // 没有 mesh batch,没有 bounds calculator
    virtual FNiagaraBoundsCalculator* CreateBoundsCalculator() override { return nullptr; }
};
```

## 字段

### 光源基本

```cpp
uint32 bUseInverseSquaredFalloff : 1;  // 物理正确衰减 vs 用 LightExponent 绑定
uint32 bAffectsTranslucency : 1;       // ⚠️ 半透明也被光照
float RadiusScale;                     // 粒子半径 × 此值
FVector ColorAdd;                      // 静态色彩偏移
```

### 属性绑定(6 个 AdvancedDisplay)

```cpp
FNiagaraVariableAttributeBinding LightRenderingEnabledBinding;  // 控制 per-particle 启用
FNiagaraVariableAttributeBinding LightExponentBinding;          // 自定义衰减时用
FNiagaraVariableAttributeBinding PositionBinding;
FNiagaraVariableAttributeBinding ColorBinding;
FNiagaraVariableAttributeBinding RadiusBinding;
FNiagaraVariableAttributeBinding VolumetricScatteringBinding;
```

### DataSetAccessor 直接缓存

```cpp
FNiagaraDataSetAccessor<FVector> PositionDataSetAccessor;
FNiagaraDataSetAccessor<FLinearColor> ColorDataSetAccessor;
FNiagaraDataSetAccessor<float> RadiusDataSetAccessor;
FNiagaraDataSetAccessor<float> ExponentDataSetAccessor;
FNiagaraDataSetAccessor<float> ScatteringDataSetAccessor;
FNiagaraDataSetAccessor<FNiagaraBool> EnabledDataSetAccessor;
```

和 Ribbon 一样——强类型 accessor 缓存。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraLightRendererProperties]](合并 Renderer)
- [[Wiki/Entities/Stock/Niagara/FNiagaraRenderer]]
