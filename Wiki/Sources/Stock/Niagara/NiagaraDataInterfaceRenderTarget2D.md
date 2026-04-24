---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, data-interface, render-target, rw, gpu]
sources: 1
aliases: [NiagaraDataInterfaceRenderTarget2D.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceRenderTarget2D.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceRenderTarget2D.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceRenderTarget2D.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 138 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 7 — 数据接口系统 7.10(**Experimental + 首个 RW DI**)

## 职责

**读写 RenderTarget2D** 的 DI,粒子可以向一张 RT 绘制(写)或采样(读)。这是 Phase 10 **Simulation Stages**(流体/grid 模拟)的基础 DI 之一——不再是"读取外部数据",而是"粒子和外部数据双向交互"。

UMETA 标记 `Experimental`。Blueprintable + BlueprintType(可从 BP 直接操作)。

## 继承:UNiagaraDataInterfaceRWBase(**不是普通 DI 基类**)

```cpp
UCLASS(EditInlineNew, Category = "Grid", meta = (DisplayName = "Render Target 2D", Experimental), Blueprintable, BlueprintType)
class NIAGARA_API UNiagaraDataInterfaceRenderTarget2D : public UNiagaraDataInterfaceRWBase
```

`UNiagaraDataInterfaceRWBase`(Phase 10 详)是"可读写 DI"的基类,带 Simulation Stage 相关额外接口。

## 双缓冲 InstanceData

### GT 侧

```cpp
struct FRenderTarget2DRWInstanceData_GameThread
{
    FIntPoint Size;
    ETextureRenderTargetFormat Format = RTF_RGBA16f;
    UTextureRenderTarget2D* TargetTexture;
    FNiagaraParameterDirectBinding<UObject*> RTUserParamBinding;  // User 参数提供 RT
#if WITH_EDITORONLY_DATA
    uint32 bPreviewTexture : 1;
#endif
};
```

### RT 侧

```cpp
struct FRenderTarget2DRWInstanceData_RenderThread
{
    FIntPoint Size;
    FTextureReferenceRHIRef TextureReferenceRHI;
    FUnorderedAccessViewRHIRef UAV;                // ← 关键!UAV 让 compute shader 写
};
```

## Proxy

```cpp
struct FNiagaraDataInterfaceProxyRenderTarget2DProxy : public FNiagaraDataInterfaceProxyRW
{
    virtual void PostSimulate(FRHICommandList&, const FNiagaraDataInterfaceArgs&) override;
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const override;

    TMap<FNiagaraSystemInstanceID, FRenderTarget2DRWInstanceData_RenderThread> SystemInstancesToProxyData_RT;
};
```

多实例支持:一个 DI 实例对应多个 SystemInstance,每个 SystemInstance 一份 RT。

## 主类字段

```cpp
UPROPERTY(EditAnywhere) FIntPoint Size;                                   // RT 尺寸
UPROPERTY(EditAnywhere) TEnumAsByte<ETextureRenderTargetFormat> OverrideRenderTargetFormat;
UPROPERTY(EditAnywhere) uint8 bOverrideFormat : 1;
UPROPERTY(Transient, EditAnywhere) uint8 bPreviewRenderTarget : 1;        // editor 预览
UPROPERTY(EditAnywhere) FNiagaraUserParameterBinding RenderTargetUserParameter;  // User 参数提供 RT

UPROPERTY(Transient)
TMap<uint64, UTextureRenderTarget2D*> ManagedRenderTargets;               // 内部管理的 RT(按 SystemInstanceID)
```

**两种 RT 来源**:
1. 用户通过 `RenderTargetUserParameter` 提供现有 RT
2. 不提供时 DI 自己 new 一张,存 `ManagedRenderTargets`

## 关键方法

```cpp
virtual bool CanExecuteOnTarget(ENiagaraSimTarget Target) const override { return true; }  // CPU+GPU

// 生命周期 / 参数
virtual bool InitPerInstanceData(void*, FNiagaraSystemInstance*) override;
virtual void DestroyPerInstanceData(void*, FNiagaraSystemInstance*) override;
virtual bool PerInstanceTick(void*, FNiagaraSystemInstance*, float) override;
virtual bool PerInstanceTickPostSimulate(...) override;
virtual bool HasPreSimulateTick() const override { return true; }
virtual bool HasPostSimulateTick() const override { return true; }

// VM / GPU HLSL
virtual void GetFunctions(...) override;
virtual void GetVMExternalFunction(...) override;
virtual void GetParameterDefinitionHLSL(...) override;
virtual bool GetFunctionHLSL(...) override;

// VM 函数
void GetSize(FVectorVMContext&);
void SetSize(FVectorVMContext&);

// Expose variables
virtual bool CanExposeVariables() const override { return true; }
virtual void GetExposedVariables(TArray<FNiagaraVariableBase>& OutVariables) const override;
```

### Function 名

```cpp
static const FName SetValueFunctionName;   // 写像素
static const FName SetSizeFunctionName;
static const FName GetSizeFunctionName;
static const FName LinearToIndexName;      // 1D index ↔ 2D (x,y)
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRenderTarget2D]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]
- Phase 2 `UNiagaraFunctionLibrary::SetVariableTextureRenderTarget`
- Phase 10 `UNiagaraDataInterfaceRWBase`(基类,SimStages 相关)
