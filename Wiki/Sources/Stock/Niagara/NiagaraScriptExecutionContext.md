---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, script-execution, vm, context]
sources: 1
aliases: [NiagaraScriptExecutionContext.h, VM 执行上下文]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptExecutionContext.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScriptExecutionContext.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScriptExecutionContext.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 531 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 5 — CPU 脚本执行 5.3(**核心**)
- **派生 Entity**(见下文 4 大块):
  - 块 B(CPU VM) → [[Wiki/Entities/Stock/Niagara/FNiagaraScriptExecutionContext]]
  - 块 C(GPU 计算) → [[Wiki/Entities/Stock/Niagara/FNiagaraComputeExecutionContext]]
  - 块 D(GPU Tick 打包) → [[Wiki/Entities/Stock/Niagara/FNiagaraGPUSystemTick]]

## 职责

定义 Niagara 脚本执行的**运行时上下文**家族:CPU VM 执行上下文 + System 级脚本上下文 + GPU 计算执行上下文 + 打包给 RT 的 GPU Tick 结构。这是 Phase 3 `FNiagaraEmitterInstance::SpawnExecContext/UpdateExecContext/GPUExecContext` 的类型定义处。

CPU 侧(主题)一句话:**持有一段 UNiagaraScript 的运行时状态(外部函数表 + 参数存储 + DataSet 绑定),配合 VectorVM dispatch 字节码**。

## 文件结构 — 4 大块

| 块 | 类型 | 作用 |
|---|---|---|
| A | `FNiagaraEventHandlingInfo / FNiagaraDataSetExecutionInfo / FScriptExecutionConstantBufferTable` | 辅助数据结构 |
| B | `FNiagaraScriptExecutionContextBase / FNiagaraScriptExecutionContext / FNiagaraSystemScriptExecutionContext` | **CPU VM 执行上下文**(Phase 5 主题) |
| C | `FNiagaraComputeSharedContext / FNiagaraComputeExecutionContext` | GPU 执行上下文(Phase 8 展开) |
| D | `FNiagaraGpuSpawnInfo / FNiagaraComputeInstanceData / FNiagaraGPUSystemTick` | GPU Tick 打包(Phase 8 展开) |

## 块 A:辅助结构

### `ENiagaraSystemSimulationScript`(L22)

```cpp
enum class ENiagaraSystemSimulationScript : uint8 { Spawn, Update, Num };
```

System 脚本只有 Spawn/Update 两类。(Emitter Spawn/Update 脚本被**合并到 System 脚本**—— Phase 1 `UNiagaraScript::IsCompilable()` 对 Emitter-level 脚本返 false 的原因。)

### `FNiagaraEventHandlingInfo`(L31)

Event 处理 runtime 状态。`TInlineAllocator<16> SpawnCounts` + `FNiagaraDataBuffer* EventData`(自动管理 ReadRef)+ `SourceEmitterName` 键。

### `FNiagaraDataSetExecutionInfo`(L66)

**VM 执行时的 DataSet 绑定**。关键约定:

```cpp
FNiagaraDataSet* DataSet;
FNiagaraDataBuffer* Input;      // 供 VM 读的 DataBuffer(带 ReadRef 自动管理)
FNiagaraDataBuffer* Output;     // 供 VM 写的 DataBuffer(必须 IsBeingWritten)
int32 StartInstance;
bool bUpdateInstanceCount;
```

`Init()` 里加 ReadRef / 析构 ReleaseReadRef,符合 `FNiagaraSharedObject` 协议。

### `FScriptExecutionConstantBufferTable`(L129)

```cpp
TArray<const uint8*, TInlineAllocator<12>> Buffers;
TArray<int32, TInlineAllocator<12>> BufferSizes;

template<typename T> void AddTypedBuffer(const T&);
void AddRawBuffer(const uint8* BufferData, int32 BufferSize);
```

传给 VM 的**常量 buffer 组**——最多 12 个 buffer(`TInlineAllocator<12>`,对应 Global/System/Owner/Emitter × Prev/Curr 等多路参数)。

## 块 B:CPU VM 执行上下文(**Phase 5 主题**)

### `FNiagaraScriptExecutionContextBase`(L154)

**基类**,三路继承:普通脚本 / System 脚本 / GPU 脚本(本块不含 GPU 子类,GPU 走 `FNiagaraComputeExecutionContext`)。

核心字段:

```cpp
UNiagaraScript* Script;                                                  // 关联的 Asset
TArray<const FVMExternalFunction*> FunctionTable;                         // DI 外部函数绑定表(指针,指向共享表)
TArray<void*> UserPtrTable;                                              // per-instance user data 指针
FNiagaraScriptInstanceParameterStore Parameters;                         // 参数存储(Phase 4 子类)
TArray<FDataSetMeta, TInlineAllocator<2>> DataSetMetaTable;              // DataSet metadata
TArray<FNiagaraDataSetExecutionInfo, TInlineAllocator<2>> DataSetInfo;   // DataSet 输入输出绑定

int32 HasInterpolationParameters : 1;
int32 bAllowParallel : 1;                                                // 允许并行(粒子间独立才能)
```

核心方法:

```cpp
virtual bool Init(UNiagaraScript* InScript, ENiagaraSimTarget InTarget);
virtual bool Tick(FNiagaraSystemInstance* Instance, ENiagaraSimTarget SimTarget) = 0;  // 纯虚

void BindData(int32 Index, FNiagaraDataSet& DataSet, int32 StartInstance, bool bUpdateInstanceCounts);
void BindData(int32 Index, FNiagaraDataBuffer* Input, int32 StartInstance, bool bUpdateInstanceCounts);

bool Execute(uint32 NumInstances, const FScriptExecutionConstantBufferTable& ConstantBufferTable);  // **实际 VM dispatch**

const TArray<UNiagaraDataInterface*>& GetDataInterfaces() const;
bool CanExecute() const;
TArrayView<const uint8> GetScriptLiterals() const;
void DirtyDataInterfaces();
void PostTick();
```

**`Execute` 是 VM 入口**——把 NumInstances + 常量 buffer 表传给 VectorVM,VM 从 `Parameters / FunctionTable / DataSetInfo` 拿到执行所需一切。

### `FNiagaraScriptExecutionContext`(L208)

普通 CPU 脚本上下文(Emitter Spawn/Update):

```cpp
TArray<FVMExternalFunction> LocalFunctionTable;  // 本实例专属的 DI 函数(per-instance data 的绑定走这里)
```

之前基类 `FunctionTable` 存指针,`LocalFunctionTable` 存实际函数对象;每 instance 的 per-instance DI 函数填这里,然后 `FunctionTable` 指过来。

### `FNiagaraSystemScriptExecutionContext`(L232)

System 级脚本上下文。多了**处理 per-instance DI 的 hook 机制**:

```cpp
TArray<FNiagaraSystemInstance*>* SystemInstances;   // 正在处理的 instance 数组
ENiagaraSystemSimulationScript ScriptType;          // Spawn 还是 Update

struct FExternalFuncInfo { FVMExternalFunction Function; };
TArray<FExternalFuncInfo> ExtFunctionInfo;

void PerInstanceFunctionHook(FVectorVMContext& Context, int32 PerInstFunctionIndex, int32 UserPtrIndex);

virtual bool GeneratePerInstanceDIFunctionTable(FNiagaraSystemInstance* Inst, TArray<FNiagaraPerInstanceDIFuncInfo>& OutFunctions);
```

**PerInstanceFunctionHook 的作用**:System 脚本跑 50 个 instance,它们的 User DI 可能不一样。VM 遇到 DI 函数调用时,走 hook,根据当前 instance 索引找到**那个 instance 的**实际 DI 函数,调用之。

## `FNiagaraPerInstanceDIFuncInfo`(L225)

```cpp
struct FNiagaraPerInstanceDIFuncInfo {
    FVMExternalFunction Function;
    void* InstData;
};
```

Per-instance DI 函数的完整绑定(Phase 3 `FNiagaraSystemInstance::PerInstanceDIFunctions[ScriptType][FuncIndex]` 就是这个数组)。TODO 注释提到想用 lambda 捕获代替 user ptr 表。

## 块 C:GPU Compute 执行上下文(Phase 8 展开)

- `FNiagaraComputeSharedContext`(L308):`ScratchIndex / ScratchTickStage / ParticleCountReadFence / ParticleCountWriteFence`
- `FNiagaraComputeSharedContextDeleter`(L317):ENQUEUE_RENDER_COMMAND 延迟删除(GT 请求,RT 执行)
- `FNiagaraComputeExecutionContext`(L328):GPU Emitter 的运行时状态——`MainDataSet`、`GPUScript / GPUScript_RT`、`CombinedParamStore`、`DataInterfaceProxies`、`DataToRender / TranslucentDataToRender`、`DefaultSimulationStageIndex / MaxUpdateIterations / SpawnStages / SimStageInfo`
- 相关 `IsOutputStage / IsIterationStage / FindIterationInterface` — SimStage 查询

## 块 D:GPU Tick 打包(Phase 8 展开)

- `FNiagaraGpuSpawnInfoParams` / `FNiagaraGpuSpawnInfo`(L267, L275):GPU spawn 参数打包,`NIAGARA_MAX_GPU_SPAWN_INFOS=8` 个槽
- `FNiagaraDataInterfaceInstanceData`(L419):`TMap<Proxy*, int32>` per-instance DI 数据 offset 表
- `FNiagaraSimStageData`(L433):SimStage 的 source/destination buffer + Alternative Iteration Source
- `FNiagaraComputeInstanceData`(L445):**打包一个 GPU Emitter 的完整 tick 数据**——SpawnInfo + EmitterParamData + ExternalParamData + Context + DataInterfaceProxies + SimStageData + `bStartNewOverlapGroup`(对接 Phase 1 `EmitterExecutionIndex::kStartNewOverlapGroupBit`)
- `FNiagaraGPUSystemTick`(L480):**一次 GPU tick 的完整打包**,从 GT 传到 RT
  - `FNiagaraDataInterfaceInstanceData* DIInstanceData`
  - `uint8* InstanceData_ParamData_Packed`(打包缓冲,`FNiagaraComputeInstanceData + ParamData` 紧凑存储)
  - Uniform buffer 源:`GlobalParamData / SystemParamData / OwnerParamData`
  - `Count / TotalDispatches / NumInstancesWithSimStages / ParticleCountFence`
  - bool 布尔:`bRequiresDistanceFieldData / bRequiresDepthBuffer / bRequiresEarlyViewData / bRequiresViewUniformBuffer / bNeedsReset / bIsFinalTick`
  - `enum EUniformBufferType { UBT_Global, UBT_System, UBT_Owner, UBT_Emitter, UBT_External }` 5 类

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraScriptExecutionContext]] — CPU VM 执行上下文
- [[Wiki/Entities/Stock/Niagara/FNiagaraComputeExecutionContext]] — GPU 执行上下文(Phase 8 详)
- [[Wiki/Entities/Stock/Niagara/FNiagaraGPUSystemTick]] — GT→RT 的 tick 打包(Phase 8 详)
- [[Wiki/Entities/Stock/Niagara/FNiagaraEmitterInstance]] — 持有 Spawn/Update ExecContext
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]] — 持有 System 级 ExecContext

## 开放问题

- `Execute` 里 `FVectorVMContext` 和 `ConstantBufferTable` 的具体对接(VM 实际调用路径)→ VectorVM.h 本身
- `PerInstanceFunctionHook` 的 FVectorVMContext 操作细节 → `.cpp`
- `bAllowParallel` 的触发条件(什么脚本能并行)→ 编译器决定,cpp 注释可能有
- Phase 8 的所有 GPU 相关结构延迟到 Phase 8 展开
