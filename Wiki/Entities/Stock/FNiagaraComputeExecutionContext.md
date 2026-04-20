---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, compute-context, gpu]
sources: 1
aliases: [FNiagaraComputeExecutionContext]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptExecutionContext.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraComputeExecutionContext

> Niagara **GPU Compute 脚本执行上下文**。Phase 3 `FNiagaraEmitterInstance::GPUExecContext` 指向的东西。**Phase 8 真正展开**。

## 一句话角色

`FNiagaraComputeExecutionContext`(非 UObject)。GPU Emitter 的运行时状态持有者——和 `FNiagaraScriptExecutionContext`(CPU)平行但不继承自共同基类。

## 核心字段

| 字段 | 作用 |
|---|---|
| `FNiagaraDataSet* MainDataSet` | 粒子 DataSet |
| `UNiagaraScript* GPUScript` | GT 侧脚本引用 |
| `FNiagaraShaderScript* GPUScript_RT` | RT 侧 shader 引用(Phase 8) |
| `FNiagaraScriptInstanceParameterStore CombinedParamStore` | 合并后的参数存储 |
| `TArray<FNiagaraDataInterfaceProxy*> DataInterfaceProxies` | DI 的 RT 替身(Phase 7 / 8) |
| `FNiagaraDataBuffer* DataToRender` | 最新渲染用 buffer |
| `FNiagaraDataBuffer* TranslucentDataToRender` | 半透明低延迟 buffer(可选) |
| `FNiagaraGpuSpawnInfo GpuSpawnInfo_GT` | GT 产生的 spawn info |
| `uint32 DefaultSimulationStageIndex` | SimStages 默认(Phase 10) |
| `uint32 MaxUpdateIterations` | SimStages 迭代上限 |
| `TSet<uint32> SpawnStages` | 哪些 stage 是 spawn |
| `TArray<FSimulationStageMetaData> SimStageInfo` | SimStages 元数据(Phase 10) |

## 主方法

```cpp
void Reset(NiagaraEmitterInstanceBatcher* Batcher);
void InitParams(UNiagaraScript*, ENiagaraSimTarget, uint32 DefaultSimStage, int32 MaxIter, const TSet<uint32> SpawnStages);
bool Tick(FNiagaraSystemInstance* ParentSystemInstance);
void PostTick();
void SetDataToRender(FNiagaraDataBuffer*);
void SetTranslucentDataToRender(FNiagaraDataBuffer*);
FNiagaraDataBuffer* GetDataToRender(bool bIsLowLatencyTranslucent) const;
```

## SimStages 查询

```cpp
bool IsOutputStage(FNiagaraDataInterfaceProxy*, uint32 CurrentStage) const;
bool IsIterationStage(FNiagaraDataInterfaceProxy*, uint32 CurrentStage) const;
FNiagaraDataInterfaceProxyRW* FindIterationInterface(const TArray<FNiagaraDataInterfaceProxyRW*>&, uint32) const;
const FSimulationStageMetaData* GetSimStageMetaData(uint32) const;
```

## 相关

- [[Wiki/Entities/Stock/FNiagaraEmitterInstance]] — 持有者(通过 `GPUExecContext` 指针)
- [[Wiki/Entities/Stock/FNiagaraGPUSystemTick]] — 每帧被打包进这里
- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]] — RT 调度

## 深入阅读

- 源摘要:[[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] § 块 C
- 主题读本:[[Readers/Niagara/Phase5-cpu-script-execution-读本]] § 5(Phase 8 将完整展开)

## 开放问题(Phase 8)

- 全部待 Phase 8
