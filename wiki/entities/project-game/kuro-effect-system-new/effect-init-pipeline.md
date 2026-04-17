---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, init-pipeline]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectInitModel, FEffectInitHandle, Pending Init, 初始化管线]
---

# FEffectInitModel + FEffectInitHandle(新版初始化管线)

> New 系统的**异步初始化基础设施**。把"Spawn 一个特效"从"同步函数返回 Handle"变成"**Pending → Loaded → Initialized → Played** 的分阶段流水线"。这是 Old 所没有的核心新机制。

## 为什么需要这个管线

在 Old 系统里,Spawn 一个特效是同步的:
```cpp
// Old 的心智模型
UEffectModelXxx* DA = LoadObject<>(Path);   // 可能慢!阻塞
FEffectHandle* h = new FEffectHandle();
h->Init(Id, ParentId, DA, Actor, ...);
system.RegisterEffectHandle(Id, ..., DA, ...);
```

问题:
- `LoadObject` 是同步加载,从冷状态加载大 DA(比如 PostProcess 700 字段)可能卡 1-2 帧
- 没办法在"loading 中"就允许业务侧对 Handle 做操作(SetHidden、Attach、AddFinishCallback)
- 更糟:如果业务调 Spawn 后立即 Stop,Old 要么无视、要么走已初始化路径

New 用 **2 个 Holder 类** 表达这条流水线:
1. **`FEffectInitModel`**:一次 Spawn 的**不可变意图**——业务想干什么,回调怎么响应。
2. **`FEffectInitHandle`**:一次 Spawn 的**可变进度**——当前加载到哪步、还缺什么、临时抓着的引用。

## FEffectInitModel(意图)

```cpp
namespace KuroEffect
{
    class FEffectInitModel
    {
        bool AutoPlay = true;
        FName Reason;
        TSharedPtr<FEffectInitCallback> Callback;               // 成功/失败回调
        TSharedPtr<FEffectBeforePlayCallback> BeforePlayCallback;// Play 之前的钩子
        bool bHasHandleInitCallback = false;
        TFunction<void(const TSharedPtr<FEffectHandle>&, ELoadEffectResult, const TSharedPtr<FEffectInitModel>&)> HandleInitCallback;
    
    public:    
        FEffectInitModel(bool AutoPlay, const FName& Reason,
                         const TSharedPtr<FEffectInitCallback>& Callback,
                         const TSharedPtr<FEffectBeforePlayCallback>& BeforePlayCallback)
            : AutoPlay(AutoPlay), Reason(Reason),
              Callback(Callback), BeforePlayCallback(BeforePlayCallback) {}
    
        ~FEffectInitModel()
        {
            if (Callback.IsValid())           Callback.Reset();
            if (BeforePlayCallback.IsValid()) BeforePlayCallback.Reset();
            bHasHandleInitCallback = false;
            HandleInitCallback = nullptr;
        }
        /* 调用器 ... */
    };
}
```

### 回调调用器(FORCEINLINE)

```cpp
// 给业务的三个回调
FORCEINLINE void DoBeforePlayCallback(int32 EffectId) const;   // 会调 BeforePlayCallback
FORCEINLINE void DoCallback(ELoadEffectResult Result, int32 EffectId) const;  // 调 Callback

// 给 EffectSystem 内部的回调(把 Handle 传回去)
FORCEINLINE void DoHandleInitCallback(const TSharedPtr<FEffectHandle>&,
                                      ELoadEffectResult,
                                      const TSharedPtr<FEffectInitModel>&) const;

FORCEINLINE void SetHandleInitCallback(const TFunction<...>& InHandleInitCallback);
```

### 两种回调层级
- **`Callback` 和 `BeforePlayCallback`** → `TSharedPtr<FEffectInitCallback>` 是 UE Delegate(见 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine]]);**给业务的**
- **`HandleInitCallback`** → `TFunction<...>` 原生 C++ lambda/functor;**内部用的,系统给自己注册**

为什么区分?
- 业务只关心"结果 + EffectId",不需要看到 Handle 本身(脚本/UObject 不应持有 Handle)
- 系统内部需要 Handle 去做后续设置(Attach、SetContext 等)
- 分两个接口避免把内部类型泄漏给业务

### `AutoPlay` 的意义
- `true`(默认):Init 完成立刻 Play
- `false`:Init 完成留在"已准备好但没播"状态——典型场景是**预加载**(比如大招前一秒就 `Prepare=true` spawn,演出触发时再手动 Play)

### `Reason` 的意义
**每次创建都带一个 FName 原因码**,便于线上调试:
```
"SkillX_HitEffect" / "WeatherSystem_Rain" / "UltraSkill_Preload"
```
log 和 debug draw 里会显示,方便"为什么场里突然多了 500 个特效"这种排查。

## FEffectInitHandle(进度)

```cpp
namespace KuroEffect
{
    class FEffectInitHandle
    {
    public:
        FEffectInitHandle(UObject* InWorldContext, const FName& InPath, const FName& InReason,
                          const FTransformDouble& InTransform,
                          const TSharedPtr<FEffectBeforeInitCallback>& InBeforeInitCallback,
                          const TSharedPtr<FEffectInitCallback>& InCallback,
                          const TSharedPtr<FEffectBeforePlayCallback>& InBeforePlayCallback,
                          bool InAutoPlay = true);
        FVectorDouble GetLocation();
        
        TWeakObjectPtr<UObject> WorldContext;
        FName Path;
        FName Reason;
        bool AutoPlay = false;
    
        TSharedPtr<FEffectBeforeInitCallback> BeforeInitCallback;
        TSharedPtr<FEffectInitCallback>       Callback;
        TSharedPtr<FEffectBeforePlayCallback> BeforePlayCallback;
    
        FEffectActorHandle EffectActorHandle;     // ← 已经占好的 Actor 门面!
        double StartTime = -1;                    // 何时 Pending 的(FPlatformTime)
        float  TimeDiff = 0;                      // 已 Pending 多久
    };
}
```

**它是什么**:一个 Pending 态特效的"临时工作本"。[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 持有 `TUniquePtr<FEffectInitHandle> InitCache`,在 DA 异步加载期间,对 Handle 做的所有操作都被缓存到这里:

```cpp
// Handle 里定义(简化)
class FEffectHandle
{
    TUniquePtr<FEffectInitHandle> InitCache;
    float PendingInitTimeScale = 1.0f;
    bool  PendingInitIgnoreGlobalTimeScale = false;
    bool  PendingInitSetAdditionTimeScale = false;
    
    void PendingInit(...);                    // 启动 Pending
    void PlayEffectWhenPendingInit();         // 在 Pending 中被要求播放
    void InitEffectActorAfterPendingInit();   // 真正 Init Actor
    void PlayEffectAfterPendingInit();        // Pending 结束后播放
    void ClearInitCache();                    // 用完丢
};
```

### 为什么 InitCache 要持有 `FEffectActorHandle`(不是 AActor*)

因为这时候 Actor 可能还没创建(DA 还在异步加载,Actor 所需的 Mesh/Material 等参数未定)。
- **`FEffectActorHandle`** 可以在没 Actor 的情况下被操作:SetHidden 会缓存到 action queue,Actor 创建后 flush
- 业务调 `SetHidden(true)`、`AttachToComponent(...)` 期间,这些命令被累积到 InitCache 里的 `EffectActorHandle` 上
- Actor 创建完立刻 `EffectActorHandle.InitEffectActor(Actor, Handle)` 把所有 pending 命令执行

这是**命令模式(Command Pattern)**的应用——下一页 [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] 会详述。

### StartTime + TimeDiff 的用途
- `StartTime`:记录何时开始 pending(`FPlatformTime::Seconds()`)
- `TimeDiff`:在 PendingInit 期间累积经过的时间

**用处**:当 Pending 最终 Init 完成后,需要"补齐"特效从 `StartTime` 到 now 这段时间的推进——比如用户开了 Spawn,半秒后才加载完毕,应该"吃掉"这 0.5 秒的播放时间,让用户感觉特效没延迟。具体 .cpp 处理待 Batch 6。

### `GetLocation()` 为什么需要
Pending 期间特效位置可能会变(比如 attach 到会跑的 Actor)。这个接口用于 Init 完成时重新校准 Transform。

## 4 阶段 Spawn 流程(整合)

```
业务调 FEffectSystem::SpawnEffect(WorldContext, Transform, Path, Reason,
                                    BeforeInit, Callback, BeforePlay, Context, ...)

  ├─ ① Spawn:分配 HandleId,构造 FEffectHandle(path),InContainer = false
  │     创建 FEffectInitModel(AutoPlay, Reason, Callback, BeforePlay)
  │     Handle->PendingInit(WorldContext, Path, Reason, Transform,
  │                         BeforeInitCallback, Callback, BeforePlayCallback, AutoPlay)
  │     ─► Handle 内:InitCache = MakeUnique<FEffectInitHandle>(...) 
  │                  IsPendingInit() == true
  │                  EffectActorHandle 占位(尚未 bind Actor)
  │     返回 HandleId 给业务

  ├─ ② LoadEffectData:异步 LoadObject(Path) → UEffectModelBase*
  │     期间:业务可以拿 HandleId 调 SetHidden / AttachToActor 等
  │                → 这些操作作用在 InitCache 的 EffectActorHandle 上,排队

  ├─ ③ LoadEffectDataCallback:DA load 完
  │     调 BeforeInitCallback(业务可以最后改写一些 Handle 字段)
  │     Handle->Init(EffectData, EffectInitModel)
  │     ─► 构造 EffectSpec(走 FEffectSpecFactory,Batch 3)
  │     ─► 创建/复用 EffectActor
  │     ─► EffectActorHandle.InitEffectActor(Actor, Handle) — flush 所有 pending actions
  │     Handle->InitEffectActorAfterPendingInit()
  │     Handle->ClearInitCache()

  └─ ④ PlayEffect (if AutoPlay):
        调 BeforePlayCallback
        Handle->Play(Reason)
        调 Callback(Success, EffectId)
```

中间任意阶段失败都走 `Callback(ELoadEffectResult_Xxx, 0)` 告诉业务。

## 关键设计思想:Separation of Concerns

| Model | Progress |
|---|---|
| `FEffectInitModel` | `FEffectInitHandle` |
| 描述业务想做什么 | 跟踪系统做到哪步 |
| 不可变,const | 可变,持有引用/状态 |
| 永远只读回调 | 可缓存 Pending 操作 |
| Callback 目标 | WorldContext 所有者 |

这种分离让同一个 Spawn 请求可以被 **重试**(重新用 Model 构造新的 Handle)、**批量处理**(同一 Model 驱动多个 InitHandle)——虽然代码里没看到这种用法,但架构是开放的。

## 与其他实体的关系

- **创建者**:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 `SpawnEffect` 和 `SpawnEffectWithActor` 路径
- **被 Handle 持有**:`TUniquePtr<FEffectInitHandle> InitCache`(见 [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]])
- **依赖**:[[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]](InitHandle 内部持有)、[[entities/project-game/kuro-effect-system/effect-define|EffectDefine]](3 种 Callback 类型)
- **FTransformDouble / FVectorDouble**:双精度坐标,Kuro 的世界坐标系扩展

## Twin(Old 版对应)

**Old 完全没有这套管线**。Old 的"异步加载"要么靠业务自己在 TS 侧 pre-load,要么直接同步 LoadObject 卡住。`FEffectHandle` 里也没有 `IsPendingInit` 等状态。**这是 New 最大的体系化新增**。

你能从 Old 代码里找到什么等价物?
- 只有 `FKuroEffectSystem::UnregisterEffectHandle` 时的 cleanup 间接反映了"半初始化状态"
- TS 层通过轮询 `IsValid` 来判断特效是否可用

## 引用来源

- 原始代码:
  - `F:\...\KuroEffectSystemNew\EffectInitModel.h`
  - `F:\...\KuroEffectSystemNew\EffectInitHandle.h`
- 配套实体:[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 的"Pending Init 相关" region
- Spawn 入口:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 SpawnEffect

## 开放问题

- `FEffectInitModel::HandleInitCallback` (TFunction) 的**注册者是谁**?看起来是 `FEffectSystem::LoadEffectData` 内部把自己的一个方法 bind 进去,作为 Load callback 的链接?
- `FEffectInitHandle::StartTime` 实际在 `~FEffectInitHandle` 或 `Init` 时怎么用?
- Pending 期间业务连续调 `SetHidden(true)`、`SetHidden(false)`,最终 flush 时合并还是逐个执行?
- `SetContext` 在 Pending 态是否合法?(Handle 有 `SetContext`,但 InitCache 里没直接 Context 字段)
- 超时怎么办?PendingInit 卡住一直不 Load 完成有没有兜底超时?(`TimeDiff` 像是监控字段)
