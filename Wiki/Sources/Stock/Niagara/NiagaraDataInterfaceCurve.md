---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, curve, float]
sources: 1
aliases: [NiagaraDataInterfaceCurve.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCurve.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceCurve.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceCurve.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 58 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.3(**最简单的 DI 示例**)

## 职责

最小的 Curve DI —— 采样一条 `FRichCurve`(float 曲线)。作为**理解 DI 模式的最佳入门样例**。

## 类声明

```cpp
UCLASS(EditInlineNew, Category = "Curves", meta = (DisplayName = "Curve for Floats"))
class NIAGARA_API UNiagaraDataInterfaceCurve : public UNiagaraDataInterfaceCurveBase
{
public:
    UPROPERTY(EditAnywhere, Category = "Curve")
    FRichCurve Curve;              // 编辑器可配的曲线

    enum { CurveLUTNumElems = 1 };  // 每个 LUT 样本 1 float
```

### 覆盖的接口

```cpp
virtual void UpdateTimeRanges() override;                        // 从 Curve.GetTimeRange 更新 LUTMin/Max/InvTimeRange
virtual TArray<float> BuildLUT(int32 NumEntries) const override; // 调 Curve.Eval 采样到 LUT
virtual void GetCurveData(TArray<FCurveData>& OutCurveData) override;

virtual void GetFunctions(TArray<FNiagaraFunctionSignature>& OutFunctions) override;
virtual void GetVMExternalFunction(const FVMExternalFunctionBindingInfo&, void*, FVMExternalFunction&) override;

template<typename UseLUT>
void SampleCurve(FVectorVMContext& Context);                     // VM 入口(模板特化 LUT vs 直接 eval)

virtual bool GetFunctionHLSL(...) override;                      // GPU HLSL 生成
virtual int32 GetCurveNumElems() const { return CurveLUTNumElems; }  // = 1

template<typename UseLUT>
FORCEINLINE_DEBUGGABLE float SampleCurveInternal(float X);        // 内部采样
```

## VM 函数注册

```cpp
static const FName SampleCurveName;  // "SampleCurve"
```

DI 给脚本提供一个函数:`Curve.SampleCurve(float X) → float Y`。在编辑器里就是 "Curve DI" 模块的 `SampleCurve` 节点。

## 编译期多态:`TCurveUseLUTBinder`

```cpp
template<typename UseLUT>
FORCEINLINE_DEBUGGABLE float SampleCurveInternal(float X)
{
    if constexpr (UseLUT::Value) {
        // 查 LUT:时间 normalize → 索引 → 线性插值
    } else {
        return Curve.Eval(X);  // 直接 spline eval(慢但精确)
    }
}
```

`bUseLUT` 布尔在编译期通过 `TCurveUseLUTBinder` 变成 `TIntegralConstant<bool, true/false>` 模板参数——VM 运行时零分支。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceCurve]](合并 CurveBase + Curve)
- Phase 7 读本 § 3
