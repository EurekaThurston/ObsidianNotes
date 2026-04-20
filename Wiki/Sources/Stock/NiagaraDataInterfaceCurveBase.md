---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, curve, lut]
sources: 1
aliases: [NiagaraDataInterfaceCurveBase.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCurveBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceCurveBase.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCurveBase.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 251 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.4

## 职责

Curve DI 的**公共基类**。所有 Curve 家族 DI(Float / Vector / LinearColor)都继承本类。核心机制:**把 UE `FRichCurve` 烘焙成 LUT(Look-Up Table)数组**,避免运行时 spline 计算。

## 关键类型

### `FNiagaraDataInterfaceProxyCurveBase`(L14)— RT 替身

```cpp
struct FNiagaraDataInterfaceProxyCurveBase : public FNiagaraDataInterfaceProxy
{
    float LUTMinTime;
    float LUTMaxTime;
    float LUTInvTimeRange;
    float CurveLUTNumMinusOne;
    FReadBuffer CurveLUT;         // GPU SRV

    virtual int32 PerInstanceDataPassedToRenderThreadSize() const override { return 0; }
};
```

LUT 数据推到 RT,GPU shader 靠 `CurveLUT` SRV 采样。析构要在 RT(`IsInRenderingThread()` check)。

### `UNiagaraDataInterfaceCurveBase`(L40)

```cpp
UCLASS(EditInlineNew, Category = "Curves", meta = (DisplayName = "Float Curve"))
class NIAGARA_API UNiagaraDataInterfaceCurveBase : public UNiagaraDataInterface
```

### 核心字段

```cpp
UPROPERTY() TArray<float> ShaderLUT;              // 烘焙的 LUT 数组
UPROPERTY() float LUTMinTime / LUTMaxTime / LUTInvTimeRange / LUTNumSamplesMinusOne;

UPROPERTY(EditAnywhere) uint32 bUseLUT : 1;       // LUT vs 直接 eval
UPROPERTY(EditAnywhere) uint32 bExposeCurve : 1;  // 发布为 UTexture2D 给材质用

#if WITH_EDITORONLY_DATA
uint32 bOptimizeLUT : 1;                          // 减少冗余样本
uint32 bOverrideOptimizeThreshold : 1;
float OptimizeThreshold;                          // default 0.01
#endif

UPROPERTY() FName ExposedName;                    // 暴露时的名字
UPROPERTY() UTexture2D* ExposedTexture;           // 生成的曲线 texture

enum { CurveLUTDefaultWidth = 128 };              // LUT 默认 128 样本
```

### `FCurveData` struct(L160)

```cpp
struct FCurveData {
    FRichCurve* Curve;
    FName Name;
    FLinearColor Color;
};
```

Editor UI:一个 DI 可能有多条曲线(Vector = 3 条),每条一个 FCurveData。

### 时间 normalize/unnormalize

```cpp
FORCEINLINE float NormalizeTime(float T) const { return (T - LUTMinTime) * LUTInvTimeRange; }
FORCEINLINE float UnnormalizeTime(float T) const { return (T / LUTInvTimeRange) + LUTMinTime; }
```

把 curve 原始时间映射到 LUT 的 0-1 索引。

### 纯虚方法

```cpp
virtual void GetCurveData(TArray<FCurveData>& OutCurveData);
virtual int32 GetCurveNumElems() const;                             // 1 / 3 / 4
virtual void UpdateTimeRanges();
virtual TArray<float> BuildLUT(int32 NumEntries) const;
```

### LUT 维护

```cpp
void SetDefaultLUT();
#if WITH_EDITORONLY_DATA
void UpdateLUT(bool bFromSerialize = false);      // 重建 LUT
void OptimizeLUT();                                // 去冗余样本
void UpdateExposedTexture();                       // 同步 ExposedTexture
#endif
```

### `TCurveUseLUTBinder`(L221)

```cpp
template<typename NextBinder>
struct TCurveUseLUTBinder {
    template<typename... ParamTypes>
    static void Bind(UNiagaraDataInterface* Interface, ...) {
        if (CurveInterface->bUseLUT)
            NextBinder::Bind<ParamTypes..., TIntegralConstant<bool, true>>(...);
        else
            NextBinder::Bind<ParamTypes..., TIntegralConstant<bool, false>>(...);
    }
};
```

根据 `bUseLUT` 编译期选择 LUT 路径或直接 eval 路径——同一个 `SampleCurveInternal<UseLUT>` 模板两次实例化。

### GPU shader 参数

```cpp
struct FNiagaraDataInterfaceParametersCS_Curve : public FNiagaraDataInterfaceParametersCS
{
    LAYOUT_FIELD(FShaderParameter, MinTime);
    LAYOUT_FIELD(FShaderParameter, MaxTime);
    LAYOUT_FIELD(FShaderParameter, InvTimeRange);
    LAYOUT_FIELD(FShaderParameter, CurveLUTNumMinusOne);
    LAYOUT_FIELD(FShaderResourceParameter, CurveLUT);  // SRV
};
```

GPU 采样直接从 SRV 读 LUT,省去运行时 spline。

## 子类家族

- `UNiagaraDataInterfaceCurve`(float,7.3 本 Phase)
- `UNiagaraDataInterfaceVector2DCurve` / `UNiagaraDataInterfaceVectorCurve` / `UNiagaraDataInterfaceVector4Curve` / `UNiagaraDataInterfaceColorCurve`(本学习路径只覆盖 Float)

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceCurve]](合并了 Base + Float)
- [[Wiki/Entities/Stock/UNiagaraDataInterface]]
