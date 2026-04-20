---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, script-base, sim-stage]
sources: 1
aliases: [NiagaraScriptBase.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraScriptBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScriptBase.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraScriptBase.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 53 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 8 — GPU 模拟 8.5

## 职责

定义 **`UNiagaraScriptBase`**——`UNiagaraScript`(Phase 1)的抽象基类,位于 NiagaraShader 模块。让 Shader 模块能依赖 script 但不拉入主模块。同时定义 **`FSimulationStageMetaData`**——SimStages 核心元数据(Phase 10 预告)。

## `FSimulationStageMetaData`(L10)

```cpp
USTRUCT()
struct NIAGARASHADER_API FSimulationStageMetaData
{
    FName SimulationStageName;
    FName IterationSource;                  // 迭代目标 DI(None=particles)

    uint32 bSpawnOnly : 1;                  // 只在 spawn 阶段跑
    uint32 bWritesParticles : 1;
    uint32 bPartialParticleUpdate : 1;      // 读写同一 buffer(省一次拷贝)

    TArray<FName> OutputDestinations;

    int32 MinStage;                         // 本 metadata 覆盖的 stage 索引范围
    int32 MaxStage;                          // 允许"本元数据对应 N 次迭代"
};
```

SimStages 允许一个 emitter 的 tick 由**多个 pass**(stages)组成——每 pass 可以指定不同的迭代源(粒子 vs Grid DI),可以写到不同 buffer。本 struct 描述每 pass 的元数据。

`MinStage / MaxStage` 是区间——SimStage 可以**重复 N 次**(比如流体模拟的 8 次 Jacobi 迭代),只用一条 metadata 条目。

## `UNiagaraScriptBase`(L47)

```cpp
UCLASS(MinimalAPI, abstract)
class UNiagaraScriptBase : public UObject
{
    virtual TConstArrayView<FSimulationStageMetaData> GetSimulationStageMetaData() const PURE_VIRTUAL(...);
};
```

Phase 1 `UNiagaraScript` 继承本类。`GetSimulationStageMetaData` 的实际实现在 `UNiagaraScript`,但**抽象在 Shader 模块**——让 Shader 只依赖这个接口不拉依赖。

## 涉及实体

- [[Wiki/Entities/Stock/UNiagaraScript]] — 派生子类(Phase 1)
- [[Wiki/Entities/Stock/FSimulationStageMetaData]](放 Phase 10)

## 开放问题

- SimStages 完整语义和 MinStage/MaxStage 使用案例 → **Phase 10**
