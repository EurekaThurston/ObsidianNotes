---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, parameter-store, data-model, runtime]
sources: 1
aliases: [NiagaraParameterStore.h, Niagara 参数存储源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraParameterStore.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraParameterStore.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraParameterStore.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 1260 行(含大量内联 FORCEINLINE)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.7

## 职责

定义 **`FNiagaraParameterStore`**——Niagara 运行时的正牌**参数存储**。这是:

- Phase 2 `UNiagaraComponent::OverrideParameters`(继承 `FNiagaraUserRedirectionParameterStore`)的基类
- Phase 3 `FNiagaraSystemInstance::InstanceParameters`、`FNiagaraEmitterInstance::RendererBindings` 等的类型
- Phase 3 `FNiagaraParameterStoreToDataSetBinding` 的数据源
- 所有 System/Emitter/Component 级**参数和 DataInterface 引用的统一容器**

关键能力:
1. 同时存**参数数据**(byte array)+ **DataInterface 对象**(`TArray<UNiagaraDataInterface*>`)+ **UObject 引用**(`TArray<UObject*>`)三种
2. Store 之间的**绑定图**(A → B 自动 push)
3. **dirty 脏标记**驱动的按需同步

## 配套结构体

### `FNiagaraBoundParameter`(USTRUCT, L15)

外部明确绑定点:`Parameter` + `SrcOffset` + `DestOffset`。

### `FNiagaraParameterStoreBinding`(非 USTRUCT, L32)

**两个 store 之间的绑定关系**。一个 binding 可以包含三种小 binding:

```cpp
// FParameterBinding: 普通参数 byte-to-byte(16-bit offset 压缩)
TArray<FParameterBinding> ParameterBindings;  // SrcOffset, DestOffset, Size

// FInterfaceBinding: DataInterface 索引映射
TArray<FInterfaceBinding> InterfaceBindings;  // SrcOffset, DestOffset

// FUObjectBinding: UObject 索引映射
TArray<FUObjectBinding> UObjectBindings;
```

核心方法:
- `Initialize(Dest, Src, BoundParameters)`:建立绑定
- `Tick(Dest, Src, bForce=false)`:按绑定从 Src 推参数到 Dest
- `VerifyBinding` / `Dump`
- TODO 注释:"Merge contiguous ranges into a single binding?"(合并相邻 byte range 优化)

### `FNiagaraVariableWithOffset`(USTRUCT, L117)

`FNiagaraVariableBase`(TypeDef + Name)+ `int32 Offset`。用于 `SortedParameterOffsets` 数组。

## 主类:`FNiagaraParameterStore`(USTRUCT, L149)

```cpp
// NiagaraParameterStore.h:149
USTRUCT()
struct NIAGARA_API FNiagaraParameterStore
{
    // ...
};
```

### 核心字段

```cpp
// Owner(给 DI 的 outer)
UPROPERTY(Transient)
UObject* Owner;

#if WITH_EDITORONLY_DATA
TMap<FNiagaraVariable, int32> ParameterOffsets;  // editor 用的 map,运行时走下面
#endif

UPROPERTY()
TArray<FNiagaraVariableWithOffset> SortedParameterOffsets;  // 运行时排序后二分查找

UPROPERTY()
TArray<uint8> ParameterData;                     // 参数 byte blob

UPROPERTY()
TArray<UNiagaraDataInterface*> DataInterfaces;   // DI 对象列表

UPROPERTY()
TArray<UObject*> UObjects;                       // 其他 UObject 类型参数

// 绑定图
typedef TPair<FNiagaraParameterStore*, FNiagaraParameterStoreBinding> BindingPair;
TArray<BindingPair> Bindings;                    // 我 → 目标 store 的 push
TArray<FNiagaraParameterStore*> SourceStores;    // 谁 push 数据到我

// 脏标记(分三路)
uint32 bParametersDirty : 1;
uint32 bInterfacesDirty : 1;
uint32 bUObjectsDirty : 1;

uint32 LayoutVersion;  // layout 变化的版本号,绑定需重建
```

### 关键 API

**参数增删**:
- `virtual bool AddParameter(const FNiagaraVariable&, bool bInitialize=true, bool bTriggerRebind=true, int32* OutOffset=nullptr)`
- `virtual bool RemoveParameter(const FNiagaraVariableBase&)`
- `virtual void RenameParameter(const FNiagaraVariableBase&, FName NewName)`
- `virtual void Empty(bool bClearBindings=true)` / `Reset(bool bClearBindings=true)`

**查找**:
- `int32 IndexOf(const FNiagaraVariable&)` — 在 SortedParameterOffsets 查
- `virtual const int32* FindParameterOffset(const FNiagaraVariableBase&, bool IgnoreType=false)`

**读值**:
- 模板 `void GetParameterValue(T& OutValue, const FNiagaraVariable&)` — 通过 offset 直接 memcpy
- 模板 `T GetParameterValue(const FNiagaraVariable&)` — 返回值版
- `const uint8* GetParameterData(int32 Offset)` / `const uint8* GetParameterData(const FNiagaraVariable&)` — 拿 raw byte ptr
- `UNiagaraDataInterface* GetDataInterface(int32 Offset)` / `GetDataInterface(const FNiagaraVariable&)`
- `UObject* GetUObject(int32 Offset)` / `GetUObject(const FNiagaraVariable&)`
- `const FNiagaraVariableBase* FindVariable(const UNiagaraDataInterface*) const` — 反向

**绑定管理**:
- `Bind(FNiagaraParameterStore* Dest, const FNiagaraBoundParameterArray* = nullptr)` — 主动 push 到 Dest
- `Unbind(Dest)` / `UnbindAll()` / `UnbindFromSourceStores()`
- `Rebind()` — 布局变化后重建
- `TransferBindings(FNiagaraParameterStore& OtherStore)` — 迁移绑定(Instance transfer 用)
- `FORCEINLINE_DEBUGGABLE void Tick()` — 按 dirty 标记 push 参数到所有 bound stores
- `bool VerifyBinding(const FNiagaraParameterStore* InDestStore) const`

**状态**:
- `GetParametersDirty() / GetInterfacesDirty() / GetUObjectsDirty()`
- `MarkParametersDirty() / MarkInterfacesDirty() / MarkUObjectsDirty()`
- `uint32 GetLayoutVersion() const`

**迁移**:
- `virtual void InitFromSource(const FNiagaraParameterStore* SrcStore, bool bNotifyAsDirty)` — 整体复制

**数据接口相关**:
- `void SanityCheckData(bool bInitInterfaces=true)` — 数据一致性校验 + DI 初始化

**Editor-only**(L207-L220):
- `DECLARE_MULTICAST_DELEGATE(FOnChanged)` + `OnChangedDelegate`
- `FString DebugName`
- `AddConstantBuffer<BufferType>()` 模板 — 一次加一批参数(Phase 5 constant buffer 用)

### 持久化 / 排序

- `void PostLoad()` — 反序列化后补偿
- `void SortParameters()` — 排 SortedParameterOffsets

## 子类(不在本头文件,但引用)

- `FNiagaraUserRedirectionParameterStore`(在 `NiagaraUserRedirectionParameterStore.h`)— Phase 2 `UNiagaraComponent::OverrideParameters` 的类型,会自动为参数加 `User.` 前缀

## 绑定图的运行时作用

拿 Component → Instance → SystemSimulation 这条线举例:

```
UNiagaraComponent::OverrideParameters
    ↓ (Bind)
FNiagaraSystemInstance::InstanceParameters
    ↓ (Bind via FNiagaraParameterStoreBinding)
FNiagaraSystemSimulation::ScriptDefinedDataInterfaceParameters
    ↓ (Tick 时按 dirty 标记 push)
VM 脚本的 constant buffer
```

绑定一次建立,之后每 tick `Tick()` 按 dirty 增量 push。这就是 Phase 3 `FNiagaraParameterStoreToDataSetBinding` 最终落到 VM 的上游链路。

## 性能关键:三路 dirty

`bParametersDirty / bInterfacesDirty / bUObjectsDirty` 三个 bit 分开——只改了参数数据不需要重传 DI 指针,反之亦然。避免全量推送。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]] — 本文件主类
- [[Wiki/Entities/Stock/Niagara/FNiagaraVariable]] / [[Wiki/Entities/Stock/Niagara/FNiagaraVariableBase]] — 参数身份
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponent]] — 通过继承子类持有
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemInstance]] / [[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]] — 也持有

## 开放问题

- `FORCEINLINE_DEBUGGABLE` 大量使用的深意(debug 也能 inline?)→ UE 约定
- Store 之间 binding 图的生命周期管理 —— 谁负责 unbind?Dest Store 析构?→ 看 cpp
- Editor-only `TMap ParameterOffsets` 与运行时 `SortedParameterOffsets` 的数据同步时机 → cpp
- `TickBatch` 和 `Tick` 方法有关?→ 前者是 Phase 3 SystemSimulation 的,这里 Tick 是本 store 自己的
- DI 参数的 outer(`Owner` 字段)在 pool 复用时需要重置吗?→ Phase 9
