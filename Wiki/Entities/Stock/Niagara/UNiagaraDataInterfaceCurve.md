---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, curve, lut]
sources: 1
aliases: [UNiagaraDataInterfaceCurve, UNiagaraDataInterfaceCurveBase, Curve DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCurveBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceCurve / CurveBase

> **最简单的 DI 示例**。让脚本采样 `FRichCurve`(float 曲线)。
>
> ⚠️ **合并页说明**:Curve 族有两个类——`UNiagaraDataInterfaceCurveBase`(抽象公共基类,251 行,无独立 Entity 页)与 `UNiagaraDataInterfaceCurve`(float 具体实现,58 行)。本 Entity 页**合并覆盖这两个类**,aliases 字段同时登记两者。对应 Source 有两个:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurveBase]] + [[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurve]]。

## 一句话角色

- `UNiagaraDataInterfaceCurveBase : UNiagaraDataInterface`(abstract)—— 公共 LUT 烘焙机制
- `UNiagaraDataInterfaceCurve : UNiagaraDataInterfaceCurveBase` —— float 曲线具体实现

## 核心机制:LUT

```cpp
UPROPERTY() TArray<float> ShaderLUT;
UPROPERTY() float LUTMinTime / LUTMaxTime / LUTInvTimeRange / LUTNumSamplesMinusOne;
UPROPERTY(EditAnywhere) uint32 bUseLUT : 1;       // LUT vs 直接 spline eval
enum { CurveLUTDefaultWidth = 128 };              // 默认 128 样本
```

编辑器调 `UpdateLUT` 把 `FRichCurve` 烘焙成数组。运行时:
- CPU:`TCurveUseLUTBinder` 编译期根据 `bUseLUT` 分派到 `SampleCurveInternal<UseLUT::true>` 或 `<false>`
- GPU:RT 侧 `FNiagaraDataInterfaceProxyCurveBase::CurveLUT`(FReadBuffer SRV),shader 直接采样

`bOptimizeLUT` + `OptimizeThreshold`(editor)去冗余样本。

## Expose to Materials

```cpp
UPROPERTY(EditAnywhere) uint32 bExposeCurve : 1;
UPROPERTY() class UTexture2D* ExposedTexture;
UPROPERTY() FName ExposedName;
```

`UpdateExposedTexture` 把 LUT 做成 `UTexture2D`,材质可绑定——**跨特效/材质共享曲线**。

## `FCurveData` struct

Editor UI:一个 DI 可有多条曲线(Vector = 3)。每条 `{ FRichCurve*, FName, FLinearColor }`。

## VM 函数

`UNiagaraDataInterfaceCurve` 提供一个函数:

```
Curve.SampleCurve(float X) → float Y
```

## 子类家族

- `UNiagaraDataInterfaceVector2DCurve` / `VectorCurve` / `Vector4Curve` / `ColorCurve` —— 本学习路径不覆盖

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]] — 祖父类
- Phase 4 `FNiagaraTypeDefinition` — curve 数据类型

## 深入阅读

- Base 源:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurveBase]]
- Float 源:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceCurve]]
- 读本:[[Readers/UE/Niagara/Phase 7 - 最强扩展点 Data Interface]] § 3
