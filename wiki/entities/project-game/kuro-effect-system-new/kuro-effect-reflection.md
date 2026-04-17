---
type: entity
category: reflection
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, reflection, ustruct, delegate]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/Reflection/KuroEffectDefine.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [KuroEffectDefine, Reflection Layer, FKuroEffectContext, FKuroEffectSpecData (USTRUCT)]
---

# KuroEffectDefine — 脚本反射层定义

> New 独有。**把 C++ 原生类型包装成 UE USTRUCT / Dynamic Delegate** 暴露给蓝图/C#/Puerts。这层是"脚本看得到的 C++ 世界"——每个跨语言类型都在这里有一个反射副本。

## 文件内容概览

`Reflection/KuroEffectDefine.h` 305 行,分三块:

1. **29 个 Dynamic Delegate** 声明(给 C# Bridge 用)
2. **Spawn / Init / Finish 的 Callback 类型**(业务侧入参)
3. **USTRUCT 版的 Context / SpecData / Parameter**

## 1️⃣ 29 个 Bridge 回调 Delegate

对应 [[entities/project-game/kuro-effect-system-new/effect-system-script-bridge|FEffectSystemScriptBridge]] 的 29 个上行 API,每个都有一个 UE Dynamic Delegate:

### 带返回值的(RetVal_*)
```cpp
// 返回 UActorComponent*,接收 4 个参数
DECLARE_DYNAMIC_DELEGATE_RetVal_FourParams(UActorComponent*, FSkeletalMeshSpecOnBodyEffectChangeRetVal,
    float, Opacity,
    UActorComponent*, CharRenderingComponent,
    USkeletalMeshComponent*, SkeletalMeshComponent,
    AActor*, Owner);

DECLARE_DYNAMIC_DELEGATE_RetVal_ThreeParams(UActorComponent*, FSkeletalMeshSpecCreateRenderCompRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(AActor*, FEffectHandleGetEntityOwnerActorRetVal, int32, EntityId);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(int32, FEffectHandleGetEntityModelConfigIdRetVal, int32, EntityId);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(FName, FEffectHandleGetOrAddEffectDynamicGroupRetVal, float, EffectEnableRange);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(UAkComponent*, FAudioSystemGetAkComponentRetVal, bool, FromPrimaryRole, AActor*, Actor);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(int32, FAudioSystemPostEventTransformRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(int32, FAudioSystemPostEventAkComponentRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(bool, FNiagaraSpecIsNeedQualityBiasRetVal, int32, EntityId);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(bool, FPostProcessSpecIsNeedPostEffectRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(bool, FPostProcessSpecIsDisableInUltraSkillRetVal, int32, EntityId);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(AActor*, FActorSystemGetRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(bool, FActorSystemPutRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(bool, FEffectSystemCheckIsNetPlayerRetVal, int32, EntityId);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(bool, FEffectSystemCheckMobileBlackEffectRetVal, const FString&, Path);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(UKuroCharRenderingComponent*, FMaterialSpecGetRenderingComponentByContextRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(UKuroCharRenderingComponent*, FMaterialSpecGetRenderingComponentBySkeletalRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(AActor*, FMaterialSpecSpawnRenderActorRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(UKuroCharRenderingComponent*, FMaterialSpecGetRenderingComponentByRenderActorRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_TwoParams(int32, FMaterialSpecAddMaterialControllerDataRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_ThreeParams(int32, FEffectAudioControllerAddPlayEffectAudioRetVal, ...);
DECLARE_DYNAMIC_DELEGATE_RetVal_FourParams(int32, FEffectAudioControllerAddPlayEffectAudioPriorityRetVal, ...);
```

### 无返回值的
```cpp
DECLARE_DYNAMIC_DELEGATE_TwoParams(FSkeletalMeshSpecDestroyRenderingComponent, AActor*, EffectActor, UActorComponent*, CharRenderingComponent);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FAudioSystemExecuteActionStop, int32, EventHandle, float, FadeOutTime);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FEffectSystemSetEffectView, AActor*, Actor, int32, EffectId);
DECLARE_DYNAMIC_DELEGATE_FiveParams(FEffectSpecRegisterBodyEffect, ...);
DECLARE_DYNAMIC_DELEGATE_FiveParams(FEffectSpecUnregisterBodyEffect, ...);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FMaterialSpecRemoveMaterialControllerData, ...);
DECLARE_DYNAMIC_DELEGATE_OneParam(FMaterialSpecDestroyRenderingComponent, ...);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FEffectAudioControllerOnStopEffectAudio, int32, Uid, const FString&, Context);
```

### UE 宏命名规则
- `DECLARE_DYNAMIC_DELEGATE_<N>Params(Name, T1, arg1, T2, arg2, ..., Tn, argN)` — 无返回值
- `DECLARE_DYNAMIC_DELEGATE_RetVal_<N>Params(RetType, Name, T1, arg1, ..., Tn, argN)` — 有返回值
- N ∈ {`One/Two/Three/Four/Five/...`};0 参用 `DECLARE_DYNAMIC_DELEGATE(Name)` 没 RetVal

## 2️⃣ Spawn / Init / Finish 业务回调

**给业务侧调 SpawnEffect 用的回调类型**(作为参数传):

```cpp
// Spawn 四段式回调
DECLARE_DYNAMIC_DELEGATE_OneParam(FKuroEffectBeforeInitCallback, int32, Handle);
DECLARE_DYNAMIC_DELEGATE_TwoParams(FKuroEffectInitCallback, uint8, Result, int32, Handle);
DECLARE_DYNAMIC_DELEGATE_OneParam(FKuroEffectBeforePlayCallback, int32, Handle);
DECLARE_DYNAMIC_DELEGATE(FKuroEffectOnClearCallback);                       // 无参无返回

// AddFinishCallback 用
DECLARE_DYNAMIC_DELEGATE_OneParam(FKuroEffectFinishCallback, int32, Handle);
```

**注意**:`Result` 类型是 `uint8` 而非 `ELoadEffectResult`。**原因**:UE Dynamic Delegate 能穿过的值类型有限制,enum 要 cast 成 `uint8`。

**两套 Init 回调命名**:
- C++ 原生:`FEffectInitCallback`(定义在 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine]])
- 反射版:`FKuroEffectInitCallback`(本文件)
- **不同类型!** Holder 负责转换

## 3️⃣ Context USTRUCT 家族(**关键!**)

和 [[entities/project-game/kuro-effect-system-new/effect-context|C++ 原生 Context 类]]并行存在的 USTRUCT 版本。

### `FKuroEffectContext`(基类)
```cpp
USTRUCT(BlueprintType)
struct FKuroEffectContext
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    int32 EntityId;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    UObject* SourceObject;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    bool DisablePostProcess;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    int32 CreateFromType;      // ← 原生是 uint16

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    int32 PlayFlag;            // ← 原生是 uint16

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    bool CreateFromBpEffectActor;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    uint8 ContextType;         // ← 原生是 enum EEffectContextType

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    int32 HitEffectType;       // ← 原生是 uint8

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "KuroEffectContext")
    FName AnsSlotName;
};
```

**类型差异**:
- USTRUCT 版的 `int32 CreateFromType` vs 原生 `uint16`
- USTRUCT 版的 `uint8 ContextType` vs 原生 `enum EEffectContextType`
- USTRUCT 版的 `int32 HitEffectType` vs 原生 `uint8`

**为什么不一致**?蓝图/BP 里 `uint16` 不方便暴露(要填 int/byte),改 int32 对业务更友好。ctor 转换时做收敛检查。

### 派生类
```cpp
USTRUCT(BlueprintType)
struct FKuroSkeletalMeshEffectContext : public FKuroEffectContext
{
    GENERATED_BODY()
    UPROPERTY() USkeletalMeshComponent* SkeletalMeshComponent;
    UPROPERTY() bool IsSyncTimeDilation;
    UPROPERTY() bool IsSyncEventTimeToEffectTime;
};

USTRUCT(BlueprintType)
struct FKuroEffectAudioContext : public FKuroEffectContext
{
    GENERATED_BODY()
    UPROPERTY() bool FromPrimaryRole;
};

USTRUCT(BlueprintType)
struct FKuroEffectRuntimeGhostEffectContext : public FKuroSkeletalMeshEffectContext
{
    GENERATED_BODY()
    UPROPERTY() float SpawnRate;
    UPROPERTY() bool  UseSpawnRate;
    UPROPERTY() float SpawnInterval;
    UPROPERTY() float GhostLifeTime;
    UPROPERTY() bool  UseBaseColorTex;
};
```

**USTRUCT 继承关系**和 C++ 原生 Context 完全对齐:
```
FKuroEffectContext  ─┬──── FKuroSkeletalMeshEffectContext ───── FKuroEffectRuntimeGhostEffectContext
                     └──── FKuroEffectAudioContext
```

## 4️⃣ SpecData USTRUCT

```cpp
USTRUCT(BlueprintType)
struct FKuroEffectSpecData
{
    GENERATED_BODY()
    UPROPERTY() int32 Id;
#if WITH_EDITORONLY_DATA
    UPROPERTY() FName Path;
#endif
    UPROPERTY() uint8 SpecType;
    UPROPERTY() uint8 EffectRegularType;
    UPROPERTY() float LifeTime;
};

USTRUCT(BlueprintType)
struct FKuroEffectSpecChildData
{
    GENERATED_BODY()
    UPROPERTY() int32 Id;
    UPROPERTY() TArray<int32> Children;
};
```

**和 C++ 原生 [[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]]**:
- 字段顺序完全一致
- `#if WITH_EDITORONLY_DATA` 条件也保留
- 只有类名多个 `Kuro` 前缀

## 5️⃣ Parameter USTRUCT

**和 [[entities/project-game/kuro-effect-system/effect-parameters|Old 的 FKuroEffectNiagaraParameters]] 顶部注释对应**——那段注释说:"业务 Ts 传参用 UObject 快,但反射构造开销大,频繁调用可以改成纯 C++ 类"。**USTRUCT 就是那个"UObject 级别"版本**,反射友好但每次跨语言调用走反射 marshaling。

### 4 种基础参数
```cpp
USTRUCT(BlueprintType) struct FKuroParameterFloat       { FName Name; float Value; };
USTRUCT(BlueprintType) struct FKuroParameterLinearColor { FName Name; FLinearColor Value; };
USTRUCT(BlueprintType) struct FKuroParameterVector      { FName Name; FVector Value; };
USTRUCT(BlueprintType) struct FKuroParameterArrayVector { FName Name; TArray<FVector> Value; };
```

### Niagara 参数包 USTRUCT
```cpp
USTRUCT(BlueprintType)
struct FKuroEffectNiagaraParametersStruct
{
    GENERATED_BODY()
    UPROPERTY() TArray<FKuroParameterFloat>       UserParameterFloat;
    UPROPERTY() TArray<FKuroParameterLinearColor> UserParameterColor;
    UPROPERTY() TArray<FKuroParameterVector>      UserParameterVector;
    UPROPERTY() TArray<FKuroParameterArrayVector> UserParameterArrayVector;
    UPROPERTY() TArray<FKuroParameterFloat>       MaterialParameterFloat;
    UPROPERTY() TArray<FKuroParameterLinearColor> MaterialParameterColor;
};
```

**和 C++ 原生 `FKuroEffectNiagaraParameters`** 字段结构**完全对齐**。差异只是类名带 `Struct` 后缀,元素类型换成 `FKuroParameterXxx`(带 UPROPERTY 的版本)。

在 [[entities/project-game/kuro-effect-system/effect-parameters|FKuroEffectNiagaraParameters]] 里见过的这个 ctor:
```cpp
FKuroEffectNiagaraParameters(const FKuroEffectNiagaraParametersStruct& Other)
```
—— **USTRUCT → 原生的转换**。业务 TS 传 `FKuroEffectNiagaraParametersStruct`,进到 C++ 后用这个 ctor 转成运行时原生版本。

## 6️⃣ SceneTeamItem USTRUCT
```cpp
USTRUCT(BlueprintType)
struct FKuroSceneTeamItem
{
    GENERATED_BODY()
    UPROPERTY() int32 EntityId;
    UPROPERTY() bool IsMyRole;
};
```

和 [[entities/project-game/kuro-effect-system-new/player-effect-container|FSceneTeamItem]] 对应。`UKuroEffectSystemFunctionLibrary::OnPlayerEffectContainerFormationLoaded(TArray<FKuroSceneTeamItem>)` 接受的就是 USTRUCT 版。

## 外部全局
```cpp
extern bool GKuroCppEffectSystemDetailedLog;
```
**模块级日志开关**。在 EffectSpec.hpp 里见过使用位置——详细日志开关。

## 总结:为什么要这层

| 需求 | 解决方式 |
|---|---|
| 业务 TS/C# 能**定义 Context 参数** | USTRUCT + UPROPERTY |
| 业务 TS/C# 能**接收 C++ 回调** | DECLARE_DYNAMIC_DELEGATE |
| 业务 TS/C# 能**把 C# 方法作为 Bridge 回调注册** | DECLARE_DYNAMIC_DELEGATE(RetVal_) |
| 在蓝图节点参数里**看得到 Context 字段** | UPROPERTY(EditAnywhere, BlueprintReadWrite) |
| **性能敏感的** 内部循环**不走反射** | 和原生 C++ 类双版本,边界 ctor 转换 |

## 与其他实体的关系

- **对应** [[entities/project-game/kuro-effect-system-new/effect-context|C++ 原生 Context 类]](4 对)
- **对应** [[entities/project-game/kuro-effect-system-new/effect-spec-data|C++ 原生 FEffectSpecData]](USTRUCT 版)
- **对应** [[entities/project-game/kuro-effect-system/effect-parameters|C++ 原生 FKuroEffectNiagaraParameters]](USTRUCT Struct 版)
- **对应** [[entities/project-game/kuro-effect-system-new/player-effect-container|FSceneTeamItem]](USTRUCT 版)
- **29 个 DECLARE_DYNAMIC_DELEGATE 被**:[[entities/project-game/kuro-effect-system-new/effect-system-script-bridge|FEffectSystemCSharpBridge]] 作为字段
- **5 个业务 Callback 被**:[[entities/project-game/kuro-effect-system-new/effect-bp-libraries|UKuroEffectSystemFunctionLibrary]] 作为 UFUNCTION 参数

## Twin(Old 版对应)

**Old 的反射层非常弱**:
- 没有 USTRUCT 版 Context(所有 Context 信息塞在 Handle 字段)
- 没有 USTRUCT 版 SpecData(无此概念)
- Niagara 参数有 Old `FKuroEffectNiagaraParameters`(Parameter 裸类),**也没 USTRUCT 版本**
- 没有 Dynamic Delegate 支持 C#,因为根本不支持 C#

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\Reflection\KuroEffectDefine.h`(~305 行)
- 搭档:
  - [[entities/project-game/kuro-effect-system-new/effect-context|C++ 原生 Context]] 的转换 ctor
  - [[entities/project-game/kuro-effect-system/effect-parameters|FKuroEffectNiagaraParameters]] 的转换 ctor

## 开放问题

- USTRUCT 版 Context 的 `CreateFromType` 为 `int32`(原生 `uint16`)—— UE 反射对 uint16 的支持确实不完全,但 **int32 会浪费字段空间**(32bit vs 16bit)。是否在 runtime 做 downcast 回 uint16 存储?Batch 6 确认。
- `#if WITH_EDITORONLY_DATA FName Path` 让 Shipping 包少 16 字节——但 UPROPERTY 反射系统需要知道每个字段的 offset,跨配置构建的脚本也要考虑这点
- DECLARE_DYNAMIC_DELEGATE 的 `FKuroEffectInitCallback` 参数是 `uint8 Result`,和 C++ 原生 `FEffectInitCallback` 的 `ELoadEffectResult` 不同——Holder thunk 里做 cast
- USTRUCT Context 的析构/拷贝行为(UE 默认值类型,有虚函数吗?)
- `GKuroCppEffectSystemDetailedLog` 在哪些地方被赋值/开关?(可能还是 CVar)
