---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-5, reader, vm, script-execution]
sources: 5
aliases: [Phase 5 读本, Niagara VM 执行读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 5 读本 — Niagara 脚本如何"跑起来"

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 5 的**主题读本**。一次读完掌握 CPU VM 如何 dispatch Niagara 脚本、per-instance DI 机制、GPU 执行上下文与 Tick 打包的形态(GPU 详细等 [[Readers/Niagara/Phase8-gpu-simulation-读本|Phase 8]])。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 5 要回答的问题

Phase 3 的 `FNiagaraEmitterInstance` 持有三个 "ExecContext" 字段,Phase 4 的 `FNiagaraParameterStore` 装了全部参数——**它们怎么合起来让脚本真的跑起来**?

> [!question] Phase 5 要回答
> - `FNiagaraEmitterInstance::SpawnExecContext` 这个字段类型是什么,里面装着什么?
> - Niagara 脚本里一条 `Position += Velocity * DeltaTime` 怎么变成 CPU 上对 1000 粒子的实际运算?
> - Niagara 脚本调用 DI 函数(比如 `SkeletalMesh.SamplePosition()`)时,运行时如何找到对应的 C++ 实现?
> - `System.GetParameterCollection().GetFloat()` 这种 User DI 调用,同一 System 的不同 instance 用不同 DI 实例,VM 怎么区分?
> - GPU 模拟时,GT 侧怎么打包数据给 RT?

Phase 5 涉及 5 文件(1185 行),特点是**CPU 执行主路径清晰,GPU 相关占据一半但属于 Phase 8 预告**:

| # | 文件 | 行 | 核心 |
|---|---|---|---|
| 1 | `NiagaraCore.h` | **6** | 只有 `typedef uint64 FNiagaraSystemInstanceID` |
| 2 | `NiagaraDataInterfaceBase.h` | 136 | DI 基类 + CS 参数绑定(Phase 7 主角) |
| 3 | `NiagaraScriptExecutionContext.h` | **531** | **核心 — CPU VM 上下文** + GPU Compute 上下文 + GPU Tick 打包 |
| 4 | `NiagaraScriptExecutionParameterStore.h` | 195 | VM 参数存储的 padding 机制 |
| 5 | `NiagaraEmitterInstanceBatcher.h` | 317 | **GPU** Batcher(名字误导,非 CPU;Phase 8 详) |

---

## 1. 先看"接口模块分离":NiagaraCore

Niagara 其实是一个 7 模块的家族:`NiagaraCore` / `Niagara` / `NiagaraShader` / `NiagaraVertexFactories` / `NiagaraAnimNotifies` / `NiagaraEditor` / `NiagaraEditorWidgets`。

`NiagaraCore.h` 整个文件就 6 行:

```cpp
#pragma once
typedef uint64 FNiagaraSystemInstanceID;
```

NiagaraCore 模块的全部内容只有:
- 本文件(`FNiagaraSystemInstanceID` typedef)
- `NiagaraDataInterfaceBase.h`(DI 抽象基类 + CS 参数绑定)
- `NiagaraMergeable.h`(merge 机制基类)

**存在原因**:让 `NiagaraShader` / `NiagaraVertexFactories` 这样的 RHI/Shader 模块能依赖 DI 接口类型,**不需要**引入整个 `Niagara` 主模块(242 文件)。这是典型的**接口/实现分离**,也是为什么 `UNiagaraDataInterfaceBase` 住在 `NiagaraCore` 而**具体 DI**(`UNiagaraDataInterfaceCurve` 等,Phase 7)住在 Niagara 主模块。

本 Phase 只需登记 DI 基类的存在(§ 源摘要),Phase 7 专题展开 DI 生态。

---

## 2. CPU VM 执行上下文:主角登场

`NiagaraScriptExecutionContext.h`(531 行)分 4 大块:

| 块 | 角色 |
|---|---|
| A | 辅助结构(EventHandlingInfo / DataSetExecutionInfo / ConstantBufferTable) |
| B | **CPU VM 执行上下文(本 Phase 主题)** |
| C | GPU Compute 执行上下文(Phase 8) |
| D | GPU Tick 打包(Phase 8) |

### 2.1 `FNiagaraScriptExecutionContextBase` 的字段意义

```cpp
struct FNiagaraScriptExecutionContextBase
{
    UNiagaraScript* Script;                                  // 关联 Asset(Phase 1)
    TArray<const FVMExternalFunction*> FunctionTable;         // DI 函数绑定表(指针共享)
    TArray<void*> UserPtrTable;                               // per-instance user data 指针
    FNiagaraScriptInstanceParameterStore Parameters;          // 实例参数存储(Phase 4.7)
    TArray<FDataSetMeta> DataSetMetaTable;
    TArray<FNiagaraDataSetExecutionInfo> DataSetInfo;         // DataSet 输入输出绑定

    int32 HasInterpolationParameters : 1;
    int32 bAllowParallel : 1;

    virtual bool Init(UNiagaraScript*, ENiagaraSimTarget);
    virtual bool Tick(FNiagaraSystemInstance*, ENiagaraSimTarget) = 0;  // 纯虚
    void BindData(int32 Index, FNiagaraDataSet&, int32 StartInstance, bool);
    bool Execute(uint32 NumInstances, const FScriptExecutionConstantBufferTable&);  // VM dispatch
};
```

六个关键字段的心智:

- **Script** 是"脚本身份",编译产物(字节码 / VM Literals)都从 Script 访问
- **FunctionTable** 是"脚本里每条 DI 调用对应的 C++ 函数指针"——编译时定好索引,运行时查表
- **UserPtrTable** 是"DI 的 per-instance 状态指针"——有些 DI 每个 instance 要独立状态(比如 `NiagaraDataInterfaceCurve` 可能无状态,但 `StaticMesh` 每 instance 缓存采样上下文)
- **Parameters** 是所有输入参数数据(User./Engine./System./Emitter./...)按 offset 组织的 byte blob(Phase 4.7 讲过 padding)
- **DataSetInfo** 是 "Spawn 脚本读 Emitter DataSet,写 Particle DataSet;Update 脚本两份都是 Particle DataSet" 之类的 IO 绑定
- **bAllowParallel** 是编译器静态分析的结果——脚本里没有"粒子间通信"(event spawn / inter-particle lookup)时,多个粒子完全独立,可并行跑

### 2.2 `Execute` 的语义

```cpp
bool Execute(uint32 NumInstances, const FScriptExecutionConstantBufferTable& ConstantBufferTable);
```

这是实际 VM dispatch 的入口。调进去之后,Niagara 会:

1. 从 `Script->GetVMExecutableData()` 取字节码
2. 把 `Parameters / FunctionTable / UserPtrTable / DataSetInfo / ConstantBufferTable` 打包成 `FVectorVMContext`
3. 调 `VectorVM::Exec(Context, ByteCode, NumInstances)` (引擎内置 VectorVM 模块)
4. VM 按 SoA 布局对 NumInstances 个粒子逐行执行字节码,遇到外部函数调用就查 `FunctionTable` 索引

`FScriptExecutionConstantBufferTable` 是"传给 VM 的一批常量 buffer,最多 12 个槽":

```cpp
TArray<const uint8*, TInlineAllocator<12>> Buffers;
TArray<int32, TInlineAllocator<12>> BufferSizes;
```

对应 Phase 3 `FNiagaraSystemInstance::GlobalParameters/SystemParameters/OwnerParameters/EmitterParameters` 各 × 2 (current + previous) = 8 个槽位,其他 4 槽留给 script literals / external。

### 2.3 `FNiagaraScriptExecutionContext`(Emitter 版)

```cpp
struct FNiagaraScriptExecutionContext : public FNiagaraScriptExecutionContextBase
{
    TArray<FVMExternalFunction> LocalFunctionTable;  // per-instance DI 的实际函数对象
    virtual bool Tick(FNiagaraSystemInstance*, ENiagaraSimTarget) override;
};
```

普通 Emitter Spawn/Update 脚本用。`LocalFunctionTable` 装"本 emitter instance 专有的 DI 函数对象",基类 `FunctionTable` 的指针指进来。

Phase 3 `FNiagaraEmitterInstance::SpawnExecContext / UpdateExecContext` 就是这个类型。

### 2.4 `FNiagaraSystemScriptExecutionContext`(System 版)

System 脚本在 `FNiagaraSystemSimulation` 里,一次处理**一批 instance**(4 个一 batch)。每个 instance 的 User DI 可能不同 —— 比如 50 个爆炸实例,每个绑定了自己的 SkeletalMesh。

```cpp
struct FNiagaraSystemScriptExecutionContext : public FNiagaraScriptExecutionContextBase
{
    TArray<FExternalFuncInfo> ExtFunctionInfo;
    TArray<FNiagaraSystemInstance*>* SystemInstances;       // 当前这批实例
    ENiagaraSystemSimulationScript ScriptType;              // Spawn / Update

    void PerInstanceFunctionHook(FVectorVMContext&, int32 PerInstFuncIndex, int32 UserPtrIndex);

    virtual bool GeneratePerInstanceDIFunctionTable(FNiagaraSystemInstance*, TArray<FNiagaraPerInstanceDIFuncInfo>&);
};
```

**`PerInstanceFunctionHook` 的魔法**:VM 跑第 N 个粒子(对应第 N 个 SystemInstance)时,遇到 DI 函数调用,hook 根据 `VectorVMContext` 的当前 instance 索引,查 `SystemInstances[N]->PerInstanceDIFunctions[ScriptType][FuncIndex]`,拿到:

```cpp
struct FNiagaraPerInstanceDIFuncInfo {
    FVMExternalFunction Function;   // 对这个 instance 有效的函数
    void* InstData;                 // 这个 instance 的 DI 状态
};
```

代为调用。**这就是 Phase 3 `FNiagaraSystemInstance::PerInstanceDIFunctions[(int32)ENiagaraSystemSimulationScript::Num]` 那个二维数组的用途**。

---

## 3. 参数存储的 padding 机制

Phase 4.7 讲 `FNiagaraParameterStore` 时留了个伏笔:**CPU 紧凑布局 vs GPU constant buffer 对齐布局**。`NiagaraScriptExecutionParameterStore.h` 里兑现:

### 3.1 两个特化

```cpp
// 绑 Script,装 layout(编译期定)
FNiagaraScriptExecutionParameterStore : FNiagaraParameterStore {
    int32 ParameterSize;                                // CPU 紧凑大小
    uint32 PaddedParameterSize;                         // GPU 对齐后大小
    TArray<FNiagaraScriptExecutionPaddingInfo> PaddingInfo;  // 两者映射
    TArray<uint8> CachedScriptLiterals;
};

// 绑 ExecContext,装实际 instance 数据
FNiagaraScriptInstanceParameterStore : FNiagaraParameterStore {
    FNiagaraCompiledDataReference<FNiagaraScriptExecutionParameterStore> ScriptParameterStore;
    void CopyCurrToPrev();                              // 插值 spawn 用
    void CopyParameterDataToPaddedBuffer(uint8*, uint32);  // 写 GPU-ready buffer
};
```

### 3.2 `FNiagaraScriptExecutionPaddingInfo`

```cpp
USTRUCT()
struct FNiagaraScriptExecutionPaddingInfo {
    uint16 SrcOffset;   // 在紧凑 CPU 布局中的 byte 位置
    uint16 DestOffset;  // 在 padded GPU 布局中的 byte 位置
    uint16 SrcSize;     // 紧凑大小
    uint16 DestSize;    // padded 大小(通常向上对齐 16)
};
```

例子:CPU 里一个 `float LifeTime`(4 byte)在紧凑 offset 16,在 GPU cbuffer 里放 offset 32(因为前一个是 `Vector4` 占 16 byte 对齐)。PaddingInfo 就是 `{16, 32, 4, 16}`(SrcSize=4,DestSize=16 表示后面要 pad 12 byte 对齐到下一个 16-byte 边界)。

### 3.3 插值 Spawn 的 `CopyCurrToPrev`

粒子可以"在帧内中间时间点 spawn"(`FNiagaraSpawnInfo::InterpStartDt`)。为了让脚本里读 `Previous.Engine.DeltaTime` 这种"上一帧值",每 tick 末把当前参数值 copy 到 prev 槽位:

```cpp
void FNiagaraScriptInstanceParameterStore::CopyCurrToPrev();
```

这和 Phase 3 `FNiagaraSystemInstance::GlobalParameters[2]` 的参数双缓冲是**不同机制**:
- Phase 3 的 `[2]` 双缓冲:解决 GT/RT 跨帧读写竞态
- 本 `CopyCurrToPrev`:解决脚本内读"上帧快照"的语义需求

`FNiagaraScriptExecutionContextBase::HasInterpolationParameters : 1` 标记是否需要调用 CopyCurrToPrev—— 大多数脚本不用,省一次拷贝。

### 3.4 禁用的基类方法

两个子类都 `check(0)` 禁用了 `AddParameter / RemoveParameter / RenameParameter` —— 因为改 layout 会破坏 VM 的 offset 编译产物。Execution store 里参数一旦添加就不能改。

---

## 4. DI 基类:登记,留 Phase 7

`NiagaraDataInterfaceBase.h` 里 `UNiagaraDataInterfaceBase` 提供:

```cpp
UCLASS(abstract, EditInlineNew)
class UNiagaraDataInterfaceBase : public UNiagaraMergeable {
    virtual FNiagaraDataInterfaceParametersCS* CreateComputeParameters() const { return nullptr; }
    virtual void BindParameters(FNiagaraDataInterfaceParametersCS*, ...);
    virtual void SetParameters(const FNiagaraDataInterfaceParametersCS*, ...) const;
    ...
};
```

特别的是 `FNiagaraDataInterfaceParametersCS` —— **non-virtual** 参数绑定基类,用 `LAYOUT_FIELD` 宏支持**二进制布局序列化**(memory image)。这是为了让 shader 参数绑定能进 shader cache,虚表指针无法序列化,所以不能用虚函数。

宏 `DECLARE_NIAGARA_DI_PARAMETER()` / `IMPLEMENT_NIAGARA_DI_PARAMETER(T, ParameterType)` 是给 DI 子类的"声明 + 实现"成对模板,把 DI UCLASS 的虚方法连接到 non-virtual `FNiagaraDataInterfaceParametersCS` 子类。

本 Phase 到此为止,具体 DI 实现、CPU/GPU 双路径绑定、per-instance data 分配 → **Phase 7 主题**。

---

## 5. GPU 执行上下文(预告,Phase 8 详展)

`NiagaraScriptExecutionContext.h` 文件的 C/D 两块几乎全是 GPU 相关,本 Phase 做心智准备:

### 5.1 `FNiagaraComputeExecutionContext`

GPU Emitter 的运行时状态持有者,和 CPU 的 `FNiagaraScriptExecutionContext` **平行但不共基类**(GPU 的是非 UObject 的独立 struct,CPU 是 struct 继承体系)。

核心字段:
- `MainDataSet / GPUScript / GPUScript_RT / CombinedParamStore`
- `DataToRender / TranslucentDataToRender`(两个 "当前可渲染的 buffer",TranslucentData 是低延迟通路用的本帧数据)
- SimStages 相关(`DefaultSimulationStageIndex / MaxUpdateIterations / SpawnStages / SimStageInfo`)— **Phase 10**

### 5.2 `FNiagaraGPUSystemTick`

**GT→RT 的打包**。每帧每个有 GPU emitter 的 SystemInstance 产生一份,注释讲得很清楚:

> `FNiagaraGPUSystemTick` represents all the information needed to dispatch a single tick of a `FNiagaraSystemInstance`. Created on the game thread and passed to the renderthread.

关键设计:**`InstanceData_ParamData_Packed` 是一大块 `uint8`**,里面按顺序:

```
[uint32 Count]
[FNiagaraComputeInstanceData × Count]
[16-byte pad]
[GlobalParamData + SystemParamData + OwnerParamData]
```

16-byte 对齐让 ParamData 能**直接 upload 成 UniformBuffer**,省一次拷贝。

5 种 Uniform Buffer 类型:

```cpp
enum EUniformBufferType {
    UBT_Global,     // System-level
    UBT_System,
    UBT_Owner,
    UBT_Emitter,    // Per-Emitter
    UBT_External,   // DI 或 script 额外 cbuffer
};
```

### 5.3 `FNiagaraDataInterfaceInstanceData`

```cpp
struct FNiagaraDataInterfaceInstanceData {
    void* PerInstanceDataForRT;
    TMap<FNiagaraDataInterfaceProxy*, int32> InterfaceProxiesToOffsets;
    uint32 PerInstanceDataSize;
    uint32 Instances;
};
```

DI 的 per-instance 数据在 GPU 侧打包:所有 DI 的数据拼一大块,按 proxy 指针查 offset。这是 Phase 3 `FNiagaraSystemInstance::DataInterfaceInstanceData` blob 的 GPU 对位。

---

## 6. GPU Batcher(误导的命名 + Phase 8 主战场)

`NiagaraEmitterInstanceBatcher.h`(317 行)文件头注释说 "Queueing and batching for Niagara simulation",但**实际上只处理 GPU**——CPU VM 直接在 `FNiagaraSystemInstance::Tick_Concurrent` 里跑,没有 CPU batcher。

### 6.1 三阶段 Tick Stage

GPU tick 分三个渲染阶段执行:

```cpp
enum class ETickStage {
    PreInitViews,      // 最早,view 数据未就绪
    PostInitViews,     // 有 view,opaque 未渲
    PostOpaqueRender,  // opaque 渲完,可读深度/场景纹理
};
```

每个 tick 根据自己的依赖(`bRequiresDistanceFieldData / DepthBuffer / EarlyViewData / ViewUniformBuffer`)分到对应 stage。

### 6.2 FFXSystemInterface 合约

Batcher 继承 `FFXSystemInterface`,UE 渲染器调:

- `PreInitViews` / `PostInitViews(ViewUniformBuffer)` / `PreRender` / `PostRenderOpaque`

每个 hook 里 Batcher 处理对应 stage 的 tick。

### 6.3 核心 API

```cpp
void GiveSystemTick_RenderThread(FNiagaraGPUSystemTick&);  // GT→RT 入口
void InstanceDeallocated_RenderThread(FNiagaraSystemInstanceID);

void Run(const FNiagaraGPUSystemTick&, const FNiagaraComputeInstanceData*,
         uint32 UpdateStartInstance, uint32 TotalNumInstances,
         const FNiagaraShaderRef&, FRHICommandList&, FRHIUniformBuffer*, ...);
```

`Run` 是 compute shader dispatch 入口——大量参数反映 Niagara GPU tick 的复杂度(shader 绑定、spawn info、SimStage 索引、DI iteration)。

### 6.4 其他职责(Phase 8 展开)

- `GPUInstanceCounterManager` — GPU 上的粒子数量 buffer(Phase 8.6)
- `GPUSortManager` 接入 — 半透明粒子深度排序(Phase 8.7)
- `GlobalCBufferLayout / SystemCBufferLayout / OwnerCBufferLayout / EmitterCBufferLayout` — 4 种 cbuffer 布局缓存
- `DummyUAVPool` — 没实际用但 shader 要绑 UAV 时的占位池
- `DeferredIDBufferUpdates` / FreeID buffer 管理

---

## 7 条关键洞察

1. **NiagaraCore 模块的存在**是接口/实现分离的典型——DI 基类抽离让 `NiagaraShader` 等 RHI 模块依赖不拖累主模块
2. **CPU VM Execute 的关键**是把脚本编译产物(字节码、offset 表)+ 运行时 `FunctionTable / Parameters / DataSetInfo / ConstantBufferTable` 喂给 `VectorVM::Exec` —— Niagara 自己不实现 VM,用引擎内置 VectorVM 模块
3. **System 脚本的 `PerInstanceFunctionHook`** 解决"一批 instance 的 User DI 不同"的问题:VM 遇到 DI 调用 → hook → 按当前 instance 索引查该 instance 的 DI 函数
4. **CPU 紧凑布局 vs GPU 对齐布局的映射**由 `FNiagaraScriptExecutionPaddingInfo` 编译期生成——4 个 uint16(SrcOffset/DestOffset/SrcSize/DestSize)描述一条映射
5. **`CopyCurrToPrev` 与 Phase 3 `GlobalParameters[2]` 双缓冲是两件事**:前者服务脚本内读"上帧快照"(`Previous.*` 命名空间),后者服务 GT/RT 跨线程读写
6. **GPU Tick 打包的核心是 `InstanceData_ParamData_Packed`**:紧凑 byte 布局 + 16-byte 对齐让 ParamData 能直接 upload 成 UniformBuffer,省一次拷贝
7. **`NiagaraEmitterInstanceBatcher` 不是 CPU batcher**,而是 RT 驻留的 GPU compute 总调度器,分 PreInitViews / PostInitViews / PostOpaqueRender 三 stage dispatch

---

## Phase 5 留下的问题

- VectorVM 自身结构(字节码 opcode / 寄存器分配)→ 不在本学习路径,但 `VectorVM.h` 可单独读
- `UNiagaraDataInterface` 完整接口 + 所有具体 DI → **Phase 7**
- `FNiagaraDataInterfaceProxy` RT 替身机制 → **Phase 7/8**
- GPU compute shader 编译流水线(`UNiagaraScript` GPU 版的产物、`FNiagaraShaderMap`)→ **Phase 8.1-8.5**
- `FNiagaraGPUInstanceCountManager` GPU 计数 buffer → **Phase 8.6**
- `FGPUSortManager` 半透明粒子排序 → **Phase 8.7** `NiagaraGPUSortInfo`
- SimStages 多 pass dispatch → **Phase 10**
- Niagara Component Renderer(Phase 3 遇到的 `FNiagaraComponentRenderPool`)→ Phase 6 可能简略覆盖

## 下一步预告

**Phase 6**:渲染系统。粒子数据怎么变成屏幕像素。10 个文件成对:

```
Renderer 类型          Properties(Asset 侧)      Renderer(运行时侧)
────────────────────── ──────────────────────── ──────────────────────
Sprite                 SpriteRendererProperties  RendererSprites
Ribbon                 RibbonRendererProperties  RendererRibbons
Mesh                   MeshRendererProperties    RendererMeshes
Light                  LightRendererProperties   RendererLights

基类                   NiagaraRendererProperties NiagaraRenderer
```

难度 ⭐⭐⭐,理解 UE 渲染线程分离后会容易。

---

## 深入阅读

### 原子页

- Source × 5:[[Wiki/Sources/Stock/NiagaraCore]] / [[Wiki/Sources/Stock/NiagaraDataInterfaceBase]] / [[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] / [[Wiki/Sources/Stock/NiagaraScriptExecutionParameterStore]] / [[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]]
- Entity × 5:[[Wiki/Entities/Stock/UNiagaraDataInterfaceBase]] / [[Wiki/Entities/Stock/FNiagaraScriptExecutionContext]] / [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext]] / [[Wiki/Entities/Stock/FNiagaraGPUSystemTick]] / [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher]]

### 前置议题

- [[Readers/Niagara/Phase3-runtime-instance-读本]] — CPU ExecContext 在 EmitterInstance 被引用但未展开
- [[Readers/Niagara/Phase4-data-model-读本]] — ParameterStore 基类、类型系统、SoA DataSet
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU/GPU 双路径

### 下一步 / 导航

- 下一阶段:[[Readers/Niagara/Phase6-rendering-读本]] — 粒子数据 → 像素
- 深入 GPU 侧:[[Readers/Niagara/Phase8-gpu-simulation-读本]]
- 学习路径总图:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 5 的 5 个头文件(合计 1185 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
