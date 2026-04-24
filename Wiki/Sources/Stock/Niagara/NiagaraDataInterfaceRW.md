---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, rw-di, grid, sim-stages]
sources: 1
aliases: [NiagaraDataInterfaceRW.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceRW.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceRW.h

- **Repo**: stock · **规模**: 246 行 · **Phase**: 10.2(**RW DI 基础**)

## 职责

定义 **RW DI 生态的基类**:`UNiagaraDataInterfaceRWBase` + `UNiagaraDataInterfaceGrid2D` + `UNiagaraDataInterfaceGrid3D` + `FNiagaraDataInterfaceProxyRW`(RT 替身基类)。Phase 7 `UNiagaraDataInterfaceRenderTarget2D` 继承这个体系。Grid / NeighborGrid / RenderTarget 都基于此。

## 全局 HLSL / VM 常量(L10-L34)

```cpp
extern const FString NumAttributesName, NumCellsName, CellSizeName, WorldBBoxSizeName;
extern const FName NumCellsFunctionName / CellSizeFunctionName / WorldBBoxSizeFunctionName;
extern const FName SimulationToUnitFunctionName / UnitToSimulationFunctionName;
extern const FName UnitToIndexFunctionName / UnitToFloatIndexFunctionName / IndexToUnitFunctionName;
extern const FName IndexToUnitStaggeredXFunctionName / StaggeredYFunctionName;
extern const FName IndexToLinearFunctionName / LinearToIndexFunctionName;
extern const FName ExecutionIndexToGridIndexFunctionName / ExecutionIndexToUnitFunctionName;
```

Grid DI 共享的 HLSL 函数名——所有 grid DI 都提供"索引 ↔ 单元 ↔ 世界空间"转换。

## `ESetResolutionMethod`(L35)

```cpp
enum class ESetResolutionMethod { Independent, MaxAxis, CellSize };
```

Grid 3 种分辨率设置方式。

## `FNiagaraDataInterfaceProxyRW`(L45)

```cpp
struct FNiagaraDataInterfaceProxyRW : public FNiagaraDataInterfaceProxy
{
    virtual FIntVector GetElementCount(FNiagaraSystemInstanceID) const = 0;
    virtual uint32 GetGPUInstanceCountOffset(FNiagaraSystemInstanceID) const { return INDEX_NONE; }
    virtual void ClearBuffers(FRHICommandList&) {}
    virtual FNiagaraDataInterfaceProxyRW* AsIterationProxy() override { return this; }
};
```

**关键**:`AsIterationProxy` 返回自己,让 SimStage 知道"本 DI 能作为 iteration source"。`GetElementCount` 告诉 dispatch 要多少 thread。

## `UNiagaraDataInterfaceRWBase`(L63)

```cpp
UCLASS(abstract, EditInlineNew)
class UNiagaraDataInterfaceRWBase : public UNiagaraDataInterface
{
    UPROPERTY(EditAnywhere, Category = "Deprecated")
    TSet<int> OutputShaderStages;         // ← Deprecated,新走 SimStageMetaData
    UPROPERTY(EditAnywhere, Category = "Deprecated")
    TSet<int> IterationShaderStages;

    virtual bool CanExecuteOnTarget(ENiagaraSimTarget) const override { return true; }  // CPU + GPU
};
```

注意:旧版 "shader stages" 已被 Phase 8.5 `FSimulationStageMetaData` 替代。

## `UNiagaraDataInterfaceGrid2D / Grid3D`(L126, L195)

两个 abstract 基类:

```cpp
// Grid3D
UCLASS(abstract) UNiagaraDataInterfaceGrid3D : public UNiagaraDataInterfaceRWBase
{
    FIntVector NumCells;
    float CellSize;
    int32 NumCellsMaxAxis;
    ESetResolutionMethod SetResolutionMethod;
    FVector WorldBBoxSize;
};

// Grid2D(参数类似但 2D)
UCLASS(abstract) UNiagaraDataInterfaceGrid2D : public UNiagaraDataInterfaceRWBase
{
    int32 NumCellsX, NumCellsY;
    int32 NumCellsMaxAxis;
    int32 NumAttributes;
    bool SetGridFromMaxAxis;
    FVector2D WorldBBoxSize;
};
```

具体 Grid DI(Collection / NeighborGrid)继承这两个。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraDataInterfaceRWBase]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraSimulationStage]]
