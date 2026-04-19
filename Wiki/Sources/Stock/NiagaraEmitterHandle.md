---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, source-code, asset, handle]
sources: 1
aliases: [NiagaraEmitterHandle.h, FNiagaraEmitterHandle 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterHandle.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraEmitterHandle.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraEmitterHandle.h`
- **快照**: commit `b6ab0dee9`
- **同仓协作文件**: `NiagaraEmitterHandle.cpp`、`NiagaraSystem.h / .cpp`(持有 `TArray<FNiagaraEmitterHandle>`)、`NiagaraEmitter.h`(`friend` 关系)
- **Ingest 日期**: 2026-04-19
- **学习阶段**: Phase 1 — 资产层 1.3

## 职责 / 这个文件干什么的

定义 `FNiagaraEmitterHandle`——一个小型 `USTRUCT`,是 **System 引用 Emitter 的"包装器"**。它持有 `UNiagaraEmitter* Instance` 指针、一个唯一 `FGuid Id`、在 System 内的 `Name` 显示名、启用开关 `bIsEnabled`。

为什么不直接让 `UNiagaraSystem::EmitterHandles` 类型是 `TArray<UNiagaraEmitter*>`?因为:

1. **唯一 Id**:同一个 "父 Emitter" 可以被一个 System 引用多次(例如复制出两个独立的火花 Emitter),需要 Guid 区分
2. **启用状态**:`bIsEnabled` 属于 "System 对此 Emitter 的使用方式",不属于 Emitter 本身
3. **Name**:在 System 内的显示名不等于 Emitter 资产名(Emitter 可以是嵌入在 System 里的匿名实例)
4. **isolation**:editor 里的 "Solo" 模式,属于 handle 级别的 UI 状态
5. **merge source 记录**:有 `Source_DEPRECATED` / `LastMergedSource_DEPRECATED` 字段(已废弃),说明 merge 源跟踪历史上挂在 handle 上

这是典型的"Handle 模式":指针 + 与上下文关联的元数据。

## 关键类型 / 函数 / 宏

### 主结构体

- `FNiagaraEmitterHandle`(L19),`USTRUCT()` + `NIAGARA_API`

### 字段

| 字段 | 类型 | 可见性 | 作用 |
|---|---|---|---|
| `Id` | `FGuid` | `UPROPERTY(VisibleAnywhere)` | 唯一 Id,让同一父 Emitter 的多个副本可区分 |
| `IdName` | `FName` | `UPROPERTY(VisibleAnywhere)` | **HACK**:历史原因 DataSet 用 Name 来查,而 Name 不唯一;临时以此作为稳定 key |
| `bIsEnabled` | `bool` | `UPROPERTY()` | 禁用时该 Emitter 不 simulate |
| `Name` | `FName` | `UPROPERTY()` | System 内的显示名(用于 Particles/Parameters 命名空间前缀) |
| `Instance` | `UNiagaraEmitter*` | `UPROPERTY()` | handle 指向的 Emitter 实例(注意是 Asset 意义上的 instance,不是运行时 Instance) |
| `Source_DEPRECATED` | `UNiagaraEmitter*` | (editor-only)已废弃的 merge 源 |
| `LastMergedSource_DEPRECATED` | `UNiagaraEmitter*` | (editor-only)已废弃 |
| `bIsolated` | `bool` | (editor-only)"Solo" 模式;不序列化 |

### 接口(只读)

- `IsValid()`
- `GetId() / GetIdName() / GetName()`
- `GetIsEnabled()`
- `GetInstance()` — 最常用,取 `UNiagaraEmitter*`
- `GetUniqueInstanceName()` — 用于脚本/参数存储的稳定名

### 接口(修改)

- `SetName(FName, UNiagaraSystem&)` — 需要 OwnerSystem 是因为要保证全 System 内 Name 唯一
- `SetIsEnabled(bInEnabled, OwnerSystem, bRecompileIfChanged)` — 改启用状态可能触发重编译
- `SetIsolated / IsIsolated`(editor-only)

### 编辑器功能

- `NeedsRecompile()` — 图是否和脚本不同步
- `ConditionalPostLoad(int32 NiagaraCustomVersion)` — 条件 post-load
- `UsesEmitter(const UNiagaraEmitter&)` — 判断此 handle 是否用到某个 Emitter(考虑继承链)
- `ClearEmitter()`

### 构造

- 默认构造:无效 handle(`InvalidHandle` 静态)
- `FNiagaraEmitterHandle(UNiagaraEmitter& InEmitter)`(editor-only):**不会修改传入 Emitter**,只持有它。从代码看,实际上 System 层面会 duplicate Emitter 再做 handle(见 `UNiagaraSystem::AddEmitterHandle`)

## 关键代码片段

> `NiagaraEmitterHandle.h:L15-L29` @ `b6ab0dee9` — 结构体头 + 构造
> ```cpp
> /**
>  * Stores emitter information within the context of a System.
>  */
> USTRUCT()
> struct NIAGARA_API FNiagaraEmitterHandle
> {
>     GENERATED_USTRUCT_BODY()
> public:
>     FNiagaraEmitterHandle();
> #if WITH_EDITORONLY_DATA
>     FNiagaraEmitterHandle(UNiagaraEmitter& InEmitter);
> #endif
> ```

> `NiagaraEmitterHandle.h:L81-L113` @ `b6ab0dee9` — 私有字段全貌
> ```cpp
> private:
>     UPROPERTY(VisibleAnywhere, Category="Emitter ID")
>     FGuid Id;
>
>     UPROPERTY(VisibleAnywhere, Category = "Emitter ID")
>     FName IdName;
>
>     UPROPERTY()
>     bool bIsEnabled;
>
>     UPROPERTY()
>     FName Name;
>
> #if WITH_EDITORONLY_DATA
>     UPROPERTY()
>     UNiagaraEmitter* Source_DEPRECATED;
>     UPROPERTY()
>     UNiagaraEmitter* LastMergedSource_DEPRECATED;
>     bool bIsolated;
> #endif
>
>     UPROPERTY()
>     UNiagaraEmitter* Instance;
> ```

## 依赖与被依赖

**上游:**
- `NiagaraModule.h`
- 前向声明 `UNiagaraSystem` / `UNiagaraEmitter`(.cpp 里才真正 include)

**下游:**
- `UNiagaraSystem`:`TArray<FNiagaraEmitterHandle> EmitterHandles` 是 System 的核心状态
- `UNiagaraEmitter`:声明 `friend struct FNiagaraEmitterHandle`,允许 handle 操纵 Emitter 内部(主要是 merge 场景)
- `FNiagaraEmitterInstance`(Phase 3):运行时按 handle 初始化,使用 handle 的 Id/Name 作为命名空间前缀
- 编辑器 Stack UI:大部分"Emitter 级"操作(重命名、启用、isolate、solo)最终都打到 handle 上

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]] — 本结构体
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — `Instance` 指向的类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — 容器
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — `USTRUCT + GENERATED_USTRUCT_BODY` + `UPROPERTY` 的典型非 UObject 反射结构
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 注意此处的 "Instance" 指 Emitter 资产副本,不是运行时粒子模拟实例

## 与既有 wiki 的关系

- 印证了 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 的 "`FNiagaraEmitterHandle` 是 System 对 Emitter 的引用包装,含启用/禁用状态" 描述
- **歧义澄清**:`FNiagaraEmitterHandle::Instance` 的命名非常误导——它是"该 handle 指向的 Emitter 资产副本",不是 [[Wiki/Concepts/UE4/UE4-资产与实例]] 里的"运行时 Instance"。运行时 Instance 叫 `FNiagaraEmitterInstance`,在 Phase 3。这个命名冲突后续读代码时要一直记住

## 开放问题 / 待深入

- `IdName` 注释里的 HACK——"DataSet used to use emitter name":具体指 DataSet 的哪个 key?是不是 `FNiagaraDataSetID`?Phase 4 读 `NiagaraDataSet.h` 时验证
- `Source_DEPRECATED / LastMergedSource_DEPRECATED`:被废弃后 merge 源跟踪迁移到了哪里?看 `UNiagaraEmitter::Parent / ParentAtLastMerge` —— 结论:**迁到 Emitter 本身了**。已验证。
- `FNiagaraEmitterHandle(UNiagaraEmitter&)` 构造:注释没说它是否 deep copy `InEmitter`。看 `UNiagaraSystem::AddEmitterHandle(UNiagaraEmitter& SourceEmitter, FName EmitterName)` 的注释:"New handle exposes an Instance value which is a copy of the original asset" → **是 copy**。
- `bIsolated` 不序列化,但没明显的 transient 标记?—— 它整个字段没有 `UPROPERTY`,所以 UE 反射完全不知道它,自然不序列化。符合 "UI-only state" 的设计
