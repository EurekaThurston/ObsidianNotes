---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, lifetime]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectLifeTime.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
twin: [[entities/project-game/kuro-effect-system-new/effect-life-time]]
aliases: [FEffectLifeTime (Old)]
---

# FEffectLifeTime(老版时间/播放状态)

> 单个特效的**时间轴状态机**:记录它播了多久、是否在循环、是否在 Stopping 阶段、要不要被时间停滞影响。构造时从 [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelBase]] 的 StartTime/LoopTime/EndTime 初始化。

## 概览

一个"特效当前播到哪了"的工作记录本。被 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 或具体 Spec(Batch 3)持有,每帧 Tick 更新 `PassTime` 等内部状态。

**3 个全局静态字段** 特别重要——它们是**所有特效共享的全局时间开关**:

```cpp
static float GlobalTimeScale;          // 全局时间缩放(慢动作/快进)
static float GlobalStoppingPlayTime;   // 时间停滞状态下还能播的总时长上限
static bool  GlobalStoppingTime;       // 是否进入全局时间停滞状态
```

—— 比如"子弹时间"、"大招演出暂停场景":业务代码 `FEffectLifeTime::GlobalTimeScale = 0.3f`,全地图所有特效立刻慢三倍。

## 完整字段

```cpp
class FEffectLifeTime
{
    float PassTime = 0;              // 当前循环内已播时间
    float TotalPassTime = 0;         // 总已播时间(跨循环累计)
    bool  IsPlaying = false;
    bool  IsStopping = false;
    float StartTime = 0;             // 启动段时长(从 DA 初始化)
    float LoopTime = 0;              // 循环段时长
    bool  WillLoop = false;          // 这次 Tick 会触发循环吗
    float LoopTimeStamp = 0;         // 本次循环开始的时间戳
    float LifeTimeStamp = 0;         // 生命时间戳
    bool  IgnoreGlobalTimeScale = false;
    bool  TemporaryIgnoreGlobalTimeScale = false;  // 业务临时关一次 GlobalTimeScale
    FEffectHandleInfo* EffectHandleInfo = nullptr;   // 回调通道
public:
    static float GlobalTimeScale;
    static float GlobalStoppingPlayTime;
    static bool  GlobalStoppingTime;
    
    void Init(const UEffectModelBase*, FEffectHandleInfo*);
    ~FEffectLifeTime();
    /* ... */
};
```

## 核心 API

### Init(从 DA 初始化)
```cpp
void Init(const UEffectModelBase* EffectModel, FEffectHandleInfo* InEffectHandleInfo);
```
- 从 DA 读 `StartTime` / `LoopTime`(不读 EndTime——EndTime 的逻辑由谁控待 Batch 3/6)
- 记下 `EffectHandleInfo` 指针,用于下面 Tick 里触发 JS 回调

### Tick(核心)
```cpp
void Tick(float DeltaTime, v8::Isolate* Isolate);
```
注意签名有 `v8::Isolate*`——因为 Tick 里会通过 `EffectHandleInfo` 调 TS 的 `UpdateLifeCycle` / `EnterStopping` 等回调。**这就是"lifetime 知道 v8"的典型** Old 风格污染。

推测 Tick 做的事(具体在 .cpp,Batch 6):
1. 用 `DeltaTime * 有效 TimeScale`(考虑 Global / Temporary Ignore)推进 `PassTime` 和 `TotalPassTime`
2. 检查 `PassTime` 是否超过 `StartTime + LoopTime`——超过就触发循环:
   - 调 `Info->UpdateLifeCycle(Isolate, DeltaTime)` 通知 TS
   - `PassTime -= LoopTime` 重置到新循环
   - 置 `WillLoop = true`
3. 检查 `TotalPassTime` 是否超过 `GlobalStoppingPlayTime`——超过且 `GlobalStoppingTime==true` 就 force Stop
4. 如果 `IsStopping` 状态,检查是否可以转到 Destroy

### SeekTo(快进/倒退)
```cpp
bool SeekTo(v8::Isolate* Isolate, float Time, bool IsTick, bool AutoLoop = true);
```
调整 PassTime 到指定值。**注意** 即使 `IsTick=false` 也能 Seek,说明 Seek 不只是调试用,业务也能用(比如特效跳帧)。

### 查询接口(FORCEINLINE)

```cpp
FORCEINLINE bool  IsAfterStart()                 { return PassTime > StartTime - 0.001; }
FORCEINLINE bool  IsAfterGlobalStoppingPlayTime(){ return TotalPassTime > GlobalStoppingPlayTime; }
FORCEINLINE bool  GetIsPlaying()                 { return IsPlaying || IsStopping; }  // 注意!
FORCEINLINE bool  GetIsReallyPlaying()           { return IsPlaying; }
FORCEINLINE bool  GetIsStopping()                { return IsStopping; }
FORCEINLINE float GetPassTime()                  { return PassTime; }
FORCEINLINE float GetTotalPassTime()             { return TotalPassTime; }
float GetGlobalTimeScale() const;
```

**陷阱**:`GetIsPlaying()` 返回 `IsPlaying || IsStopping`。**"在 Stopping 阶段也算在播"**——因为特效还在渲染、还要消耗 CPU。业务拿这个接口判断"这个特效还在吗"就能覆盖两种情况。如果你只关心"还在主动播、没进入 Stopping",用 `GetIsReallyPlaying()`。

### 状态切换器
```cpp
FORCEINLINE void SetIsPlaying(bool InIsPlaying)                  { IsPlaying = InIsPlaying; }
void SetIsStopping(v8::Isolate* Isolate, bool InIsStopping);     // 非 inline,会调 TS 的 EnterStopping
FORCEINLINE void SetTemporaryIgnoreGlobalTimeScale(bool);
```
`SetIsStopping` 不是 inline,是因为从 `!IsStopping → IsStopping` 切换时要回调 TS 的 `EnterStopping`。这个副作用决定了它必须在 .cpp 实现(需要 v8 调用的完整 include)。

## 有效 TimeScale 的计算

概念上:
```
effective_dt = delta_time
             * (IgnoreTimeDilation ? 1 : world_time_dilation)         // 来自 DA
             * (IgnoreGlobalTimeScale || TemporaryIgnoreGlobalTimeScale ? 1 : GlobalTimeScale)
             * EffectSpec->EffectTimeScale                              // 业务动态设置
             * (IgnoreGlobalTimeScale ? 1 : Addition... )  // 这条 New 才有
```

具体代码在 `GetGlobalTimeScale()` 里。影响所有 `PassTime += ... * effective_scale` 的前因。

**为什么要有 `IgnoreGlobalTimeScale` 和 `TemporaryIgnoreGlobalTimeScale` 两个**:
- `IgnoreGlobalTimeScale` 来自 DA 配置,长期属性(比如"主角技能特效不受场景慢动作影响")
- `TemporaryIgnoreGlobalTimeScale` 业务临时干预(比如"这一刀的特效我想全场慢放时正常速度")
- 任一为 true 都不应用 Global

## 与其他实体的关系

- **持有者**:典型情况下 `FEffectSpecBase` 子类会持有一个 `FEffectLifeTime`(待 Batch 3 确认),也可能是 Handle 直接持
- **依赖**:[[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelBase]](读取 StartTime/LoopTime)、[[entities/project-game/kuro-effect-system/effect-handle-info|FEffectHandleInfo]](调回调)
- **静态全局影响**:被 [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]] 的 `SetEffectGlobalTimeScale` / `SetEffectGlobalStoppingTime` 入口修改

## Twin(New 版对应)

- [[entities/project-game/kuro-effect-system-new/effect-life-time|KuroEffect::FEffectLifeTime]] — 彻底重写

### 关键 Delta(预告,New 实体页有详解)
- **无 v8**:New 的 Tick 签名纯 C++
- **整合 Timer System**:New 用 `FKuroTimerHandle` 管延迟事件,不再靠轮询 PassTime 判断
- **三段时间显式化**:Old 只用 `StartTime + LoopTime`,New 有显式 `StartTime + LoopTime + EndTime` 三段
- **状态迁移更丰富**:`IsInitTime`, `WillEverPlay`, `IsPendingPlayFinish`, `ContinuousLoop`
- **无 ToStopping 回调**:回调路径走 Handle 的 `FinishDelegate`,不再直接调 JS

## 引用来源

- 原始代码:`F:\...\KuroEffectSystem\EffectLifeTime.h`(~85 行)
- [[sources/project-game/kuro-effect-system/overview|Old 系统 overview]]

## 开放问题

- `EndTime` 的逻辑在谁那里管?(Base 类字段 +  Spec 处理?)
- `LoopTimeStamp` / `LifeTimeStamp` 的具体用法(它们不是相对时间而是 timestamp,单位是 `FPlatformTime::Seconds()`?)
- `WillLoop` 用来干什么——业务接口?
- `FEffectLifeTime` 的 dtor 为什么 explicit?(推测:如果持有 v8 相关的反注册,需要在析构时做清理)
