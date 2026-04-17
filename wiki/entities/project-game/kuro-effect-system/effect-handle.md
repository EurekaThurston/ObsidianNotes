---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, handle]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectHandle.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
twin: [[entities/project-game/kuro-effect-system-new/effect-handle]]
aliases: [FEffectHandle (Old)]
---

# FEffectHandle(老版句柄)

> Old 特效系统的句柄类;每个活动特效对应一个实例;持有运行时 Spec + JS 伴生对象,是"特效实例"的具体承载者。

## 概览

一个极度紧凑的数据 + 转发层:

```cpp
class FEffectHandle
{
    friend class FEffectSystemDebugDraw;
    
    int HandleId = 0;
    int ParentId = 0;
    FEffectSpecBase* EffectSpec = nullptr;       // ← 运行时 Spec(裸指针)
    FEffectHandleInfo* EffectHandleInfo = nullptr; // ← JS 伴生对象(裸指针)

    bool IsFreeze = false;
    float SeekToTargetTime = -1;
    float SeekToDelta = 0;
    bool SeekContinue = false;
    bool IsSeeking = false;
public:
    /* 7 个 Init 重载 + Tick + SeekTo + 大量转发方法 */
};
```

核心字段就 9 个:2 个 Id、2 个裸指针(Spec / Info)、5 个 Seek/Freeze 状态。其他所有行为,**这个类都是去调 `EffectSpec->...` 或 `EffectHandleInfo->...`**。

## 两个伴生对象(关键!)

### `FEffectSpecBase* EffectSpec`

**运行时 Spec**,基于 DataAsset 构造的具体类型(`FEffectModelGroupSpec`、`FEffectModelGhostSpec`、`FEffectModelNiagaraSpec` 等 12+ 种)。Handle 几乎所有"跟特效当前状态相关"的方法都转发到它:

```cpp
FORCEINLINE void SetFlag(int InFlag) const
{
    if (EffectSpec)
    {
        EffectSpec->EffectFlag = InFlag;
    }
}

FORCEINLINE void SetIsPlaying(bool IsPlaying) const
{
    if (EffectSpec)
    {
        EffectSpec->SetIsPlaying(IsPlaying);
    }
}
```

### `FEffectHandleInfo* EffectHandleInfo`

**JS 伴生体**,见兄弟文件 `EffectHandleInfo.h`。持有:

- `TWeakObjectPtr<AActor> EffectActor`(UObject 指针弱引用)
- `v8::Global<v8::Object> GlobalJsHandle`(TS 侧的句柄镜像对象)
- 5 个 `v8::Global<v8::Function>` —— `PreStop`、`UpdateLifeCycle`、`EnterStopping`、`GroupDelayPlay`、`MultiEffectAdjustNumber` 的 JS 回调
- 自身实现 `GroupDelayPlay` / `PreStop` / `UpdateLifeCycle` / `EnterStopping` / `MultiEffectAdjustNumber` 方法 —— 就是"调用已注册的 JS 函数"的 thunk。

Handle 的 JS-注册 API 全部是"转给 Info":
```cpp
FORCEINLINE void RegisterJsObject(v8::Isolate* Isolate, v8::Local<v8::Object> JsObject) const
{
    if (EffectHandleInfo)
    {
        EffectHandleInfo->RegisterJsObject(Isolate, JsObject);
    }
}
```

**观察**:这种分工本来很好——Handle 是 C++ 数据,Info 专管 JS 接口。但由于 Handle 自己的方法签名里也仍然出现 `v8::Isolate*`,v8 还是渗透到了 Handle 层面。

## 7 种 Init 签名

这是 Old 最典型的一块——**类型决定 Init 签名**:

```cpp
// 最基础:纯 UEffectModelBase
bool Init(int Id, int ParentId, UEffectModelBase* DA, AActor* EffectActor);
bool Init(..., UEffectModelBase* DA, ..., UActorComponent* Component);

// 组合型:持有子句柄 Id 列表
bool Init(..., UEffectModelGroup* DA, ..., USceneComponent*, TArray<int> Children);

// 幽灵:特殊生命周期参数
bool Init(..., UEffectModelGhost* DA, ..., USkeletalMeshComponent*,
          float GhostLifeTime, float SpawnInterval);

// 后处理:独立材料实例和参数名
bool Init(..., UEffectModelPostProcess* DA, ..., UKuroPostProcessComponent*,
          bool DisablePostProcess, int PostProcessMaterialHandle,
          UMaterialInstanceDynamic* Material, FName LocationParamName);

// 静态网格:动态材质数组
bool Init(..., UEffectModelStaticMesh* DA, ..., UStaticMeshComponent*,
          TArray<UMaterialInstanceDynamic*> DynamicMaterials);

// 轨迹:贝塞尔 + 骨骼 + 材质
bool Init(..., UEffectModelTrail* DA, ..., UKuroBezierMeshComponent*,
          USkeletalMeshComponent*, UMaterialInstanceDynamic*);
```

—— 每种特效类型都要在编译期写一套签名,`EffectSpec = new FEffectModelXxxSpec(...)` 也硬编码在每个 Init 里(推测,具体待 Batch 6 读 .cpp 确认)。**扩展一个新类型要改这里 + 入口 class + Spec 工厂,三处改动**。

## 核心 API(按角色)

### 生命周期
```cpp
~FEffectHandle();  // 构造没有专门 ctor,只有 Init
void OverriderTick(FName GroupTag, EffectToken Token);
void Tick(v8::Isolate* Isolate, float DeltaSeconds);   // ← 还是 v8
```

### Seek 家族
```cpp
void SeekTo(v8::Isolate* Isolate, float Time, bool AutoLoop) const;
void SeekToTimeInFreeze(v8::Isolate* Isolate, float DeltaSeconds);
// Seek 相关 FORCEINLINE
SetIsFreeze(bool), SetSeekToTimeWithProcessInfo(Time, Delta, Continue)
```

### 参数下发到 Spec
```cpp
InitParameter(Flag, TimeScale, IgnoreTimeScale, IsStoppingTime, IsPlaying, IsStopping,
              IsFreeze, SeekToTargetTime, SeekToDelta, SeekContinue, IsSeeking);
SetFlag, SetTimeScale, SetIsStoppingTime, SetIsInStoppingTime, SetIsPlaying, SetIsStopping;
```

### JS 注册(转给 Info)
```cpp
RegisterJsObject(Isolate, JsObject);
RegisterCommonJsFunction(Isolate, PreStop, UpdateLifeCycle, EnterStopping);
RegisterGroupDelayPlayJsFunction(Isolate, GroupPlayFunction);
UnregisterEffectGroupDelayPlayJsFunction();
RegisterMultiEffectAdjustNumJsFunction(Isolate, AdjustNum);
```

### 类型专属
```cpp
// MultiEffect
void AddMultiEffect(int Id) const;
void RemoveMultiEffect(int Id) const;
// Ghost
bool GetGhostSpecCanStop() const;
// Niagara
void SetNiagaraExtraState(int InExtraState) const;
// Material(曲线/常量下发)
void CollectFloatCurve / CollectVectorCurve / CollectLinearColorCurve / CollectEffectFloatConst / ... / RemoveEffectXxxCurveOrConst;
```

### PostTick
```cpp
FORCEINLINE bool NeedPostTick() const { if (EffectSpec) return EffectSpec->NeedPostTick(); return false; }
FORCEINLINE void OnPostTick(float DeltaTime) const { if (EffectSpec) EffectSpec->OnPostTick(DeltaTime); }
```

### 访问
```cpp
FORCEINLINE TWeakObjectPtr<AActor> GetEffectActor() const;
FORCEINLINE FEffectSpecBase* GetEffectSpec() const;
```

## `EffectToken`

文件顶部 `typedef uint32 EffectToken;`。只在 `OverriderTick(HandleId, GroupTag, EffectToken)` 出现。**具体语义未知,是 Tick override 的授权令牌?或是优先级?待 Batch 6 回答**。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]] **持有**:`TMap<int, FEffectHandle*> ProxyTickHandleMap`。
- **拥有**(裸指针,所有权不明):`FEffectSpecBase* EffectSpec`、`FEffectHandleInfo* EffectHandleInfo`。
- **依赖的 DataAsset 类型**(全部在 Init 重载里硬编码):
  - `UEffectModelBase`(基类) / `UEffectModelGroup` / `UEffectModelGhost` / `UEffectModelPostProcess` / `UEffectModelStaticMesh` / `UEffectModelTrail`
  - 相关组件:`UActorComponent` / `USceneComponent` / `USkeletalMeshComponent` / `UKuroPostProcessComponent` / `UStaticMeshComponent` / `UKuroBezierMeshComponent`
- **`FEffectSpecBase`** 的具体子类(运行时 new 出来):`FEffectModelGroupSpec` 等 — Batch 3 细读。

## Twin(New 版对应)

- [[entities/project-game/kuro-effect-system-new/effect-handle|KuroEffect::FEffectHandle]] — New 版

### Delta 速记
- **所有权**:裸 `FEffectSpecBase*` + `FEffectHandleInfo*` → `TSharedPtr<IEffectSpecBase>` + `TUniquePtr<FEffectContext>`(Info 被拆分到 Context / Bridge)
- **构造**:7 个类型化 Init 重载 → 单个 ctor `FEffectHandle(const FName& Path)` + `AfterConstruct(const FEffectSpecData&)` 的两阶段
- **JS 依附**:Handle 方法签名含 `v8::Isolate*`,Info 持有 5 个 v8::Global → 全部迁出到 `FEffectSystemScriptBridge` + `EffectJsFunctionHolder`(Batch 5)
- **新增**:`TSharedFromThis<>`、`EEffectHandleCreateSource`、`SourceEntityId`、`IsPreview`、`InContainer`、`EffectEnableRange`、`LifeTime`、Pending Init、Finish Delegate、OwnerEffect 同步、BodyEffect 下发
- **DataAsset 依赖**:编译期硬编码 6 种 → 运行时按 `FName Path` 查 `FEffectSpecData`
- **字段规模**:~10 个字段 → ~25 个字段(但其中一半是状态机相关,而非类型专属参数)

## 引用来源

- [[sources/project-game/kuro-effect-system/overview|Old 系统 overview]]
- 原始代码:`F:\Aki\dev\Source\Client\Plugins\Kuro\KuroGameplay\Source\KuroGameplay\Public\KuroEffectSystem\EffectHandle.h`(~230 行)
- 伴生文件:`EffectHandleInfo.h`(~100 行,见上文"两个伴生对象"节)

## 开放问题

- **Init 到 Spec 的工厂逻辑**:每个 Init 重载怎么决定 `EffectSpec = new FEffectModelXxxSpec(...)`?(.cpp,Batch 6)
- **析构**:`~FEffectHandle()` 是否 delete `EffectSpec` / `EffectHandleInfo`?谁负责?(.cpp,Batch 6)
- **`EffectToken`** 语义(uint32 代表什么?)
- **`OverriderTick`** 与 `Tick` 的差别 —— 从命名看前者用于 Group 内某个子特效按 parent 的 GroupTag tick,但怎么 override 原有 tick?(Batch 6)
- **MultiEffect 运行时状态** 存在 Spec 还是 Info 里?`AddMultiEffect(int Id)` 传的 id 是子特效 HandleId 吗?
