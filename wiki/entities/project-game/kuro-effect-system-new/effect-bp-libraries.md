---
type: entity
category: blueprint-library
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, blueprint, function-library]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/Reflection
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [UKuroEffectSystemFunctionLibrary, UKuroEffectSystemHandleHelperLibrary, BP API]
---

# BP 函数库(脚本/蓝图主入口)

> `UKuroEffectSystemFunctionLibrary`(~500 行)+ `UKuroEffectSystemHandleHelperLibrary`(~130 行)—— 两个 `UBlueprintFunctionLibrary` 派生类,构成**脚本和蓝图访问特效系统的主要入口**。方法全是 `UFUNCTION(BlueprintCallable, static)`。

## UBlueprintFunctionLibrary 是什么

UE 的 `UBlueprintFunctionLibrary` 是**静态函数容器**——:
- 派生自 `UObject`,但从不实例化
- 所有方法必须是 `static UFUNCTION`
- 方法在 BP / TS / C# 里以**函数节点**直接调用(不需要 instance)
- 等效于 "C++ 静态工具类"

放在这里的 API = "脚本和蓝图可以拿到的所有特效 API"。

## 文件 1:UKuroEffectSystemFunctionLibrary

**500 行,~100 个 UFUNCTION**。按职责分类罗列:

### 系统生命周期
```cpp
static bool Initialize(UGameInstance*, TArray<FKuroEffectSpecData>, TArray<FKuroEffectSpecChildData>,
                       bool IsGameRunning, 4 个 float, UClass* EffectViewClass, bool IsResetSpecData);
static void Clear();
static void ClearPool(bool IsInternal);
static bool HasInitialize();
static bool HasEffectForSpecData();
static void UpdateIsGameRunning(bool);
static void RefreshEffectForSpecData(TArray<FKuroEffectSpecData>, TArray<FKuroEffectSpecChildData>, bool);
static void InitStaticGlobalData(bool UseLog, bool IsInEditorTick, bool UseDbConfig);
```

### Bridge 注册(C# 路径)
```cpp
static void RegisterJsFunction(
    const FSkeletalMeshSpecOnBodyEffectChangeRetVal& Cb1,
    const FSkeletalMeshSpecCreateRenderCompRetVal& Cb2,
    // ... 27 个同类 Dynamic Delegate 参数
);
```

**注意**:虽然叫 `RegisterJsFunction` 但参数类型是 **FKuroEffectXxx Dynamic Delegate**——也就是**C# 版 Bridge** 的注册入口。命名可能是历史遗留(早期只有 JS,后加 C# 时重用了名字)。

### Spawn 四大家族(核心!)

**每种 Context 类型都有 SpawnUnloopedEffect + SpawnEffect + SpawnEffectWithActor 共 3 种调用**,所以:

| Context 类型 | Spawn 3 种 |
|---|---|
| `FKuroEffectContext`(基础) | `SpawnUnloopedEffect` / `SpawnEffect` / `SpawnEffectWithActor` |
| `FKuroEffectAudioContext` | `SpawnUnloopedEffectFromAudioContext` / `SpawnEffectFromAudioContext` / `SpawnEffectWithActorFromAudioContext` |
| `FKuroSkeletalMeshEffectContext` | `SpawnUnloopedEffectFromSkeletalContext` / `SpawnEffectFromSkeletalContext` / `SpawnEffectWithActorFromSkeletalContext` |
| `FKuroEffectRuntimeGhostEffectContext` | `SpawnUnloopedEffectFromGhostContext` / `SpawnEffectFromGhostContext` / `SpawnEffectWithActorFromGhostContext` |

**共 12 个 SpawnXxx 方法**。

**为什么不共用一个带多态 Context 参数的方法?**

因为**UE 蓝图不支持结构体多态参数**——蓝图节点参数类型必须固定。所以每种 Context 类型**得**一个单独方法。C# 侧同理(不能用引用基类接受派生结构体)。

### 标准 SpawnEffect 签名
```cpp
UFUNCTION(BlueprintCallable, Category = "KuroEffectSystem")
static int32 SpawnEffect(
    UObject* WorldContext,
    const FTransformDouble& Transform,
    const FString& Path,                          // ← FString,非 FName
    const FString& Reason,                        // ← FString
    const FKuroEffectContext& Context,             // 按值传!
    const FKuroEffectBeforeInitCallback& BeforeInitCallback,
    const FKuroEffectInitCallback& Callback,
    const FKuroEffectBeforePlayCallback& BeforePlayCallback,
    const FKuroEffectOnClearCallback& OnClearCallback,
    uint8 EffectType = 3,                         // EEffectType_Scene
    bool bPrepare = false,
    bool bForceCreateActor = false
);
```

**类型对比 C++ 原生 `FEffectSystem::SpawnEffect`**:
- `Path`:`FString`(BP 友好)vs `FName`(C++ 原生)
- `Reason`:`FString` vs `FName`
- `EffectType`:`uint8` vs `EEffectType` 枚举
- `Context`:`const FKuroEffectContext&` vs `TUniquePtr<FEffectContext>&`(引用传入的 UniquePtr!)

实现时:
- `FName::FromString(Path)` / `::FromString(Reason)`
- `static_cast<EEffectType>(EffectType)`
- `FEffectContextHelper::CreateUniqueContext(&Context, OutUniqueContext)`(从 USTRUCT 版 copy 到 C++ 原生)
- 然后转发到 `FEffectSystem::SpawnEffect`

### SpawnEffectWithActor(带 Actor 的重载)

签名不同:**不接收 Spawn 回调四连**,只收基本参数 + 已有的 Actor。业务已经有一个 Actor 想附加特效时用这个。

```cpp
static int32 SpawnEffectWithActor(
    UObject* WorldContext, AActor* Actor, 
    const FString& Path, const FString& Reason,
    const FKuroEffectContext& Context,
    bool bAutoPlay = true, bool bIsExternalActor = true,
    uint8 EffectType = 3);
```

### 常用操作(~40 个方法)

```cpp
static bool StopEffectById(int32 Handle, const FString& Reason, bool Immediately, bool bDestroyActor);
static bool IsEffectValid(int32 Id);
static bool IsEffectActorValid(int32 Id);
static AActor* GetSureEffectActor(int32 Id);
static UNiagaraComponent* GetSureNiagaraComponent(int32 Id);
static bool HasNiagaraComponentHandle(int32 Id);
static void ReplayEffect(int32, const FString& Reason, const FTransformDouble&, bool bResetTransform);
static bool IsPlaying(int32 Id);
static void SetHandleLifeCycle(int32, float Time);
static void SetTimeScale(int32, float, bool bIgnoreGlobalTimeScale);
static void SetAdditionTimeScale(int32 SourceType, int32 Id, float);
static void SetAdditionTimeScaleEnable(int32 SourceType, bool);
static bool GetAdditionTimeScaleEnable(int32 SourceType);
static void FreezeHandle(int32, bool, bool bForce);
static bool IsHandleFreeze(int32);
static bool HandleSeekToTime(int32, float, bool, bool bForce);
static void HandleSeekToTimeWithProcess(int32, float, bool, float Delta);
static float GetSeekToTargetTime(int32);
static FString GetPath(int32);
static void SetThreeStageTime(int32, float StartTime, float LoopTime, float EndTime, bool bResetPassTime);
static void SetEffectParameterNiagara(int32, const FKuroEffectNiagaraParametersStruct&);
static void SetEffectDataFloatConstParam(int32, FName, float);
static void SetEffectExtraState(int32, int32);
static void SetEffectIgnoreVisibilityOptimize(int32, bool);
static void SetEffectStoppingTime(int32, bool);
static void AttachToEffectSkeletalMesh(int32, AActor*, FName Socket, EAttachmentRule);
static void AttachSkeletalMesh(int32, const FKuroSkeletalMeshEffectContext&);
static void CollectMaterialFloatCurve(int32, FName Key, const FKuroCurveFloat&);
static void CollectMaterialVectorCurve(int32, FName Key, const FKuroCurveVector&);
static void CollectMaterialLinearColorCurve(int32, FName Key, const FKuroCurveLinearColor&);
static UEffectModelBase* GetEffectModel(int32);
static float GetTotalPassTime(int32);
static float GetPassTime(int32);
static void SetEffectQualityLevel(int32, int32);
static void StopEffectFromEntity(int32 EntityId, FName FilterName, bool Immediately);
static void StopLimitedEffectFromEntityImmediately(int32 EntityId, bool RootOnly);
static uint8 GetEffectSpecDataType(const FName&);
static bool IsEffectSpecDataContainsType(const FName&, uint8);
```

### Callback 管理
```cpp
static void DynamicRegisterSpawnCallback(int32 EffectId, const FKuroEffectInitCallback&, const FKuroEffectOnClearCallback&);
static void AddFinishCallback(int32 Id, const FKuroEffectFinishCallback&, const FKuroEffectOnClearCallback&);
static void RemoveFinishCallback(int32 Id);
static void ForceCheckPendingInit(int32 Handle);
```

### GlobalTimeScale / Stopping Time
```cpp
static void  SetGlobalTimeScale(float);
static void  OnGlobalTimeScaleChange();
static float GlobalStoppingPlayTime();
static bool  GlobalStoppingTime();
static void  SetGlobalStoppingTime(bool, float PlayTime);
```

### BodyEffect
```cpp
static void UpdateBodyEffect(int32, float Opacity, bool Visible, bool CastShadow);
```

### Debug / 统计
```cpp
static void DebugUpdate(int32, bool);
static int32 GetEffectCount();
static int32 GetActiveEffectCount();
static void DebugPrintAllErrorEffects();
static void DebugPrintCurrentImportanceEffects();
static void DebugPrintEffect();
static int32 GetPlayerEffectLruSize(int32 Pos);
static int32 GetEffectLruCount(const FString& Path);
static int32 GetEffectLruCapacity();
static void  SetEffectLruCapacity(int32);
static int32 GetEffectLruSize();
static float GetLastPlayTime(int32);
static float GetLastStopTime(int32);
static void TickHandleInEditor(int32, float Delta);
static bool EffectIsLoop(int32);
static void SetUseDebugDrawNew(bool);
```

### 全局状态同步
```cpp
static void OnUiSceneStateChange(EKuroUI3DState);
static void OnIsInEditorTickChange(bool);
static void OnTickSystemPausedChange(bool);
static void OnEffectQualityBiasRemoteChange(float);
static void OnDisableOtherEffectChange(bool);
static void OnPlayerEffectContainerFormationLoaded(const TArray<FKuroSceneTeamItem>&);
```

---

## 文件 2:UKuroEffectSystemHandleHelperLibrary

**ActorHandle + NiagaraComponentHandle 的 BP 版** —— 用 `int32 Id` 间接操作。

### ActorHandle_* 方法(16 个)

```cpp
static bool ActorHandle_IsValid(int32 Id);
static void ActorHandle_SetActorHiddenInGame(int32 Id, bool);

// Attach
static void ActorHandle_K2_AttachToActor(int32 Id, AActor*, FName Socket, 3 个 EAttachmentRule, bool bWeldSimulatedBodies);
static void ActorHandle_K2_AttachToComponent(int32 Id, USceneComponent*, FName Socket, 3 个 EAttachmentRule, bool);

// Getters
static FVectorDouble ActorHandle_GetActorLocation(int32);
static FVectorDouble ActorHandle_D_K2_GetActorLocation(int32);
static FRotator      ActorHandle_K2_GetActorRotation(int32);
static FVectorDouble ActorHandle_D_GetActorScale3D(int32);

// Setters
static bool ActorHandle_D_K2_SetActorLocation(int32, const FVectorDouble&, bool bSweep, FHitResult& SweepHit, bool bTeleport);
static bool ActorHandle_K2_SetActorRotation(int32, const FRotator&, bool bTeleportPhysics);
static void ActorHandle_D_SetActorScale3D(int32, const FVectorDouble&);
static bool ActorHandle_D_K2_SetActorLocationAndRotation(int32, const FVectorDouble&, const FRotator&, bool, FHitResult&, bool);
static void ActorHandle_D_K2_AddActorWorldOffset(int32, const FVectorDouble&, bool, FHitResult&, bool);
static bool ActorHandle_D_K2_SetActorTransform(int32, const FTransformDouble&, bool, FHitResult&, bool);
static void ActorHandle_D_K2_SetActorRelativeLocation(int32, const FVectorDouble&, bool, FHitResult&, bool);
static void ActorHandle_K2_SetActorRelativeRotation(int32, const FRotator&, bool, FHitResult&, bool);
static void ActorHandle_D_K2_SetActorRelativeTransform(int32, const FTransformDouble&, bool, FHitResult&, bool);
static void ActorHandle_K2_AddActorLocalTransform(int32, const FTransform&, bool, FHitResult&, bool);
```

### NiagaraComponentHandle_* 方法(14 个)

```cpp
static bool NiagaraComponentHandle_IsValid(int32);
static bool NiagaraComponentHandle_GetForceSolo(int32);
static void NiagaraComponentHandle_SetForceSolo(int32, bool);
static void NiagaraComponentHandle_SetNiagaraVariableFloat(int32, const FString&, float);
static void NiagaraComponentHandle_SetNiagaraVariableVec3(int32, const FString&, const FVector&);
static void NiagaraComponentHandle_SetNiagaraVariableLinearColor(int32, const FString&, const FLinearColor&);
static void NiagaraComponentHandle_SetIntParameter(int32, const FName&, int32);
static void NiagaraComponentHandle_SetFloatParameter(int32, const FName&, float);
static void NiagaraComponentHandle_SetColorParameter(int32, const FName&, const FLinearColor&);
static void NiagaraComponentHandle_SetVectorParameter(int32, const FName&, const FVector&);
static void NiagaraComponentHandle_SetKuroNiagaraEmitterFloatParam(int32, const FString&, const FString&, float);
static void NiagaraComponentHandle_SetKuroNiagaraEmitterVectorParam(int32, const FString&, const FString&, const FVector4&);
static void NiagaraComponentHandle_SetKuroNiagaraEmitterCustomTexture(int32, const FString&, const FString&, UTexture*);
static void NiagaraComponentHandle_SetCastShadow(int32, bool);
static void NiagaraComponentHandle_SetEnviInteractionComp(int32, UKuroEnviInteractionComponent*);
```

**和 [[entities/project-game/kuro-effect-system-new/niagara-component-handle|FNiagaraComponentHandle]] 里的方法一一对应**,前缀 `NiagaraComponentHandle_`,第一参数是 `int32 Id`。

**实现套路**:
```cpp
// 伪代码
void UKuroEffectSystemHandleHelperLibrary::NiagaraComponentHandle_SetFloatParameter(int32 Id, const FName& Name, float Param)
{
    FNiagaraComponentHandle* Handle = FEffectSystem::GetEffectHandle(Id)->GetNiagaraComponentHandle();
    if (Handle) Handle->SetFloatParameter(Name, Param);
}
```

脚本只握 `int32 Id`,C++ 内部每次查表找到真 Handle,然后调用。

## 和原生 C++ EffectSystemHandleHelper 的区别

[[entities/project-game/kuro-effect-system-new/scripting-bridge-architecture|Batch 1 已读的 EffectSystemHandleHelper]]:
- 纯 C++ static helper 类(`class FEffectSystemHandleHelper`)
- 方法签名和这里**完全一致**,只是不带 `UFUNCTION(BlueprintCallable)` 标注
- **C++ 业务可以直接调** `FEffectSystemHandleHelper::ActorHandle_K2_AttachToActor(Id, ...)`
- 脚本/蓝图**必须**走 `UKuroEffectSystemHandleHelperLibrary` 的反射版本

两个文件是**"C++ 内部 API" vs "反射对外 API"**的镜像关系。

## 整体 API 分层视图

```
┌─ BP/TS/C# 业务代码 ───────────────────────────────┐
│                                                  │
│ KuroEffectSystem.SpawnEffect(path, ctx, ...)       │
│ ActorHandle.SetActorHidden(id, true)               │
│                                                  │
└──────────┬────────────────────┬──────────────────┘
           │                    │
           ▼                    ▼
┌─ BP 函数库(反射,跨语言) ────────────────────────┐
│                                                  │
│  UKuroEffectSystemFunctionLibrary                  │
│     100+ UFUNCTION(BlueprintCallable, static)      │
│                                                  │
│  UKuroEffectSystemHandleHelperLibrary              │
│     30 UFUNCTION(BlueprintCallable, static)        │
│                                                  │
└──────────┬───────────────────────────────────────┘
           │(FString→FName, USTRUCT→原生 Context 等 marshal)
           ▼
┌─ C++ 原生 API(内部) ──────────────────────────────┐
│                                                  │
│  FEffectSystem::SpawnEffect(FName, TUniquePtr...)  │
│  FEffectSystemHandleHelper::ActorHandle_K2_...     │
│  ScriptBridge::EffectHandle_GetEntityOwnerActor(.) │
│                                                  │
└──────────────────────────────────────────────────┘
```

**分层好处**:
1. 反射层(BP Lib)是"脏工作"——负责类型转换和 BP/TS 友好封装
2. 原生 C++ 层干净——没有 FString/USTRUCT 等 BP 友好但性能差的类型
3. **性能关键路径永不走 BP Lib**,业务 C++ 代码直接调原生 API

## 与其他实体的关系

- **调用目标**:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的全部静态方法
- **反射类型输入**:[[entities/project-game/kuro-effect-system-new/kuro-effect-reflection|KuroEffectDefine 的所有 USTRUCT]]
- **类型转换辅助**:[[entities/project-game/kuro-effect-system-new/effect-context|FEffectContextHelper::CreateUniqueContext]](USTRUCT → 原生)
- **实现对应的纯 C++ helper**:`FEffectSystemHandleHelper`(Batch 1 已读)
- **暴露的目标**:TS(Puerts)/ C# / Blueprint

## Twin(Old 版对应)

**Old 没有 `U*FunctionLibrary`**。BP/TS 访问路径:
- [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]] 提供 `KuroEffectLibrary.h`(UBlueprintFunctionLibrary,本批未读)——有一些 BP 工具函数
- 但**主要操作靠 TS 直接调 FKuroEffectSystem 的实例方法** via Puerts 绑定
- 缺少**蓝图层面的统一 Spawn API**
- **BP 用户无法从头 Spawn 一个特效**,只能用预配好的 BP Actor

New 提供 UKuroEffectSystemFunctionLibrary 大大扩展了 BP 的特效能力——可以用 BP 动作图直接 Spawn/操作特效,不需要写 TS/C++ 代码。

## 引用来源

- 原始代码:
  - `F:\...\KuroEffectSystemNew\Reflection\KuroEffectSystemFunctionLibrary.h`(~507 行)
  - `F:\...\KuroEffectSystemNew\Reflection\KuroEffectSystemHandleHelperLibrary.h`(~134 行)
- 底层调:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]]、[[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]]、[[entities/project-game/kuro-effect-system-new/niagara-component-handle|FNiagaraComponentHandle]]
- Puerts 绑定:[[entities/project-game/kuro-effect-system-new/scripting-bridge-architecture|EffectSystemForPuerts.h]]

## 开放问题

- `Initialize`(BP 版)的参数签名和 C++ 原生版参数对齐情况—— 如 `EffectViewClass` 在原生是 `UClass*`,BP 版可能是 `TSubclassOf<AActor>` 更友好
- **SpawnEffect 的 12 个重载**:实现层是否存在公共辅助函数?还是每个都独立 copy 大部分逻辑?(Batch 6 .cpp)
- `SpawnEffectWithActor` 不收 Spawn 回调四连—— 是**因为外部 Actor 的 init 已在外部完成**,Spawn 回调无意义?
- `SetUseDebugDrawNew(bool)` 的功能——切换到某个"新"的 debug draw 路径?为什么需要这个开关?
- `KuroEffectLibrary.h`(Old)和这里的差别详细对比(Old 那个文件 Batch 1 没读)
