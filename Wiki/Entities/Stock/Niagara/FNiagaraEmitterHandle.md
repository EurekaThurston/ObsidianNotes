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

> System 引用 Emitter 的**包装 struct**。把 Emitter 指针和"System 对它的使用方式"(Id / 显示名 / 启用状态)打包在一起。

## 一句话角色

`USTRUCT() struct NIAGARA_API FNiagaraEmitterHandle`。`UNiagaraSystem::EmitterHandles` 的元素类型。**存在的理由**:同一父 Emitter 可被 System 多次引用(需要 Guid 区分),启用状态/显示名/solo 属于"System 对 Emitter 的使用方式"而非 Emitter 本身。

> ⚠️ **命名陷阱**:此 struct 的 `Instance` 字段指 "Emitter 资产副本"(仍在 Asset 层),**不是**运行时粒子模拟实例(那叫 `FNiagaraEmitterInstance`,Phase 3)。

## 核心字段速查

| 字段 | 类型 | 作用 |
|---|---|---|
| `Id` | `FGuid` | 稳定唯一 key |
| `IdName` | `FName` | HACK:DataSet 历史上按 Name 查,Name 不唯一,此为临时稳定 key |
| `bIsEnabled` | `bool` | 禁用时此 Emitter 不 simulate |
| `Name` | `FName` | System 内显示名;也是 `Particles.*` 命名空间前缀 |
| `Instance` | `UNiagaraEmitter*` | Emitter 资产副本(⚠️ 见上) |
| `bIsolated`(editor-only,非 UPROPERTY) | Solo 模式 UI 状态 |
| `Source_DEPRECATED / LastMergedSource_DEPRECATED` | 废弃;merge 源跟踪迁到 `UNiagaraEmitter::Parent` |

常用方法:
- 查询:`IsValid / GetId / GetIdName / GetName / GetIsEnabled / GetInstance / GetUniqueInstanceName`
- 修改(需 OwnerSystem 保证唯一):`SetName(FName, UNiagaraSystem&)` / `SetIsEnabled(bool, UNiagaraSystem&, bRecompile)`
- editor-only:`IsIsolated / SetIsolated / NeedsRecompile / ConditionalPostLoad / UsesEmitter`
- 静态:`FNiagaraEmitterHandle::InvalidHandle`

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraSystem]] — 容器,持有 `TArray<FNiagaraEmitterHandle>`
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — `Instance` 指向类型,声明 `friend struct FNiagaraEmitterHandle`
- [[Wiki/Concepts/UE/UE4-uobject-系统]] — USTRUCT + UPROPERTY 的反射语义

## 深入阅读

- 全字段清单 + 设计理由:[[Wiki/Sources/Stock/Niagara/NiagaraEmitterHandle]]
- 主题读本(推荐初读):[[Readers/UE/Niagara/Phase 1 - 从 System 到图源抽象基类]] § 2

## 开放问题

- `IdName` HACK 里 "DataSet used to use emitter name" 的具体受影响点?→ Phase 4 读 `FNiagaraDataSetID` 时验证
