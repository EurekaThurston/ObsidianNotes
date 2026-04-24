---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, parameter-store, vm, padding]
sources: 1
aliases: [NiagaraScriptExecutionParameterStore.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraScriptExecutionParameterStore.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScriptExecutionParameterStore.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraScriptExecutionParameterStore.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 195 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 5 — CPU 脚本执行 5.4

## 职责

`FNiagaraParameterStore` 的两个**执行期特化子类**:
- `FNiagaraScriptExecutionParameterStore` — 绑在 `UNiagaraScript` 上,script 编译产物的参数布局
- `FNiagaraScriptInstanceParameterStore` — 绑在执行上下文上,引用 script 那份作为 layout 源

两者合作:**script 层定义布局一次,每个 execution context 引用它作为实例参数容器**。

## `FNiagaraScriptExecutionPaddingInfo`(L11)

USTRUCT,(SrcOffset, DestOffset, SrcSize, DestSize) 4 个 uint16。描述 CPU 里参数布局与 **GPU constant buffer 对齐布局** 的映射。

为什么要 padding?GPU constant buffer 按 HLSL 规则(vec4 对齐)组织,CPU 侧按紧凑 byte 布局——两套不同,要一份 mapping 表。

## `FNiagaraScriptExecutionParameterStore`(L32)

```cpp
USTRUCT()
struct FNiagaraScriptExecutionParameterStore : public FNiagaraParameterStore
```

字段:

```cpp
int32 ParameterSize;                                  // 不含 prev/internal 的参数数据大小
uint32 PaddedParameterSize;                           // 按 GPU 对齐填充后的大小
TArray<FNiagaraScriptExecutionPaddingInfo> PaddingInfo;  // CPU→GPU 布局映射表
uint8 bInitialized : 1;

#if WITH_EDITORONLY_DATA
TArray<uint8> CachedScriptLiterals;                   // 编译产生的字面量(给 VM 做 const)
#endif
```

**禁用的基类方法**:`RemoveParameter / RenameParameter` 都 `check(0)` —— 因为会改变 offset,破坏 VM 生成的绑定。

Editor-only 初始化:
- `InitFromOwningScript(UNiagaraScript*, SimTarget, bNotifyAsDirty)` — 从 script 编译产物建 layout
- `AddScriptParams(UNiagaraScript*, SimTarget, bTriggerRebind)` — 批量加入 script 定义的参数
- `CoalescePaddingInfo()` — 合并相邻 padding 条目(内存优化)

保护接口:
- `AddPaddedParamSize(const FNiagaraTypeDefinition&, uint32 InOffset)` — 加一个参数并生成对应的 padding info
- `AddAlignmentPadding()` — 在末尾加 16-byte 对齐 pad

## `FNiagaraScriptInstanceParameterStore`(L134)

```cpp
USTRUCT()
struct FNiagaraScriptInstanceParameterStore : public FNiagaraParameterStore
```

**执行上下文**持有的参数存储(Phase 3 `FNiagaraScriptExecutionContextBase::Parameters` 就是这个)。设计特点:**layout 来自另一个 store**,本 store 只是填数据。

```cpp
private:
    FNiagaraCompiledDataReference<FNiagaraScriptExecutionParameterStore> ScriptParameterStore;
    uint8 bInitialized : 1;
```

`FNiagaraCompiledDataReference` 是 Niagara 的"共享 layout,局部 instance"模式——layout 数据只一份,多个 instance 引用。

关键方法:

```cpp
void InitFromOwningContext(UNiagaraScript* Script, ENiagaraSimTarget SimTarget, bool bNotifyAsDirty);
void CopyCurrToPrev();                                // Interpolated spawn 用(当前参数存一份给下帧作 "Previous.*")
uint32 GetExternalParameterSize() const;
uint32 GetPaddedParameterSizeInBytes() const;
void CopyParameterDataToPaddedBuffer(uint8* InTargetBuffer, uint32 InTargetBufferSizeInBytes) const;
                                                      // 按 PaddingInfo 把 CPU 数据写进 GPU-ready buffer
TArrayView<const FNiagaraVariableWithOffset> ReadParameterVariables() const override;
                                                      // 重写父类接口,从 ScriptParameterStore 读变量
```

**禁用的**:`AddParameter / RemoveParameter / RenameParameter / Empty / Reset` 全部 `check(0)` —— 只接 layout,不能自己改。

## 关键机制:CopyCurrToPrev(Interpolated Spawn)

粒子可以 "在帧内中间时间点 spawn"(`FNiagaraSpawnInfo::InterpStartDt`)。为此脚本里能读 `Previous.*` 命名空间——上帧的参数值。`CopyCurrToPrev` 在每 tick 末把当前值 copy 到 prev 槽位,下 tick 就能读到。

这解释了 `FNiagaraScriptExecutionContextBase::HasInterpolationParameters : 1`——只有这种脚本才有 CopyCurrToPrev 开销。

## 关键机制:CopyParameterDataToPaddedBuffer

GT CPU 侧用紧凑布局,GPU constant buffer 要 16-byte 对齐。`CopyParameterDataToPaddedBuffer` 按 `PaddingInfo` 把参数从紧凑写到 padded buffer —— 准备上传给 GPU。

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraScriptExecutionParameterStore]] / [[Wiki/Entities/Stock/Niagara/FNiagaraScriptInstanceParameterStore]](合并一页)
- [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]] — 父类
- [[Wiki/Entities/Stock/Niagara/UNiagaraScript]] — layout 源

## 开放问题

- `PaddingInfo` 生成算法(编译器侧) → `.cpp`
- "Prev frame values" 的具体存储位置(buffer 末尾 offset?)→ `GetExternalParameterSize` 名字暗示只包括 external 不包括 internal/prev
