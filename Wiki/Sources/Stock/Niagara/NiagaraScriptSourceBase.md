---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, source-code, editor, script, source, abstract]
sources: 1
aliases: [NiagaraScriptSourceBase.h, UNiagaraScriptSourceBase 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptSourceBase.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScriptSourceBase.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptSourceBase.h`
- **快照**: commit `b6ab0dee9`
- **同仓协作文件**: `NiagaraGraph.h / UNiagaraGraph`(NiagaraEditor 模块里真正的图源实现)、`NiagaraScript.h`(持有 `Source` 指针)、`NiagaraEmitter.h`(持有 `GraphSource`)
- **Ingest 日期**: 2026-04-19
- **学习阶段**: Phase 1 — 资产层 1.5

## 职责 / 这个文件干什么的

定义 `UNiagaraScriptSourceBase`——**脚本图源的抽象基类**。它只存在于 editor(所有字段/方法都被 `#if WITH_EDITOR` 保护或者返回空/false 的默认实现),运行时是一个空壳。

为什么需要一个基类?**架构解耦**:
- `Niagara`(运行时模块)里的 `UNiagaraScript` / `UNiagaraEmitter` 需要"引用一个图源" —— 但运行时模块不应该知道"图"长什么样(那些是 `UEdGraph`、`UNiagaraGraph` 等 editor 类)
- 所以定义一个纯虚抽象 `UNiagaraScriptSourceBase`,声明所需的接口;**具体实现** `UNiagaraScriptSource` 在 `NiagaraEditor` 模块里继承并填实

这是典型的 "Abstract Base in Runtime Module, Impl in Editor Module" 模式。

此文件同时还定义了两个小型 helper:`FNiagaraCompileRequestDataBase`(编译请求数据接口)、`FNiagaraCompileOptions`(编译选项),以及两个裸 struct `EditorExposedVectorConstant` / `EditorExposedVectorCurveConstant`。

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraScriptSourceBase : public UObject`(L68),`UCLASS(MinimalAPI)`
  - 运行时此类几乎无用;editor 下子类 `UNiagaraScriptSource`(在 NiagaraEditor 模块)提供真正实现

### 抽象接口(全部 virtual,多数默认返回 false/nullptr/空 FGuid)

- `IsSynchronized(const FGuid& InChangeId)` — 图 change id 是否匹配
- `MarkNotSynchronized(FString Reason)` — 标记失同步
- `GetChangeID()` — 当前图的 change id
- `ComputeVMCompilationId(out Id, InUsage, InUsageId, bForceRebuild)` — 基于图源计算 VMID(配合 `UNiagaraScript::ComputeVMCompilationId` 委托用)
- `GetCompileBaseId(InUsage, InUsageId)` / `GetCompileHash(InUsage, InUsageId)` — 分别取基础 ID / 内容哈希
- `PreCompile(Emitter, EncounterableVars, ReferencedCompileRequests, bClearErrors)` — 编译前收集数据,返 `TSharedPtr<FNiagaraCompileRequestDataBase>`
- `PostLoadFromEmitter(Emitter&)` — Emitter 的 PostLoad 回调
- `AddModuleIfMissing(ModulePath, Usage, out bFoundModule)` — 图里添加模块(如果还没有),Stack UI 的"添加模块"走这里
- `MakeRecursiveDeepCopy(DestOuter, ExistingConversions)` — 深拷贝,用于 merge
- `SubsumeExternalDependencies(ExistingConversions)` — 把外部依赖吸进本包(打包资产时用)
- `CleanUpOldAndInitializeNewRapidIterationParameters(Emitter, Usage, UsageId, out Params)` — 清理旧的 RapidIterationParameter,初始化新的(editor-only)
- `ForceGraphToRecompileOnNextCheck()` — 强制下次检查时重编译
- `RefreshFromExternalChanges()` — 外部改动时刷新
- `CollectDataInterfaces(out DataInterfaces)` — 收集图里的 DI

### 数据成员

- `ExposedVectorConstants`(`TArray<TSharedPtr<EditorExposedVectorConstant>>`)
- `ExposedVectorCurveConstants`(`TArray<TSharedPtr<EditorExposedVectorCurveConstant>>`)
- `OnChangedDelegate`(`FOnChanged`,editor-only)

### 辅助类型

#### `FNiagaraCompileRequestDataBase`(L25)
编译请求的**纯虚接口**,运行时代码(`UNiagaraScript::RequestExternallyManagedAsyncCompile`)持有 `TSharedPtr<FNiagaraCompileRequestDataBase, ESPMode::ThreadSafe>`,但具体实现又是编辑器模块里给的。

虚方法:
- `GatherPreCompiledVariables(NamespaceFilter, out Vars)`
- `GetReferencedObjects(out Objects)`
- `GetObjectNameMap()` → `const TMap<FName, UNiagaraDataInterface*>&`
- `GetDependentRequestCount() / GetDependentRequest(Index)` — 递归编译请求(编译器是多 pass 的)
- `ResolveEmitterAlias(VariableName)` — 把 "Emitter.XXX" 里的 "Emitter" 替换成实际 Emitter 名
- `GetUseRapidIterationParams()`

#### `FNiagaraCompileOptions`(L38)
编译选项(**非** `USTRUCT`,纯 C++ struct):
- `TargetUsage / TargetUsageId`
- `PathName / FullName / Name`(用于日志/调试)
- `TargetUsageBitmask`(允许的 Usage 组合)
- `AdditionalDefines`(HLSL `#define` 注入,给 GPU 编译用)

#### `EditorExposedVectorConstant / EditorExposedVectorCurveConstant`
两个裸 struct,存 editor 暴露给用户的 Vector4 常量 / CurveVector 常量(`ConstName + Value` 对)。挂在 `UNiagaraScriptSourceBase` 的数组里。

### 关键代码片段

> `NiagaraScriptSourceBase.h:L66-L70` @ `b6ab0dee9` — 类头
> ```cpp
> /** Runtime data for a Niagara system */
> UCLASS(MinimalAPI)
> class UNiagaraScriptSourceBase : public UObject
> {
>     GENERATED_UCLASS_BODY()
> ```
> 注释里说 "Runtime data" 其实是误导——这个基类本身在运行时几乎没有功能,真正的 runtime data 在 `FNiagaraVMExecutableData`。注释可能是远古遗留。

> `NiagaraScriptSourceBase.h:L98-L108` @ `b6ab0dee9` — 两个关键编译接入点
> ```cpp
> virtual TSharedPtr<FNiagaraCompileRequestDataBase, ESPMode::ThreadSafe>
> PreCompile(UNiagaraEmitter* Emitter,
>            const TArray<FNiagaraVariable>& EncounterableVariables,
>            TArray<TSharedPtr<FNiagaraCompileRequestDataBase, ESPMode::ThreadSafe>>& ReferencedCompileRequests,
>            bool bClearErrors = true) { return nullptr; }
>
> virtual void PostLoadFromEmitter(UNiagaraEmitter& OwningEmitter) { }
>
> NIAGARA_API virtual bool AddModuleIfMissing(FString ModulePath, ENiagaraScriptUsage Usage, bool& bOutFoundModule) { bOutFoundModule = false; return false; }
> ```

## 依赖与被依赖

**上游:**
- `NiagaraCommon.h`(`ENiagaraScriptUsage`、`FNiagaraVariable` 等)
- `NiagaraCompileHash.h`(`FNiagaraCompileHash`)

**下游:**
- `UNiagaraScript::Source`(editor-only) — 每个 script 指向它
- `UNiagaraEmitter::GraphSource`(editor-only) — Emitter 的共享图源
- **实现类** `UNiagaraScriptSource` + `UNiagaraGraph`(在 `NiagaraEditor` 模块)
- 编译管线(`FHlslNiagaraTranslator` 等)—— 走 `PreCompile → 生成请求数据 → 编译器吞下`

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/Niagara/UNiagaraScriptSourceBase]] — 本文件定义的主类
- [[Wiki/Entities/Stock/Niagara/UNiagaraScript]] — 编译产物;其 `Source` 指向此类
- [[Wiki/Entities/Stock/Niagara/UNiagaraEmitter]] — `GraphSource` 指向此类
- [[Wiki/Concepts/UE/UE4-uobject-系统]] — 抽象基类 + editor/runtime 模块分离的典型模式

## 与既有 wiki 的关系

- 补充 [[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]]:Phase 1 完成后,可以清楚看到 **editor 侧图源 → 编译请求 → 字节码 / shader** 的接口边界
- 印证 [[Wiki/Concepts/UE/UE4-资产与实例]]:Asset 在 editor 下携带额外的 editor-only 信息(图源),cook 后这部分被剥离

## 开放问题 / 待深入

- **子类在哪个文件?** `UNiagaraScriptSource` 在 `NiagaraEditor/Private/NiagaraScriptSource.h / .cpp`,持有一个 `UNiagaraGraph* NodeGraph`。真正读就是 Phase 1.5 之外的扩展阅读
- **两条 ExposedVectorConstant 数组的作用?** 看起来是极老的"暴露常量"机制,很可能已被 RapidIterationParameters 取代;具体是否还有实际使用点需搜代码
- **`FNiagaraCompileRequestDataBase` 的实现类**:`FNiagaraCompileRequestData` / `FNiagaraCompileRequestDuplicateData` 之类,同样在 NiagaraEditor 模块。Phase 5 真正读编译路径时要追这里
- `FNiagaraCompileOptions::AdditionalDefines` 最终怎么进入 HLSL?编译器注入到 shader source 头?Phase 8 shader 编译路径验证
