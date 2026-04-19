---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, class, asset, uclass]
sources: 1
aliases: [UNiagaraSystem, NiagaraSystem, Niagara System 资产]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraSystem.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraSystem

> Niagara 特效的**顶层资产类**。一个 `.uasset` 文件 = 一个 UNiagaraSystem = 若干 Emitter + System 脚本 + 编译产物。

## 概览

`UNiagaraSystem : public UFXSystemAsset`。它是玩家在内容浏览器里看到的那个 "Niagara System" 资产,定义一个完整特效的**静态描述**——包含哪些 Emitter、Emitter 的执行顺序、System 级的 Spawn/Update 脚本、暴露给外部的用户参数(User Parameters)、固定包围盒、预热配置、scalability 归属(`EffectType`)等。

它是 Niagara 三层架构里的 **Asset 层顶点**。运行时由 `UNiagaraComponent` 引用,实例化为 `FNiagaraSystemInstance`(Phase 3)。

## 关键事实 / 属性

### 核心数据成员

| 字段 | 类型 | 作用 |
|---|---|---|
| `EmitterHandles` | `TArray<FNiagaraEmitterHandle>` | System 持有的所有 Emitter 引用;handle 里含启用状态和唯一 Id |
| `SystemSpawnScript` | `UNiagaraScript*` | System 首次激活时执行一次 |
| `SystemUpdateScript` | `UNiagaraScript*` | 每帧 tick 执行 |
| `ExposedParameters` | `FNiagaraUserRedirectionParameterStore` | 暴露给 BP/Sequencer 的 "User.*" 命名空间参数 |
| `EffectType` | `UNiagaraEffectType*` | scalability 预算归属(Phase 9) |
| `ParameterCollectionOverrides` | `TArray<UNiagaraParameterCollectionInstance*>` | 覆盖全局 Parameter Collection |
| `FixedBounds` | `FBox` | 启用 `bFixedBounds` 时跳过每帧包围盒重算 |
| `WarmupTime / WarmupTickCount / WarmupTickDelta` | 预热:生成时快进 N 步再展示 |

### 编译产物(运行时只读,编辑器填充)

| 字段 | 作用 |
|---|---|
| `EmitterCompiledData`(`TArray<TSharedRef<const FNiagaraEmitterCompiledData>>`) | 每 Emitter 的 DataSet 布局 + 绑定变量 |
| `SystemCompiledData`(`FNiagaraSystemCompiledData`) | System 级 DataSet + 四类参数绑定(Global/System/Owner/Emitter) |
| `EmitterExecutionOrder`(`TArray<FNiagaraEmitterExecutionIndex>`) | 依赖求解后的 tick 顺序;高位用来标记 parallel overlap 边界 |
| `RendererDrawOrder`(`TArray<int32>`) | 渲染器绘制顺序 |

### 开关(`UPROPERTY`,可在资产里编辑)

- `bFixedBounds`:开启固定包围盒
- `bAutoDeactivate`:所有 emitter 不再 spawn 就自动停用
- `bRequireCurrentFrameData`:true=按依赖排 TickGroup,false=尽早执行(性能换准确)
- `bDumpDebugSystemInfo / bDumpDebugEmitterInfo`:调试
- `bBakeOutRapidIteration*`、`bCompressAttributes`、`bTrimAttributes*`(仅 editor-only):性能调优
- `bOverrideScalabilitySettings`:用 `SystemScalabilityOverrides` 覆盖 EffectType 的默认 scalability

### 常用方法

- `GetEmitterHandles() / GetEmitterHandle(int)` — 取 Emitter
- `GetSystemSpawnScript() / GetSystemUpdateScript()` — 取系统脚本
- `ForEachScript(TAction)` — 遍历 System + 所有 Emitter 的所有脚本(模板)
- `IsValid() / IsReadyToRun()` — 资产可用性检查
- `ComputeEmittersExecutionOrder() / ComputeRenderersDrawOrder()` — 依赖求解(编辑器里发生)
- `CacheFromCompiledData()` — 把编译数据展开成运行时 accessor
- `AddEmitterHandle / DuplicateEmitterHandle / RemoveEmitterHandle`(editor-only)

## 相关

- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — `EmitterHandles` 的元素类型,持有 `UNiagaraEmitter*` + 启用状态
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 被 handle 引用的 Emitter 资产
- [[Wiki/Entities/Stock/UNiagaraScript]] — SystemSpawnScript / SystemUpdateScript 的类型
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 本类是典型 Asset;Instance 是 `FNiagaraSystemInstance`(Phase 3)
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — `UFXSystemAsset` 基类由 Cascade 时代延续

## 引用来源

- [[Wiki/Sources/Stock/NiagaraSystem]](`NiagaraSystem.h` @ `b6ab0dee9`)

## 开放问题 / 矛盾

- `bHasDIsWithPostSimulateTick` / `bHasAnyGPUEmitters` / `bNeedsSortedSignificanceCull` 这几个 cached 位用来做什么优化决策?Phase 3/7/9 再回看
- `ActiveInstances` / `ActiveInstancesTemp` 似乎是 scalability 计数用,但维护点(增减)不在此文件;需追踪到 `FNiagaraSystemInstance` 的注册逻辑
