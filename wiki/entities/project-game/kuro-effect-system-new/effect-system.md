---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, entry-point]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSystem.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[entities/project-game/kuro-effect-system/kuro-effect-system]]
aliases: [KuroEffect::FEffectSystem, FEffectSystem (New)]
---

# KuroEffect::FEffectSystem(新版入口类)

> New KuroEffectSystem 的入口类;**全 static 模块单例**,位于 `KuroEffect` 命名空间;特效系统的所有全局状态和入口 API 都以 static 的形式挂在这里。

## 概览

对 [[entities/project-game/kuro-effect-system/kuro-effect-system|Old FKuroEffectSystem]] 的彻底重写。关键模式改变:

- **没有实例**——全部数据是 `static` 成员(包括 `MainWorld`、容器、计数器、配置)。
- **命名空间**——位于 `KuroEffect`,所以和 Old 的同名类型(`FEffectHandle`、`FEffectLifeTime`)不冲突。
- **明确的数据 / 运行时分离**——`FEffectSpecData`(config) + `FEffectInitModel`(创建参数) + `FEffectContext`(运行时上下文) + `IEffectSpecBase`(运行时 Spec)四层。
- **池化**——`TLru<FName, FEffectHandle> LruPool` 按 Path 组织回池。
- **回调分段**——`BeforeInit → Init → BeforePlay → Finish` 四段可注入。
- **编辑器 / PIE 感知**——`OnBeginPIE` / `OnEndPIE` / `OnEditorCurrentMapFinishExit`(Old 完全没有)。
- **零 v8 入侵**——类体内除 `Initialize(v8::Isolate*, ...)` 外,看不到 `v8::Isolate*`;脚本相关全部推给 `FEffectSystemScriptBridge` 和 `FEffectSystemHandleHelper`(Batch 5)。

## 关键 static 字段(按类别)

### 单例 / 核心容器
```cpp
static TWeakObjectPtr<UWorld> MainWorld;
static uint32 HandleCount;           // 含子特效的总句柄数
static uint32 EffectCount;           // 不含子特效
static uint32 HoldObjectCount;
static TArray<TSharedPtr<FEffectHandle>> Effects;             // 强持有
static TMap<int32, TSharedPtr<FEffectHandle>> TickHandleMap;  // 需要 Tick 的集合
static TQueue<int32> RemoveHandleQueue;
static TSet<TSharedPtr<FEffectHandle>> PendingDeleteHandles;
static TSet<int32> PendingRemoveHandles;
```

### 数据层
```cpp
static TMap<int32, FEffectSpecData> EffectForSpecMap;
static TMap<int32, TArray<int32>> EffectForSpecChildMap;
static TMap<int32, FEffectSpecData> EffectForSpecDbCacheMap;
static TStrongObjectPtr<UHoldPreloadObject> HoldPreloadObject;
```

### 池化
```cpp
static TLru<FName, FEffectHandle> LruPool;       // ← Old 无
static FPlayerEffectContainer PlayerEffectContainer;  // ← Old 无
```

### Id 生成(位编码)
```cpp
static uint8 Digit;       // 总位数
static uint8 IndexDigit;  // Index 占位
static uint8 VersionDigit;// Version 占位
static int32 MaxIndex;
static int32 MaxVersion;
static TArray<int32> Versions;
static TArray<int32> Indexes;
```
—— HandleId 看起来是 `(Version << IndexDigit) | Index` 形式;Version 递增防止 ABA(回池复用 slot 时)。具体算法待读 .cpp(Batch 6)。

### 全局运行时配置
```cpp
static bool UseLog, IsGameRunning, IsMobile, IsInEditorTick, IsTickSystemPaused, UseDbConfig, DisableOtherEffect;
static float GlobalTimeScale;
static EKuroUI3DState UiSceneState;
static int32 EffectQualityBiasRemote;
```

### 回调 Delegate
```cpp
static FDelegateHandle JsEnvCleanDelegateHandle;
static FDelegateHandle PostLoadLevelDelegate;
#if WITH_EDITOR
static FDelegateHandle OnEndPIEDelegate, OnBeginPIEDelegate, OnEditorCurrentMapFinishExitDelegate;
#endif
```

### 附属系统(值成员)
```cpp
static FKuroEffectVisibilityOptimizeController VisibilityOptimizeController;  // 复用 Old 同名类!
static FContinuousEffectController ContinuousEffectController;                // New 独有
static FEffectSystemScriptBridge EffectSystemScriptBridge;                    // New 独有
```

## 核心 API(按 region 切分,照搬代码注释)

### `Initialize`:启动入口
```cpp
static bool Initialize(v8::Isolate* Isolate, UGameInstance* GameInstance,
                       const TArray<FEffectSpecData>& SpecData,
                       const TArray<FEffectSpecChildData>& SpecChildData,
                       bool InIsGameRunning,
                       float InBoundsVisibleThreshold, float InMaxVisibleCullDeltaTime,
                       float InWasRecentlyRenderInterval, bool InUseVisibilityTestPass,
                       UClass* EffectViewClass, bool IsResetSpecData);
```
—— **唯一一处 `v8::Isolate*` 出现**,用于注册 JsEnv Clean delegate。

### EffectSystem 内部 API(`#pragma region EffectSystem内部用 API`)
- `SpawnEffectWithActor` / `SpawnChildEffect` —— 内部 / 蓝图用
- `AddRemoveHandle` / `StopEffect` / `OnEffectActorEndPlay`
- `InitHandleWhenEnable` / `ForceCheckPendingInit` —— Pending Init 机制,New 特有,见 [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 的 `PendingInit` region
- `UpdateBodyEffect(EffectId, Opacity, Visible, CastShadow)` —— 给角色身上的光效特效下发可视性/阴影
- `GetEffectLruCount` / `GetEffectLruCapacity` / `SetEffectLruCapacity` / `GetEffectLruSize` —— LRU 池管理

### 外部业务 API(`#pragma region 外部业务逻辑调用 API`)

对外最常用的函数,全带 `KUROGAMEPLAY_API` 导出,有 doxygen 注释:

```cpp
// 非循环特效(做了检测)
KUROGAMEPLAY_API static int32 SpawnUnloopedEffect(...);

// 通用 Spawn
KUROGAMEPLAY_API static int32 SpawnEffect(
    UObject* WorldContext, const FTransformDouble& Transform, const FName& Path,
    const FName& Reason,
    const TSharedPtr<FEffectBeforeInitCallback>& BeforeInitCallback,
    const TSharedPtr<FEffectInitCallback>& Callback,
    const TSharedPtr<FEffectBeforePlayCallback>& BeforePlayCallback,
    TUniquePtr<FEffectContext>& Context,
    EEffectType EffectType = EEffectType_Scene,
    bool Prepare = false,
    bool ForceCreateActor = false);

KUROGAMEPLAY_API static bool StopEffectById(int32 Handle, const FName& Reason, bool Immediately, bool DestroyActor = false);
KUROGAMEPLAY_API static bool IsEffectValid(int32 Id);
KUROGAMEPLAY_API static AActor* GetSureEffectActor(int32 Id);
KUROGAMEPLAY_API static void ReplayEffect(int32 Id, const FName& Reason, const FTransformDouble& Transform, bool ResetTransform = false);
KUROGAMEPLAY_API static uint8 GetEffectSpecDataType(const FName& EffectPath);
KUROGAMEPLAY_API static bool IsEffectSpecDataContainsType(const FName& EffectPath, uint8 Type);
KUROGAMEPLAY_API static void AttachToActor(...);
KUROGAMEPLAY_API static void AttachToActorEx(...);
```

其他非导出的业务 API(~30 个)包括:`SetHandleLifeCycle` / `SetTimeScale` / `SetAdditionTimeScale` / `FreezeHandle` / `HandleSeekToTime` / `HandleSeekToTimeWithProcess` / `SetThreeStageTime` / `SetEffectParameterNiagara` / `SetEffectExtraState` / `AttachToEffectSkeletalMesh` / `AttachSkeletalMesh` / 各种 `CollectMaterialXxxCurve` / `StopEffectFromEntity` / `StopLimitedEffectFromEntityImmediately` / 等等。

### 容器池化(`#pragma region 容器相关`)
```cpp
static TSharedPtr<FEffectHandle> TryCreateFromContainer(...);  // 尝试从 LRU/Player 容器取
static bool TryRecycleToContainer(const TSharedPtr<FEffectHandle>& Handle);  // 回池
static TSharedPtr<FEffectHandle> CreateEffectHandleFromContainer(const FName& Path, ...);
static bool PutEffectHandleToContainer(const TSharedPtr<FEffectHandle>& Handle);
static bool LruRemoveExternalFromContainer(const TSharedPtr<FEffectHandle>& Handle);
static void OnPlayerEffectContainerFormationLoaded(TArray<FSceneTeamItem>& SceneTeamItems);
```

### OwnerEffect(`#pragma region OwnerEffect相关`)
```cpp
static bool RegisterSyncTimeScaleHandle(int32 OwnerHandle, int32 OwnedHandle);
static bool UnregisterSyncTimeScaleHandle(int32 OwnerHandle, int32 OwnedHandle);
static void ClearOwnerEffectHandle(int32 OwnedHandle, bool FromOwner);
static void DeepCopyAdditionTimeScale(int32 Handle, const TMap<int32, float>& AdditionTimeScaleMap);
```
—— 一个 Handle 可以"属于"另一个 Handle,TimeScale 等状态会同步。Old 没有这个概念。

### Debug(`#pragma region Debug调用 API`)
```cpp
static void DebugTotalEffectHandle(bool bError);
static void DebugUpdate(int32 Id, bool DebugUpdate);
static int32 GetEffectCount();
static int32 GetActiveEffectCount();
static void DebugPrintAllErrorEffects();
static void DebugPrintCurrentImportanceEffects();
static void DebugPrintEffect();
static FString GetEffectHandleStr(const TSharedPtr<FEffectHandle>& Handle);
static TArray<TSharedPtr<FEffectHandle>> GetTickHandles();
static void SetUseDebugDrawNew(bool bUseDebugDrawNew);
```

### 全局时间停滞
```cpp
static float GlobalStoppingPlayTime();
static bool GlobalStoppingTime();
static void SetGlobalStoppingTime(bool StoppingTime, float PlayTime);
```
Old 里这是 `FEffectLifeTime::GlobalStoppingTime` 的静态字段,New 在入口类上抽出显式 API。

## Tick 管线(三段式)

```cpp
static FKuroTickNativeFunction PrePhysicsTickFunction;
static FKuroTickNativeFunction PostPhysicsTickFunction;
static FKuroTickNativeFunction PostUpdateWorkTickTickFunction;

static void Tick(float DeltaTime);
static void PostPhysicsTick(float DeltaTime);
static void PostUpdateWorkTick(float DeltaTime);
```
—— 和 UE 的 `TG_PrePhysics` / `TG_PostPhysics` / `TG_PostUpdateWork` tick group 对齐。Old 入口类里只看到 `PostUpdateWorkTickPtr` 单个。**New 参与 Tick 的时机扩展了 3 倍**——意味着特效系统能在物理前后都处理逻辑,适合与 attach 到动力学/骨骼的特效互动。

## 关键回调(`#if WITH_EDITOR`)

```cpp
static FDelegateHandle OnEndPIEDelegate, OnBeginPIEDelegate, OnEditorCurrentMapFinishExitDelegate;
static void OnEndPIE(const bool bIsSimulating);
static void OnBeginPIE(const bool bIsSimulating);
static void OnEditorCurrentMapFinishExit();
```
—— PIE(Play In Editor)进出、编辑器地图退出时都会触发清理。**Old 完全没有这些,是编辑器里特效残留 bug 的主要来源**——New 明确关掉了这条路径。

## 与其他实体的关系

- **拥有(值成员)**:`FEffectSystemScriptBridge`、`FContinuousEffectController`、`FKuroEffectVisibilityOptimizeController`(**复用 Old 同名类**!)、`FPlayerEffectContainer`(Batch 4 详读)。
- **强依赖**:[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]、`FEffectSpecData`、`FEffectInitModel`、`FEffectContext`、`FEffectInitCallback`、`FEffectBeforeInitCallback`、`FEffectBeforePlayCallback`、`FEffectHandleFinishDelegate`、`FEffectSpecChildData`、`FSkeletalMeshEffectContext`、`FEffectHandle::LruCreator/LruClearer`(LRU 回调)。
- **UE 集成**:`UGameInstance` 初始化入口、`UWorld` 绑定、`AActor` 作为 EffectActor、`UKuroPostProcessComponent`、`UKuroBezierMeshComponent` 等复用 Old 的组件。

## Twin(Old 版对应)

- [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]](Old)

### Delta 速记
- instance → static
- `TMap<int, FEffectHandle*>` 裸指针 → `TArray<TSharedPtr<FEffectHandle>>` + `TMap<int32, TSharedPtr<>>`
- 编译期类型化 Register × 7 → 单个 Path-based `SpawnEffect`
- v8 贯穿 → v8 只在 `Initialize` 出现一次
- 单 Tick 时机 → 三段式 Tick(PrePhysics/PostPhysics/PostUpdateWork)
- 无池 → LRU + PlayerEffectContainer
- 无 PIE 钩子 → PIE 完整生命周期
- 无 Owner 概念 → OwnerEffect + AdditionTimeScale
- 无 Pending 态 → PendingInit pipeline(见 Handle 页)
- 2 段回调 → 4 段回调(Before*/*/Before*/Finish)

## 引用来源

- [[sources/project-game/kuro-effect-system-new/overview|New 系统 overview]]
- 原始代码:`F:\Aki\dev\Source\Client\Plugins\Kuro\KuroGameplay\Source\KuroGameplay\Public\KuroEffectSystemNew\EffectSystem.h`(~800 行头)

## 开放问题

- **Id 编码**:`Digit/IndexDigit/VersionDigit`、`MaxIndex/MaxVersion`、`Versions/Indexes` 数组的具体算法?(Batch 6 读 .cpp)
- **`InitStaticGlobalData` 的调用者**?—— `UseLog/IsInEditorTick/UseDbConfig` 的初始化时机。
- **`HoldPreloadObject`**:`UHoldPreloadObject` 是 UObject,用来在 GC 期间持有被预加载的资源?(Batch 2 看 DA 时追)
- **`BPEffectViewClass`**:`TStrongObjectPtr<UClass>`——是特效的 view 蓝图类?(Batch 4 看 EffectSystemActor 时追)
- **`EffectTimeScaleEnableMap`**:`TMap<int32, bool>`——AdditionTimeScale 的开关表?
- **三段 Tick 的具体职责划分**(哪些工作在 PrePhysics,哪些在 PostPhysics)——Batch 6 读 .cpp。
- **Static 初始化顺序**:这么多 static 成员,初始化依赖顺序稳定吗?`InitStaticGlobalData` 是不是显式兜底?
