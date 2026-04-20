---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, grid3d, rw-di, volumetric]
sources: 1
aliases: [UNiagaraDataInterfaceGrid3DCollection, FGrid3DBuffer]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceGrid3DCollection.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceGrid3DCollection

> **3D 体积网格**。体积烟雾、流体、火焰 3D。用 `FTextureRWBuffer3D` 存储。

## 一句话角色

继承 `UNiagaraDataInterfaceGrid3D`。数据结构 = `FTextureRWBuffer3D`(3D RW 纹理,HLSL 里是 `RWTexture3D<float>`)。多属性用 **tile 打包**到同一 3D 体积(不同于 Grid2D 的 TextureArray 模式)。

## `FGrid3DBuffer`

```cpp
class FGrid3DBuffer {
    FTextureRWBuffer3D GridBuffer;
};
```

## 双缓冲 + NumTiles

```cpp
struct FGrid3DCollectionRWInstanceData_RenderThread {
    FIntVector NumCells, NumTiles;       // ← NumTiles:多属性的 tile 布局
    FVector CellSize, WorldBBoxSize;
    EPixelFormat PixelFormat;

    TArray<TUniquePtr<FGrid3DBuffer>> Buffers;
    FGrid3DBuffer* CurrentData / DestinationData;
    FTextureRHIRef RenderTargetToCopyTo;
};
```

`NumTiles` 让 HLSL 访问属性 K 时知道该属性在 3D 体积里的 tile 偏移。

## 主字段

```cpp
UPROPERTY() int32 NumAttributes;
UPROPERTY() FNiagaraUserParameterBinding RenderTargetUserParameter;
UPROPERTY() ENiagaraGpuBufferFormat BufferFormat;
```

## VM 函数

```
SetValue / GetValue / SampleGrid
```

比 Grid2D 简单——没有 per-channel 重载(3D 场景常用 float/vec4,变种需求少)。

## 2D 与 3D 存储策略差异

| | 2D | 3D |
|---|---|---|
| 物理结构 | `Texture2DArray`(N slices = N attributes) | `RWTexture3D`(1 体积 + NumTiles 打包多属性) |
| 每属性容量 | NumCells.X × NumCells.Y | NumCells.X×Y×Z / NumTiles |
| 采样 shader | `tex.SampleLevel(uint3(x,y,attr),0)` | `tex[uint3(x+tileOff,y,z)]` |

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]]
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid2DCollection]] — 对位 2D 版
- [[Wiki/Entities/Stock/UNiagaraSimulationStage]] — 常作 iteration source

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceGrid3DCollection]]
- 读本:[[Readers/Niagara/Phase10-advanced-features-读本]]
