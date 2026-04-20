---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, shader, compute-shader, gpu]
sources: 1
aliases: [FNiagaraShader, FNiagaraShaderType, FNiagaraShaderMap, FNiagaraShaderScript, Niagara Shader 家族]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/NiagaraShader/Public/NiagaraShader.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraShader 家族

> Niagara GPU compute shader 体系——每个 GPU emitter 一个 compiled shader。本页合并 `FNiagaraShader` / `FNiagaraShaderType` / `FNiagaraShaderMap` / `FNiagaraShaderMapId` / `FNiagaraDataInterfaceParamRef` 等。

## 一句话角色

- **`FNiagaraShader : FShader`**:已编译的 compute shader 实例,含粒子 buffer / constant buffer / SimStage / Event UAV 等大量 `LAYOUT_FIELD`
- **`FNiagaraShaderType : FShaderType`**:编译 meta,`BeginCompileShader / FinishCompileShader`
- **`FNiagaraShaderMap`**:一个 GPU Script 对应一份 shader cache(类比 `FMaterialShaderMap`)
- **`FNiagaraShaderMapId`**:Map 的唯一 id(CompilerVersion + FeatureLevel + BaseCompileHash + ReferencedHashes + AdditionalDefines + LayoutParams + bUsesRapidIterationParams)
- **`FNiagaraDataInterfaceParamRef`**:每个 DI 在 shader 侧的参数绑定,支持 binary memory image 序列化

## 关键 Shader 参数(FNiagaraShader 的 LAYOUT_FIELD)

| 类别 | 字段 |
|---|---|
| 粒子 Buffer | `Float/Int/Half InputBufferParam` (SRV) + `OutputBufferParam` (UAV) × 3 |
| Instance Count | `InstanceCountsParam` + `ReadInstanceCountOffsetParam` + `WriteInstanceCountOffsetParam` |
| Persistent IDs | `FreeIDBufferParam` + `IDToIndexBufferParam` |
| Constant Buffers | `Global/System/Owner/Emitter/External ConstantBufferParam[2]`(5 × 2 = 10) + `ViewUniformBufferParam` |
| Spawn/Tick | `SimStartParam / EmitterTickCounterParam / SpawnInfoOffsets / SpawnInfoParams / CopyInstancesBeforeStartParam / NumSpawnedInstancesParam / UpdateStartInstanceParam` |
| SimStage | `DefaultSimulationStageIndexParam / SimulationStageIndexParam / IterationInfoParam / NormalizedIterationIndexParam / DispatchThreadIdToLinearParam` |
| Events(×4 并发) | `EventFloat/Int UAV[4] / SRV[4]` + 读写 stride |
| DI | `DataInterfaceParameters` array |

## Thread Group Size

```cpp
static uint32 GetGroupSize(EShaderPlatform Platform) {
    return Platform == SP_PS4 ? 64 : 32;
}
```

PS4 用 64(wave size),其他用 32(典型 NV/AMD)。

## ShaderMap / 缓存

`FNiagaraShaderMapId` 是 DDC key —— 脚本改、编译器版本改、平台变都重编。编译异步走 `FNiagaraCompilationQueue` + `FNiagaraShaderQueueTickable`。

## 相关

- [[Wiki/Entities/Stock/UNiagaraScript]] — Asset 侧对位(Phase 1)
- [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]] — runtime 用 shader(Phase 5 `GPUScript_RT`)
- [[Wiki/Entities/Stock/UNiagaraDataInterface]] — 通过 `DataInterfaceParameters` 接入

## 深入阅读

- 源 × 4:[[Wiki/Sources/Stock/NiagaraShared]] / [[Wiki/Sources/Stock/NiagaraShader]] / [[Wiki/Sources/Stock/NiagaraShaderType]] / [[Wiki/Sources/Stock/NiagaraShaderMap]]
- 读本:[[Readers/Niagara/Phase8-gpu-simulation-读本]] § 2-3
