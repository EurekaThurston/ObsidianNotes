---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, rw-di, grid, sim-stages]
sources: 1
aliases: [UNiagaraDataInterfaceRWBase, UNiagaraDataInterfaceGrid2D, UNiagaraDataInterfaceGrid3D, FNiagaraDataInterfaceProxyRW]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceRW.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceRWBase + Grid2D/3D 基类

> **RW DI 生态基础**。合并 `RWBase`(abstract 基类)+ `Grid2D/3D`(abstract 中间类)+ `FNiagaraDataInterfaceProxyRW`(RT 替身基类)。Phase 7 RenderTarget2D、Phase 10 所有 Grid DI 都基于此。

## 一句话角色

- `UNiagaraDataInterfaceRWBase : UNiagaraDataInterface`(abstract)—— "可读写"的 DI 基类
- `UNiagaraDataInterfaceGrid2D / Grid3D : RWBase`(abstract)—— Grid 的 2D/3D 维度变种
- `FNiagaraDataInterfaceProxyRW : FNiagaraDataInterfaceProxy`(RT 侧)—— 带 SimStage 钩子 + GetElementCount

## `FNiagaraDataInterfaceProxyRW` 关键接口

```cpp
virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const = 0;   // 多少 threads dispatch
virtual uint32 GetGPUInstanceCountOffset(FNiagaraSystemInstanceID) const;
virtual void ClearBuffers(FRHICommandList&);
virtual FNiagaraDataInterfaceProxyRW* AsIterationProxy() override { return this; }  // ← 标记可迭代
```

**`AsIterationProxy` 返回 `this`** 让 SimStage 识别"这个 DI 可作 iteration source"。

## `ESetResolutionMethod`

```cpp
enum class ESetResolutionMethod { Independent, MaxAxis, CellSize };
```

Grid 3 种分辨率设置方式。

## Grid2D / Grid3D 基类字段

```cpp
// Grid3D
FIntVector NumCells;
float CellSize;
int32 NumCellsMaxAxis;
ESetResolutionMethod SetResolutionMethod;
FVector WorldBBoxSize;

// Grid2D
int32 NumCellsX, NumCellsY, NumCellsMaxAxis, NumAttributes;
FVector2D WorldBBoxSize;
```

## 共享 HLSL/VM 函数名

```
NumCells / CellSize / WorldBBoxSize
SimulationToUnit / UnitToSimulation
UnitToIndex / UnitToFloatIndex / IndexToUnit
IndexToUnitStaggeredX / Y
IndexToLinear / LinearToIndex
ExecutionIndexToGridIndex / ExecutionIndexToUnit
```

所有 Grid DI 共享——HLSL 里写 `GetCellSize_DI_0()` 等统一 API。

## 子类家族

- `UNiagaraDataInterfaceRenderTarget2D`(Phase 7)
- `UNiagaraDataInterfaceGrid2DCollection / GridReader / Grid3DCollection / NeighborGrid3D`(Phase 10)

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterface]] — 祖父类
- [[Wiki/Entities/Stock/UNiagaraSimulationStage]] — 使用本类作 iteration source
- Phase 7 `UNiagaraDataInterfaceRenderTarget2D` 也是本家族

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceRW]]
- 读本:[[Readers/Niagara/Phase 10 - Niagara 的高级特性]]
