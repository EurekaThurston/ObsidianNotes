---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, class, uclass, script, compile, asset]
sources: 1
aliases: [UNiagaraScript, NiagaraScript, Niagara Script 资产]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScript.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraScript

> **编译后的脚本资产**。连接 editor 图源与运行时执行体的核心单元。Niagara 里几乎所有"能跑的东西"都是一个 UNiagaraScript。

## 概览

`UNiagaraScript : public UNiagaraScriptBase`,`UCLASS(MinimalAPI)`。角色由 `ENiagaraScriptUsage` 枚举决定,可以是:

- **素材级**:Function / Module / DynamicInput(编辑器装配用)
- **粒子级**:ParticleSpawn / ParticleSpawnInterpolated / ParticleUpdate / ParticleEvent / ParticleSimulationStage / ParticleGPUCompute
- **Emitter 级**:EmitterSpawn / EmitterUpdate(**不可单独编译**,合并进 System 脚本)
- **System 级**:SystemSpawn / SystemUpdate

每个 `UNiagaraScript` 同时承载:
- **editor 图源**(`Source : UNiagaraScriptSourceBase*`,editor-only)
- **编译产物**(`CachedScriptVM : FNiagaraVMExecutableData` + `CachedScriptVMId : FNiagaraVMExecutableDataId`)
- **GPU shader 资源**(`ScriptResource : TUniquePtr<FNiagaraShaderScript>`,Phase 8)

## 关键事实 / 属性

### Usage 系统

- `Usage`(`ENiagaraScriptUsage`)+ `UsageId`(`FGuid`)组成唯一角色 key(比如 "ParticleEvent #3")
- 提供大量 `IsXxxScript()` 成员/静态方法做分发判断
- `IsCompilable()` 返 false 仅对 `EmitterSpawn/EmitterUpdate`:他们的逻辑被融进 SystemSpawn/SystemUpdate
- `IsInterpolatedParticleSpawnScript`:特殊 usage `ParticleSpawnScriptInterpolated`,对应 Emitter 的 `bInterpolatedSpawning`

### 编译产物

| 字段 | 类型 | 说明 |
|---|---|---|
| `CachedScriptVM` | `FNiagaraVMExecutableData` | 字节码 + 属性列表 + DI 信息 + 外部函数表 + ... |
| `CachedScriptVMId` | `FNiagaraVMExecutableDataId` | "身份证"(编译器版本、Usage、IDs、开关、基图哈希) |
| `ScriptResource` | `TUniquePtr<FNiagaraShaderScript>` | RenderThread 端 shader map 资源 |
| `ScriptShader` | `FComputeShaderRHIRef` | RHI 句柄 |
| `ScriptExecutionParamStore` | `FNiagaraScriptExecutionParameterStore` | cooked 平台的执行绑定存储 |
| `ScriptExecutionBoundParameters` | `TArray<FNiagaraBoundParameter>` | 与上条配对 |
| (editor)`ScriptExecutionParamStoreCPU / GPU` | Transient,未 cook 时用 |

### 用户可调参数

- `RapidIterationParameters`(`FNiagaraParameterStore`):暴露给 UI 快速调节;通常不会触发重编译,除非 `bBakeOutRapidIteration` 开启

### DDC(Derived Data Cache)

- `FNiagaraVMExecutableDataId::operator==` + `GetTypeHash` + `BaseScriptCompileHash` 构成 DDC key
- `BuildNiagaraDDCKeyString(CompileId)` / `GetNiagaraDDCKeyString()`
- `BinaryToExecData / ExecToBinaryData`:DDC 二进制 ↔ in-memory 结构

### 编辑器功能

- 重编译:`ComputeVMCompilationId / AreScriptAndSourceSynchronized / RequestCompile / RequestExternallyManagedAsyncCompile / SetVMCompilationResults`
- 委托:`OnVMScriptCompiled / OnGPUScriptCompiled / OnPropertyChanged`
- 元数据:`Category / bDeprecated / DeprecationRecommendation / bExperimental / LibraryVisibility / ProvidedDependencies / RequiredDependencies / ModuleUsageBitmask / ScriptMetaData`

### 运行时接口

- `GetVMExecutableData()` — 拿 `CachedScriptVM`(`CachedScriptVM.ByteCode` 就是字节码本体)
- `IsReadyToRun(SimTarget)`
- `CanBeRunOnGpu()`
- `GetExecutionReadyParameterStore(SimTarget)`
- `GetRenderThreadScript()` — 拿 `FNiagaraShaderScript*`(Phase 8)

### Usage 枚举的结构

`IsUsageDependentOn(A, B)` 声明了执行顺序的依赖链。系统的一般 tick 顺序:

```
SystemSpawnScript → SystemUpdateScript
  ↓
每个 Emitter:
  EmitterSpawnScript(已合并) → EmitterUpdateScript(已合并)
  ↓
  ParticleSpawnScript(或 ParticleSpawnScriptInterpolated) → ParticleUpdateScript
  ↓(GPU)
  ParticleGPUComputeScript(compute shader)
  ↓(事件)
  ParticleEventScript × N
  ↓(Phase 10)
  ParticleSimulationStageScript × N
```

## 相关

- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] — editor Source 字段类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — 持有 SystemSpawn/Update
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 持有所有粒子脚本 + GPUComputeScript
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — Script 的 CPU(VectorVM byte code)/ GPU(Compute Shader)双形态在此类统一
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — "脚本=显式编译字节码"的实现点

## 引用来源

- [[Wiki/Sources/Stock/NiagaraScript]](`NiagaraScript.h` @ `b6ab0dee9`)

## 开放问题 / 矛盾

- `DIParamInfo` 在 `FNiagaraVMExecutableData` 里但注释作者吐槽"GPU 信息不该在 VM 数据里"——积压的技术债,Phase 7/8 跟进
- editor/cooked 下三份 ExecutionParamStore(CPU transient / GPU transient / cooked)的切换逻辑在 `GetExecutionReadyParameterStore` 里,Phase 5 细看
- `ProcessSerializedShaderMaps` 与 `FNiagaraShaderMap` 的协作关系是 Phase 8 的核心,此处只记清楚入口
