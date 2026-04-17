---
type: entity
category: class-hierarchy
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, spec, template]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectSpec/EffectSpec.hpp
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
twin: [[entities/project-game/kuro-effect-system-new/effect-spec-base]]
aliases: [FEffectSpecBase, FEffectSpec, Old Spec]
---

# FEffectSpecBase + FEffectSpec<T>(老版 Spec 架构)

> Old 特效的**运行时状态机实现层**。一个两层架构:**抽象基类 `FEffectSpecBase`**(纯虚)+ **CRTP 模板 `FEffectSpec<T>`**(给每种 DataAsset 类型提供 Tick 公共实现)。具体子类(`FEffectModelNiagaraSpec` 等 12 种)继承模板,只重写 `OnTick` / `OnAfterStart` 等小钩子。

## 设计模式:Template Method + CRTP

这是 C++ 里标准的 **Template Method Pattern**(模板方法模式):
- 父类(`FEffectSpec<T>`)写**算法骨架**——比如 Tick 的标准流程
- 子类只填**钩子点**——`OnTick` / `OnAfterStart`

加上**CRTP**(Curiously Recurring Template Pattern):把 DataAsset 类型 `T` 作为模板参数,让 `EffectModel` 字段直接是 `T*`(而不是 `UEffectModelBase*`),编译期就有类型安全。

## 两层架构

```
         FEffectSpecBase          ← 纯虚基类(纯接口 + LifeTime 持有)
              ▲
              │
        FEffectSpec<T>            ← 模板基类(填 Tick 骨架 + 常用字段)
              ▲
              │
   ┌──────────┼──────────┬─────────────┬──────────────┐
   │          │          │             │              │
 Niagara  Group      MultiEffect   Ghost ...        (12 个)
  Spec    Spec        Spec         Spec
```

## FEffectSpecBase(纯虚基类)

```cpp
class FEffectSpecBase
{
protected:
    FEffectLifeTime LifeTime;         // 时间状态

public:
    // === 状态位(所有子类共享)===
    int EffectFlag = 0;                // EEffectFlag 位掩码(None|Init|Start|Play|Stop|End|Clear|Destroy)
    bool IgnoreTimeScale = false;
    float EffectTimeScale = 1.0;
    bool IsInStoppingTime = false;
    
    // === TimeScale 下发到 LifeTime ===
    FORCEINLINE void SetIsPlaying(bool);
    FORCEINLINE void SetIsStopping(v8::Isolate* Isolate, bool);  // ← v8!
    FORCEINLINE bool GetIsPlaying() const;
    FORCEINLINE bool GetIsReallyPlaying() const;
    FORCEINLINE bool GetIsStopping() const;
    FORCEINLINE float GetPassTime() const;
    FORCEINLINE void SetTemporaryIgnoreGlobalTimeScale(bool);
    FORCEINLINE float GetTimeScale() const {
        return IsInStoppingTime ? 0 : (EffectTimeScale * LifeTime.GetGlobalTimeScale());
    }
    
    // === 纯虚接口(子类必须实现)===
    virtual ~FEffectSpecBase();
    virtual void Tick(v8::Isolate*, float DeltaSeconds) = 0;
    virtual void SeekTo(v8::Isolate*, float Time, bool AutoLoop = true, float TickDelta = 0) = 0;
    virtual void SeekDelta(v8::Isolate*, float DeltaTime, bool AutoLoop = true, float TickDelta = 0) = 0;
    virtual void OnPostTick(float DeltaTime) = 0;
    virtual bool NeedPostTick() = 0;
    virtual void SetIsStoppingTime(bool) = 0;
    
    // === 虚可选(默认返回 nullptr,MaterialController/StaticMesh 等会重写)===
    virtual FKuroEffectMaterialParameters* GetEffectMaterialParameters() { return nullptr; }
    
#if KURO_ENABLE_EFFECT_SYSTEM_DEBUG_DRAW
    virtual void DebugDraw(FKuroRuntimeDebugDrawVerticalBox*, int HandleId) = 0;
    virtual bool PassDebugDrawFilter(const FString& Filter) = 0;
#endif
};
```

**观察**:
- 大量 FORCEINLINE wrapper,**让 Spec 上的行为等价于直接操作 LifeTime**——把 LifeTime 的接口"外包"出来,但 Spec 又有自己的 EffectFlag / EffectTimeScale 等补充字段。
- `GetTimeScale()` 的实现很漂亮:`IsInStoppingTime ? 0 : (EffectTimeScale * GlobalTimeScale)`——Stopping 期间 TimeScale 强制为 0,表示"特效冻结但还活着"。
- `v8::Isolate*` **再次穿透**——`SetIsStopping` 需要 Isolate 是因为 LifeTime.SetIsStopping 内部要回调 JS。

## FEffectSpec\<T\>(CRTP 模板)

```cpp
template <typename T>
class FEffectSpec : public FEffectSpecBase
{
protected:
    T* EffectModel;                             // 裸 UObject* — 没保护!
    FEffectHandleInfo* EffectHandleInfo;        // JS 伴生体
    bool IsStoppingTime = false;
    bool IsAfterStart = false;

public:
    // Constructor:拿 DataAsset + HandleInfo,初始化 LifeTime
    FEffectSpec(T* InEffectModel, FEffectHandleInfo* InEffectHandleInfo)
    {
        EffectModel = InEffectModel;
        EffectHandleInfo = InEffectHandleInfo;
        LifeTime.Init(InEffectModel, EffectHandleInfo);
    }
    
    virtual ~FEffectSpec() override
    {
        EffectModel = nullptr;
        EffectHandleInfo = nullptr;
    }
    
    /* 省略的三个虚钩子 */
    virtual void OnTick(v8::Isolate* Isolate, float DeltaTime) {}
    virtual void OnSeekTime(v8::Isolate* Isolate, float DeltaTime) {}
    virtual void OnAfterStart() {}
    
    /* ... */
};
```

### 核心:Tick 的标准流程

```cpp
virtual void Tick(v8::Isolate* Isolate, float DeltaSeconds) override
{
    // 1. 不在播 → 跳过
    if (!GetIsPlaying()) return;
    
    // 2. 全局 StoppingTime 检测(子弹时间)
    if (FEffectLifeTime::GlobalStoppingTime && IsStoppingTime)
    {
        if (!IsInStoppingTime) {
            // 还没进入 StoppingTime,但已过 GlobalStoppingPlayTime
            if (LifeTime.IsAfterStart() && LifeTime.IsAfterGlobalStoppingPlayTime()) {
                if (EffectHandleInfo) {
                    EffectHandleInfo->EnterStopping(Isolate);  // ← 回调 TS 的 EnterStopping
                    return;
                }
            }
        } else {
            return;   // 已在 Stopping,不再 tick
        }
    }
    
    // 3. 应用 TimeScale 得到真实 DeltaTime
    float DeltaTime = DeltaSeconds;
    if (!IgnoreTimeScale) {
        const float GlobalTimeScale = LifeTime.GetGlobalTimeScale();
        if (GlobalTimeScale != 1 || EffectTimeScale != 1)
            DeltaTime = DeltaSeconds * GlobalTimeScale * EffectTimeScale;
    }
    
    // 4. 子类钩子:OnTick(做各自类型的工作)
    OnTick(Isolate, DeltaTime);
    
    // 5. 推进 LifeTime
    LifeTime.Tick(DeltaTime, Isolate);
    
    // 6. 首次越过 StartTime → 触发 OnAfterStart 钩子
    if (!IsAfterStart) {
        if (LifeTime.IsAfterStart()) {
            IsAfterStart = true;
            OnAfterStart();
        }
    }
    
#if KURO_ENABLE_EFFECT_SYSTEM_DEBUG_DRAW
    LastTickTime = GFrameCounter;
#endif
}
```

这就是**Template Method 的骨架**:`Tick()` 的流程是固定的,只有第 4 步 `OnTick()` 和第 6 步 `OnAfterStart()` 让子类插手。

### SeekTo/SeekDelta

```cpp
virtual void SeekTo(v8::Isolate*, float Time, bool AutoLoop = true, float TickDelta = 0) override
{
    float DeltaTime = TickDelta != 0 ? TickDelta : Time - LifeTime.GetPassTime();
    OnSeekTime(Isolate, DeltaTime);           // 子类钩子
    if (Time == LifeTime.GetPassTime() || !LifeTime.SeekTo(Isolate, Time, false, AutoLoop)) {
        OnTick(Isolate, DeltaTime > 0 ? DeltaTime : 0);
    }
}
```

Seek 时:
1. 调 `OnSeekTime` 让子类做准备(如 Niagara 要清理视觉状态)
2. LifeTime 真正 Seek;如果失败或已在目标时间,手动用 DeltaTime 再跑一次 `OnTick` 补齐

### 默认空实现(让子类按需覆盖)

```cpp
virtual void OnPostTick(float DeltaTime) override {}         // 默认啥也不做
virtual bool NeedPostTick() override { return false; }
virtual void OnAfterStart() {}
virtual void OnTick(v8::Isolate*, float) {}
virtual void OnSeekTime(v8::Isolate*, float) {}
virtual void SetIsStoppingTime(bool InIsStoppingTime) override { IsStoppingTime = InIsStoppingTime; }
```

## 子类示例:`FEffectModelNiagaraSpec`(Niagara 粒子)

```cpp
class FEffectModelNiagaraSpec : public FEffectSpec<UEffectModelNiagara>
{
    TWeakObjectPtr<UNiagaraComponent> NiagaraComponent;   // 持目标组件
    bool RequestForceUpdate = false;
    int32 ExtraState = -1;
    bool LogicIsPaused = false;

public:
    // Ctor 多一个参数:NiagaraComponent(在 Handle Init 时由外部传入)
    FEffectModelNiagaraSpec(UEffectModelNiagara* M, FEffectHandleInfo* Info, UNiagaraComponent* Nc);
    
    // 每帧做:timeScale==0 暂停 Niagara;非 0 用 SetNiagaraFrameDeltaTime 下发 DeltaTime;
    //        调 UKuroEffectLibrary::UpdateEffectModelNiagaraSpec 下发 Location/Rotation/Scale/Parameters 曲线
    virtual void OnTick(v8::Isolate* Isolate, float DeltaTime) override;
    
    // 首次播放越过 StartTime 时,通知 Niagara "不再是 Starting 状态"(影响某些初始 FX)
    virtual void OnAfterStart() override;
    
    void SetExtraState(int);    // 运行时切换 ExtraState → 置 RequestForceUpdate 强刷参数
};
```

**OnTick 伪代码**:
```
if (NiagaraComponent invalid || visibility culled) 跳过
if (!IgnoreTimeScale) UpdateNiagara(DeltaTime)
if (EffectModel valid) 下发参数
```

## 子类示例:`FEffectModelGroupSpec`(子特效容器)

```cpp
class FEffectModelGroupSpec : public FEffectSpec<UEffectModelGroup>
{
    TArray<FEffectSpecBase*> Children;          // ← 裸指针!
    TWeakObjectPtr<USceneComponent> GroupComponent;
    bool HasTransformAnim = false;
    
public:
    // Ctor:接收 children HandleId[],通过 FKuroEffectSystemInterface::FindEffectSpecBase 查出子 Spec 裸指针
    FEffectModelGroupSpec(UEffectModelGroup*, FEffectHandleInfo*, USceneComponent*, TArray<int> ChildIds);
    
    // 重写 Tick,而不是 OnTick —— 因为要在 base Tick 之后还要 tick 各 child
    virtual void Tick(v8::Isolate* Isolate, float DeltaSeconds) override
    {
        FEffectSpec::Tick(Isolate, DeltaSeconds);
        for (FEffectSpecBase* Child : Children) {
            if (Child->EffectFlag & EEffectFlag_Stop) continue;
            if (Child->EffectFlag & EEffectFlag_Destroy ||
                !(Child->EffectFlag & EEffectFlag_Start) ||
                Child->EffectFlag & EEffectFlag_Clear) continue;
            Child->Tick(Isolate, DeltaSeconds);
        }
    }
    
    virtual void OnTick(v8::Isolate*, float DeltaTime) override {
        // 调 Info->GroupDelayPlay(Isolate, DeltaTime) 让 TS 控制子特效延迟
        // 若 Location/Rotation/Scale 是曲线,调 UpdateEffectTransform 改组件 Transform
    }
    
    virtual void SeekTo(...) override {
        FEffectSpec::SeekTo(...);
        for (Child : Children) Child->SeekTo(...);   // propagate
    }
    
    virtual void SetIsStoppingTime(bool v) override {
        FEffectSpec::SetIsStoppingTime(v);
        for (Child : Children) Child->SetIsStoppingTime(v);
    }
};
```

**设计重点**:
- **`TArray<FEffectSpecBase*> Children`** 是裸指针!假设 child Handle 先注册后注销,Group 自己构造/析构期间 children 有效(文件里的注释明确说了这种顺序约定)
- **状态检查**:Tick child 前要看 flag——已 `Stop/Destroy/Clear/未 Start` 的 child 不 tick
- **操作传播**:SeekTo / SetIsStoppingTime 等对 Group 的操作要递归到所有 child

## 子类示例:`FEffectModelMultiEffectSpec`(动态数量子特效)

```cpp
// 抽象策略:不同 MultiEffect 类型的摆放算法
class FMultiEffectBase {
    virtual void Init(UEffectModelMultiEffect*) {}
    virtual int GetDesireNum(float PassTime) { return 0; }
    virtual void Update(float DeltaTime, float PassTime, const TArray<int>& Handles) {}
};

// 具体策略:BuffBall(围绕中心旋转的 Buff 球)
class FMultiEffectBuffBall : public FMultiEffectBase {
    int BaseNum, float SpinSpeed, Radius, BaseAngle;
public:
    virtual int GetDesireNum(float PassTime) override {
        // 按 PassTime 累积:越播越多
        return FMath::CeilToInt(this->BaseNum * PassTime - 0.01);
    }
    
    virtual void Update(float Dt, float PassTime, const TArray<int>& Handles) override {
        BaseAngle -= Dt * SpinSpeed;
        // 均匀角度分布 + 插入缓动(最后一个正在"滑入轨道")
        for (i < NumOnTrack) {
            Actor[i].RelativeLocation = (cos(angle), sin(angle), 0) * Radius;
        }
        // ...
    }
};

class FEffectModelMultiEffectSpec : public FEffectSpec<UEffectModelMultiEffect> {
    TArray<int> Handles;            // 当前子特效 Handle Id 列表
    FMultiEffectBase* MultiEffect;   // 策略
    bool LastHiddenInGame = false;
    
    // Ctor:根据 Type 创建对应策略
    ctor: if (Type == BuffBall) MultiEffect = new FMultiEffectBuffBall();
    
    virtual void OnTick(v8::Isolate* Isolate, float DeltaTime) override {
        int DesiredNum = MultiEffect->GetDesireNum(LifeTime.GetPassTime());
        if (Handles.Num() != DesiredNum) {
            EffectHandleInfo->MultiEffectAdjustNumber(Isolate, DesiredNum);  // 调 TS 回调调节数量
        }
        MultiEffect->Update(DeltaTime, LifeTime.GetPassTime(), Handles);
        // Hidden 传播到子特效 Actor...
    }
    
    void AddMultiEffect(int HandleId);    // 外部调用,TS 侧新增子特效回调
    void RemoveMultiEffect(int HandleId);
};
```

**设计模式**:策略模式(Strategy Pattern)——`FMultiEffectBase` 和 `FMultiEffectBuffBall` 是**摆放算法**的抽象。未来加新 `EMultiEffectType`(比如直线形 / 棋盘形)只要加新策略子类。

**重点**:**调整数量这件事本身由 TS 做**——Spec 只调 `EffectHandleInfo->MultiEffectAdjustNumber(Isolate, DesiredNum)` 通知,TS 侧实际 spawn/destroy 子 Handle,然后调回 `AddMultiEffect(Id)` 登记。这是 Old "业务逻辑在 TS 侧" 的典型。

## 与其他实体的关系

- **基类** → 被 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 持有为 `FEffectSpecBase*`(裸指针)
- **持有**:
  - `FEffectLifeTime` 值成员
  - `T* EffectModel`(裸 UObject)
  - `FEffectHandleInfo*` 裸指针
  - 其他类型专有字段(如 Niagara 的 TWeakObjectPtr<UNiagaraComponent>)
- **调 TS 回调**:通过 `EffectHandleInfo->EnterStopping / GroupDelayPlay / MultiEffectAdjustNumber`
- **调 JS-safe API**:`UKuroEffectLibrary::XXX` 提供了包装

## Twin(New 版对应)

- [[entities/project-game/kuro-effect-system-new/effect-spec-base|IEffectSpecBase + KuroEffect::FEffectSpec<T>]]

### Delta 速记

| 维度 | Old | New |
|---|---|---|
| 抽象模式 | `FEffectSpecBase` 抽象类(含 LifeTime 字段) | `IEffectSpecBase` **纯接口**(I 前缀,80+ 纯虚) |
| 模板层 | `FEffectSpec<T>` 持 EffectModel + Info | `FEffectSpec<T>` 持 TStrongObjectPtr\<T\> + TWeakPtr\<Handle\> |
| EffectModel 所有权 | 裸 `T*` | `TStrongObjectPtr<T>` |
| Handle 连接 | 裸 `FEffectHandleInfo*` | `TWeakPtr<FEffectHandle> Handle` + Spec 中 Init 时 SetHandle |
| v8 | Tick/SeekTo 都带 Isolate | 全部 Tick(float) / SeekTo(float) 纯 C++ |
| 生命周期接口 | Tick/SeekTo/OnPostTick/SetIsStoppingTime | +Init/Start/End/Clear/Destroy/Replay/PreFirstPlay/Play/CanStop/PreStop/Stop/EnableChanged/OnParentInit/OnBeginDelayPlay/OnHiddenInGameChange/RegisterBodyEffect/UnregisterBodyEffect/UpdateBodyEffect |
| Addition TimeScale | 无 | TMap\<int32, float\> AdditionTimeScaleMap,多来源叠乘 |
| 工厂 | 散落在 RegisterEffectHandle 各重载 | `FEffectSpecFactory::CreateEffectSpec(FEffectSpecData)` 集中 |

## 引用来源

- 原始代码:`F:\...\KuroEffectSystem\EffectSpec\EffectSpec.hpp`(~248 行)
- 子类 .hpp:同目录 12 个
- 使用者:[[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 的 `EffectSpec` 字段
- 另见:[[entities/project-game/kuro-effect-system/effect-spec-subclasses|Old Spec 子类目录]]

## 开放问题

- `Tick` 前的 "GlobalStoppingTime 检测" 里有 `if (!IsInStoppingTime)` 和 `} else { return; }` 两个分支——第二个分支的 return 意味着已在 Stopping 态的特效不再 tick。但 Stopping 期间不是应该还 tick 直到 EndTime 才真正停?待 Batch 6 .cpp 确认。
- `FEffectLifeTime::GlobalStoppingPlayTime` 是全局的"停滞期最大播放时长"——用它做保底防止 Stopping 态卡死。但既然 `IsInStoppingTime = true` 之后 tick 就返回,这个 GlobalStoppingPlayTime 检测只在进入 Stopping 那一帧触发一次?
- `FEffectModelGroupSpec::Children` 裸指针的生命周期保障:文件注释说 "GroupSpec 注册在 Child 之后、注销在 Child 之前"——这个顺序是谁保证的?`FKuroEffectSystem::RegisterEffectHandle` 传 children 前确保已注册?
- Multi 的 `AddMultiEffect` / `RemoveMultiEffect` 调用者是谁?(TS 侧收到 `MultiEffectAdjustNumber` 后回调?)
