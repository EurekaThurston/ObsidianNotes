---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, sim-stages, advanced]
sources: 1
aliases: [NiagaraSimulationStageBase.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraSimulationStageBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraSimulationStageBase.h

- **Repo**: stock · **规模**: 78 行 · **Phase**: 10.1

## 职责

定义 `UNiagaraSimulationStageBase` + `UNiagaraSimulationStageGeneric` —— **Simulation Stages** 的 Asset 侧。SimStages 允许 Emitter tick 由多 pass 组成(每 pass 自己的 script + iteration source),是 **Grid 流体模拟 / 邻域查询 / 多 pass 计算** 的基础。

## `UNiagaraSimulationStageBase`(L17)

```cpp
UCLASS()
class UNiagaraSimulationStageBase : public UNiagaraMergeable
{
    UPROPERTY() UNiagaraScript* Script;
    UPROPERTY(EditAnywhere, Category = "Simulation Stage")
    FName SimulationStageName;
    UPROPERTY() uint32 bEnabled : 1;

    virtual bool AppendCompileHash(FNiagaraCompileHashVisitor*) const;

#if WITH_EDITOR
    virtual FName GetStackContextReplacementName() const { return NAME_None; }
    void SetEnabled(bool);
    void RequestRecompile();
#endif
};
```

## `UNiagaraSimulationStageGeneric`(L47)

```cpp
UCLASS(meta = (DisplayName = "Generic Simulation Stage"))
class UNiagaraSimulationStageGeneric : public UNiagaraSimulationStageBase
{
    UPROPERTY(EditAnywhere)
    ENiagaraIterationSource IterationSource;          // Particles | DataInterface

    UPROPERTY(EditAnywhere)
    int32 Iterations = 1;                              // 重复次数(Jacobi 等迭代式)

    UPROPERTY(EditAnywhere, EditCondition="DataInterface")
    uint32 bSpawnOnly : 1;                            // 只在 reset 后首 tick 跑

    UPROPERTY(EditAnywhere, AdvancedDisplay, EditCondition="Particles")
    uint32 bDisablePartialParticleUpdate : 1;          // Partial update 控制(debugging)

    UPROPERTY(EditAnywhere, EditCondition="DataInterface")
    FNiagaraVariableDataInterfaceBinding DataInterface;   // 迭代目标 DI
};
```

## Iteration Source 两种

- **Particles**:每粒子执行一次,默认模式
- **DataInterface**:每 DI 元素(Grid 的每 cell / NeighborGrid 的每格)执行一次

## Iterations 的意义

比如 Jacobi 压力求解常需要 20-40 次迭代,传统做法是写 20-40 个独立 SimStage。`Iterations` 参数让**一条 SimStage 覆盖 N 次**——参见 Phase 8.5 `FSimulationStageMetaData::MinStage/MaxStage`。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/UNiagaraSimulationStage]]
- Phase 8.5 `FSimulationStageMetaData`
