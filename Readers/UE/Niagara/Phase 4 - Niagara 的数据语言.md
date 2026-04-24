---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-4, reader, data-model, soa]
sources: 7
aliases: [Phase 4 读本, Niagara 数据模型读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 4 - Niagara 的数据语言

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]] Phase 4 的**主题读本**——详细、精确、一次读完即完整掌握 Niagara 如何表示"数据"(类型 → 变量 → 布局 → 数据集 → 参数存储),不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 4 要回答的问题

Phase 3 让我们看清了 Niagara 的"心脏" — 运行时实例层。但那些类里到处充斥着没解释的名字:`FNiagaraVariable`、`FNiagaraDataSet`、`FNiagaraParameterStore`、`FNiagaraVariableLayoutInfo`、`FNiagaraDataSetCompiledData`…… Phase 4 来清账。

> [!question] Phase 4 要回答
> Niagara 脚本能操作的"数据"长什么样?从"`Particles.Position`" 这样一个字符串名字,到 GPU 上一个 `float3` 的内存实体,中间有几层结构?Niagara 为什么不直接用 `TMap<FString, FVector>` 存粒子属性?

Phase 4 涉及 7 文件,约 5663 行。按"由基础到上层"的叙事顺序:

| # | 文件 | 行 | 角色 |
|---|---|---|---|
| 1 | `NiagaraCommon.h` | 1200 | 共享枚举 / 常量 / 小结构(函数签名等) |
| 2 | `NiagaraTypes.h` | 1739 | 类型系统(TypeDefinition / Variable / LayoutInfo)+ 执行状态枚举 |
| 3 | `NiagaraConstants.h` | 209 | 预定义命名空间 + 所有 Engine/System/Emitter/Particle 级变量注册 |
| 4 | `NiagaraDataSet.h` | 554 | **核心** — 粒子数据存储(SoA + 双 buffer + 并发保护) |
| 5 | `NiagaraDataSetAccessor.h` | 619 | 类型安全读写模板家族 |
| 6 | `NiagaraParameters.h` | 82 | Editor-only 的简单参数列表 |
| 7 | `NiagaraParameterStore.h` | 1260 | 运行时正牌**参数存储** + 绑定图 + 三路 dirty |

读完你应该能回答:
1. `FNiagaraVariable` 和 `FNiagaraVariableWithOffset` 的区别?
2. 为什么 Niagara 要有自己一套类型系统而不直接用 UStruct?
3. "SoA 布局" 在 `FNiagaraDataBuffer` 里具体长什么样?
4. `FNiagaraDataSet` 管理的 DataBuffer 池为什么要 2-3 个?
5. Persistent ID(`FNiagaraID`)为什么要双整数?
6. `FNiagaraParameters` vs `FNiagaraParameterStore` —— 为什么编辑器有两套?
7. ParameterStore 的三路 dirty(参数 / DI / UObject)的存在意义?

---

## 1. 共享基础:`NiagaraCommon.h`

最先讲它是因为其他文件全部 `#include "NiagaraCommon.h"`。它装的是:

- **常量**:`NiagaraComputeMaxThreadGroupSize = 64`(GPU 线程组上限)、`NIAGARA_MAX_GPU_SPAWN_INFOS = 8`、`NiagaraNumTickGroups`(TG_PrePhysics 到 TG_LastDemotable 的范围)
- **枚举**(20+ 个):`ENiagaraTickBehavior` / `ENiagaraSimTarget`(CPU/GPU)/ `ENiagaraAgeUpdateMode` / `ENiagaraDataSetType`(ParticleData/Shared/Event)/ `ENiagaraInputNodeUsage` / `ENiagaraScriptCompileStatus` / `ENiagaraBaseTypes`(Half/Float/Int32/Bool)/ `ENiagaraGpuBufferFormat`(Float/HalfFloat/UNB)……
- **小结构**:`FNiagaraDataSetID`(Name + Type)、`FNiagaraDataSetProperties`、`FNiagaraFunctionSignature`(Phase 7 的 DI 函数签名核心)、`FNiagaraOpInOutInfo`

`FNiagaraFunctionSignature`(L269)是其中最重要的——VM 和 DI 函数的完整身份:

```cpp
FName Name;
TArray<FNiagaraVariable> Inputs, Outputs;
uint32 bSupportsCPU : 1, bSupportsGPU : 1;
uint32 bWriteFunction : 1, bRequiresContext : 1;
int32 ModuleUsageBitmask;
TMap<FName, FName> FunctionSpecifiers;
```

留到 Phase 5/7 展开。本 Phase 只需知道它存在。

---

## 2. 类型系统:`FNiagaraTypeDefinition` + `FNiagaraVariable`

回答问题 2 的第一步。Niagara 为什么不直接用 UStruct?

核心原因:**粒子数据需要能被 VM 指令集和 GPU Compute Shader 操作**。UStruct 有反射元数据、虚函数、对齐规则,这些在一个 GPU compute buffer 里都是累赘。Niagara 需要一层"把 UStruct 拍平成 (Float, Int32, Half) 三类 component 数组"的机制。

### 2.1 `FNiagaraTypeDefinition` — 类型身份

它既能指 UStruct(如 `FVector`、`FNiagaraSpawnInfo`),也能指 UEnum(如 `ENiagaraExecutionState`),也能指 UClass(DataInterface 类)。

全局单例通过静态工厂访问:

```cpp
FNiagaraTypeDefinition::GetFloatDef()     // 对应 FNiagaraFloat UStruct
FNiagaraTypeDefinition::GetVec3Def()      // 对应 FVector
FNiagaraTypeDefinition::GetColorDef()     // 对应 FLinearColor
FNiagaraTypeDefinition::GetQuatDef()      // 对应 FQuat
FNiagaraTypeDefinition::GetBoolDef()      // 对应 FNiagaraBool
FNiagaraTypeDefinition::GetIntDef()       // 对应 FNiagaraInt32
FNiagaraTypeDefinition::GetHalfDef()      // 对应 FNiagaraHalf
FNiagaraTypeDefinition::GetHalfVec2Def()  // 对应 FNiagaraHalfVector2
// ... 等
```

### 2.2 Niagara 自己的基础 USTRUCT 包装

`NiagaraTypes.h` 头部定义了一组 USTRUCT 包装基础类型(§ 源摘要):

- `FNiagaraFloat` / `FNiagaraInt32` / `FNiagaraHalf` / `FNiagaraHalfVector2/3/4` / `FNiagaraNumeric` / `FNiagaraParameterMap` / `FNiagaraMatrix`
- 特殊:**`FNiagaraBool`** 不是简单的 `bool Value`,而是一个 `int32 Value`,约定 `True = INDEX_NONE(-1)` / `False = 0` —— 这是 VM compare+select 指令对期望的编码(-1 在 bit 级是全 1,`And/Or` 直接能做 mask)

```cpp
// NiagaraTypes.h:44-56
enum BoolValues { True = INDEX_NONE, False = 0 };
void SetValue(bool bValue) { Value = bValue ? True : False; }
bool GetValue() const { return Value != False; }
```

这个约定让 bool 在 DataSet 里和 int32 共用存储,在 VM/HLSL 里也能直接用位运算优化。

### 2.3 `FNiagaraVariable` — 命名的数据

```cpp
FNiagaraVariableBase = FNiagaraTypeDefinition + FName Name
FNiagaraVariable    = FNiagaraVariableBase + TArray<uint8> VarData(默认值)
```

几乎 Niagara 里所有能写"参数"或"变量"的地方都是 `FNiagaraVariable`:Asset 的 ExposedParameters、DataSet 的变量列表、VM 脚本输入输出、DI 的函数签名参数……

### 2.4 `FNiagaraVariableWithOffset`(在 ParameterStore.h)

`FNiagaraVariableBase + int32 Offset`。纯身份 + 在 ParameterStore byte 数组里的位置。`TArray<FNiagaraVariableWithOffset>` 排序后方便二分查找。

---

## 3. 拍平术:`FNiagaraTypeLayoutInfo`

回答问题 3 的入口。**这是把 UStruct 拍成 SoA 的桥梁**。

```cpp
// NiagaraTypes.h:215
USTRUCT()
struct FNiagaraTypeLayoutInfo
{
    TArray<uint32> FloatComponentByteOffsets;       // 在结构体内的 byte 偏移
    TArray<uint32> FloatComponentRegisterOffsets;   // 在 VM register table 中的索引
    TArray<uint32> Int32ComponentByteOffsets;
    TArray<uint32> Int32ComponentRegisterOffsets;
    TArray<uint32> HalfComponentByteOffsets;
    TArray<uint32> HalfComponentRegisterOffsets;

    static void GenerateLayoutInfo(FNiagaraTypeLayoutInfo& Layout, const UScriptStruct* Struct);
};
```

`GenerateLayoutInfo` 通过反射遍历 UStruct 字段,按类型(FFloatProperty / FUInt16Property / FIntProperty / FBoolProperty)分到 Float/Half/Int32 三类,递归处理 nested struct。

**示例**:`FVector`(x/y/z 都是 float)生成:

```
FloatComponentByteOffsets = [0, 4, 8]         // x@0, y@4, z@8
FloatComponentRegisterOffsets = [0, 1, 2]     // 占 3 个 float 寄存器
Int32ComponentByteOffsets = []                // 没有 int
HalfComponentByteOffsets = []
```

**示例**:`FNiagaraSpawnInfo`(Count: int32, InterpStartDt: float, IntervalDt: float, SpawnGroup: int32)生成:

```
FloatComponentByteOffsets = [4, 8]            // InterpStartDt@4, IntervalDt@8
Int32ComponentByteOffsets = [0, 12]           // Count@0, SpawnGroup@12
```

> [!abstract] 类型系统的三路分拣
> 任何 Niagara 类型都被拍成 **Float / Int32 / Half** 三类 component 的 byte offset 列表。这就是 Niagara 能把完全不同的类型(Vec3 / Color / SpawnInfo / bool)都塞进同一套 DataSet、同一套 VM、同一套 GPU Compute Buffer 的技术底座。

---

## 4. 粒子数据:`FNiagaraDataSet` 与 SoA

回答问题 3、4、5 的主战场。

### 4.1 四层嵌套

```
FNiagaraDataSet                    ← 管理者,不存数据
  ├─ FNiagaraDataSetCompiledData   ← 编译期不变布局(Variables + LayoutInfos + Totals)
  └─ TArray<FNiagaraDataBuffer*>   ← buffer 池(通常 2-3 个)
      ├─ FloatData / Int32Data / HalfData  ← byte blob
      └─ GPUBufferFloat / Int / Half       ← 对应 FRWBuffer
```

### 4.2 SoA 布局的内存图像

一个典型 CPU Emitter 有 Position(Vec3)、Velocity(Vec3)、Color(Color,4 float)、Lifetime(float)、Age(float),共 **12 个 float component**。假设 `NumInstancesAllocated = 1024`,`FloatStride = 1024 * 4 = 4096 byte`:

```
FloatData (总 = 12 × 4096 = 49152 byte):
┌─────────────────────────────────────────────────────────────┐
│ ComponentIdx=0:Position.x 数组(1024 个 float 连续)        │ 4096 B
├─────────────────────────────────────────────────────────────┤
│ ComponentIdx=1:Position.y                                  │ 4096 B
├─────────────────────────────────────────────────────────────┤
│ ComponentIdx=2:Position.z                                  │ 4096 B
├─────────────────────────────────────────────────────────────┤
│ ComponentIdx=3:Velocity.x                                  │ ...
│ ...                                                         │
│ ComponentIdx=11:Age                                         │ 4096 B
└─────────────────────────────────────────────────────────────┘
```

访问 API(两级 ptr):

```cpp
uint8* GetComponentPtrFloat(uint32 ComponentIdx) {
    return FloatData.GetData() + FloatStride * ComponentIdx;
}
float* GetInstancePtrFloat(uint32 ComponentIdx, uint32 InstanceIdx) {
    return (float*)GetComponentPtrFloat(ComponentIdx) + InstanceIdx;
}
```

**SIMD 优势**:VM 要给 1000 粒子每个 `Velocity.x += DeltaTime` —— 直接从 `Velocity.x` 数组连读 8 个,vec8 加法,写回。不用跨 strided 地址。

### 4.3 Double Buffer:CurrentData ↔ DestinationData

Niagara 模拟总是**读上帧写下帧**:

```cpp
FNiagaraDataBuffer* CurrentData;      // 当前可读的(上帧的结果)
FNiagaraDataBuffer* DestinationData;  // 当前在写(下帧)
TArray<FNiagaraDataBuffer*, TInlineAllocator<2>> Data;  // 池
```

```cpp
auto& Dest = DataSet.BeginSimulate();   // 从池里取一个 free buffer 作 Dest
DataSet.Allocate(1024);                 // 分配空间
// ... VM 写 Dest ...
DataSet.EndSimulate();                  // Current ← Dest,原 Current 进池
```

### 4.4 为什么 Buffer 池 2-3 个(问题 4)

理论上 double buffer 2 个够——为什么要 3?因为**渲染线程也要读一份**(Phase 6 `FNiagaraRenderer` 从 `FNiagaraSharedObject` 加 read ref),这份在 RT 释放引用之前不能被回收。所以同时可能有:

- CurrentData(本帧刚完成)
- 一个正被 RT 渲染(还持有 read ref)
- 一个即将被 BeginSimulate 拿来写

### 4.5 并发保护:`FNiagaraSharedObject`

`FNiagaraDataBuffer` 继承 `FNiagaraSharedObject`,用一个原子 int `ReadRefCount` 做保护:

```cpp
// NiagaraDataSet.h:58-87
FORCEINLINE bool IsBeingRead() const  { return ReadRefCount.Load() > 0; }
FORCEINLINE bool IsBeingWritten() const { return ReadRefCount.Load() == INDEX_NONE; }

// 只有 count==0 时才能拿写锁
FORCEINLINE bool TryLock() {
    int32 Expected = 0;
    return ReadRefCount.CompareExchange(Expected, INDEX_NONE);
}
```

`INDEX_NONE`(-1)做"写锁"特殊值。`Destroy()` 不立即析构——把对象放进 `DeferredDeletionList`,`FlushDeletionList()` 在安全时机(RT 不再引用)真正删。

### 4.6 Persistent ID:双整数保护(问题 5)

```cpp
USTRUCT(BlueprintType, meta = (DisplayName = "Niagara ID"))
struct FNiagaraID {
    int32 Index;        // 稠密 index(死了就复用)
    int32 AcquireTag;   // 获得时的 tick 标签
};
```

为什么要双整数?因为 **Index 会被粒子复用**(A 粒子死了,它的 Index 可能下帧被 B 粒子用)。如果 DI 在上帧缓存了 `FNiagaraID { Index=5, AcquireTag=100 }`,下帧发现 Index=5 但 AcquireTag 是 101,说明**这已经是别的粒子了**,缓存失效。

DataSet 管理相关字段:

```cpp
TArray<int32> FreeIDsTable;     // 可分配的 Index 池
int32 NumFreeIDs;
int32 MaxUsedID;                // 最高水位,方便缩表
int32 IDAcquireTag;             // 本 tick 新 ID 用的 tag
TArray<int32> SpawnedIDsTable;  // 本 tick 新 spawn 的 Index
```

**性能代价**:`bRequiresPersistentIDs` 是 Emitter Asset 字段,**默认关**。开了之后,每帧需要维护 FreeIDs/IDToIndexTable,GPU 侧还有 `GPUIDToIndexTable` / `GPUFreeIDs` 额外 buffer。只有需要按 ID 跨帧追踪个别粒子时才开(比如 ribbon 的"同一粒子连成带")。

### 4.7 CPU 与 GPU 的数据对偶

同一份 DataSet 可以 CPU 模拟或 GPU 模拟(由 `CompiledData->SimTarget` 指定):

- CPU:数据在 `FloatData/Int32Data/HalfData` byte blob
- GPU:数据在 `GPUBufferFloat/Int/Half`(`FRWBuffer`,即 RHI 的 UAV/SRV)

只有 `FScopedNiagaraDataSetGPUReadback`(editor-only,stall CPU)能同步读回。

线程检查:

```cpp
// NiagaraDataSet.h:395-405
void CheckCorrectThread() const {
    bool CPUSimOK = (SimTarget == CPUSim && !IsInRenderingThread());
    bool GPUSimOK = (SimTarget == GPUComputeSim && IsInRenderingThread());
    checkfSlow(CPUSimOK || GPUSimOK, "wrong thread");
}
```

---

## 5. 预定义词汇表:`NiagaraConstants.h`

Niagara 脚本面板里你看到的 `Engine.DeltaTime` / `Particles.Position` / `Emitter.Age` 都**不是字符串字面量**,而是注册在 `FNiagaraConstants` 里的 `FNiagaraVariable` 全局单例。

### 5.1 参数命名空间

```
User.          ← 用户暴露参数(Component OverrideParameters)
Engine.        ← 引擎驱动(Time, Delta, Quality)
Engine.Owner.  ← Component 的 Transform/Velocity
Engine.System. ← System 级引擎值(Age, TickCount, NumInstances)
Engine.Emitter.← Emitter 级引擎值(NumParticles)
System.        ← System 脚本可写
Emitter.       ← Emitter 脚本可写
Particles.     ← 粒子级属性
Module.        ← Module 本地
Initial.       ← 粒子初始值
Previous.      ← 上帧值
Constants.     ← RapidIteration 参数(编辑调试)
Array.         ← 数组索引
```

命名空间不仅是命名约定——Niagara 编译器根据前缀决定变量的**存储位置**(ParticleDataSet / SystemParameters / EmitterParameters / UserOverride...)。

### 5.2 约 70+ 个预定义变量

`SYS_PARAM_ENGINE_*` / `SYS_PARAM_EMITTER_*` / `SYS_PARAM_PARTICLES_*` 三族宏展开为静态 getter。Particle 族最多(约 40 个):Position/Velocity/Color/Lifetime/NormalizedAge/Scale、Sprite 相关(SpriteRotation/SpriteSize/SpriteFacing/SpriteAlignment/SubImageIndex)、Mesh(MeshOrientation)、Light(LightRadius/Exponent/Enabled)、Ribbon(RibbonID/RibbonWidth/RibbonTwist/RibbonFacing…)、DynamicMaterial(DynamicMaterialParam×4)、其他(UniqueID/MaterialRandom/VisibilityTag/ComponentsEnabled)。

### 5.3 FNiagaraConstants 的静态方法

```cpp
static const TArray<FNiagaraVariable>& GetEngineConstants();
static const TArray<FNiagaraVariable>& GetCommonParticleAttributes();
static FNiagaraVariable GetAttributeAsParticleDataSetKey(const FNiagaraVariable&);
static FNiagaraVariableAttributeBinding GetAttributeDefaultBinding(const FNiagaraVariable&);
static bool IsEngineManagedAttribute(const FNiagaraVariable&);
```

Editor 面板显示 "Engine.DeltaTime"、编译器查 "这个名字对应的类型是 float" 都走本表。

---

## 6. 类型安全访问:`FNiagaraDataSetAccessor` 家族

C++ 代码(比如 Renderer、DI 的 per-instance 数据处理)要读 DataSet 的粒子属性,不想每次都手动算 offset。模板家族提供**类型安全 accessor**:

```cpp
// 典型用法
FNiagaraDataSetAccessor<FVector> PositionAccessor(DataSet, FName("Position"));
auto Reader = PositionAccessor.GetReader(DataSet);

for (int32 i = 0; i < Reader.NumInstances; ++i) {
    FVector Pos = Reader[i];  // O(1),直接从 ComponentData 数组索引
    // ...
}
```

内部三件事:
1. **构造时**:按 `FName("Position")` 找到 `DataSetCompiledData.Variables` 里的条目,拿到 `FloatComponentStart`
2. **构造时**:把 FVector 的 3 个 component 基址记到 `ComponentData[3]`
3. **运行时 `Reader[i]`**:按模板 `TypeInfo<FVector>::NumElements=3` 循环,从 `ComponentData[k][i]` 组装 FVector

### 6.1 三家族分工

| 家族 | Reader | Writer |
|---|---|---|
| Float(含 Half 自动降级) | 有 | **被注释掉,C++ 不能写** |
| Int32(含 Bool、ExecutionState) | 有 | 有 |
| Struct(SpawnInfo、NiagaraID) | 有 | 无 |

Float writer 注释掉有意思——设计选择 "粒子 float 属性都由 VM 脚本写"。C++ 只在 DI 的内部数据里写 int,不直接改粒子 position。

### 6.2 Half 自动降级

```cpp
FNiagaraDataSetReaderFloat<FVector> Reader = Accessor.GetReader(DataSet);
FVector V = Reader[i];
// 内部:如果 DataSet 里 "Position" 是 half 存的,LoadHalf→float;否则直接读
```

Half 支持列表:`float / Vec2 / Vec3 / Vec4`——Color/Quat 不支持(check(false)阻止,精度要求高)。

### 6.3 用户扩展

想给自己的 UStruct 加 accessor?特化 TypeInfo:

```cpp
template<>
struct FNiagaraDataSetAccessorTypeInfo<FMyStruct> {
    using TAccessorBaseClass = FNiagaraDataSetAccessorStruct<FMyStruct>;
    static constexpr int32 NumFloatComponents = ...;
    static constexpr int32 NumInt32Components = ...;
    static FNiagaraTypeDefinition GetStructType() { return FNiagaraTypeDefinition(FMyStruct::StaticStruct()); }
    static FMyStruct ReadStruct(int32 Idx, const float*const* FloatComps, const int32*const* IntComps) { ... }
};
```

---

## 7. 参数存储:`FNiagaraParameterStore`

最后一层。回答问题 6、7。

### 7.1 为什么 editor 有两套(`FNiagaraParameters` vs `FNiagaraParameterStore`)

- **`FNiagaraParameters`**(`NiagaraParameters.h`,82 行):纯 editor-only 的 `TArray<FNiagaraVariable>` 包装,编辑期间 汇总/合并参数用。代码很诚实地 TODO:"Sort + binary search,不改 TMap 避免内存膨胀"
- **`FNiagaraParameterStore`**(`NiagaraParameterStore.h`,1260 行):运行时正牌,支持绑定图 / 三路 dirty / 多类型管理 / DI 实例化

两者不是兼容关系。`FNiagaraParameters` 是编辑期产物,编译时会"平铺"成 `FNiagaraParameterStore` 的初始数据。这是**遗留技术债**——理论上可以统一,但 editor 用不上 Store 的全部能力,保留两套反而简单。

### 7.2 ParameterStore 的三类管理

同一个 "offset" 可能指向三种不同类型的存储:

```cpp
TArray<uint8> ParameterData;                      // 参数 byte blob(类似 DataSet 但不分 Float/Int 三路)
TArray<UNiagaraDataInterface*> DataInterfaces;    // DI 对象(Phase 7)
TArray<UObject*> UObjects;                        // 其他 UObject(比如 Material)
TArray<FNiagaraVariableWithOffset> SortedParameterOffsets;  // 排序后的变量查找
```

一个变量的类型决定它存哪——`GetType().IsDataInterface()` 就去 `DataInterfaces[Offset]`,`IsUObject()` 就去 `UObjects[Offset]`,其他去 `ParameterData` 的 byte 位置。

### 7.3 绑定图(Binding)

```cpp
TArray<BindingPair> Bindings;                  // 我 push 到 Dest
TArray<FNiagaraParameterStore*> SourceStores;  // 谁 push 给我
```

`FNiagaraParameterStoreBinding` 结构体内含三种 binding(16-bit offset 压缩):

```cpp
TArray<FParameterBinding> ParameterBindings;   // 普通参数 byte-to-byte
TArray<FInterfaceBinding> InterfaceBindings;   // DI 索引映射
TArray<FUObjectBinding> UObjectBindings;
```

典型链路:

```
UNiagaraComponent::OverrideParameters (FNiagaraUserRedirectionParameterStore)
    ↓ Bind
FNiagaraSystemInstance::InstanceParameters
    ↓ Bind
FNiagaraSystemSimulation::ScriptDefinedDataInterfaceParameters
    ↓ Tick push
VM Script 执行时的 constant buffer
```

每个 store `Tick()` 被调用时,按 dirty 标记**增量** push 数据到所有 Bindings。

### 7.4 三路 dirty 的存在意义(问题 7)

```cpp
uint32 bParametersDirty : 1;    // 普通参数改了
uint32 bInterfacesDirty : 1;    // DI 换了
uint32 bUObjectsDirty : 1;      // UObject 换了
```

每帧 Tick 里,如果**只改了一个参数**,没必要重传 200 个 DI 指针。三路 dirty 分开让 push 代价最小化。

### 7.5 常用 API 速查

- 参数增删:`AddParameter / RemoveParameter / RenameParameter / Empty / Reset`
- 查找:`IndexOf / FindParameterOffset / FindVariable(反向,给 DI 找 Variable)`
- 读值:`GetParameterValue<T>` 模板 / `GetParameterData(Offset)` 裸 byte / `GetDataInterface` / `GetUObject`
- 绑定:`Bind / Unbind / UnbindAll / Rebind / TransferBindings / Tick`
- 迁移:`InitFromSource / TransferBindings`(实例迁移 / Solo-Batched 切换)

### 7.6 子类

- `FNiagaraUserRedirectionParameterStore`(不在本文件)——Phase 2 `UNiagaraComponent::OverrideParameters` 的类型,**自动给参数加 `User.` 前缀**
- `FNiagaraScriptExecutionParameterStore`(Phase 5)——VM 执行上下文特化

---

## 8 条关键洞察

1. **Niagara 有自己的类型系统**,不直接用 UStruct。`FNiagaraTypeDefinition` 能指向 UStruct/UEnum/UClass/内置基础类型。这是为了把任何类型**拍成 Float/Int32/Half 三路 component** 以适配 VM 和 GPU compute shader
2. **`FNiagaraBool` 不是 bool,是 int32**,编码约定 `True=-1 / False=0`——VM compare+select 指令友好
3. **SoA 布局的具体形态**:`FNiagaraDataBuffer::FloatData` 是一大块 `uint8`,按 component 为主序组织(所有粒子的 Position.x 连续 → 所有 Position.y 连续 → ...),`FloatStride = NumInstancesAllocated × 4`
4. **Double buffer + 池化**:DataSet 持有 2-3 个 DataBuffer,`CurrentData` 供 VM 读,`DestinationData` 供 VM 写,RT 正在读的那个单独保留——避免写锁等待
5. **`FNiagaraSharedObject`** 用原子 `ReadRefCount`(`INDEX_NONE` 作写锁特殊值)+ 延迟删除队列解决 GT/RT 并发
6. **Persistent ID 双整数**(Index + AcquireTag)是性能与正确性的权衡——只有需要跨帧追踪个别粒子(ribbon、component renderer)时才开,其他时候 Index 即索引
7. **参数命名空间系统**(`User.` / `Engine.` / `Particles.` 等)不只是命名约定,它决定变量的**存储位置**(哪个 ParameterStore / 哪个 DataSet)
8. **三路 dirty 标记**(参数 / DI / UObject)让 ParameterStore 的 Tick 增量推送——改一个参数不重传 200 个 DI 指针

---

## 自检问题(读完回答)

下面这些题需要把"类型系统三路分拣 + SoA 布局 + 命名空间 + 双缓冲 + Persistent ID"全部内化才能答得准。

1. **`FNiagaraBool` 编码的 SIMD 收益**:为什么是 True=-1 / False=0,而不是 True=1 / False=0?试推一段 `bool ? a : b` 在两套编码下的字节码序列——前者能省哪条指令(提示:bit-level mask & select)?为什么这个"小"决定能影响 VM 整体吞吐?
2. **内存帐**:一个 Emitter 的粒子有 12 个 float 属性 + 4 个 int 属性,`NumInstancesAllocated = 1024`。FloatData / Int32Data 各占多少字节?如果再加一个 `Position2: FVector`,字节量怎么变?如果把 Position 改用 Half,变多少?——这个帐能不能算清,决定了你能不能合理预估 GPU buffer 上限。
3. **DataBuffer 池为什么是 2-3 个**:理论上 double buffer 2 个就够,第 3 个在 RT 渲染时持 read ref。如果硬要把池砍到 2 个,在什么并发场景下会撞到死锁/写阻塞?(提示:GT 模拟下一帧、RT 还在画上一帧的同时性)
4. **Persistent ID 的总成本**:假设你把 `bRequiresPersistentIDs` 给所有 Emitter 默认开启,内存(`FreeIDsTable / IDToIndexTable / GPUIDToIndexTable / GPUFreeIDs`)、tick CPU(每帧维护 free list)、GPU buffer(额外两块)各多多少?这个帐解释了为什么默认必须关。
5. **命名空间 vs 存储位置的耦合**:有人在脚本里写 `User.MyAttr` 想给每个粒子赋值。Niagara 编译器/运行时会怎么响应?(命中错误 / 静默忽略 / 退化为 emitter 级单值 全都可能)——回答能否解释 §5.1 那张命名空间表里"前缀决定存储"的真正含义。
6. **三路 dirty 合并的损失**:ParameterStore 把 dirty 拆成参数 / DI / UObject 三路,每路 1 bit。如果合并为单路 dirty bit,哪些"局部修改"场景会退化成"全量重传"?把这个想清楚就懂了为什么三路是 minimum split。
7. **类型系统的拍平极限**:`FNiagaraTypeLayoutInfo` 把任何类型拆成 Float/Int32/Half 三路 component。如果你要在脚本里用 `FString`(变长),Niagara 能拍平吗?如果不能,根本障碍是什么?——这个反例能让你看清"三路分拣"成立的前提条件。
8. **DataSet vs ParameterStore 的本质差异**:两者都"存数据 + 按 offset 取",看起来很像。但一个是 SoA(每属性一个连续 component 数组),一个是 byte blob(各参数挨着排)。为什么粒子数据用 SoA 而参数用 byte blob?——把 SIMD 访问模式 vs 单值访问模式想清楚。

---

## Phase 4 留下的问题

- `FNiagaraScriptExecutionContext` 怎么用 ParameterStore 构建 VM constant buffer → **Phase 5**
- VectorVM 怎么用 DataBuffer 的 RegisterTable 跑粒子计算 → **Phase 5**
- DI 具体长什么样,per-instance 数据怎么分配到 `DataInterfaceInstanceData` blob → **Phase 7**
- GPU 侧 `GPUBufferFloat/Int/Half` 的 upload / readback 路径 → **Phase 8**
- `FNiagaraGPUInstanceCountManager` 的 count buffer 全局结构 → **Phase 8**
- `NiagaraTypes.h` 后半(800 行未完整扒)里 `FNiagaraVariableMetaData` / `FNiagaraVariableAttributeBinding` / 编译 visitor 细节 → 按需 offset 读

## 下一步预告

**Phase 5**:CPU 脚本执行。终于要打开 VectorVM——Niagara 脚本字节码如何在 CPU 上 dispatch:

- `NiagaraCore.h` — 模块基类
- `NiagaraDataInterfaceBase.h` — DI 基类(留给 Phase 7 详细)
- `NiagaraScriptExecutionContext.h` — **核心**,VM 执行上下文
- `NiagaraScriptExecutionParameterStore.h` — VM 参数存储特化
- `NiagaraEmitterInstanceBatcher.h`(CPU 侧)

难度 ⭐⭐⭐⭐。

---

## 深入阅读

### 本议题的原子页

- 源摘要 × 7:
  - [[Wiki/Sources/Stock/Niagara/NiagaraTypes]] / [[Wiki/Sources/Stock/Niagara/NiagaraCommon]]
  - [[Wiki/Sources/Stock/Niagara/NiagaraConstants]]
  - [[Wiki/Sources/Stock/Niagara/NiagaraDataSet]] / [[Wiki/Sources/Stock/Niagara/NiagaraDataSetAccessor]]
  - [[Wiki/Sources/Stock/Niagara/NiagaraParameters]] / [[Wiki/Sources/Stock/Niagara/NiagaraParameterStore]]
- Entity × 7:[[Wiki/Entities/Stock/Niagara/FNiagaraTypeDefinition]] / [[Wiki/Entities/Stock/Niagara/FNiagaraVariable]] / [[Wiki/Entities/Stock/Niagara/FNiagaraTypeLayoutInfo]] / [[Wiki/Entities/Stock/Niagara/FNiagaraConstants]] / [[Wiki/Entities/Stock/Niagara/FNiagaraDataSet]] / [[Wiki/Entities/Stock/Niagara/FNiagaraDataSetAccessor]] / [[Wiki/Entities/Stock/Niagara/FNiagaraParameterStore]]

### 前置议题

- [[Readers/UE/Niagara/Phase 0 - 上阵前的四层脑内地图]] — Asset/Instance、UObject 前缀
- [[Readers/UE/Niagara/Phase 1 - 从 System 到图源抽象基类]] — `FNiagaraVariable` 在资产里的存储
- [[Readers/UE/Niagara/Phase 2 - Component 层的五职责]] — `FNiagaraUserRedirectionParameterStore` 在 Component 里的暴露
- [[Readers/UE/Niagara/Phase 3 - Niagara 的心脏]] — ParameterStore/DataSet 反复出现的消费者
- [[Wiki/Concepts/UE/Niagara/Niagara-cpu-vs-gpu模拟]] — Half 类型降级的动机

### 下一步 / 导航

- 下一阶段:[[Readers/UE/Niagara/Phase 5 - Niagara 脚本如何跑起来]] — CPU 脚本如何消费本 Phase 的数据结构
- 学习路径总图:[[Wiki/Syntheses/UE/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 4 的 7 个头文件(合计 5663 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
