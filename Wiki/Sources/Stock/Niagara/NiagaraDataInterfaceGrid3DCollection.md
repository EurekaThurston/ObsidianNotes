---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, grid3d, rw-di, volumetric]
sources: 1
aliases: [NiagaraDataInterfaceGrid3DCollection.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceGrid3DCollection.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceGrid3DCollection.h

- **Repo**: stock · **规模**: 158 行 · **Phase**: 10.5

## 职责

**3D 体积网格**。典型用途:体积烟雾、流体、火焰 3D 模拟。存储用 `FTextureRWBuffer3D`。

## `FGrid3DBuffer`(L16)

```cpp
class FGrid3DBuffer {
    FGrid3DBuffer(int NumX, int NumY, int NumZ, EPixelFormat)
    {
        GridBuffer.Initialize(BlockBytes, NumX, NumY, NumZ, PixelFormat);
    }

    FTextureRWBuffer3D GridBuffer;   // ← 3D RW 纹理
};
```

和 2D 不同,3D 用 `FTextureRWBuffer3D`(非 Texture2DArray),HLSL 里是 `RWTexture3D<float>`。

## GT/RT InstanceData

```cpp
struct FGrid3DCollectionRWInstanceData_GameThread {
    FIntVector NumCells, NumTiles;
    FVector CellSize, WorldBBoxSize;
    EPixelFormat PixelFormat;
    FNiagaraParameterDirectBinding<UObject*> RTUserParamBinding;
};

struct FGrid3DCollectionRWInstanceData_RenderThread {
    // 同上参数 +
    TArray<TUniquePtr<FGrid3DBuffer>> Buffers;
    FGrid3DBuffer* CurrentData;
    FGrid3DBuffer* DestinationData;              // 双缓冲
    FTextureRHIRef RenderTargetToCopyTo;

    void BeginSimulate(FRHICommandList&);
    void EndSimulate(FRHICommandList&);
};
```

**`NumTiles`**:3D collection 里多属性用 **tile 方式**打包(和 2D 的 Texture2DArray 策略不同)—— 多属性被 tile 成一个更大的 3D 体积。

## Proxy

```cpp
struct FNiagaraDataInterfaceProxyGrid3DCollectionProxy : public FNiagaraDataInterfaceProxyRW
{
    virtual void PreStage / PostStage / PostSimulate / ResetData;
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const override;
    TMap<FNiagaraSystemInstanceID, FGrid3DCollectionRWInstanceData_RenderThread> SystemInstancesToProxyData_RT;
};
```

## 主类

```cpp
UCLASS(Experimental, Blueprintable, BlueprintType)
class UNiagaraDataInterfaceGrid3DCollection : public UNiagaraDataInterfaceGrid3D
{
    UPROPERTY(EditAnywhere) int32 NumAttributes;
    UPROPERTY(EditAnywhere) FNiagaraUserParameterBinding RenderTargetUserParameter;
    UPROPERTY(EditAnywhere) ENiagaraGpuBufferFormat BufferFormat;

    // VM / HLSL interface 同 2D
};
```

## VM 函数

```cpp
SetValueFunctionName
GetValueFunctionName
SampleGridFunctionName
```

比 2D 简单(没有 per-channel 重载),因为 3D 常用 float/vec4,per-channel 变种需求少。

## Deprecated BP

```cpp
FillVolumeTexture / FillRawVolumeTexture / GetRawTextureSize / GetTextureSize  // Deprecated
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceGrid3DCollection]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRWBase]]
