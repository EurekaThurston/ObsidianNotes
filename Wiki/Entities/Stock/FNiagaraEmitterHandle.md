---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, ustruct, asset, handle]
sources: 1
aliases: [FNiagaraEmitterHandle, NiagaraEmitterHandle, Emitter Handle]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterHandle.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraEmitterHandle

> System 引用 Emitter 的**包装器 struct**。把 Emitter 指针和"System 对它的使用方式"(唯一 Id / 显示名 / 启用状态)打包在一起。

## 概览

`USTRUCT() struct NIAGARA_API FNiagaraEmitterHandle`。`UNiagaraSystem` 的核心状态 `TArray<FNiagaraEmitterHandle> EmitterHandles` 的元素类型。

为什么不直接用 `TArray<UNiagaraEmitter*>`?三条理由:

1. **同一父 Emitter 可被 System 引用多次**,需要 `FGuid` 稳定区分
2. **启用状态属于"System 对 Emitter 的使用方式"**,不属于 Emitter 本身
3. **Name 是 System 内显示名**,独立于 Emitter 资产名(也是 `Particles.*` 命名空间的 key)

> ⚠️ **命名陷阱**:此 struct 里的 `Instance` 字段指的是 "Emitter 资产副本"(还是 Asset 层),**不是**运行时粒子模拟实例(那叫 `FNiagaraEmitterInstance`,Phase 3)。

## 关键事实 / 属性

### 字段

| 字段 | 类型 | 用途 |
|---|---|---|
| `Id` | `FGuid` | 稳定唯一 key |
| `IdName` | `FName` | HACK:DataSet 历史上以 Name 查,但 Name 不唯一,此字段是临时稳定 key |
| `bIsEnabled` | `bool` | 禁用时此 Emitter 不 simulate |
| `Name` | `FName` | 在所属 System 内的显示名;也是 Particles 命名空间前缀 |
| `Instance` | `UNiagaraEmitter*` | 指向的 Emitter 资产副本 |
| `Source_DEPRECATED / LastMergedSource_DEPRECATED` | `UNiagaraEmitter*` | 已废弃;merge 源跟踪现在挂在 `UNiagaraEmitter::Parent / ParentAtLastMerge` |
| `bIsolated` | `bool`(非 UPROPERTY) | editor-only 的 Solo 模式 |

### 关键方法

- 查询:`IsValid / GetId / GetIdName / GetName / GetIsEnabled / GetInstance / GetUniqueInstanceName`
- 修改(需要 OwnerSystem 保证唯一性):`SetName(FName, UNiagaraSystem&)`、`SetIsEnabled(bool, UNiagaraSystem&, bRecompile)`
- editor-only:`IsIsolated / SetIsolated / NeedsRecompile / ConditionalPostLoad / UsesEmitter / ClearEmitter`

### 静态

- `FNiagaraEmitterHandle::InvalidHandle` — 一个全局无效 handle,比较用

### 构造语义

- 默认构造 → invalid
- editor 构造 `FNiagaraEmitterHandle(UNiagaraEmitter&)` → 包装传入 emitter;**但** System 层面的 `AddEmitterHandle(UNiagaraEmitter& Source, FName)` 会先 duplicate Source,handle 拿到的是副本(注释自述)

## 相关

- [[Wiki/Entities/Stock/UNiagaraSystem]] — 持有 `TArray<FNiagaraEmitterHandle>` 的容器
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — `Instance` 字段的类型;声明 `friend struct FNiagaraEmitterHandle`
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — USTRUCT + UPROPERTY 的反射语义
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 注意 handle 的 `Instance` 仍然在 Asset 层,不是运行时 Instance

## 引用来源

- [[Wiki/Sources/Stock/NiagaraEmitterHandle]](`NiagaraEmitterHandle.h` @ `b6ab0dee9`)

## 开放问题 / 矛盾

- `IdName` 的 HACK 注释指出 DataSet 曾经用 Name 查;待 Phase 4 在 `NiagaraDataSet.h` / `FNiagaraDataSetID` 验证具体是什么 key
- Niagara 里有**两层** "Emitter 的 Instance" 概念:
  - handle 里的 `Instance` = Emitter 资产副本(Asset)
  - `FNiagaraEmitterInstance` = 运行时粒子模拟(Instance,Phase 3)
  - 两者独立,别混
