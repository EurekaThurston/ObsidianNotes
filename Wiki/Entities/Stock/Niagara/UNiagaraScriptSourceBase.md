---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, class, uclass, editor, abstract, script, source]
sources: 1
aliases: [UNiagaraScriptSourceBase, NiagaraScriptSourceBase]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptSourceBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# UNiagaraScriptSourceBase

> 脚本**图源的抽象基类**。只在 editor 下有实质功能;runtime 是空壳。是 editor/runtime 模块解耦的接缝。

## 一句话角色

`UNiagaraScriptSourceBase : public UObject`,位于 `Niagara`(runtime)模块。所有方法是虚方法 + 默认空/nullptr 实现。真正实现类 `UNiagaraScriptSource` 在 `NiagaraEditor` 模块,内部持有 `UNiagaraGraph* NodeGraph`(继承自 `UEdGraph`)。

**存在的全部理由**:让 `UNiagaraScript::Source` 和 `UNiagaraEmitter::GraphSource` 能通过基类指针引用图源,而不依赖编辑器模块——典型的 "abstract in runtime, impl in editor" 解耦模式。

## 核心接口速查

**图状态 / 编译辅助(全部 virtual)**

- `GetChangeID()` — 图 change id
- `IsSynchronized(FGuid)` / `MarkNotSynchronized(FString)`
- `ComputeVMCompilationId(out Id, Usage, UsageId, bForceRebuild)`
- `GetCompileBaseId(Usage, UsageId)` / `GetCompileHash(Usage, UsageId)`
- `PreCompile(Emitter, EncounterableVars, out ReferencedRequests, bClearErrors)` → `TSharedPtr<FNiagaraCompileRequestDataBase>`
- `PostLoadFromEmitter(Emitter&)`

**编辑 / 维护(editor-only)**

- `AddModuleIfMissing(Path, Usage, out bFound)` — Stack UI 的"添加模块"
- `MakeRecursiveDeepCopy / SubsumeExternalDependencies` — merge / 打包
- `CleanUpOldAndInitializeNewRapidIterationParameters`
- `ForceGraphToRecompileOnNextCheck / RefreshFromExternalChanges`
- `CollectDataInterfaces(out)` — 收集图里的 DI

**数据成员**

- `ExposedVectorConstants / ExposedVectorCurveConstants`(`TArray<TSharedPtr<...>>`)— 老式常量暴露机制,很可能已被 RapidIterationParameters 取代
- `OnChangedDelegate`(editor-only,protected)

## 同文件辅助类型

- `FNiagaraCompileRequestDataBase` — 编译请求纯虚接口,具体实现也在 NiagaraEditor 模块
- `FNiagaraCompileOptions` — 编译选项(TargetUsage、HLSL `#define` 注入)

## 相关

- [[Wiki/Entities/Stock/Niagara/UNiagaraScript]] — `Source` 字段指向本类
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — `GraphSource` 字段指向本类
- [[Wiki/Concepts/UE/UE4-uobject-系统]] — "abstract base in runtime, impl in editor module" 典型模式

## 深入阅读

- 全接口清单 + 设计理由:[[Wiki/Sources/Stock/Niagara/NiagaraScriptSourceBase]]
- 主题读本(推荐初读):[[Readers/UE/Niagara/Phase 1 - 从 System 到图源抽象基类]] § 5

## 开放问题

- `ExposedVectorConstants / ExposedVectorCurveConstants` 是否还有实际使用点?→ grep 验证
- 子类 `UNiagaraScriptSource` + `UNiagaraGraph` 若需深入,属 Phase 1 之外的编辑器加餐
- 头部注释 "Runtime data for a Niagara system" 与此类实际角色不符,属文档过期(不是 bug)
