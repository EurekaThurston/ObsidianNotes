---
type: synthesis
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, synthesis, architecture]
sources: 2
repo: project-game
source_root: project-game
aliases: [Old vs New Effect System, EffectSystem 架构对比]
---

# KuroEffectSystem — 老版 vs 新版架构综合

> **这个文件回答一个问题:**为什么团队要重写 EffectSystem?
>
> 基于 Batch 1-6 对全部 public headers + 部分 private 实现的阅读,这是一份**架构级 diff**——不是行级代码对比,而是每个设计决策背后的"问题 → 解法"叙述。
>
> **核心结论**:New 不是"Old + 新功能",是**架构重写**,解决 Old 里深植的 5 个结构性问题。

## TL;DR(1 分钟版本)

| Old 的问题 | New 的解法 |
|---|---|
| **JS 耦合贯穿全链**(Handle/Info/LifeTime/Spec 都含 `v8::Isolate*`) | **ScriptBridge 层 + Holder 抽象**,Spec/Handle/LifeTime 零 v8 引用 |
| **裸指针所有权**(`FEffectHandle*` / `FEffectSpecBase*` / `FEffectHandleInfo*`) | **TSharedPtr / TWeakPtr / TStrongObjectPtr / TUniquePtr** 精细所有权,`TSharedFromThis` |
| **类型化硬编码**(7 个 `RegisterEffectHandle` 重载) | **数据驱动**:`FEffectSpecData` + `FEffectSpecFactory::CreateEffectSpec(Data)` 按 SpecType dispatch |
| **同步 Spawn**(LoadObject 阻塞 / 无 Pending 态) | **4 阶段异步管线**:Spawn → LoadData → Init → Play,`FEffectInitHandle` Pending 缓存 |
| **无联机支持**(单玩家特效假设) | **FPlayerEffectContainer** 每玩家一池 + **FContinuousEffectController** 连续技过渡 + C# Bridge |

## 规模对照

| 指标 | Old | New |
|---|---|---|
| Public 头 | 43 文件 | 42 文件(加 3 子目录) |
| Public 行数 | ~5k | ~10k |
| Private 实现 | 13 文件 | 23 文件 |
| Private 行数 | ~4k | ~13k |
| **总代码量** | ~9k 行 | **~23k 行**(2.5x) |
| 入口类 | `FKuroEffectSystem`(~370 行) | `KuroEffect::FEffectSystem`(~800 行) |
| 核心 Handle | `FEffectHandle`(~230 行) | `FEffectHandle`(~265 行,但字段数 2.5x) |
| Spec 接口 | `FEffectSpecBase`(6 纯虚) | `IEffectSpecBase`(80+ 纯虚) |
| DA 子类支持 | 12 个(5 个 DA 无 Spec) | 17 个(全覆盖,4 个"待兼容") |
| 运行时 CVars | 0 | 15+ |
| Context 类型 | 散落在 Handle 字段 | 4 类 C++ + 4 类 USTRUCT |
| 脚本宿主 | 1(Puerts) | 2(Puerts + C#) |

**New 代码量是 Old 的 2.5 倍**。这不是"功能膨胀",而是**基础设施被显式化**——Old 里隐藏在"默契约定"里的逻辑,New 写成了显式代码。

## 五大结构性改变

### 改变 1:JS 解耦(核心!)

#### Old 的问题

`v8::Isolate*` 类型作为参数**出现在**:
```cpp
// FKuroEffectSystem:
void SetIsStopping(v8::Isolate*, int HandleId, bool);
void EffectSeekTo(v8::Isolate*, int HandleId, float, bool);
void RegisterEffectJsObject(int HandleId, v8::Isolate*, const v8::Local<v8::Object>&);

// FEffectHandle:
void Tick(v8::Isolate*, float);
void SeekTo(v8::Isolate*, float, bool);
void RegisterJsObject(v8::Isolate*, v8::Local<v8::Object>);

// FEffectHandleInfo:
void PreStop(v8::Isolate*);
void UpdateLifeCycle(v8::Isolate*, float);

// FEffectSpecBase:
virtual void Tick(v8::Isolate*, float DeltaSeconds) = 0;

// FEffectLifeTime:
void Tick(float, v8::Isolate*);
void SetIsStopping(v8::Isolate*, bool);
```

**v8 污染贯穿 5 个类**。想加 C# 支持?必须把这些签名全改,类型系统都会炸。

#### New 的解法
`v8::Isolate*` **只出现在 2 处**:
1. `FEffectSystem::Initialize(v8::Isolate*, ...)` — 启动时接收
2. `FEffectSystemScriptBridge::Initialize(v8::Isolate*)` — 传给 Bridge 子系统

**其余所有类零 v8 引用**。脚本回调被 **Holder 模式** 吸收:
- 业务注册 JS 函数 → `JsBridge::CreateSpawnCallbackHolder(Isolate, ...)` 创建 Holder
- Holder 内部存 `v8::Global<v8::Function>`
- Holder 暴露 `TSharedPtr<FEffectInitCallback>`(UE 原生 Delegate)
- 主系统拿到 `TSharedPtr<FEffectInitCallback>`,调用时 Holder thunk 反过来调 JS

**结果**:主系统测试可以脱离 v8 跑(只要绑 C++ 原生 callback),C# 支持是**在 JS 旁边平行加一条路径**,不改主系统。

### 改变 2:所有权精细化

Old 的典型所有权:
```cpp
// FKuroEffectSystem 里
TMap<int, FEffectHandle*> ProxyTickHandleMap;   // 裸指针

// FEffectHandle 里
FEffectSpecBase* EffectSpec = nullptr;           // 裸指针
FEffectHandleInfo* EffectHandleInfo = nullptr;   // 裸指针

// FEffectModelGroupSpec 里
TArray<FEffectSpecBase*> Children;               // 裸指针!
```

**问题**:
- 生命周期靠"注册先后"约定
- 删除时必须人肉确保没人持着
- GC 和 ABA 风险:Handle 回池后同 slot 给别人用,别人还持着旧 Id

New 的所有权设计:
```cpp
// FEffectSystem 里
TArray<TSharedPtr<FEffectHandle>> Effects;                        // 强持有
TMap<int32, TSharedPtr<FEffectHandle>> TickHandleMap;            // tick 子集
TLru<FName, FEffectHandle> LruPool;                              // 池
TSet<TSharedPtr<FEffectHandle>> PendingDeleteHandles;            // 待删

// FEffectHandle: TSharedFromThis<FEffectHandle>
TSharedPtr<IEffectSpecBase> EffectSpec;                          // 共享
TUniquePtr<FEffectContext> Context;                              // 独占
TStrongObjectPtr<AActor> EffectActor;                            // UObject 强持
TWeakPtr<FEffectHandle> Parent;                                  // 弱(避免循环)
TUniquePtr<FEffectInitHandle> InitCache;                         // Pending 独占

// FEffectModelGroupSpec 里
TMap<int32, TWeakPtr<FEffectHandle>> EffectSpecMap;              // 弱(parent 不强持 child)

// FEffectSpec<T> 里
TWeakPtr<FEffectHandle> Handle;                                   // 弱(Spec 不强持 Handle)
TStrongObjectPtr<T> EffectModel;                                  // UObject 强持
```

每种所有权**含义明确**:
- **TSharedPtr**:可多方共享,引用计数控制
- **TWeakPtr**:观察者,不参与引用计数(避免循环)
- **TStrongObjectPtr**:C++ 对 UObject 的强持(防 GC)
- **TWeakObjectPtr**:UObject 弱(随 GC 失效)
- **TUniquePtr**:独占,move semantics
- **TSharedFromThis**:类本身能拿到自己的 TSharedPtr

ID 位编码配合(Batch 6):`Version << IndexDigit | Index`,slot 复用时 Version++,旧 Id 持有者查询立刻"失效"。

### 改变 3:数据驱动构造

Old:**每种 DA 类型专用 `Init` 签名**
```cpp
// FEffectHandle 有 7 个 Init 重载
bool Init(int, int, UEffectModelBase*, AActor*);
bool Init(int, int, UEffectModelBase*, AActor*, UActorComponent*);
bool Init(int, int, UEffectModelGroup*, AActor*, USceneComponent*, TArray<int>);
bool Init(int, int, UEffectModelGhost*, AActor*, USkeletalMeshComponent*, float, float);
bool Init(int, int, UEffectModelPostProcess*, AActor*, UKuroPostProcessComponent*, bool, int, UMaterialInstanceDynamic*, FName);
bool Init(int, int, UEffectModelStaticMesh*, AActor*, UStaticMeshComponent*, TArray<UMaterialInstanceDynamic*>);
bool Init(int, int, UEffectModelTrail*, AActor*, UKuroBezierMeshComponent*, USkeletalMeshComponent*, UMaterialInstanceDynamic*);

// FKuroEffectSystem 有 7 个 RegisterEffectHandle 重载,对应每个 Init
```

**加一种新类型 = 改 Handle + 改 System + 改 Spec 家族,3 处改**。

New:**统一 Path + SpecData**
```cpp
// FEffectSystem::SpawnEffect 只有 1 个通用签名
static int32 SpawnEffect(UObject*, const FTransformDouble&, const FName& Path,
                         const FName& Reason, 
                         TSharedPtr<FEffectBeforeInitCallback>, TSharedPtr<FEffectInitCallback>, TSharedPtr<FEffectBeforePlayCallback>,
                         TUniquePtr<FEffectContext>&, 
                         EEffectType = Scene, bool Prepare = false, bool ForceCreateActor = false);

// FEffectHandle 构造只有一个:
FEffectHandle(const FName& Path);   // 只存 Path
void AfterConstruct(const FEffectSpecData& SpecData);   // 后阶段,走 Factory
```

类型路由通过 `FEffectSpecData.SpecType` 走 `FEffectSpecFactory`:
```cpp
switch (SpecData.SpecType) {
case EEffectSpecDataType_Niagara:      return MakeShared<FEffectModelNiagaraSpec>();
case EEffectSpecDataType_Group:        return MakeShared<FEffectModelGroupSpec>();
// ...
}
```

**加新类型 = 只改 Factory 一处 + 写新 Spec 类**。

### 改变 4:异步 Spawn 管线

Old:**同步**,LoadObject 阻塞
- 业务调 Spawn → 等 DA 加载完成 → 拿到 HandleId 返回
- 冷加载大 DA(`UEffectModelPostProcess` 700+ 字段)可能卡 1-2 帧

New:**4 阶段异步**
```
业务调 SpawnEffect(Path, Reason, 3 个 Callback, Context)
  │
  ├─ 阶段 1:构造 Handle(Pending 态),立即返回 HandleId
  │     Handle 持 TUniquePtr<FEffectInitHandle> InitCache
  │     业务可对 HandleId 调 SetHidden/Attach 等,被缓存到 InitCache
  │
  ├─ 阶段 2:异步 LoadObject<UEffectModelBase>(Path)
  │
  ├─ 阶段 3:Load 完成回调
  │     BeforeInitCallback(HandleId)        业务最后改写参数
  │     Handle->Init(EffectData, InitModel)
  │        Spec->OnInit(InitModel)          创建 Component 等
  │     EffectActorHandle.InitEffectActor    flush 所有 pending 动作
  │
  └─ 阶段 4:Play(if AutoPlay)
        BeforePlayCallback(HandleId)
        Handle->Play(Reason)
        Callback(Success, HandleId)          业务知道 Spawn 完成
```

关键基础设施:
- [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel]]:不可变的"业务意图"
- [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitHandle]]:可变的"进度"
- [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]]:Actor 未创建时的**命令队列**(`FEffectActorAction` 抽象 + 3 种具体 Action)

**效果**:业务侧的"Spawn"永远非阻塞;DA 加载失败/延迟都有明确路径处理。

### 改变 5:联机 + 多玩家支持

Old 的基本架构**假设单主控**:
- 所有 Handle 放一个 Map
- 一个全局 LRU
- 无 Entity/Owner 概念

New 为联机重设计:
- **FPlayerEffectContainer**:`TArray<TUniquePtr<TLru<FName, FEffectHandle>>>` 每玩家一个池(SCENE_TEAM_MAX_NUM ≈ 4)
- **FSceneTeamItem + OnFormationLoaded**:队伍变化广播,池重组
- **FEffectContext::EntityId**:每个特效溯源到 Entity
- **StopEffectFromEntity(EntityId)**:批量终结某 Entity 的所有特效
- **FContinuousEffectController**:连续技刀光平滑过渡(每 AnsSlot 一个活跃特效,新特效软停旧的)
- **OwnerEffect 同步**:两个 Handle 之间可注册 `SyncTimeScaleHandle`(主从同步)
- **FEffectSystemScriptBridge**:脚本侧判断 `EffectSystem_CheckIsNetPlayer(EntityId)` 等联机状态
- **HideForProtoPlayer**(DA 字段):联机队友视角下不可见
- **C# Bridge**:联机框架可能用 C# 写,Bridge 支持双语言

这些能力**在 Old 里要么没有、要么靠业务自己管**。

## 次要但有价值的改进

### CVar 可调性

Old:**0 个 CVar**。行为完全代码写死。
New:**15+ 个 CVar**,覆盖:
- 系统总开关(Enable)
- 删除策略(ForceDelete / Immediately / RemoveHandleImmediately)
- AdditionTimeScale 开关
- PostTick 的 DeltaTime==0 跳过
- BodyEffect 分离隐藏
- Niagara LocalTime
- 获取 NiagaraComponent 是否经过 Handle
- TickSystemPaused 广播
- 编辑器 Preview 无配置
- 日志详细程度

**线上问题应急**:不改代码,直接控制台 `Kuro.CppEffectSystem.Enable 0` 整个系统禁用。

### Tick 粒度细化

Old:`TG_PostUpdateWork` **单点** tick
New:`TG_PrePhysics` + `TG_PostPhysics` + `TG_PostUpdateWork` **三段** tick

意义:
- **PrePhysics**:物理前,业务逻辑推进
- **PostPhysics**:物理后,attach 到刚骨骼的特效位置已更新
- **PostUpdateWork**:帧末,清理/flush 推迟操作(Niagara WaitPostTick 在这)

### 生命周期回调扩展

Old:2 段(Play / Stop)
New:`BeforeInit → Init → BeforePlay → Finish` + `PreStop / Stop / Replay / Clear / Destroy / AfterLeavePool / OnEnabledChange`

**业务接入点**多很多——能拦截、能补参数、能做预/后处理。

### 编辑器 / PIE 钩子

Old:**无编辑器支持**。在编辑器里 Play-Stop 可能留残特效。
New:`OnBeginPIE / OnEndPIE / OnEditorCurrentMapFinishExit / JsEnvClean` 4 路 delegate,每次 PIE 进出清一次。VisibilityOptimizeController.SetActive 也随 PIE 切换。

## 一致性 / 复用

虽然大部分是重写,New 仍**复用 Old 的几个组件**:
- **DataAsset 家族** [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelBase 及 18 个子类]]——保留不动
- **`FKuroEffectVisibilityOptimizeController`**——New 直接 `#include "KuroEffectSystem/KuroEffectVisibilityOptimizeController.h"` 复用
- **`UKuroEffectLibrary`**——UE API 的 Kuro wrapper,双版本共用
- **`FKuroEffectNiagaraParameters`**(Old 版 standalone 类)——New 的 Handle 直接用 `TUniquePtr<FKuroEffectNiagaraParameters>`
- **Enum 和 Delegate**([[entities/project-game/kuro-effect-system/effect-define|EffectDefine.h]])——两版共享

**复用的原因**:这些是"数据配置"或"UE API 薄 wrapper",没有 JS 污染、没有架构债。改它们没价值。

## 历史迁移轨迹(猜测性)

从代码看到的线索拼凑:
1. **v1**:Old 系统(`KuroEffectSystem/`)——Aki 项目早期,Puerts 单脚本,单玩家
2. **v2**:开始联机后发现 Old 不够(没 Entity 追溯、没分池)
3. **v3**:团队决定重写,保留 DA 家族 + Visibility Controller,其他从头做
4. **v4 (当前)**:New 系统基本就绪,Old 冻结(最后修改 2026-04-10 仅限 bug fix)
5. **v5(未来)**:New 里的 "待兼容" Specs(MaterialController/SequencePose/NDC/CurveTrailDecal)完全实现后,Old 完全退役

## 未完成的工作(New 的 Technical Debt)

从代码里直接可见的"待完成":

### 1. 四个 "待兼容" Spec
`EffectSpecFactory.cpp` 注释:
```cpp
//待兼容
case EEffectSpecDataType_MaterialController:
    EffectSpec = MakeShared<FEffectModelMaterialControllerSpec>(); break;
case EEffectSpecDataType_SequencePose:
    EffectSpec = MakeShared<FEffectModelSequencePoseSpec>(); break;
case EEffectSpecDataType_NDC:
    EffectSpec = MakeShared<FEffectModelNDCSpec>(); break;
case EEffectSpecDataType_CurveTrailDecal:
    EffectSpec = MakeShared<FEffectModelCurveTrailDecalSpec>(); break;
```

"待兼容"可能意味着这些 Spec 类已经**写了但未完全调通**,或者某些分支在特定条件下才 fallback 到 Old。

### 2. 部分 Private .cpp 的未读行为
本 wiki 未读完:
- `EffectSystem.cpp` 3400 行未读
- `EffectHandle.cpp` 1926 行全未读
- Bridge 和 Holder 的实际 thunk 实现
- `EffectActorHandle.cpp` 的 Action 执行细节

### 3. 双 NiagaraComponentHandle 字段
[[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] 有两个同类型字段:
```cpp
TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandle;
TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandles;   // ← 单数 vs 复数!
```
—— 仍不明这两个字段的分工差异。

### 4. `EFFECT_REASON_LENGTH_LIMIT = 4` / `EFFECT_LIFETIME_FLOAT_TO_INT = 10000` / `CHECK_EFFECT_OWNER_INTERVAL = 60000` 等常量的业务含义未完全锁定

## 性能比较(推测)

| 操作 | Old | New |
|---|---|---|
| Spawn 冷启动 | 同步 LoadObject 可能卡帧 | 异步,立即返回 HandleId |
| Spawn 热启动(池命中) | 无池,总是新建 | LRU 池 + PlayerContainer 双池查 |
| Tick 单特效 | ~简单(直接 OnTick) | +Addition TimeScale 乘法 +Per-frame cache 判断 +状态检查 |
| JS 调用 1 次 | 直接 v8::Function::Call | 同样经过 HandleScope(Holder thunk 增加一层但可忽略) |
| Group 操作传播 | `for Child : Children` 裸遍历 | 查 TWeakPtr Map,每个 IsValid + Pin,稍贵 |
| LRU 容量超限 | N/A(无 LRU) | LruClearer → EndHandle + ClearHandle + 引用清理 |
| 同特效重复 Spawn | 每次都新建 | LruCreator 可能复用 Handle 对象 |

**权衡**:
- New 在每个特效的 Tick 成本略高(AdditionTimeScale、状态检查、NeedTick 保护)
- 但**整体 Spawn/Stop 成本大幅降低**(池化 + 异步)
- **内存开销**:New 有更多 TSharedPtr 管理,但 `TUniquePtr<TMap>` 的懒分配优化抵消一部分

## 业务迁移面对的实际问题

假设 Old 有业务代码:
```typescript
// Old TS
const handle = effectSystem.RegisterEffectHandle(id, parentId, niagaraDA, actor);
handle.RegisterCommonJsFunction(preStop, updateLifeCycle, enterStopping);
handle.SetTimeScale(0.5);
```

New 等价:
```typescript
// New TS
const handle = KuroEffectSystem.SpawnEffect(
    worldCtx, transform, "FX/Niagara/XXX",
    "BusinessReason",
    { EntityId: 42, ContextType: Base, ... },  // FKuroEffectContext
    beforeInitCb, initCb, beforePlayCb, onClearCb,
    EEffectType.Fight);
// 回调形态变了——不是 PreStop/UpdateLifeCycle 而是 BeforeInit/Init/BeforePlay
KuroEffectSystem.SetTimeScale(handle, 0.5);
```

**迁移难点**:
- API 形态变(类型化重载 → 通用 Spawn + Callback 风格)
- 回调语义变(生命周期点不完全对应)
- Handle 不再是 JS 对象(而是 int32 Id)
- 业务需要 refactor,不是简单的 find-replace

## 设计启示(**超出 Kuro 项目的可迁移经验**)

读完两版可以总结几个**一般性架构原则**:

### 1. 脚本边界的"Holder 模式"
跨语言边界永远是杂质聚集处。**把杂质集中在 Holder 类里**,内部用 `v8::Global` 或 C# Delegate,对外暴露**UE 原生类型**。这让主系统能假装只和 C++ 打交道。

### 2. 双版本类型(USTRUCT 反射 / 纯 C++ 运行时)
热路径永远不走反射。**USTRUCT 只作为边界类型**,跨过边界后 ctor 一次性转成纯 C++ 对象。New 对 `FEffectContext` / `FEffectSpecData` / `FKuroEffectNiagaraParameters` 都这么做。

### 3. Factory 集中化 + 数据驱动
当"新增一种类型"涉及改多处,**集中到 Factory**。`EEffectSpecDataType` + `FEffectSpecFactory::CreateEffectSpec` 把"加 Spec 类型"降到单点修改。

### 4. Pending State + Command Queue
异步操作(加载、网络)期间,不要"等"也不要"拒绝操作"。用 Pending 态 + 命令队列(FEffectActorHandle 的 `TUniquePtr<FEffectActorAction>`),操作累积,Ready 时 flush。

### 5. Dirty Flag + PostTick
每帧的昂贵操作(Niagara Pause、材质参数刷新)不要立即执行,标 Dirty,**帧末 PostTick 统一处理**。避免 Tick 中途状态不一致。

### 6. Per-Frame Cache via `GFrameCounter`
重复调用的计算(GetTotalAdditionTimeScale)用 `LastUpdateFrame == GFrameCounter` 做帧内缓存,多次调用 ≈ 一次成本。

### 7. CVar 从设计第一天开始
任何有**"开/关"**或**"阈值"**的行为,第一天就做 CVar。线上运维永远需要这些旋钮,事后加是痛苦的。

### 8. Reason/Source Code 贯穿
每个 Stop/Hide/Create/Destroy 调用带 **FName Reason**。log 可读性、debug 可追溯性、灰度分析都靠它。

## 引用来源

- **Old 系统**(p4 CL 6985991):
  - [[sources/project-game/kuro-effect-system/overview|Module overview]]
  - 10 个 entity 页(kuro-effect-system / effect-handle / effect-handle-info / effect-life-time / effect-model-base / effect-parameters / effect-define / effect-spec-base / effect-spec-subclasses)
- **New 系统**(p4 CL 7086024):
  - [[sources/project-game/kuro-effect-system-new/overview|Module overview]]
  - 16 个 entity 页(effect-system / effect-handle / effect-life-time / effect-spec-data / effect-init-pipeline / effect-context / effect-actor-handle / effect-spec-base / effect-spec-factory / effect-spec-subclasses / player-effect-container / continuous-effect-controller / effect-system-actor / niagara-component-handle / scripting-bridge-architecture / effect-system-script-bridge / effect-function-holders / kuro-effect-reflection / effect-bp-libraries)
- **综合**:[[entities/project-game/kuro-effect-system-new/implementation-insights|Private 实现洞察]](Batch 6 精读 .cpp)

## 后续扩展建议

这份综合本身是"足够的总览"。如果要继续深入,按优先级:
1. **读完 EffectHandle.cpp**(1926 行,未读)——解答 PendingInit 完整流程、OwnerEffect、BodyEffect 细节
2. **读完 EffectSystem.cpp**(3400 行未读)——解答 SpawnEffect 完整代码路径、RemoveHandle 队列机制、LruClearer 的详细处理
3. **读 Bridge/Holder .cpp**——解答 thunk 实现、v8/UE 反射 marshaling 细节
4. **测试具体业务用例**——在 Rider 里跑起来,观察 Handle Id 位编码的实际值、AdditionTimeScale 的叠乘 trace、PendingInit 卡住时的日志
