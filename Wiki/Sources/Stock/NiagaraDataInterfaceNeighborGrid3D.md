---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, neighbor-grid, spatial-hash, fluid]
sources: 1
aliases: [NiagaraDataInterfaceNeighborGrid3D.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceNeighborGrid3D.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataInterfaceNeighborGrid3D.h

- **Repo**: stock · **规模**: 99 行 · **Phase**: 10.6(**最后一个文件**)

## 职责

**空间哈希**:每 cell 存 `MaxNeighborsPerCell` 个粒子索引,让粒子可以**查询自己邻域**。流体模拟(SPH)、群体行为(boids)的基础。

## `NeighborGrid3DRWInstanceData`(L19)

```cpp
class NeighborGrid3DRWInstanceData {
    FIntVector NumCells;
    float CellSize;
    bool SetGridFromCellSize;
    uint32 MaxNeighborsPerCell;       // ⭐ 每 cell 容量上限
    FVector WorldBBoxSize;

    FRWBuffer NeighborhoodBuffer;      // [CellCount × MaxNeighborsPerCell] 粒子索引数组
    FRWBuffer NeighborhoodCountBuffer; // [CellCount] 每 cell 实际数量
};
```

**两个 buffer**:索引数组(固定大小 × cell 数)+ count 数组(每 cell 几个有效)。超容 drop(lossy)。

## Proxy

```cpp
struct FNiagaraDataInterfaceProxyNeighborGrid3D : public FNiagaraDataInterfaceProxyRW
{
    virtual void PreStage(FRHICommandList&, const FNiagaraDataInterfaceStageArgs&) override;
    int32 PerInstanceDataPassedToRenderThreadSize() const override { return sizeof(NeighborGrid3DRWInstanceData); }
    FIntVector GetElementCount(FNiagaraSystemInstanceID) const override;
    TMap<FNiagaraSystemInstanceID, NeighborGrid3DRWInstanceData> SystemInstancesToProxyData;
};
```

`PreStage` 清 count buffer(每 tick 重建邻域)。

## 主类

```cpp
UCLASS(EditInlineNew, Category = "Grid", meta = (DisplayName = "Neighbor Grid3D"))
class UNiagaraDataInterfaceNeighborGrid3D : public UNiagaraDataInterfaceGrid3D
{
    UPROPERTY(EditAnywhere, Category = "Grid")
    uint32 MaxNeighborsPerCell;

    virtual void PostInitProperties() override {
        if (HasAnyFlags(RF_ClassDefaultObject))
            FNiagaraTypeRegistry::Register(FNiagaraTypeDefinition(GetClass()), true, false, false);
    }

    // VM
    void GetWorldBBoxSize(FVectorVMContext&);
    void GetNumCells(FVectorVMContext&);
    void GetMaxNeighborsPerCell(FVectorVMContext&);
};
```

## 注意:Experimental 标记没有

和 Grid2D/3D Collection 不同,本 DI **没有 Experimental 标记**——说明它相对稳定。

## 使用模式

```
Emitter Tick:
    SimStage 0 (IterationSource=Particles):
        每粒子 → Hash(Position) → 写入对应 cell(原子操作,cap = MaxNeighborsPerCell)

    SimStage 1 (IterationSource=Particles):
        每粒子 → 读自身及邻 27 个 cell → 算邻域力(SPH 密度/压力/粘度)
```

## 为什么不是 Collection

Grid2DCollection / Grid3DCollection 存**标量/向量场**(密度、速度、压力等按 cell 存值);NeighborGrid3D 存**对象列表**(每 cell 一组粒子索引)。数据结构目的不同。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceNeighborGrid3D]]
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]]
