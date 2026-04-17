---
type: entity
category: struct
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, data-layer]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSpecData.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectSpecData, FEffectSpecChildData]
---

# FEffectSpecData(新版配置数据元)

> New 系统的**最轻量数据层**——替代了"从 DataAsset 直接读字段"的模式,提供一张**全局特效索引表**,每条记录是一个小结构体。业务启动时一次加载,运行时只按 `int32 Id` 查询。

## 整体文件内容(只有 17 行)

```cpp
#pragma once

namespace KuroEffect
{
    struct FEffectSpecData
    {
        int32 Id;
#if WITH_EDITORONLY_DATA
        FName Path;      // ← 只有编辑器构建有 Path(运行时不需要)
#endif
        uint8 SpecType;         // EEffectSpecDataType 的 cast(见 EffectDefine)
        uint8 EffectRegularType;
        float LifeTime;
    };

    struct FEffectSpecChildData
    {
        int32 Id;
        TArray<int32> Children;
    };
}
```

极简。没有 `UCLASS`/`USTRUCT` 反射,就是裸 struct。

## 为什么这么小

这是**数据索引**,不是数据本身。实际的 DA 字段(UNiagaraSystem、Curves、Materials、Params...)还在 [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelXxx]] 里。`FEffectSpecData` 只告诉你:

- `Id`:这条特效的**整数 Id**(比 Path 快很多的查找键)
- `Path` (仅编辑器):对应 DA 的路径,供开发时 debug
- `SpecType`:这条特效是什么类型(`EEffectSpecDataType_Niagara` 等 19 种,见 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine]])
- `EffectRegularType`:类型的"规制分类"(可能与距离剔除策略对应)
- `LifeTime`:**配置层面的**总生命周期(秒)

### 为什么把 Path 包在 `#if WITH_EDITORONLY_DATA` 里

这是 UE 里的常见优化。
- 编辑器构建:`Path` 存在,方便 debug / 日志输出
- Shipping 构建:**整条字段消失**,每个 FEffectSpecData 少 16 字节(FName 是 uint32*2)

如果有成千上万条特效,省下来的内存很可观。但代价是:**生产环境里你只能拿到 Id,不能从 SpecData 倒推 Path**。这也是为什么 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 有 `SpawnEffect(FName Path)` 入口——路径还在 `FEffectHandle::Path` 里保留。

## 如何被加载

回忆 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::Initialize]] 的签名:
```cpp
static bool Initialize(v8::Isolate*, UGameInstance*,
                       const TArray<FEffectSpecData>& SpecData,      // ← 这里
                       const TArray<FEffectSpecChildData>& SpecChildData,
                       bool IsGameRunning, ...);
```

业务启动时从 db / json 配置读出所有特效记录,一次性塞到 `FEffectSystem::EffectForSpecMap`:
```cpp
// 大致样子(.cpp,Batch 6 验证)
static TMap<int32, FEffectSpecData> EffectForSpecMap;
static TMap<int32, TArray<int32>> EffectForSpecChildMap;
static TMap<int32, FEffectSpecData> EffectForSpecDbCacheMap;
```

三个 Map 的用途:
- `EffectForSpecMap`:常驻,所有生效的特效 id→data
- `EffectForSpecChildMap`:每个特效的子特效 Ids(用于 Group 展开)
- `EffectForSpecDbCacheMap`:db 缓存,可能是"上一次加载的数据",用于 HotReload/Refresh 时 diff

配套 API:
```cpp
static bool HasEffectForSpecData();
static void RefreshEffectForSpecData(const TArray<FEffectSpecData>& SpecData,
                                      const TArray<FEffectSpecChildData>& SpecChildData,
                                      bool bRefresh);
static bool GetEffectSpecConfig(const FName& Path, FEffectSpecData& OutSpecData,
                                bool WithEditor = false);  // Path → Data 查询
```

**`GetEffectSpecConfig(Path, OutSpecData)`** 是反向查找接口——当业务只有 Path 时用它拿到 SpecData。

## FEffectSpecChildData(父子关系表)

```cpp
struct FEffectSpecChildData
{
    int32 Id;
    TArray<int32> Children;
};
```

- `Id`:父特效 id
- `Children`:所有子特效 ids

这张表对 **Group 类特效** 是关键——[[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelGroup]] 持有 `TMap<UEffectModelBase*, float> EffectData` 是引用形式,数据驱动下需要展开成 Id 列表。SpecChildData 就是这份"展开后的平面结构"。

为什么单独存一张表而不是把 `Children` 放进 `FEffectSpecData`?
- **数据局部性**:绝大多数特效是叶子(没有 children),把 `TArray<int32> Children` 塞进每个 SpecData 会让 SpecData 大小倍增且多数空置。
- **按需查找**:遍历所有特效做剔除判断时不需要 children,分表后主表紧凑。
- **加载次序解耦**:主表先加载,Child 关系表可以延后/按需加载。

## 实体关系示意

```
  Aki DB / Config json
          │
          ▼  (启动加载)
  TArray<FEffectSpecData>   ──►  FEffectSystem::EffectForSpecMap
  TArray<FEffectSpecChildData> ──► FEffectSystem::EffectForSpecChildMap

  业务调 SpawnEffect(Path)
          │
          ▼
  查表:Path → FEffectSpecData → SpecType
          │
          ▼
  FEffectSpecFactory 按 SpecType 构造 IEffectSpecBase 派生类 (Batch 3)
          │
          ▼
  FEffectHandle 持有 TSharedPtr<IEffectSpecBase>
          │
          ▼
  DA (UEffectModelXxx) 通过 Path 异步 LoadObject 后注入 Spec
```

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] **静态 Map 持有**
- **驱动** `FEffectSpecFactory`(Batch 3)按 `SpecType` 分派构造
- **类型枚举** 来自 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine::EEffectSpecDataType]](19 种)
- **`EffectRegularType`** 待确认来源(推测是另一个枚举,.cpp 回答)

## Twin(Old 版对应)

**Old 没有等价概念**——Old 直接:
- 从 TS 侧传 DataAsset 指针 `UEffectModelBase*` 给 `FKuroEffectSystem::RegisterEffectHandle`
- 在 RegisterEffectHandle 内部直接 `new` 对应类型的 `FEffectSpec*`
- 没有"配置表"集中化

**New 的改进**:
1. **路径→SpecType 一次查表,而非重复 `Cast<UEffectModelXxx>`**(后者慢)
2. **可以在没 load DA 的情况下就知道这个特效会是什么类型**(用于预分配、预估开销)
3. **支持热更新**:改配置表,`RefreshEffectForSpecData` 刷新所有特效元数据,不需要重新 load DA
4. **轻量,方便做统计和预加载**

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectSpecData.h`(17 行)
- [[sources/project-game/kuro-effect-system-new/overview|New 系统 overview]]

## 开放问题

- `EffectRegularType` 是哪个枚举?(可能是 `EKuroNiagaraEffectRegularType`,在 FEffectSystem 里提到过 `EffectRegularTypeMaxDistanceMap`)
- 配置数据从哪读?db 后端还是本地 json?(Batch 6 看 Module init)
- `GetEffectSpecConfig(Path, ..., bool WithEditor)` 里 `WithEditor` 参数什么意思?编辑器模式下查不同的表?
- `RefreshEffectForSpecData` 的 `bRefresh` 参数——是区分"全量替换"vs"增量更新"?
