---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, sim-stages, advanced]
sources: 1
aliases: [UNiagaraSimulationStageBase, UNiagaraSimulationStageGeneric]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSimulationStageBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraSimulationStage

> **Simulation Stages** 的 Asset 侧。让 Emitter tick 由多 pass 组成,每 pass 独立 script + iteration source。合并 `Base`(abstract)+ `Generic`(实际用)。

## 一句话角色

- `UNiagaraSimulationStageBase : UNiagaraMergeable`(UCLASS)—— 基类(Script + Name + bEnabled)
- `UNiagaraSimulationStageGeneric : StageBase`(UCLASS, "Generic Simulation Stage")—— 实际用户配置项

## Generic 字段

```cpp
ENiagaraIterationSource IterationSource;       // Particles | DataInterface
int32 Iterations = 1;                           // 迭代次数
uint32 bSpawnOnly : 1;                          // 只在 reset 后首 tick(DI iteration 用)
uint32 bDisablePartialParticleUpdate : 1;       // Partial update 控制(debug)
FNiagaraVariableDataInterfaceBinding DataInterface;  // 迭代目标 DI
```

## 两种 IterationSource

| 源 | 含义 |
|---|---|
| **Particles** | 每粒子执行一次(默认 tick 模式) |
| **DataInterface** | 每 DI 元素执行一次(Grid 每 cell / NeighborGrid 每 cell) |

## `Iterations` × SimStageMetaData MinStage/MaxStage

Jacobi 等迭代式算法:同一 stage 重复 N 次。用**一条 stage + Iterations=N**代替 N 条 stage。Phase 8.5 `FSimulationStageMetaData::MinStage/MaxStage` 是 runtime 表示。

## 典型流水(SPH 流体示例)

```
Stage 0(Particles):Advect + Gravity
Stage 1(Particles):写入 NeighborGrid(空间哈希)
Stage 2(Particles):读 NeighborGrid → 算邻域密度 + 压力
Stage 3(Particles):应用压力梯度 + 粘度力
Stage 4(Grid DI):把速度写入 Grid2D / Grid3D(可视化)
```

## 相关

- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase]] — 常作 iteration source
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid2DCollection]] / Grid3D / NeighborGrid3D
- Phase 8.5 `FSimulationStageMetaData` 是 shader 侧元数据

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraSimulationStageBase]]
- 读本:[[Readers/Niagara/Phase 10 - Niagara 的高级特性]]
