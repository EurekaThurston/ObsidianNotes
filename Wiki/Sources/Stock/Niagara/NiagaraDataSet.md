---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, dataset, data-model, soa, particle-storage]
sources: 1
aliases: [NiagaraDataSet.h, Niagara 数据集源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSet.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataSet.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSet.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 554 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.4(**核心**)

## 职责

定义 Niagara 的**粒子数据存储核心**——`FNiagaraDataSet`(管理者)+ `FNiagaraDataBuffer`(一帧数据)+ `FNiagaraSharedObject`(并发保护基类)+ `FNiagaraDataSetCompiledData`(编译期布局产物)。Phase 3 `FNiagaraEmitterInstance::ParticleDataSet` 指向的那个 DataSet。

**SoA(Structure of Arrays)布局**在此实体现——每个 component(如 Position.x / Position.y / Position.z)自己一条连续数组,而非每粒子一个结构体。SIMD 友好。

## 关键类型 — 由内到外

### 1. `FNiagaraVariableLayoutInfo`(USTRUCT, L14)

```cpp
uint32 FloatComponentStart;   // 在 DataBuffer 里 float 成员的起始索引
uint32 Int32ComponentStart;
uint32 HalfComponentStart;
FNiagaraTypeLayoutInfo LayoutInfo;  // 变量自己的类型 layout(byte offsets)
```

配合 `GetNumFloatComponents / Int32 / Half`,一起说明 "这个变量在 DataSet 里占哪几个 component 索引"。

### 2. `FNiagaraSharedObject`(L47)

基类:**多线程共享保护**。ReadRefCount(原子 int)机制:

```cpp
FORCEINLINE bool IsBeingRead() const  { return ReadRefCount.Load() > 0; }
FORCEINLINE bool IsBeingWritten() const { return ReadRefCount.Load() == INDEX_NONE; }

FORCEINLINE void AddReadRef() { check(!IsBeingWritten()); ReadRefCount++; }
FORCEINLINE void ReleaseReadRef() { check(IsBeingRead()); ReadRefCount--; }

// TryLock: 只有无 reader 时可加写锁
FORCEINLINE bool TryLock() {
    int32 Expected = 0;
    return ReadRefCount.CompareExchange(Expected, INDEX_NONE);
}
```

`INDEX_NONE`(-1)作特殊值标记"写锁独占"。

`Destroy()` 把对象放到**延迟删除队列**,`FlushDeletionList()` 在安全时机清理——因为 RT 可能还在读这个 buffer。

### 3. `FNiagaraDataBuffer : FNiagaraSharedObject`(L103)— **一帧的粒子数据**

这是 SoA 的**现场**:

```cpp
// CPU 数据:三个基础类型各自一块 byte blob
TArray<uint8> FloatData;      // 所有 float component 拼在一起
TArray<uint8> Int32Data;
TArray<uint8> HalfData;

// 表示 "每个 component 间距多少 byte"(= NumInstances × 4 或 2)
uint32 FloatStride;
uint32 Int32Stride;
uint32 HalfStride;

// GPU 数据:对应三个 RW Buffer
FRWBuffer GPUBufferFloat;
FRWBuffer GPUBufferInt;
FRWBuffer GPUBufferHalf;

// Persistent IDs
TArray<int32> IDToIndexTable;      // CPU
FRWBuffer GPUIDToIndexTable;       // GPU

uint32 GPUInstanceCountBufferOffset;  // GPU 计数 buffer 中的偏移
uint32 NumInstances;                   // 当前实例数
uint32 NumInstancesAllocated;          // 分配容量
uint32 NumSpawnedInstances;            // 本帧新 spawn 的
uint32 IDAcquireTag;                   // 本 tick 新 ID 用的 tag

// 给 VM 的寄存器表(扁平)
TArray<uint8*> RegisterTable;
uint32 RegisterTypeOffsets[3];         // Float/Int32/Half 三段的起始索引
```

关键访问(注释详见代码 L119-L149):

```cpp
// 两级 ptr:先取 component base,再取 instance
uint8* GetComponentPtrFloat(uint32 ComponentIdx) { return FloatData.GetData() + FloatStride * ComponentIdx; }
float* GetInstancePtrFloat(uint32 ComponentIdx, uint32 InstanceIdx) {
    return (float*)GetComponentPtrFloat(ComponentIdx) + InstanceIdx;
}
```

> [!abstract] SoA 的内存图像
> 假设一个 DataSet 有 Position(Vec3)、Velocity(Vec3)、Color(Color,4 float)、Lifetime(float)共 10 个 float component,NumInstances=1000。
> ```
> FloatData 总 byte 数 = 10 × 1000 × 4 = 40000 byte
> FloatStride = 1000 × 4 = 4000 byte
> ComponentIdx=0 → Position.x 数组(1000 float 连续)
> ComponentIdx=1 → Position.y 数组
> ...
> ComponentIdx=9 → Lifetime 数组
> ```
> SIMD 可以一次从 Position.x 数组连读 4 或 8 个粒子的 x。

`SwapInstances(OldIdx, NewIdx)` + `KillInstance(InstanceIdx)` 是 "以索引 swap+pop 方式"杀粒子——比 shift 快 O(1)。

### 4. `FNiagaraDataSetCompiledData`(USTRUCT, L253)

**编译期产物**,不变数据(asset 编译后固定):

```cpp
TArray<FNiagaraVariable> Variables;             // 这个 DataSet 有哪些变量
TArray<FNiagaraVariableLayoutInfo> VariableLayouts;  // 各变量在 buffer 里布局
FNiagaraDataSetID ID;
uint32 TotalFloatComponents / TotalInt32Components / TotalHalfComponents;  // 合计
uint32 bRequiresPersistentIDs : 1;
ENiagaraSimTarget SimTarget;                    // CPU or GPU
```

方法:`BuildLayout()`(根据 Variables 算 VariableLayouts + Totals)、`Empty()`。

### 5. `FNiagaraDataSet`(L300)— 管理者

**本身不直接存粒子数据**,数据在 `FNiagaraDataBuffer` 池里:

```cpp
FNiagaraCompiledDataReference<FNiagaraDataSetCompiledData> CompiledData;  // 布局

// Persistent ID 管理
TArray<int32> FreeIDsTable;
int32 NumFreeIDs;
int32 MaxUsedID;
int32 IDAcquireTag;
TArray<int32> SpawnedIDsTable;
FRWBuffer GPUFreeIDs;
uint32 GPUNumAllocatedIDs;

// 双缓冲读写指针
FNiagaraDataBuffer* CurrentData;     // 当前读
FNiagaraDataBuffer* DestinationData; // 当前写(BeginSimulate...EndSimulate 之间)

// Buffer 池
TArray<FNiagaraDataBuffer*, TInlineAllocator<2>> Data;  // 通常 2-3 个,RT 在读就留着
uint32 MaxInstanceCount;
```

核心流程(L315-L319):

```cpp
FNiagaraDataBuffer& BeginSimulate(bool bResetDestinationData = true);  // 取一个 free buffer 作 Destination
void EndSimulate(bool SetCurrentData = true);                          // Current <- Destination,旧 Current 进池
void Allocate(int32 NumInstances, bool bMaintainExisting = false);     // Destination 预分配
```

查询:

```cpp
const TArray<FNiagaraVariable>& GetVariables() const;
const FNiagaraVariableLayoutInfo* GetVariableLayout(const FNiagaraVariable& Var) const;
bool GetVariableComponentOffsets(const FNiagaraVariable&, int32& FloatStart, int32& IntStart, int32& HalfStart) const;
```

线程检查(L395-L405):`CheckCorrectThread()` 强制 CPU sim 不在 RT,GPU sim 必须在 RT。

### 6. `FNiagaraDataVariableIterator`(L454)

**注释警告 "Super slow. Don't use at runtime."**——它拷贝 DataBuffer 数据到 `FNiagaraVariable::VarData`,调试用。

### 7. `FScopedNiagaraDataSetGPUReadback`(L517,editor-only)

RAII 把 GPU buffer 同步读回 CPU(`DataBuffer->FloatData`)用于 accessor 读。**stall CPU** 警告,仅工具/调试。

## 依赖

**上游**:`NiagaraCommon.h` / `RHI.h` / `VectorVM.h` / `RenderingThread.h` / `Containers/DynamicRHIResourceArray.h`
**下游**:`FNiagaraEmitterInstance::ParticleDataSet`(Phase 3)、`FNiagaraSystemSimulation` 里的多份 DataSet(Phase 3)、`FNiagaraDataSetAccessor` 家族(Phase 4.5)、所有 DI 的 per-instance 粒子访问

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraDataSet]] — 管理者
- [[Wiki/Entities/Stock/Niagara/FNiagaraDataBuffer]] — 一帧数据
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — CPU/GPU 双路径在此呈现

## 开放问题

- Buffer 池多大够用?`TInlineAllocator<2>` 暗示 2-3 个够——什么场景下会超?
- `IDToIndexTable` GPU/CPU 同步细节 → Phase 8
- `GPUInstanceCountBufferOffset` 指向 `FNiagaraGPUInstanceCountManager` 的 global count buffer → Phase 8
- `ReleaseGPUInstanceCounts` 的时机 → Phase 8
