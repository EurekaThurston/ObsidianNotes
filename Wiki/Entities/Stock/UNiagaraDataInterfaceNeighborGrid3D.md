---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, neighbor-grid, spatial-hash, fluid, sph]
sources: 1
aliases: [UNiagaraDataInterfaceNeighborGrid3D, NeighborGrid3DRWInstanceData]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataInterfaceNeighborGrid3D.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraDataInterfaceNeighborGrid3D

> **空间哈希**——每 cell 存固定数量的粒子索引。流体(SPH)、群体(boids)、碰撞查询优化的核心。

## 一句话角色

`UCLASS UNiagaraDataInterfaceNeighborGrid3D : UNiagaraDataInterfaceGrid3D`。与 Grid3DCollection **目的完全不同**:后者存场(标量/向量按 cell),本类存**对象列表**(每 cell 一组粒子索引)。

## `NeighborGrid3DRWInstanceData`

```cpp
class NeighborGrid3DRWInstanceData {
    FIntVector NumCells;
    float CellSize;
    bool SetGridFromCellSize;
    uint32 MaxNeighborsPerCell;            // ⭐ 每 cell 容量上限
    FVector WorldBBoxSize;

    FRWBuffer NeighborhoodBuffer;           // [CellCount × MaxNeighborsPerCell] 粒子索引
    FRWBuffer NeighborhoodCountBuffer;       // [CellCount] 每 cell 当前数量
};
```

两个 buffer 配合:`count[cellIdx]` 当前放了几个,`neighbor[cellIdx * MaxNeighbors + k]` 是第 k 个粒子索引。超容 drop(lossy)。

## 主字段

```cpp
UPROPERTY(EditAnywhere, Category = "Grid")
uint32 MaxNeighborsPerCell;               // 用户配置
```

**只一个关键参数**——其他继承 Grid3D 基类(NumCells / CellSize / WorldBBoxSize)。

## 使用模式(SPH 典型)

```cpp
// Emitter Tick:
SimStage 0 (IterationSource = Particles):
    每粒子:hash(Position) → cellIdx
    原子 InterlockedAdd(count[cellIdx], 1) → slotIdx
    neighbor[cellIdx * MaxNeighbors + slotIdx] = ParticleID

SimStage 1 (IterationSource = Particles):
    每粒子:读自身 cell + 27 邻 cell 的 neighbor 列表
    → 算密度 / 压力 / 粘度
```

`PreStage` 被 Proxy 实现来**清 count buffer**(每 tick 重建)。

## `PostInitProperties` 自动注册

```cpp
virtual void PostInitProperties() override {
    if (HasAnyFlags(RF_ClassDefaultObject))
        FNiagaraTypeRegistry::Register(FNiagaraTypeDefinition(GetClass()), true, false, false);
}
```

CDO 加载时自动把本 DI 类型注册到 Niagara 类型系统—— script 里能用 `NeighborGrid3D` 作参数类型。

## 与 Grid3DCollection 的对比

| | Grid3DCollection | NeighborGrid3D |
|---|---|---|
| 目的 | 存场(标量/向量按 cell) | 存对象(粒子索引按 cell) |
| 容量 | 每 cell 恰好 1 值 | 每 cell 最多 MaxNeighbors |
| 超容 | N/A | Drop(lossy) |
| 典型用途 | 烟雾密度场、温度场 | SPH 邻域查询、碰撞空间优化 |

## 陷阱

- ⚠️ **`MaxNeighborsPerCell` 超容丢失**:重要粒子被 drop 的可能。需要较宽容量
- ⚠️ 与 Grid3DCollection 名字相似,目的完全不同 —— 不要混用
- ⚠️ 非 Experimental(相对稳定)—— 和 Collection 不同

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]]
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid3DCollection]] — 对位的场存储

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraDataInterfaceNeighborGrid3D]]
- 读本:[[Readers/Niagara/Phase10-advanced-features-读本]]
