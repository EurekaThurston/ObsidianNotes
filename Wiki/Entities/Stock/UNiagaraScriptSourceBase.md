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

> 脚本**图源的抽象基类**。只在 editor 下有实质功能;运行时是空壳。是"runtime 模块能引用图源但不直接依赖图实现"的解耦点。

## 概览

`UNiagaraScriptSourceBase : public UObject`,`UCLASS(MinimalAPI)`。位于 `Niagara`(runtime)模块,但几乎所有方法都是虚方法 + 默认空实现。

真正的图源实现类:
- `UNiagaraScriptSource`(位于 `NiagaraEditor` 模块),内部持有一个 `UNiagaraGraph* NodeGraph`
- `UNiagaraGraph` 又继承自 UE 的 `UEdGraph`,承载节点 + 连线数据

这是典型的 **"Abstract Base in Runtime Module, Impl in Editor Module"** 模式:
- 运行时的 `UNiagaraScript::Source` 和 `UNiagaraEmitter::GraphSource` 能通过基类指针持有图源
- cooked 包里图源会被剥离(或保留但不使用)
- 编辑器启动时,实现类提供真正功能

## 关键事实 / 属性

### 角色

- 存储 editor 的**图源**(`UNiagaraGraph` 级别的节点+连线)
- 为编译系统提供**抽象入口**:`PreCompile / ComputeVMCompilationId / GetCompileHash` 等
- 为 Stack UI / Merge 等提供抽象服务:`AddModuleIfMissing / MakeRecursiveDeepCopy / RefreshFromExternalChanges`
- 承载少量 editor 状态:`ExposedVectorConstants / ExposedVectorCurveConstants / OnChangedDelegate`

### 抽象接口(精选)

| 方法 | 用途 |
|---|---|
| `GetChangeID()` | 图的当前 change id |
| `IsSynchronized(InChangeId)` | 与某 change id 是否匹配 |
| `ComputeVMCompilationId(out Id, Usage, UsageId, bForceRebuild)` | 计算 VMExecutableDataId |
| `GetCompileBaseId(Usage, UsageId) / GetCompileHash(Usage, UsageId)` | 分别取基础 Id / 内容哈希 |
| `PreCompile(Emitter, EncounterableVars, out ReferencedRequests, bClearErrors)` | 编译前收集数据并返回 `FNiagaraCompileRequestDataBase` 共享指针 |
| `PostLoadFromEmitter(Emitter&)` | Emitter PostLoad 期的回调 |
| `AddModuleIfMissing(Path, Usage, out bFound)` | Stack UI 的"添加模块" |
| `MakeRecursiveDeepCopy(DestOuter, ExistingConversions)` | Merge 场景的深拷贝 |
| `SubsumeExternalDependencies(ExistingConversions)` | 打包时把外部依赖纳入本包 |
| `CleanUpOldAndInitializeNewRapidIterationParameters(...)` | 编辑 RapidIterationParameters |
| `ForceGraphToRecompileOnNextCheck()` | 主动标记脏 |
| `RefreshFromExternalChanges()` | 响应外部变动 |
| `CollectDataInterfaces(out DIs)` | 收集图里用到的 DataInterface |

### 数据成员

- `ExposedVectorConstants`(`TArray<TSharedPtr<EditorExposedVectorConstant>>`)
- `ExposedVectorCurveConstants`(`TArray<TSharedPtr<EditorExposedVectorCurveConstant>>`)
- `OnChangedDelegate`(`FOnChanged`,editor-only,protected)

## 相关

- [[Wiki/Entities/Stock/UNiagaraScript]] — 每个 Script 持有 `UNiagaraScriptSourceBase* Source`(editor-only)
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 持有 `UNiagaraScriptSourceBase* GraphSource`(所有内部脚本共享的图)
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — "抽象基类 in runtime, impl in editor module" 的模式

## 引用来源

- [[Wiki/Sources/Stock/NiagaraScriptSourceBase]](`NiagaraScriptSourceBase.h` @ `b6ab0dee9`)

## 开放问题 / 矛盾

- 子类 `UNiagaraScriptSource` 在 `NiagaraEditor/Private/` 下,未来如果 Phase 1 扩展或 Phase 5 需要,可以把它独立 ingest
- `ExposedVectorConstants / ExposedVectorCurveConstants` 看起来很像早期的用户常量暴露机制,功能上很可能被 `RapidIterationParameters` / `User.*` 参数命名空间取代;实际使用点需 grep 验证
- 头文件顶部注释 "Runtime data for a Niagara system" 与此类的实际角色不符(实际上此类几乎不承载 runtime data)——标为文档过期,不是 bug
