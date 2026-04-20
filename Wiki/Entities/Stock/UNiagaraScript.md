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

> **编译后的脚本资产**。不是源代码,而是 VectorVM 字节码 / GPU compute shader 的容器,由 `Usage` 决定角色。

## 一句话角色

`UNiagaraScript : public UNiagaraScriptBase`,`UCLASS(MinimalAPI)`。Niagara 里"能跑的东西"几乎都是 `UNiagaraScript`——System 脚本、粒子脚本、GPU 合并脚本、模块素材都共用此类。**同一对象同时承载 CPU 字节码(`CachedScriptVM`)与 GPU shader 资源(`ScriptResource`)**,按 Emitter `SimTarget` 分发。

> ⚠️ **关键陷阱**:`Usage == EmitterSpawnScript / EmitterUpdateScript` 的 `IsCompilable()` 返 false——这两种脚本**不独立编译**,被合并进 SystemSpawn/Update 脚本。

## 核心字段速查

**角色与标识**

| 字段 | 类型 | 作用 |
|---|---|---|
| `Usage` | `ENiagaraScriptUsage` | 角色枚举(Particle/Emitter/System × Spawn/Update,+Module/Function/DynamicInput/GPU/SimStage) |
| `UsageId` | `FGuid` | 同 Usage 多脚本时区分(如多个 EventHandler) |
| `ModuleUsageBitmask`(editor) | Module 脚本允许的 Usage 上下文 |

**编译产物**

| 字段 | 类型 | 作用 |
|---|---|---|
| `CachedScriptVM` | `FNiagaraVMExecutableData` | CPU 字节码 + 属性 + DI 绑定 + 外部函数表 |
| `CachedScriptVMId` | `FNiagaraVMExecutableDataId` | 身份证 + DDC key |
| `ScriptResource` | `TUniquePtr<FNiagaraShaderScript>` | GPU shader 资源(Phase 8) |
| `ScriptShader` | `FComputeShaderRHIRef` | RHI 句柄 |
| `ScriptExecutionParamStore` + `ScriptExecutionBoundParameters` | cooked 平台执行绑定 |

**参数**

- `RapidIterationParameters`(`FNiagaraParameterStore`)— UI 可调节,不触发重编译;`bBakeOutRapidIteration` 开启时烘死

**Source(editor-only)**

- `Source : UNiagaraScriptSourceBase*` — 指向图源抽象基类

常用方法:
- Usage 查询:`IsParticleSpawnScript / IsSystemUpdateScript / IsGPUScript / IsCompilable / IsInterpolatedParticleSpawnScript ...`(含静态版)
- 依赖:`IsUsageDependentOn(A, B)` / `IsEquivalentUsage` / `ConvertUsageToGroup`
- 编译:`ComputeVMCompilationId / RequestCompile / SetVMCompilationResults / AreScriptAndSourceSynchronized`
- DDC:`BuildNiagaraDDCKeyString / GetNiagaraDDCKeyString / BinaryToExecData / ExecToBinaryData`
- 运行时:`GetVMExecutableData / GetRenderThreadScript / IsReadyToRun(SimTarget) / CanBeRunOnGpu`
- 委托:`OnVMScriptCompiled / OnGPUScriptCompiled / OnPropertyChanged`

## 相关

- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] — `Source` 类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — 持有 System 脚本
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 持有粒子 / GPU / EventHandler 脚本
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU(VectorVM)/GPU(Compute Shader)双形态在此类统一
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — "显式编译字节码" 是相对 Cascade 的质变
- [[Wiki/Concepts/UE4/UE4-ddc|DDC]] — `CachedScriptVMId` 直接作为 DDC key,团队编译产物缓存的入口

## 深入阅读

- 全字段清单 + 编译产物三件套详解:[[Wiki/Sources/Stock/NiagaraScript]]
- 主题读本(推荐初读):[[Readers/Niagara/Phase1-asset-layer-读本]] § 4

## 开放问题

- `CachedScriptVM.OptimizedByteCode` 的"平台优化"具体做了什么?→ Phase 5
- 三种参数存储(`User.*` / `RapidIteration.*` / `Module.*`)的完整协作?→ Phase 5
- `FNiagaraVMExecutableData::DIParamInfo` 的技术债(GPU 信息不该在 VM 数据)→ Phase 7/8
- `SynchronizeExecutablesWithMaster` 的 merge 快路径实现?→ 编辑器加餐
- `ReleasedByRT` 与 `ScriptResource` 的线程安全配合?→ Phase 8
