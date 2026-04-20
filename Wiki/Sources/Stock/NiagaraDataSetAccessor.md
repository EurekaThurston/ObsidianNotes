---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, accessor, templates, type-safe]
sources: 1
aliases: [NiagaraDataSetAccessor.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSetAccessor.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraDataSetAccessor.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraDataSetAccessor.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 619 行(几乎全是模板特化)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.5

## 职责

C++ 代码**类型安全地读写** `FNiagaraDataSet` 的模板家族。典型用法:

```cpp
// 给 DataSet 里的 "Position" 字段建一个读取器
FNiagaraDataSetAccessor<FVector> PositionAccessor(DataSet, FName("Position"));
// 拿到 reader,逐粒子读
auto Reader = PositionAccessor.GetReader(DataSet);
for (int32 i = 0; i < Reader.NumInstances; ++i) {
    FVector Pos = Reader[i];
    ...
}
```

CPU 侧用得多(GPU 侧不用 Accessor,用 shader register)。Phase 6 Renderer / Phase 7 DI / Phase 10 SimStages 都会见到。

## 三大家族

按 C++ 类型分三组 accessor,每组内部是 `TypeInfo<T>` 特化 + `Accessor<T>` + `Reader<T>` + `Writer<T>`(部分):

### 1. Float 家族(L23-L286)

- `FNiagaraDataSetReaderFloat<T>`:读。支持 half 自动降级(若 DataSet 里是 half 存的,LoadHalf 转 float)
- `FNiagaraDataSetAccessorFloat<T>`:查找变量偏移 + 创建 reader
- 特化类型(TypeInfo):`float` / `FVector2D` / `FVector` / `FVector4` / `FLinearColor` / `FQuat`
- Half 支持:`float/Vector2D/Vector/Vector4` 支持(`bSupportsHalf=true`);`Color/Quat` **不支持**(check(false) 硬阻止)

代表方法:

```cpp
TType Get(int Index) const {
    if (!bSupportsHalf || bIsFloat) {
        // 直接从 float 数组读
    } else {
        // half→float 转换 LoadHalf
    }
}
TType GetSafe(int Index, const TType& Default) const;  // 带越界默认
float GetMaxElement(int32 ElementIndex) const;          // 遍历最大
void GetMinMaxElement(int32 ElementIndex, float&, float&) const;
```

一个小的注释说明:**Writer 被注释掉**(`FNiagaraDataSetWriterFloat` 在 L161 注释块)——float 的 writer 被显式未启用,只能写 int。设计意图可能是"粒子 float 属性都由 VM 脚本写",C++ 不提供 setter。

### 2. Int32 家族(L291-L462)

- `FNiagaraDataSetReaderInt32<T>` / `FNiagaraDataSetWriterInt32<T>`(**有 writer,与 Float 不同**)/ `FNiagaraDataSetAccessorInt32<T>`
- 特化:`int32` / `FNiagaraBool` / `ENiagaraExecutionState`

关键:

```cpp
FORCEINLINE void Set(int Index, const TType& Value) {
    for (int i = 0; i < NumElements; ++i) {
        reinterpret_cast<int32*>(ComponentData[i])[Index] = reinterpret_cast<const int32*>(&Value)[i];
    }
}
```

`FNiagaraBool` 的特化能直接写入 bool 编码(`True=-1 / False=0` VM 约定)。

### 3. Struct 家族(L464-L619)

- `FNiagaraDataSetReaderStruct<T>` + `FNiagaraDataSetAccessorStruct<T>`(无 writer)
- 特化:`FNiagaraSpawnInfo`(2 float + 2 int)/ `FNiagaraID`(0 float + 2 int)

结构体的读法靠 TypeInfo 提供的 `ReadStruct` 静态函数:

```cpp
// FNiagaraSpawnInfo 特化:
static FNiagaraSpawnInfo ReadStruct(int32 Idx, const float*const* FloatComps, const int32*const* IntComps) {
    FNiagaraSpawnInfo Info;
    Info.Count = IntComps[0][Idx];
    Info.InterpStartDt = FloatComps[0][Idx];
    Info.IntervalDt = FloatComps[1][Idx];
    Info.SpawnGroup = IntComps[1][Idx];
    return Info;
}
```

所以如果要加一个自定义 struct 类型的 accessor 读,就要 `FNiagaraDataSetAccessorTypeInfo<YourStruct>` 特化 + 提供 `ReadStruct` / `NumFloatComponents` / `NumInt32Components` / `GetStructType()` 四个成员。

## 统一入口

```cpp
// L12-L18
template<typename TType>
struct FNiagaraDataSetAccessor : public FNiagaraDataSetAccessorTypeInfo<TType>::TAccessorBaseClass
{
    FNiagaraDataSetAccessor<TType>() {}
    explicit FNiagaraDataSetAccessor<TType>(const FNiagaraDataSet& DataSet, const FName VariableName) {
        Init(DataSet.GetCompiledData(), VariableName);
    }
};
```

通过 TypeInfo 特化选基类 —— 用户只写 `FNiagaraDataSetAccessor<FVector>` 就行。

## 线程 / 性能约定

- Reader 在 `GetCurrentData()` 上建立(读完整一帧)
- Writer 在 `GetDestinationData()` 上建立(BeginSimulate/EndSimulate 之间)
- 构造时做一次"按 FName 查变量,拿 ComponentIndex",之后 `Get(Index)` 就是 O(1)

## 涉及实体

- [[Wiki/Entities/Stock/FNiagaraDataSetAccessor]] — 模板家族总称
- [[Wiki/Entities/Stock/FNiagaraDataSet]] — 操作对象
- [[Wiki/Entities/Stock/FNiagaraVariable]] — 按变量名查找

## 开放问题

- Float Writer 被注释掉的确切设计意图 → 翻 git blame
- SpawnInfo 的 float component 顺序(ReadStruct 里 `FloatComps[0]`=InterpStartDt / `[1]`=IntervalDt)与字段声明顺序一致吗?→ 看 `FNiagaraSpawnInfo` USTRUCT 字段顺序 + Layout 生成规则
- 为什么 Color/Quat 不支持 half?→ Color 精度需要,Quat 归一化损失敏感;实际决策可能是保守
