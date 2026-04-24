---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, grid2d, rw-di, sim-stages]
sources: 1
aliases: [NiagaraDataInterfaceGrid2DCollection.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceGrid2DCollection.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceGrid2DCollection.h

- **Repo**: stock · **规模**: 268 行 · **Phase**: 10.3

## 职责

**2D 网格 + N 属性 + 双缓冲**。典型用途:2D 烟雾模拟、热图、2D 流体。存储用 `Texture2DArray`,每 slice 是一个 attribute。

## `FGrid2DBuffer`(L16)

```cpp
class FGrid2DBuffer {
    FGrid2DBuffer(int NumX, int NumY, int NumAttributes, EPixelFormat)
    {
        GridTexture = RHICreateTexture2DArray(NumX, NumY, NumAttributes, ...);  // ← Texture2DArray
        GridSRV / GridUAV = ...;
    }

    FTexture2DArrayRHIRef GridTexture;
    FShaderResourceViewRHIRef GridSRV;
    FUnorderedAccessViewRHIRef GridUAV;
};
```

`Texture2DArray` 让 HLSL 用 `GridTexture.SampleLevel(sampler, uint3(x, y, attrIdx), 0)` 索引——所有 attribute 合成一张大纹理。

## GT / RT InstanceData 双缓冲

```cpp
struct FGrid2DCollectionRWInstanceData_GameThread {
    FIntPoint NumCells;
    int32 NumAttributes;
    FVector2D CellSize, WorldBBoxSize;
    EPixelFormat PixelFormat = PF_R32_FLOAT;
    FNiagaraParameterDirectBinding<UObject*> RTUserParamBinding;
    UTextureRenderTarget* TargetTexture;
    TArray<FNiagaraVariableBase> Vars;
    TArray<uint32> Offsets;
};

struct FGrid2DCollectionRWInstanceData_RenderThread {
    // 同上参数 +
    TArray<TUniquePtr<FGrid2DBuffer>> Buffers;
    FGrid2DBuffer* CurrentData;
    FGrid2DBuffer* DestinationData;                    // ← 双缓冲
    FTextureRHIRef RenderTargetToCopyTo;
    TArray<int32> AttributeIndices;
    TArray<FName> Vars;
    TArray<int32> VarComponents;
    TArray<uint32> Offsets;

    void BeginSimulate(FRHICommandList&);
    void EndSimulate(FRHICommandList&);
};
```

**Current/Destination 双缓冲** + `BeginSimulate/EndSimulate` 流程——和 Phase 4 `FNiagaraDataSet` 思路同源。

## Proxy(L95)

```cpp
struct FNiagaraDataInterfaceProxyGrid2DCollectionProxy : public FNiagaraDataInterfaceProxyRW
{
    virtual void PreStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&) override;  // 切缓冲
    virtual void PostStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&) override;
    virtual void PostSimulate(FRHICommandList&, const FNiagaraDataInterfaceArgs&) override;
    virtual void ResetData(FRHICommandList&, const FNiagaraDataInterfaceArgs&) override;
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const override;

    TMap<FNiagaraSystemInstanceID, FGrid2DCollectionRWInstanceData_RenderThread> SystemInstancesToProxyData_RT;
};
```

SimStage hooks 都实现:`PreStage` 切 current/destination,`PostStage` 收尾,`PostSimulate` 把结果 copy 到 user RT。

## 主类(L114)

```cpp
UCLASS(EditInlineNew, Category = "Grid", meta = (DisplayName = "Grid2D Collection", Experimental),
       Blueprintable, BlueprintType)
class UNiagaraDataInterfaceGrid2DCollection : public UNiagaraDataInterfaceGrid2D
{
    FNiagaraUserParameterBinding RenderTargetUserParameter;   // 可绑定 User.RT
    ENiagaraGpuBufferFormat OverrideBufferFormat;
    uint8 bOverrideFormat : 1;
#if WITH_EDITORONLY_DATA
    uint8 bPreviewGrid : 1;                                    // editor 预览
    FName PreviewAttribute;
#endif

    // 诸多 VM Set/Get/Sample 方法,按属性类型(Float/Vec2/Vec3/Vec4)分
};
```

## VM 函数家族

```
ClearCellFunctionName, CopyPreviousToCurrentForCellFunctionName
SetValueFunctionName, GetValueFunctionName, SampleGridFunctionName
Set/Get/Sample Vector4 / Vector3 / Vector2 / Float
SetNumCellsFunctionName, GetVector4/Vector3/Vector2/FloatAttributeIndexFunctionName
```

**按 channel 数重载**是 HLSL sampling 的标准模式。

## 运行时 API

```cpp
void FindAttributesByName(FName DIName, TArray<FNiagaraVariableBase>& OutVars, TArray<uint32>& OutOffsets, int32& OutNumAttribChannelsFound, TArray<FText>* OutWarnings);
void FindAttributes(TArray<FNiagaraVariableBase>& OutVars, TArray<uint32>& OutOffsets, int32& OutNumAttribChannelsFound, TArray<FText>* OutWarnings);
```

## Editor 增强:SimStage 自动 HLSL 生成

```cpp
virtual bool SupportsSetupAndTeardownHLSL() const { return true; }
virtual bool GenerateSetupHLSL(...) / GenerateTeardownHLSL(...);
virtual bool SupportsIterationSourceNamespaceAttributesHLSL() const override { return true; }
virtual bool GenerateIterationSourceNamespaceReadAttributesHLSL(...) / WriteAttributesHLSL(...);
```

当用作 SimStage iteration source 时,自动生成 "StackContext parameter map" 的读/写 HLSL 代码。这是 Grid 能作 iteration source 的核心。

## Deprecated BlueprintCallable

`FillTexture2D / FillRawTexture2D / GetRawTextureSize` 都标了 `DeprecatedFunction`——用户参数方式取代。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceGrid2DCollection]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRWBase]]
