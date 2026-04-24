---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, data-interface, render-target, rw]
sources: 1
aliases: [UNiagaraDataInterfaceRenderTarget2D, RenderTarget 2D DI]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceRenderTarget2D.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceRenderTarget2D

> **可读写** RenderTarget2D DI。粒子可以向 RT 写入(UAV)或采样(SRV)。继承 `UNiagaraDataInterfaceRWBase`(不是普通 DI 基类)—— **Phase 10 Simulation Stages 基础之一**。

## 一句话角色

`UCLASS(Experimental, Blueprintable, BlueprintType) UNiagaraDataInterfaceRenderTarget2D : UNiagaraDataInterfaceRWBase`。典型用途:粒子写入热图、Grid 模拟、粒子间数据交换(通过共享 RT)。

## GT / RT 双 InstanceData

```cpp
// GT 侧
struct FRenderTarget2DRWInstanceData_GameThread {
    FIntPoint Size;
    ETextureRenderTargetFormat Format = RTF_RGBA16f;
    UTextureRenderTarget2D* TargetTexture;
    FNiagaraParameterDirectBinding<UObject*> RTUserParamBinding;
};

// RT 侧
struct FRenderTarget2DRWInstanceData_RenderThread {
    FIntPoint Size;
    FTextureReferenceRHIRef TextureReferenceRHI;
    FUnorderedAccessViewRHIRef UAV;                // ← 关键!compute shader 写
};
```

## RT 来源两种

1. **User 参数提供**:`RenderTargetUserParameter`(`FNiagaraUserParameterBinding`) —— 绑定 User.* 参数由外部设置
2. **DI 自管理**:`ManagedRenderTargets: TMap<uint64, UTextureRenderTarget2D*>` —— 按 `SystemInstanceID` 映射,内部创建

## 主字段

```cpp
FIntPoint Size;
TEnumAsByte<ETextureRenderTargetFormat> OverrideRenderTargetFormat;
uint8 bOverrideFormat : 1;
uint8 bPreviewRenderTarget : 1;  // editor
FNiagaraUserParameterBinding RenderTargetUserParameter;
```

## VM 函数

- `SetSize / GetSize`
- `SetValue`(写像素)
- `LinearToIndex`(1D index ↔ 2D)

## Proxy

```cpp
struct FNiagaraDataInterfaceProxyRenderTarget2DProxy : FNiagaraDataInterfaceProxyRW
{
    TMap<FNiagaraSystemInstanceID, FRenderTarget2DRWInstanceData_RenderThread> SystemInstancesToProxyData_RT;

    virtual void PostSimulate(FRHICommandList&, const FNiagaraDataInterfaceArgs&) override;
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const override;
};
```

**`FIntVector GetElementCount`** 是 `ProxyRW` 接口——SimStage 迭代时"每 pass dispatch 多少元素"由此返回,RT 这里就是 `(Size.X, Size.Y, 1)`。

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterface]]
- `UNiagaraDataInterfaceRWBase` — 父类(Phase 10)
- Phase 2 `UNiagaraFunctionLibrary::SetVariableTextureRenderTarget`

## 深入阅读

- 源:[[Wiki/Sources/Stock/Niagara/NiagaraDataInterfaceRenderTarget2D]]
- 读本:[[Readers/UE/Niagara/Phase 7 - 最强扩展点 Data Interface]] § 9
