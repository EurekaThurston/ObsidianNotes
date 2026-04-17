---
type: entity
category: class-hierarchy
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, spec, interface, template]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSpec
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[entities/project-game/kuro-effect-system/effect-spec-base]]
aliases: [IEffectSpecBase, KuroEffect::FEffectSpec, New Spec]
---

# IEffectSpecBase + KuroEffect::FEffectSpec<T>(新版 Spec 架构)

> New 的运行时状态机实现。`I` 前缀的**纯接口**(`IEffectSpecBase`)+ 提供通用实现的**模板基类**(`FEffectSpec<T>`)+ 工厂(`FEffectSpecFactory`)。相比 [[entities/project-game/kuro-effect-system/effect-spec-base|Old]] 彻底重构——接口膨胀到 80+ 虚函数,模板内容从 ~250 行扩到 ~1150 行,架构能力成倍提升。

## 三层结构

```
     IEffectSpecBase          ← 纯接口(I 前缀,80+ 纯虚)
          ▲
          │
    FEffectSpec<T>            ← 模板基类(实现接口,~1150 行)
          ▲
          │
   ┌──────┼──────┬────────┬──────────┬──────────────┐
   │      │      │        │          │              │
 Niagara Group Multi   Audio(新)  SeqPose(新)  ... (17 个)
```

## IEffectSpecBase(纯接口)

~80 个纯虚方法,按类别切分:

### 身份 / 引用
```cpp
virtual EEffectSpecType SpecType() = 0;
virtual void InitLifeTime(TSharedPtr<IEffectSpecBase>& InEffectSpec) = 0;
virtual TWeakPtr<FEffectHandle> GetHandle() const = 0;
virtual void SetHandle(const TSharedPtr<FEffectHandle>& InHandle) = 0;
virtual UEffectModelBase* GetEffectModel() const = 0;
virtual USceneComponent* GetSceneComponent() const = 0;
```

### TimeScale 管理(一整块)
```cpp
virtual float GetTimeScale() const = 0;
virtual float GetGlobalTimeScale() const = 0;
virtual float GetTotalTimeScale() = 0;
virtual void SetTimeScale(float, bool Force = false, bool IgnoreGlobalTimeScale = false) = 0;
virtual void SetAdditionTimeScale(int32 SourceType, float Value) = 0;
virtual void SetAdditionTimeScaleValue(int32 SourceType, float Value) = 0;
virtual TMap<int32, float> GetAdditionTimeScales() const = 0;
virtual void DeepCopyAdditionTimeScales(const TMap<int32, float>&, bool Refresh = true) = 0;
virtual bool HasAnyAdditionTimeScale() = 0;
virtual void RefreshTimeScale() = 0;
virtual void OnAdditionTimeScaleEnableChanged(int32 SourceType, bool Enable) = 0;
```

### 状态感知
```cpp
virtual void OnTickSystemPausedChanged(bool IsPaused) = 0;
virtual void OnGlobalTimeScaleChange() = 0;
virtual bool GetInStoppingTime() = 0;
virtual void OnGlobalStoppingTimeChange(bool InStoppingTime) = 0;
virtual void SetStoppingTime(bool) = 0;
virtual void EnterStopping() = 0;
virtual void SetLifeCycle(float) = 0;
virtual bool IsPlaying() = 0;
virtual bool IsStopping() = 0;
virtual bool IsClear() = 0;
virtual bool IsSpecValid() = 0;
virtual float GetTotalPassTime() = 0;
virtual float GetPassTime() = 0;
virtual EEffectType GetEffectType() = 0;
virtual void SetEffectType(EEffectType) = 0;
virtual bool IsLoop() = 0;
virtual bool GetIgnoreTimeScale() const = 0;
virtual bool GetIgnoreGlobalTimeScale() const = 0;
virtual bool IsEnable() const = 0;
```

### 参数下发(Material + Niagara)
```cpp
virtual void SetEffectParameterNiagara(const FKuroEffectNiagaraParameters&) = 0;
virtual void SetExtraState(int32) = 0;
virtual bool HasMaterialParameters() const = 0;
virtual FKuroEffectMaterialParameters* GetMaterialParameters() = 0;
virtual void CollectMaterialFloatCurve(FName, const FKuroCurveFloat&) = 0;
// ...6 个 CollectMaterial* + 3 个 RemoveMaterial*
```

### 标志控制
```cpp
virtual void SetThreeStageTime(float StartTime, float LoopTime, float EndTime, bool ResetPassTime) = 0;
virtual bool NeedAlwaysTick() const = 0;
virtual bool IsUseBoundsCalculateDistance() const = 0;
virtual void FreezeEffect(bool) = 0;
virtual void SetPlaying(bool) = 0;
virtual void SetStopping(bool) = 0;
```

### BodyEffect(角色身上光效专用)
```cpp
virtual bool ShouldRegisterBodyEffect() = 0;
virtual void RegisterBodyEffect() = 0;
virtual void UnregisterBodyEffect() = 0;
virtual void UpdateBodyEffect(float Opacity, bool Visible, bool CastShadow) = 0;
```

### 生命周期(**核心!10 多个**)
```cpp
virtual void Init(UEffectModelBase*, const TSharedPtr<FEffectInitModel>&) = 0;
virtual bool Start() = 0;
virtual void Tick(float Delta) = 0;                     // ← 无 Isolate
virtual void EnableChanged(bool) = 0;
virtual bool End() = 0;
virtual bool Clear() = 0;
virtual void Destroy() = 0;
virtual void Replay() = 0;
virtual void PreFirstPlay(const TSharedPtr<class FNiagaraComponentHandle>&) = 0;
virtual void Play(FName Reason) = 0;
virtual bool CanStop() = 0;
virtual void PreStop() = 0;
virtual void Stop(FName Reason, bool Immediately) = 0;
virtual void SeekTo(float Time, bool AutoLoop = true, bool CheckFinish = false, float TickDelta = 0) = 0;
virtual void SeekDelta(float DeltaTime, bool AutoLoop = true, bool CheckFinish = false, float TickDelta = 0) = 0;
virtual bool NeedPostTick() = 0;
virtual void OnPostTick(float) = 0;
virtual bool NeedTick() = 0;
virtual void OnParentInit() = 0;
virtual void OnBeginDelayPlay() = 0;
virtual void OnHiddenInGameChange(bool IsHidden) = 0;
```

### Misc
```cpp
virtual double GetLastPlayTime() = 0;
virtual double GetLastStopTime() = 0;
virtual void OnModifyEffectModel() = 0;
virtual int32 DebugErrorNiagaraPauseCount() = 0;
virtual EEffectErrorCode GetDebugErrorCode() = 0;
```

**总 80+ 个纯虚方法**。这种"**巨型接口**"在 OOP 里常被视为异味,但对**特效系统这种每种类型都有高度不同表现**的场景,集中在接口层让 Handle/System 用起来**完全 type-erased**(不需要知道具体 Spec 类型)。

## 开头的 Console Variables

文件顶部有 3 个 runtime-tunable CVar:
```cpp
inline bool GEffectSystemAdditionTimeScaleEnable = true;
// "Kuro.CppEffectSystem.AdditionTimeScale.Enable" — 额外附加的 TimeScale 是否生效

inline bool GKuroCppEffectSpecLifeDisablePostTickWhenDeltaTimeZero = true;
// "Kuro.CppEffectSystem.LifeDisablePostTickWhenDeltaTimeZero" — PostTick DeltaTime=0 时不执行

inline bool GKuroCppEffectSpecSeparateBodyEffectHidden = true;
// "Kuro.CppEffectSystem.SeparateBodyEffectHidden" — 是否分离 BodyEffect 类型的隐藏
```

**每个 CVar 都是在线开关**——线上出问题可以直接用控制台命令切换行为而不用发版。Old 没这种做法。

## FEffectSpec\<T\>(模板基类,~1150 行)

```cpp
template<typename T>
class FEffectSpec : public IEffectSpecBase
{
private:
    // 常量
    const float SMALLER_ONE = 0.99;
    const float LARGER_ONE = 1.01;
    const float MAX_WAIT_TIME_SCALE_ZERO_TIME = 10;   // TimeScale==0 保底时长
    const float MAX_WAIT_TIME_SCALE_VALUE = 0.1;      // 触发保底的阈值

protected:
    // 核心字段
    TStrongObjectPtr<T> EffectModel;                  // ← 强持有(Old 是裸指针)
    FEffectLifeTime LifeTime;                          // 同名类,但是 New 版 KuroEffect::FEffectLifeTime
    TWeakPtr<FEffectHandle> Handle;                    // ← 弱引用 Handle(Old 是裸 Info*)
    TWeakObjectPtr<USceneComponent> SceneComponent;
    bool StoppingTimeInternal = false;
    bool Enable = true;
    bool IsStoppingTime = false;
    bool IsAfterStart = false;
    
    // AdditionTimeScale 缓存
    float CurrentFrameTotalAdditionTimeScale = 1;
    uint64 LastUpdateTotalAdditionTimeScale = 0;      // 帧编号做"本帧已算过"缓存
    bool IsDirtyTimeScaleAfterReplay = false;
    
    // 虚钩子(子类重写)
    virtual void OnEffectTypeChange() {}
    virtual EEffectSpecInitState OnInit(const TSharedPtr<FEffectInitModel>&) { return EEffectSpecInitState_Success; }
    virtual bool OnStart() { return true; }
    virtual void OnTick(float DeltaTime) {}
    virtual bool OnEnd() { return true; }
    virtual bool OnClear() { return true; }
    virtual void OnReplay() {}
    virtual void OnPlay(const FName& Reason) {}
    virtual bool OnCanStop() { return true; }
    virtual void OnPreStop() {}
    virtual void OnStop(const FName& Reason, bool Immediately) {}
    virtual void OnEnableChanged(bool Value) {}
    virtual void OnSeekTime(float Time) {}
    virtual void OnEnterFreeze() {}
    virtual void OnExitFreeze() {}
    virtual void OnAfterStart() {}
    virtual void OnParentInit() {}
    virtual void OnBeginDelayPlay() {}

private:
    EEffectType EffectType = EEffectType_Scene;
    float StartTime = 0.0f, LoopTime = 0.0f, EndTime = 0.0f;
    bool Playing = false, Stopping = false;
    bool IgnoreTimeScale = false, IgnoreGlobalTimeScale = false;
    double LastPlayTime = 0.0f, LastStopTime = 0.0f;

protected:
    TMap<int32, float> AdditionTimeScaleMap;           // ← 新增!多来源叠加
    float TimeScale = 1.0f;
    bool TemporaryIgnoreGlobalTimeScale = false;
    
    /* 20+ 个虚钩子默认实现 + 80+ 个接口方法实现 */
};
```

### AdditionTimeScale 机制

```cpp
float GetTotalAdditionTimeScale()
{
    if (!GEffectSystemAdditionTimeScaleEnable) return 1;   // CVar 关了直接返回 1
    
    // 本帧缓存(帧号比较)
    if (LastUpdateTotalAdditionTimeScale == GFrameCounter) 
        return CurrentFrameTotalAdditionTimeScale;
    
    LastUpdateTotalAdditionTimeScale = GFrameCounter;
    float AdditionTimeScale = 1;
    for (const auto& Pair : AdditionTimeScaleMap) {
        const int32 Type = Pair.Key;
        if (FEffectSystem::GetAdditionTimeScaleEnable(Type))  // 全局可以按 Type 关掉
            AdditionTimeScale *= Pair.Value;   // 多来源**乘法叠加**
    }
    CurrentFrameTotalAdditionTimeScale = AdditionTimeScale;
    return CurrentFrameTotalAdditionTimeScale;
}
```

**场景**:一个特效同时受"技能减速(Type=1, Value=0.5)"和"场景减速(Type=2, Value=0.7)"影响 → `TotalAddition = 0.5 * 0.7 = 0.35`,特效播放速度变 35%。

业务能独立开关任一来源(`FEffectSystem::SetAdditionTimeScaleEnable(Type, false)`),不用记得去清 AdditionTimeScaleMap。

### GetTotalTimeScale(完整 TimeScale 叠乘)

```cpp
virtual float GetTotalTimeScale() override
{
    if (GetIgnoreTimeScale()) 
        return GetTotalAdditionTimeScale();
    return GetTimeScale() * GetGlobalTimeScale() * GetTotalAdditionTimeScale();
}
```

= **LocalScale × GlobalScale × AdditionScale 乘积**,除非 `IgnoreTimeScale` 则只剩 Addition。

### Init() 的详细流程

```cpp
virtual void Init(UEffectModelBase* EffectData, const TSharedPtr<FEffectInitModel>& EffectInitModel) override
{
    if (!Handle.IsValid()) return;
    
    // 1. 强类型化 DA
    EffectModel = TStrongObjectPtr<T>(Cast<T>(EffectData));
    if (!EffectModel) {
        // 走失败回调
        EffectInitModel->DoHandleInitCallback(Handle.Pin(), ELoadEffectResult_Fail, EffectInitModel);
        return;
    }
    
    // 2. 应用 DA 字段到 Spec
    IgnoreTimeScale = EffectData->IgnoreTimeDilation;
    IgnoreGlobalTimeScale = EffectData->IgnoreGlobalTimeDilation;
    
    // 3. 子特效的一致性检查
    if (!Handle.Pin()->IsRoot()) {
        if (EffectData->IsA(UEffectModelGroup::StaticClass())) {
            // 子特效不能是 Group 类型!
            走失败回调;
            return;
        }
        // 应用父的 EffectType 和 TimeScale
        if (Parent.IsValid()) {
            SetEffectType(Parent.Pin()->GetEffectType());
            SetTimeScale(Parent.Pin()->GetTimeScale());
        }
    }
    
    // 4. 存时间三段
    StartTime = EffectData->StartTime;
    LoopTime = EffectData->LoopTime;
    EndTime = EffectData->EndTime;
    
    // 5. 多线程化:子类 OnInit 可能是异步(返回 Initializing 则不立即回调)
    EEffectSpecInitState Result = OnInit(EffectInitModel);
    if (Result == EEffectSpecInitState_Fail) {
        EffectInitModel->DoHandleInitCallback(Handle.Pin(), ELoadEffectResult_Fail, ...);
    } else if (Result == EEffectSpecInitState_Success) {
        EffectInitModel->DoHandleInitCallback(Handle.Pin(), ELoadEffectResult_Success, ...);
    }
    // Initializing 不触发回调——子类需要自己异步触发(如 Group 等所有 child init 完成)
}
```

**三态 OnInit 返回值**(`EEffectSpecInitState_Fail/Success/Initializing`)非常关键——让子类能选择"同步成功"、"同步失败"或"异步挂起后再回调"。

### Tick() 实现

```cpp
virtual void Tick(float Delta) override
{
    if (!IsPlaying()) return;
    if (!Handle.IsValid()) return;
    
    // GlobalStoppingTime 检测(同 Old)
    if (FEffectSystem::GlobalStoppingTime() && IsStoppingTime) { /* 进入 Stopping 判断 */ }
    
    // 应用 TimeScale
    float DeltaTime = Delta;
    if (!GetIgnoreTimeScale()) {
        const float GlobalTimeScale = GetGlobalTimeScale();
        const float LocalTimeScale = GetTimeScale();
        if (偏离 1 的阈值) DeltaTime = Delta * GlobalTimeScale * LocalTimeScale;
    }
    if (HasAnyAdditionTimeScale()) DeltaTime *= GetTotalAdditionTimeScale();
    
    // 子类钩子(受 NeedTick 控制)
    if (NeedTick()) OnTick(DeltaTime);
    
    LifeTime.Tick(DeltaTime);
    
    // OnAfterStart 首次触发
    if (!IsAfterStart && LifeTime.IsAfterStart()) {
        IsAfterStart = true;
        OnAfterStart();
    }
}
```

**和 Old 的差别**:
- **无 `v8::Isolate*`**——完全纯 C++
- **`NeedTick()` 守卫**`OnTick`——不需要 tick 的 Spec 可以直接跳(如 Light 类特效定值,不需要每帧更新参数)
- AdditionTimeScale 叠乘
- `GetGlobalTimeScale` 内建在 FEffectSpec(Old 在 FEffectLifeTime)

### SetTimeScale(~60 行!非常复杂)

New 的 `SetTimeScale` 触发链:
1. 计算 effective Scale(考虑 IgnoreTimeScale / Addition)
2. 如果 Handle 是 Root,**写到 Actor 的 `CustomTimeDilation`** + 如果是 `AEffectSystemActor` 调 `SetTimeScale`
3. 调 `Handle->OnTimeScaleChange` 通知 Handle(传播到 OwnerEffect 同步句柄)
4. 更新 `LifeTime.SetTimeScale(...)`
5. **Mini TimeScale 保底**:
   - 从 "≥0.1" 降到 "<0.1" → 注册 10 秒后自动复原的定时器
   - 从 "<0.1" 升到 "≥0.1" → 取消定时器

**这是 [[entities/project-game/kuro-effect-system-new/effect-life-time|New FEffectLifeTime 的 WaitMiniTimeScale]] 的 Setter 侧**。

### BodyEffect(角色身上光效的单独路径)

```cpp
virtual void RegisterBodyEffect() override
{
    if (!ShouldRegisterBodyEffect()) return;
    if (!Handle.IsValid()) return;
    
    BodyEffectOpacity = 1.0f;
    BodyEffectVisible = true;
    
    // 找 SkeletalMeshComponent(从 Context 或其他地方)
    USkeletalMeshComponent* BodySkeletalMeshComponent = ...;
    
    // ← Script Bridge 调用!
    FEffectSystem::EffectSystemScriptBridge.EffectSpec_RegisterBodyEffect(
        Handle->Id, Actor, BodySkeletalMeshComponent, Context->SourceObject, EffectData);
}
```

`FEffectSystem::EffectSystemScriptBridge` 是 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 static 字段,类型是 `FEffectSystemScriptBridge`(Batch 5)。BodyEffect 的注册路径经过 Bridge —— 让脚本有机会拦截处理。

## 与 Old 的逐点对照

| 维度 | Old | New |
|---|---|---|
| 接口规模 | 6 纯虚方法 | 80+ 纯虚方法 |
| 抽象机制 | class with fields | `I` 前缀 pure interface |
| EffectModel 所有权 | 裸 `T*` | `TStrongObjectPtr<T>` |
| Handle 连接 | 裸 `FEffectHandleInfo*` | `TWeakPtr<FEffectHandle>` + `SetHandle()` |
| v8 | Tick/SeekTo 带 Isolate | 全部纯 C++ |
| TimeScale | Local * Global | **Local * Global * AdditionMap-乘积** |
| 生命周期显式化 | 无(靠 Flag 位) | 12 个 On* 钩子 |
| 初始化同步性 | 同步(ctor 完成) | 三态(Sync Success/Fail/Async Initializing) |
| BodyEffect | 无 | 独立注册路径(经过 ScriptBridge) |
| CVars | 无 | 3 个运行时开关 |
| 代码行数 | ~250 | ~1150 |

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] **持有**:`TSharedPtr<IEffectSpecBase> EffectSpec`(通过 `GetEffectSpec()` 暴露)
- **构造**:[[entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory::CreateEffectSpec]]
- **依赖**:
  - [[entities/project-game/kuro-effect-system-new/effect-life-time|KuroEffect::FEffectLifeTime]](字段)
  - [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel]](Init 回调)
  - [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]](GlobalTimeScale 等 static)
  - [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelBase 家族]](DA 字段读)
- **UE 交互**:`TStrongObjectPtr` 保证 DA 不被 GC,`AActor::CustomTimeDilation` 下发 TimeScale,`AEffectSystemActor::SetTimeScale`(Batch 4)

## Twin(Old 版对应)

- [[entities/project-game/kuro-effect-system/effect-spec-base|Old FEffectSpecBase + FEffectSpec<T>]]

## 引用来源

- 原始代码:
  - `F:\...\KuroEffectSystemNew\EffectSpec\IEffectSpecBase.h`(~117 行接口)
  - `F:\...\KuroEffectSystemNew\EffectSpec\EffectSpec.hpp`(~1150 行模板)
- 工厂:[[entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory]]
- 子类目录:[[entities/project-game/kuro-effect-system-new/effect-spec-subclasses|New Spec 子类目录]]

## 开放问题

- `PreFirstPlay(const TSharedPtr<FNiagaraComponentHandle>&)` 和 `OnBeginDelayPlay` 的调用时机差别?(看 NiagaraSpec 的实现,两者都是"准备开始播放前"的钩子)
- `OnParentInit` 只由 Group 的 `OnSpawnChildEffectFinish` 调用——用途?
- `IsDirtyTimeScaleAfterReplay` 的逻辑:Replay 时若曾有 Addition 就标脏,**下次 SetTimeScale 强制触发**。为什么这个边角情况要显式处理?
- `IsEnable()` vs `IsPlaying()` / `IsSpecValid()`——三个 bool 的语义层次?从 SetTimeScale 的代码看 `SpecValid = Flag 含 Start 且不含 Clear`。
- `OnHiddenInGameChange(bool)` 的调用者是谁?(Handle 或 System 的 SetHidden 路径?)
