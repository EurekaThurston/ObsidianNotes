---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, handle]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectHandle.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[entities/project-game/kuro-effect-system/effect-handle]]
aliases: [KuroEffect::FEffectHandle, FEffectHandle (New)]
---

# KuroEffect::FEffectHandle(新版句柄)

> New 特效系统的句柄核心;`TSharedFromThis` 派生;~265 行头文件、~25 个字段、分 6 个 region;把 Old 里 Handle+Info+LifeTime+部分 System 的职责重新组合后的产物。

## 概览

```cpp
namespace KuroEffect
{
    class FEffectHandle : public TSharedFromThis<FEffectHandle>
    {
        /* 6 个 #pragma region:
           1. TsEffectHandle 完全迁移      ← 核心数据 + 基本操作
           2. Debug 专用                 
           3. InitHanlde 相关             ← PendingInit pipeline
           4. 生命周期                    ← Init/Start/End/Clear/Destroy/Play/Stop...
           5. OwnerEffect 相关            ← 父子句柄 TimeScale 同步
           6. BodyEffect                 ← 人物身上光效的可见性/投影下发
        */
    };
}
```

文件顶部注释 **"TsEffectHandle 完全迁移"** 直接透露意图:以前一部分 Handle 行为是在 TypeScript 那边实现的,这次整体下沉到 C++。这解释了为什么新 Handle 字段和方法这么多——把 TS 侧的状态机也收了。

## 关键字段(按 region 分)

### 核心数据(region "TsEffectHandle 完全迁移")
```cpp
int32 Id = 0;
int32 HoldObjectId = 0;
FName Path;                          // 特效 DataAsset 路径(替代 Old 里的类型重载)
int32 Flag = EEffectFlag_None;
TWeakPtr<FEffectHandle> Parent;
FName CreateReason, StopReason, PlayReason, RemoveReason;  // 操作溯源
bool IsExternalActor = false;
bool IsPendingStop = false;
bool IsPendingPlay = false;
EEffectHandleCreateSource CreateSource = EEffectHandleCreateSource_None;
int32 SourceEntityId = 0;            // 来源实体 id(供 StopEffectFromEntity 用)
bool IsPreview = false;              // 编辑器预览模式
bool InContainer = false;            // 是否在 LRU / PlayerContainer 池内
uint32 EffectEnableRange = EFFECT_ENABLE_RANGE;
float LifeTime = 0.0f;
bool IsInitializing = false;
```

```cpp
// private
TSharedPtr<IEffectSpecBase> EffectSpec;           // ← 替代 Old 的 FEffectSpecBase*
TUniquePtr<FEffectContext> Context;               // ← 替代 Old 部分 Info 的作用
TUniquePtr<FKuroEffectNiagaraParameters> NiagaraParameters;
int32 ExtraState = -1;
bool IgnoreVisibilityOptimizeInternal = false;
bool StoppingTimeInternal = false;
TStrongObjectPtr<AActor> EffectActor;             // ← Old 用 TWeakObjectPtr,New 强持有
TFunction<int32>* OnFinishCallback = nullptr;
bool IsNeedVisibilityTest = false;
TArray<TWeakObjectPtr<AActor>> AttachToActors;
bool LogicHidden = false, BodyEffectHidden = false;
TSharedPtr<FEffectHandleFinishDelegate> FinishDelegate;
TSharedPtr<FNiagaraComponentHandle> NiagaraComponentHandle;
```

**观察**:
- `EffectActor` 从 Old 的 `TWeakObjectPtr` 变成 `TStrongObjectPtr` —— New 明确由 Handle 强持有 EffectActor,避免了 Old 里"Handle 还活着但 Actor 被 GC 掉"的歧义。
- `TSharedPtr<IEffectSpecBase>` —— **接口**(`I` 前缀)+ 共享所有权,非具体 struct。
- `TUniquePtr<FEffectContext>` —— 独占所有权,`SetContext` 是用引用交接 `TUniquePtr&`。

### 静态常量(region "TsEffectHandle 完全迁移")
```cpp
static float EFFECT_USE_BOUNDS_RANGE;
static float MAX_LOOP_EFFECT_WITHOUT_OWNER_TIME_OF_EXISTENCE;
static float EFFECT_ENABLE_RANGE;
static float EFFECT_IMPORTANCE_ENABLE_RANGE;

// 命名原因字符串 (FName,复用)
static FName OnActorEndPlayStopEffectReason;
static FName PlayEffectAfterPendingInitReason;
static FName AttachEffectHandleHiddenReason;
static FName StopHandleHiddenReason;
static FName StopHandleChildHiddenReason;
```
—— `FName` 命名的 Reason 是 New 的风格:每次 Stop / Hide 都带一个理由码,便于 debug 和 log 回溯。Old 没有这个约定。

### Pending Init 相关(region "InitHanlde相关")
```cpp
TUniquePtr<FEffectInitHandle> InitCache;          // 还没初始化完成前的参数缓存
float PendingInitTimeScale = 1.0f;
bool PendingInitIgnoreGlobalTimeScale = false;
bool PendingInitSetAdditionTimeScale = false;

void PendingInit(UObject* WorldContext, const FName& Path, const FName& Reason,
                 const FTransformDouble& Transform,
                 const TSharedPtr<FEffectBeforeInitCallback>& BeforeInitCallback,
                 const TSharedPtr<FEffectInitCallback>& Callback,
                 const TSharedPtr<FEffectBeforePlayCallback>& BeforePlayCallback,
                 bool AutoPlay = false);
void PlayEffectWhenPendingInit();
void InitEffectActorAfterPendingInit();
void PlayEffectAfterPendingInit();
void ClearInitCache();
```

—— 一个完整的"待初始化"管线。业务在 `FEffectSystem::SpawnEffect` 时拿到一个 Handle Id,但 DataAsset 还在异步加载;此时 Handle 处于 Pending 态,能接受 `SetHidden` / `Attach` 等操作,都被缓存到 `FEffectInitHandle`,等 DA 加载完成后一并应用。**Old 不存在这个状态**,所有操作要等 Handle 真正 Init 完才能下。

### 生命周期(region "生命周期")
```cpp
private:
    uint32 GameBudgetManagedTokenInternal = 0;
    FName GameBudgetGroupNameInternal;       // 集成时间预算系统
    TFunction<ELoadEffectResult()> InitFinishCallback;
    bool IsSpawnInUiScene = false;
    float SeekToTargetTime = -1, SeekToDelta = 0;
    bool SeekContinue = false, IsSeeking = false;
    bool IsFreezeInternal = false, PendingFreeze = false;
    bool HasRegisteredTick = false;

public:
    void Init(UEffectModelBase* EffectData, const TSharedPtr<FEffectInitModel>& EffectInitModel);
    bool Start();
    bool End();
    bool Clear();
    void Destroy();
    void Play(FName Reason);
    void PlayEffect(FName Reason);
    void PreStop();
    void Stop(FName Reason, bool Immediately);
    void StopEffect(FName Reason, bool Immediately = false, bool DestroyActor = false);
    void Replay();
    void AfterLeavePool();
    void OnEnabledChange(bool Enable);

    bool TickWithoutGameBudget = false;
    void RegisterTick();
    void UnregisterTick();
    FVectorDouble LocationProxyFunction();
    void Tick(float DeltaTime);                   // ← 注意:没有 v8::Isolate*
    void SeekDelta(float DeltaTime, bool AutoLoop, bool CheckFinish = false) const;
    bool SeekTo(float Time, bool AutoLoop) const;
    void SeekToTimeWithProcess(float Time, float DeltaTime, bool InSeekContinue = false);
```

**关键对比**:Old 的 `Tick(v8::Isolate*, float)`,New 的 `Tick(float)`——脚本回调路径已被挪到了 `ScriptBridge`。

生命周期方法比 Old 丰富得多:
- `Init / Start / End / Clear / Destroy` —— 完整生命周期
- `Play / PlayEffect / PreStop / Stop / StopEffect / Replay` —— 播放控制,`PreStop` 是关键新增(stop 之前的钩子)
- `AfterLeavePool` —— 出 LRU 池时清理
- `OnEnabledChange` —— 可见性变化
- `RegisterTick` / `UnregisterTick` —— 显式 tick 注册(Old 里是整体一次性 tick)

### 判定函数组(state predicates)
```cpp
bool IsRoot() const;
bool IsEffectValid() const;
bool IsDestroy() const;
bool IsDone() const;
bool IsPlaying() const;
bool IsPendingInit() const;
bool IsEffectActorValid() const;
bool IsStopping() const;
bool IsLoop() const;
```
—— 显式多状态判定。Old 里这些"状态"要么散落在 `FEffectSpecBase` 的 flag,要么根本没有(如 `IsPendingInit`)。

### 状态访问
```cpp
FEffectContext* GetContext();
void SetContext(TUniquePtr<FEffectContext>& Value);
void ClearContext();
void MoveContext(TUniquePtr<FEffectContext>& Value);
TWeakPtr<FEffectHandle> GetRoot();

UEffectModelBase* GetEffectData() const;
EEffectType GetEffectType() const;
FEffectActorHandle* GetEffectActor() const;       // 返回 handle,不是 Actor*
AActor* GetSureEffectActor() const;               // 保证拿到的 Actor 实体
FNiagaraComponentHandle* GetNiagaraComponentHandle() const;
void GetNiagaraComponentHandles(TArray<FNiagaraComponentHandle*>& OutHandles) const;
UNiagaraComponent* GetNiagaraComponent() const;
void GetNiagaraComponents(TArray<UNiagaraComponent*>& OutComponents) const;
IEffectSpecBase* GetEffectSpec() const;
```

注释里写:**"这些Handle得避免被Ts的堆内存持有,不然会有崩溃的风险"**——`FEffectActorHandle*` / `FNiagaraComponentHandle*` 都是裸指针,**脚本层不能长期持有**,必须每次通过 `FEffectSystemHandleHelper` 用 int32 id 间接访问(见 Batch 5)。

### TimeScale
```cpp
float GetTimeScale() const;
void SetTimeScale(float Value, bool bForce = false, bool bIgnoreGlobalTimeScale = false);
void SetAdditionTimeScale(int32 SourceType, float Value);
void OnAdditionTimeScaleEnableChanged(int32 SourceType, bool Enable) const;
bool GetIgnoreTimeScale() const;
```
—— **"Addition TimeScale"** 是新增概念:`int32 SourceType` 区分不同来源(如 "技能 slow"、"场景 slow"),各自可独立启用/关闭,互不覆盖。

### Material 曲线下发
```cpp
void CollectMaterialFloatCurve(const FName& Key, const FKuroCurveFloat& Value) const;
void CollectMaterialVectorCurve(const FName& Key, const FKuroCurveVector& Value) const;
void CollectMaterialLinearColorCurve(const FName& Key, const FKuroCurveLinearColor& Value) const;
```
—— 保留了 Old 的 `CollectFloatCurve/VectorCurve/LinearColorCurve` 接口语义,但**命名里加了 `Material` 前缀**,和曾在 Old 里并存的 `CollectEffectFloatConst` 等显式区分。
**注意**:Old 里对"常量下发"(Const)和"曲线下发"(Curve)都有接口;New 的这个文件里 **没看到 Const 系列**——要么合并了,要么移到了 ScriptBridge / Spec 层。

### OwnerEffect(region "OwnerEffect相关")
```cpp
TSet<int32> SyncTimeScaleHandles;
int32 OwnerEffectHandleId = 0;
void ClearOwnedEffectHandles();
public:
    void OnAfterCreateHandle();
    void ClearOwnerEffectHandle(bool FromOwner = false);
    bool RegisterSyncTimeScaleHandle(int32 HandleId);
    bool UnregisterSyncTimeScaleHandle(int32 HandleId);
    void OnTimeScaleChange(float InTimeScale, bool InIgnoreGlobalTimeScale);
    void DeepCopyAdditionTimeScale(const TMap<int32, float>& SourceMap, bool Refresh = true);
```

—— "Owner"关系:句柄 A 可以"拥有"句柄 B,当 A 的 TimeScale 变化,B 自动同步。见 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::RegisterSyncTimeScaleHandle]] 入口。**Old 无此概念**。

### BodyEffect(region "BodyEffect")
```cpp
void UpdateBodyEffect(float Opacity, bool Visible, bool CastShadow) const;
```
—— 只有 1 个方法。配合 `BodyEffectHidden` 标志位,专门给**挂在角色身上的光效**做可见性/透明度/阴影控制。在编辑器角色消失(Evade、Stealth)期间,单独控制其身上特效的显隐。

### Debug 专用
```cpp
bool InDebugMode();
void SetDebugUpdate(bool Value);
void DebugTick(float DeltaTime);
void NiagaraDebugTick(float DeltaTime) const;
EEffectErrorCode GetDebugErrorCode() const;
bool IsImportanceEffect() const;
// 末尾 #if !UE_BUILD_SHIPPING 标记
uint64 BornFrameCount = 0;
// #if KURO_ENABLE_EFFECT_SYSTEM_DEBUG_DRAW
void DebugDraw(FKuroRuntimeDebugDrawVerticalBox* Column, int32 HandleId) const;
bool PassDebugDrawFilter(const FString& Filter) const;
```

## Finish Callback
```cpp
void AddFinishCallback(const TSharedPtr<FEffectHandleFinishDelegate>& InDelegate);
void RemoveFinishCallback();
void ExecuteStopCallback();
```
—— `TSharedPtr<FEffectHandleFinishDelegate>`;Old 里 stop 回调走 JS(`UpdateLifeCycle` 里自己判断是否结束),New 抽象成显式 C++ delegate,**业务代码可以不碰 JS 就接收 finish 事件**。

## 可见性 / 时间停滞 / Attach
```cpp
bool GetIgnoreVisibilityOptimize();
void SetIgnoreVisibilityOptimize(bool Value);
bool GetStoppingTime();
void SetStoppingTime(bool Value);
void OnGlobalStoppingTimeChange(bool InStoppingTime) const;

void AttachToEffectSkeletalMesh(AActor* AttachActor, FName SocketName, EAttachmentRule TransformRule);
void ExecuteAttachToEffectSkeletalMesh(...);  // 在 Init 完成后执行的真正 attach
void AttachSkeletalMesh(TUniquePtr<FSkeletalMeshEffectContext>& InContext);

void SetHidden(bool Hidden, FName Reason, bool IsLogic = false, bool IsBodyEffectHidden = false);
int32 GetFlag() const;
void SetFlag(EEffectFlag Value);
void OnModifyEffectModel() const;
void OnTickSystemPausedChanged(bool InIsPaused);
bool GetCreateFromPlayerEffectPool() const;
UKuroEnviInteractionComponent* GetInteractionEffectComponent();
void RemoveChild(int32 ChildId);
```

—— 注意 `SetHidden` 的参数:`bool IsLogic` 区分是"业务逻辑"(优先级高)还是"系统"触发的隐藏;`bool IsBodyEffectHidden` 再区分是身上光效专用。Old 里没这种细分。

## 与其他实体的关系

- **被 static 管理**:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 `Effects` / `TickHandleMap` / `LruPool` 中。
- **拥有**:
  - `TSharedPtr<IEffectSpecBase> EffectSpec`(接口 + 共享)
  - `TUniquePtr<FEffectContext> Context`
  - `TUniquePtr<FKuroEffectNiagaraParameters> NiagaraParameters`
  - `TStrongObjectPtr<AActor> EffectActor`
  - `TSharedPtr<FNiagaraComponentHandle>`
  - `TSharedPtr<FEffectHandleFinishDelegate>`
  - `TUniquePtr<FEffectInitHandle> InitCache`(Pending 态用)
- **弱引用**:`TWeakPtr<FEffectHandle> Parent`(避免循环)、`TArray<TWeakObjectPtr<AActor>> AttachToActors`。
- **间接依赖**:`FEffectActorHandle`、`FEffectInitHandle`、`FEffectInitModel`、`FEffectNiagaraParameters`、`IEffectSpecBase`、`FEffectSpecData`、`EEffectHandleCreateSource`、`EEffectFlag`、`EEffectType`、`FEffectBeforeInitCallback`、`FEffectInitCallback`、`FEffectBeforePlayCallback`、`FSkeletalMeshEffectContext`、`UKuroEnviInteractionComponent`、`UEffectModelBase`(仅保留为 `GetEffectData()` 的返回类型,作为 DA 查询接口)。

## Twin(Old 版对应)

- [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle (Old)]]

### Delta 速记
- **构造**:7 类型重载 Init → `ctor(FName Path)` + `AfterConstruct(FEffectSpecData)` + `Init(UEffectModelBase*, FEffectInitModel)`(两阶段,数据驱动)
- **Spec 所有权**:`FEffectSpecBase*` 裸 → `TSharedPtr<IEffectSpecBase>` 共享
- **Info/JS 依附**:独立 `FEffectHandleInfo*` + 5 个 `v8::Global` → 全部移走,Handle 无 v8
- **状态**:9 字段 → ~25 字段(+Pending / +Owner / +BodyEffect / +Reasons / +CreateSource / +EntityId / +Preview / +InContainer)
- **生命周期**:`Init/Tick/SeekTo/InitParameter` → +`Start/End/Clear/Destroy/Play/PlayEffect/PreStop/Stop/StopEffect/Replay/AfterLeavePool/OnEnabledChange/RegisterTick/UnregisterTick`
- **回调**:JS functions via v8::Global → `TSharedPtr<FEffectHandleFinishDelegate>`(C++ 原生)
- **新能力**:PendingInit 管线、OwnerEffect、BodyEffect、GameBudget 集成、VisibilityTest 集成、AdditionTimeScale 多来源

## 引用来源

- [[sources/project-game/kuro-effect-system-new/overview|New 系统 overview]]
- 原始代码:`F:\Aki\dev\Source\Client\Plugins\Kuro\KuroGameplay\Source\KuroGameplay\Public\KuroEffectSystemNew\EffectHandle.h`(~265 行头,方法实现全在 .cpp)

## 开放问题

- **`AfterConstruct(const FEffectSpecData&)`** 做什么?推测是根据 SpecData 构造具体 IEffectSpecBase 派生类(走 FEffectSpecFactory);具体 Batch 3 读 `EffectSpecFactory.h` 确认。
- **`GameBudgetManagedTokenInternal` / `GameBudgetGroupNameInternal`**:时间预算系统(KuroTickManager)的 token 语义?
- **`LocationProxyFunction() → FVectorDouble`**:Handle 本身不就有 Actor 吗?为什么要 Proxy?推测是提供给 Tick 系统做重要性评估的位置闭包。
- **`IsImportanceEffect()`** 的判定标准(距离?类型?时间?)
- **`FEffectHandleFinishDelegate`** 的签名(里面 `TFunction` / `TMulticastDelegate`?能挂几条?)
- **`OnFinishCallback`** 裸指针 `TFunction<int32>*`——这个裸指针是谁的?why not TSharedPtr?(可能是跟 Old 兼容的过渡字段,Batch 6 回答)
- **`HoldObjectId`** 与 `FEffectSystem::HoldPreloadObject` 的关联?
