---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, lifetime]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectLifeTime.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[entities/project-game/kuro-effect-system/effect-life-time]]
aliases: [KuroEffect::FEffectLifeTime, FEffectLifeTime (New)]
---

# KuroEffect::FEffectLifeTime(新版时间/播放状态)

> New 系统的生命周期管理器;**不含 v8**,**显式三段时间**(StartTime / LoopTime / EndTime),**集成 Kuro Timer System** 处理延迟事件。相比 [[entities/project-game/kuro-effect-system/effect-life-time|Old FEffectLifeTime]] 更干净也更强大。

## 整体

```cpp
namespace KuroEffect
{
    class IEffectSpecBase;   // 前向声明
    
    class FEffectLifeTime
    {
        static FName PlayFinishStopEffectReason;      // 播完后 Stop 的原因码
        
        TWeakPtr<IEffectSpecBase> EffectSpec;         // 所属的 Spec(弱引用,避免循环)
        float DefaultPassTime = 0;                    // 默认的起始播放时间
        float StartTime = -1;                         // Start 阶段时长
        float LoopTime = 0;
        float EndTime = 0;                            // ← Old 没有这个字段
        float LifeTimeStamp = 0;                      // 全时长 = Start + Loop + End
        bool  ContinuousLoop = false;
        bool  IsLoopInternal = false;
        bool  WillEverPlay = false;                   // ← 新状态
        bool  IsInitTime = false;                     // ← 新状态
        float TimeScale = 1.0f;                       // 每 handle 自己的 TimeScale
        
        bool  IsPendingPlayFinish = false;
        FKuroTimerHandle WaitPlayFinishedTimerHandle;   // 定时器:等播完
        FKuroTimerHandle WaitMiniTimeScaleTimerHandle;  // 定时器:等 mini timescale 解除
        
        void PlayFinished();
        void WaitMiniTimeScaleFinish();
        void MakeLoop();
        void StartWaitPlayFinished(float WaitTime);
        
        float PassTime = 0;
        float TotalPassTime = 0;
        float LoopTimeStamp = 0;                      // Start + Loop
    
    public:
        void Init(TSharedPtr<IEffectSpecBase>& EffectSpec);
        FORCEINLINE float GetStartTime() const { return StartTime; }
        bool  IsLoop() const;
        bool  NeedPostTick() const;
        float GetPassTime() const;
        float GetTotalPassTime() const;
        void  SetTime(float StartTime, float LoopTime, float EndTime, bool ContinuousLoop = false);
        void  SetLifeCycle(float LifeCycle);
        void  WhenEnterStopping();
        void  UpdateLifeCycle(float LifeCycle);
        void  SetTimeScale(float TimeScale);
        void  OnGlobalTimeScaleChange();
        void  RegisterWaitMiniTimeScale(float WaitTime);
        void  UnregisterWaitMiniTimeScale();
        void  Tick(float DeltaTime);
        void  PostTick(float DeltaTime);
        bool  SeekTo(float Time, bool CheckFinish, bool IsTick, bool AutoLoop = true);
        bool  IsAfterStart() const;
        void  OnReplay();
        void  Clear(bool Unbind = false);
        void  OnTickSystemPausedChanged(bool IsPaused);
    };
}
```

## 关键改进 vs Old

### 1. 去 v8 化
```cpp
// Old
void Tick(float DeltaTime, v8::Isolate* Isolate);

// New
void Tick(float DeltaTime);
```
Tick 纯 C++。Stopping 等回调不再直接从这里走 JS;通过 Spec 或 Handle 间接驱动。

### 2. 显式 EndTime
Old 只存 `StartTime` + `LoopTime`,`EndTime` 没出现在 LifeTime 类里(可能散落在 Spec 或 DA)。New 把三段**全部管起来**:
```cpp
float StartTime = -1;
float LoopTime = 0;
float EndTime = 0;           // 新增
float LifeTimeStamp = 0;      // = StartTime + LoopTime + EndTime
float LoopTimeStamp = 0;      // = StartTime + LoopTime
```
- 循环特效:`LoopTimeStamp` 是判断"要 loop 了"的时机,`LifeTimeStamp` 是 Stop 之后的最大持续时间
- 非循环特效:直接用 `LifeTimeStamp`

### 3. 集成 Timer System
```cpp
FKuroTimerHandle WaitPlayFinishedTimerHandle;
FKuroTimerHandle WaitMiniTimeScaleTimerHandle;

void StartWaitPlayFinished(float WaitTime);
void RegisterWaitMiniTimeScale(float WaitTime);
void UnregisterWaitMiniTimeScale();
```

**Old 的做法**:每帧在 `Tick(DeltaTime, Isolate)` 里判断 `PassTime > LoopTime`,`PassTime > GlobalStoppingPlayTime` 等——**轮询**。

**New 的做法**:用 `KuroTimerSystem` 注册一个定时器。播放时调 `StartWaitPlayFinished(TotalDuration)`,时间到系统自动调 `PlayFinished()` 回调。——**事件驱动**。

### 为什么 "WaitMiniTimeScale"?
业务能把 Handle 的 TimeScale 设成很小(接近 0),此时 PassTime 推进极慢。如果不设兜底,某个特效可能"永远播不完"(见 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::SetTimeScale]] 的注释:"会触发 SetTimeScale(0) 保底——当设置 TimeScale 为 0 为防止泄漏,会在保底时间后复原")。

这里的 `WaitMiniTimeScaleTimerHandle` 就是这个**保底定时器**——超过某个等待时间,系统强制复原 TimeScale,避免 Handle 永久挂在内存里。

### 4. 每 Handle 独立 TimeScale 存在 LifeTime
```cpp
float TimeScale = 1.0f;           // 成员字段
void SetTimeScale(float);
```
Old 的 TimeScale 在 `FEffectSpecBase::EffectTimeScale` 里;New 直接下沉到 LifeTime。

**原因**:TimeScale 只影响时间推进,和 Spec 的其他状态(材质参数、Niagara 状态)无关——放在 LifeTime 职责更单一。

### 5. 显式"我会播吗"(WillEverPlay)
Old 只有 `IsPlaying`/`IsStopping`。New 多:
```cpp
bool WillEverPlay = false;    // 这条 Handle 最终会播放吗
bool IsInitTime = false;      // 是否在 Init 阶段
bool IsPendingPlayFinish = false;  // 已经在等播完
```

`WillEverPlay` 特别重要——**处理"创建了但可能还没来得及 Play 就被 Stop"** 的情况。例如业务 Spawn 一个 Unlooped 特效后立刻 `Stop(Immediately=true)`;这时 LifeTime 里 `WillEverPlay = false`,Tick 可以直接跳过 Loop 逻辑,走简化的 Finish 路径。

## 核心 API(按功能)

### 初始化
```cpp
void Init(TSharedPtr<IEffectSpecBase>& EffectSpec);  // 绑 Spec(弱引用)
void SetTime(float StartTime, float LoopTime, float EndTime, bool ContinuousLoop = false);
void SetLifeCycle(float LifeCycle);                   // 直接设总时长
void UpdateLifeCycle(float LifeCycle);                // 外部修改 LifeCycle 时用
```

### Tick 管线
```cpp
void Tick(float DeltaTime);        // 主 Tick
void PostTick(float DeltaTime);    // 在 PostPhysics / PostUpdateWork 的 tick
bool NeedPostTick() const;         // Spec 或 LifeTime 是否需要 Post Tick
```

### 状态切换
```cpp
bool  IsLoop() const;              // 这个 LifeTime 是不是循环态
bool  IsAfterStart() const;        // 过 Start 阶段了吗
void  WhenEnterStopping();         // 进入 Stopping 态
void  OnReplay();                  // Replay 重置
void  Clear(bool Unbind = false);  // 清理(可选择解除 EffectSpec 绑定)
```

### Seek
```cpp
bool SeekTo(float Time, bool CheckFinish, bool IsTick, bool AutoLoop = true);
```
Old 是 `SeekTo(Isolate, Time, IsTick, AutoLoop)`。New 去掉 Isolate,但**新增 `CheckFinish`**——Seek 到某个时间时是否检查"这样 Seek 之后特效应该完成了",如果是就触发 Finish 逻辑。

### TimeScale 响应
```cpp
void SetTimeScale(float TimeScale);
void OnGlobalTimeScaleChange();       // 全局 TS 变动时调这个,自己重算有效 TS
void OnTickSystemPausedChanged(bool IsPaused);
```

### MiniTimeScale 兜底
```cpp
void RegisterWaitMiniTimeScale(float WaitTime);
void UnregisterWaitMiniTimeScale();
```

## 全局静态字段去哪了

Old 里:
```cpp
// Old
static float GlobalTimeScale;
static float GlobalStoppingPlayTime;
static bool  GlobalStoppingTime;
```

**New 里这些没有了,被迁移到了 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 static 字段**:
```cpp
// FEffectSystem.h
static float GlobalTimeScale;
static void  SetGlobalTimeScale(float);
static float GlobalStoppingPlayTime();     // 函数而不是字段
static bool  GlobalStoppingTime();
static void  SetGlobalStoppingTime(bool, float PlayTime);
```

**设计动因**:
- 全局状态应该归全局单例(`FEffectSystem`)管,不是归每个 LifeTime 实例
- 改成函数接口可以在 Setter 里做**通知扩散**——`SetGlobalTimeScale` 要让所有现存 LifeTime 调 `OnGlobalTimeScaleChange()`

具体通知链待 Batch 6 读 .cpp。

## 与其他实体的关系

- **前向声明 `IEffectSpecBase`**——实际持引用,`Init(TSharedPtr<IEffectSpecBase>&)` 存为 `TWeakPtr<IEffectSpecBase>` 字段(弱引用避免循环)
- **被 Spec 内部持有**:具体的 `FEffectSpec*` 子类会持一个 `FEffectLifeTime`(Batch 3 验证)
- **依赖**:`FKuroTimerHandle`(KuroTimerSystem 插件)
- **全局配置** 来自 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 static 字段
- **无 v8 依赖**

## Twin(Old 版对应)

- [[entities/project-game/kuro-effect-system/effect-life-time|Old FEffectLifeTime]]

### Delta 速记
| 维度 | Old | New |
|---|---|---|
| JS | Tick/SeekTo 带 `v8::Isolate*` | 纯 C++ |
| 三段时间 | Start + Loop(End 散落在 Spec) | Start + Loop + End 显式 |
| 延迟事件 | 每帧轮询 PassTime | Timer System 事件驱动 |
| 全局 TS | static 字段 | 移到 FEffectSystem,setter 驱动通知 |
| TS 冻结 | 通过 Info | 不直接暴露,Handle 驱动 |
| TimeScale 保底 | 无 | `WaitMiniTimeScaleTimerHandle` |
| 状态 | IsPlaying/IsStopping | +`WillEverPlay`/`IsInitTime`/`IsPendingPlayFinish`/`ContinuousLoop` |
| Spec 关联 | 无明确引用 | `TWeakPtr<IEffectSpecBase> EffectSpec` |
| IgnoreGlobalTimeScale | `IgnoreGlobalTimeScale + TemporaryIgnoreGlobalTimeScale` | 本文件未见——可能迁到 Handle 或 Spec |

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectLifeTime.h`(~53 行,比 Old ~85 行**更短**——因为状态变量减少,方法外提,Timer 替代轮询)
- 依赖:`KuroTimerSystem/TimerHandle.h`
- 使用:各种 `FEffectSpec*` 子类(Batch 3)

## 开放问题

- `ContinuousLoop` 的语义?Old DA 里说是"循环特效变成持久特效"——具体是改成 `LoopTime = infinity`?
- `PlayFinished()` 私有方法——是 `WaitPlayFinishedTimerHandle` 的回调,但被谁注册?
- `MakeLoop()` 是内部的 loop 触发器,Tick 里被调?还是外部?
- `DefaultPassTime` vs `PassTime`:前者是从哪来?DA 的 `DefaultManualTime` 吗?
- `IEffectSpecBase` 接口长什么样(Batch 3 专题)
