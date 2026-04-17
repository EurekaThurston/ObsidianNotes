---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, script-bridge, bridge]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/ScriptBridge
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectSystemScriptBridge, FEffectSystemJsBridge, FEffectSystemCSharpBridge]
---

# FEffectSystemScriptBridge(脚本桥三件套)

> Bridge 层的**核心 3 个类**。`FEffectSystemScriptBridge` 是门面,`FEffectSystemJsBridge` 和 `FEffectSystemCSharpBridge` 是它下面的两条具体实现路径(Puerts JS/TS 和 C#)。

## 三者关系

```cpp
// FEffectSystemScriptBridge.h
class FEffectSystemScriptBridge
{
    bool bEffectSystemJsBridgeValid = false;
public:
    FEffectSystemJsBridge     EffectSystemJsBridge;      // JS 路径(值成员)
    FEffectSystemCSharpBridge EffectSystemCSharpBridge;  // C# 路径(值成员)
    
    void Initialize(v8::Isolate* Isolate);
    void Clear(bool IsForce = false);
    
    /* 29 个上行 API,内部 dispatch 到 JsBridge 或 CSharpBridge */
    
    void OnPostUpdateWorkTick(float DeltaTime);
};
```

**值成员**持两个子 Bridge——`FEffectSystemScriptBridge` 就一个实例(作为 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::EffectSystemScriptBridge]] static 字段),里面装了 JsBridge 和 CSharpBridge 两个实例。

## FEffectSystemScriptBridge(门面)

### 29 个"上行"API

```cpp
// SkeletalMesh 渲染组件管理
UActorComponent* SkeletalMeshSpec_OnBodyEffectChange(float Opacity, UActorComponent*, USkeletalMeshComponent*, AActor*) const;
UActorComponent* SkeletalMeshSpec_CreateRenderingComponent(UActorComponent*, USkeletalMeshComponent*, AActor*) const;
void SkeletalMeshSpec_DestroyRenderingComponent(AActor*, UActorComponent*) const;

// Entity 查询
AActor* EffectHandle_GetEntityOwnerActor(int32 EntityId) const;
int32   EffectHandle_GetEntityModelConfigId(int32 EntityId) const;
FName   EffectHandle_GetOrAddEffectDynamicGroup(float EffectEnableRange) const;

// Wwise 音效
UAkComponent* AudioSystem_GetAkComponent(bool FromPrimaryRole, AActor*) const;
void  AudioSystem_ExecuteActionStop(int32 EventHandle, float FadeOutTime) const;
int32 AudioSystem_PostEventTransform(const FString& EventName, const FTransformDouble&) const;
int32 AudioSystem_PostEventAkComponent(const FString& EventName, UAkComponent*) const;

// Spec 条件查询
bool NiagaraSpec_IsNeedQualityBias(int32 EntityId) const;
bool PostProcessSpec_IsNeedPostEffect(int32 EntityId, bool VisibleForProtoPlayer) const;
bool PostProcessSpec_IsDisableInUltraSkill(int32 EntityId) const;

// ActorSystem(Kuro 全局 Actor 池)
AActor* ActorSystem_Get(UClass*, const FTransformDouble&) const;
bool    ActorSystem_Put(const FName& Reason, AActor*) const;

// EffectSystem 业务查询
void EffectSystem_SetEffectView(AActor* EffectActor, int32 EffectId) const;
bool EffectSystem_CheckIsNetPlayer(int32 EntityId) const;
bool EffectSystem_CheckMobileBlackEffect(const FName& Path) const;

// BodyEffect 生命周期
void EffectSpec_RegisterBodyEffect(int32, AActor*, USkeletalMeshComponent*, UObject*, UEffectModelBase*) const;
void EffectSpec_UnregisterBodyEffect(int32, AActor*, USkeletalMeshComponent*, UObject*, UEffectModelBase*) const;

// Material 渲染组件
UKuroCharRenderingComponent* MaterialSpec_GetRenderingComponentByContext(int32 EntityId, UObject* SourceObject) const;
UKuroCharRenderingComponent* MaterialSpec_GetRenderingComponentBySkeletal(USkeletalMeshComponent*) const;
AActor* MaterialSpec_SpawnRenderActor(USkeletalMeshComponent*) const;
UKuroCharRenderingComponent* MaterialSpec_GetRenderingComponentByRenderActor(AActor*, USkeletalMeshComponent*) const;
int32   MaterialSpec_AddMaterialControllerData(UKuroCharRenderingComponent*, UKuroMaterialControllerDataAsset*) const;
void    MaterialSpec_RemoveMaterialControllerData(UKuroCharRenderingComponent*, int32 Handle) const;
void    MaterialSpec_DestroyRenderingComponent(UKuroCharRenderingComponent*);

// Audio Controller
int32 EffectAudioController_AddPlayEffectAudio(UEffectModelAudio*, AActor*, uint8 HitType) const;
int32 EffectAudioController_AddPlayEffectAudioPriority(UEffectModelAudio*, AActor*, uint8 HitType, int32 Priority) const;
void  EffectAudioController_OnStopEffectAudio(int32 Uid, const FName& Desc) const;
```

### 实现策略(推断,从 bool flag 看)

```cpp
// 伪代码
AActor* FEffectSystemScriptBridge::EffectHandle_GetEntityOwnerActor(int32 EntityId) const
{
    // 优先级:C# bound > JS valid
    if (EffectSystemCSharpBridge.IsEffectHandle_GetEntityOwnerActorBound()) {
        return EffectSystemCSharpBridge.EffectHandle_GetEntityOwnerActor(EntityId);
    }
    if (bEffectSystemJsBridgeValid) {
        return EffectSystemJsBridge.EffectHandle_GetEntityOwnerActor(EntityId);
    }
    return nullptr;
}
```

**dispatch 规则**:
- **C# 已绑定 → 走 C#**(C# 可能比 JS 更快/更准)
- **JS 有效 → 走 JS**
- **都没有 → 返回默认/nullptr**

具体优先级策略待 .cpp(Batch 6)确认。

### Initialize / Clear / OnPostUpdateWorkTick
```cpp
void Initialize(v8::Isolate* Isolate);      // 启动
void Clear(bool IsForce = false);            // 清理
void OnPostUpdateWorkTick(float DeltaTime);  // 每帧末回调,分发给两个子 Bridge 做 holder 清理等
```

---

## FEffectSystemJsBridge(JS/Puerts 实现)

```cpp
class FEffectSystemJsBridge
{
    // 30 个 v8::Global<v8::Function>
    v8::Global<v8::Function> SkeletalMeshSpecOnBodyEffectChangeGlobal;
    v8::Global<v8::Function> SkeletalMeshSpecCreateRenderingComponentGlobal;
    // ... (30 个)
    
public:
    v8::Isolate* MainIsolate = nullptr;
    v8::Global<v8::Context> DefaultContextGlobal;
    
    void Initialize(v8::Isolate* Isolate);
    void Clear(bool IsForce = false);
    
    // 大 Register 方法:一次性注册所有 30 个函数(30 个参数!)
    void RegisterJsFunction(const v8::Local<v8::Function>& SkeletalMeshSpecOnBodyEffectChange, ... /* 29 个 */);
    
    /* 29 个 Call* 方法——和 FEffectSystemScriptBridge 同名 */
    
    // Holder 管理
    TArray<TSharedPtr<IJsFunctionHolder>> JsFunctionHolders;
    TSharedPtr<FEffectSpawnCallbackHolder> CreateSpawnCallbackHolder(...);
    TSharedPtr<FEffectFinishCallbackHolder> CreateFinishCallbackHolder(...);
    TSharedPtr<FEffectDynamicInitCallbackHolder> CreateDynamicInitCallbackHolder(...);
    void OnPostUpdateWorkTick(float DeltaTime);
};
```

### 30 个 v8::Global 字段

都是 `v8::Global<v8::Function>` 类型的成员,命名一致(`XxxFunctionGlobal`)。存的是**TS 侧注册过来的 JS 函数的持久句柄**——跨帧、跨函数调用一直有效,直到明确 Reset 或 Isolate 关闭。

### `RegisterJsFunction`:30 参数的 setter

```cpp
void RegisterJsFunction(
    const v8::Local<v8::Function>& SkeletalMeshSpecOnBodyEffectChange,
    const v8::Local<v8::Function>& SkeletalMeshSpecCreateRenderingComponent,
    // ... 28 个参数
);
```

**一次性注册 30 个**——TS 启动时调一次,把所有回调塞进来。函数太长所以写成一个方法而不是 30 个 Setter。

### `JsFunctionHolders`:holder 集合

```cpp
TArray<TSharedPtr<IJsFunctionHolder>> JsFunctionHolders;
```

- **所有** JS Holder(Spawn/Finish/DynamicInit)**集中放这里**
- `OnPostUpdateWorkTick` 帧末扫一遍,清理"应该清理的"Holder(可能是 IsValid()==false 的)

### 调用机制(推断)

```cpp
// 伪代码
AActor* FEffectSystemJsBridge::EffectHandle_GetEntityOwnerActor(int32 EntityId) const
{
    v8::Isolate* Isolate = MainIsolate;
    v8::HandleScope HS(Isolate);
    v8::Local<v8::Context> Ctx = DefaultContextGlobal.Get(Isolate);
    v8::Context::Scope CS(Ctx);
    
    v8::Local<v8::Function> Fn = EffectHandleGetEntityOwnerActorGlobal.Get(Isolate);
    if (Fn.IsEmpty()) return nullptr;
    
    v8::Local<v8::Value> Args[] = { v8::Integer::New(Isolate, EntityId) };
    v8::MaybeLocal<v8::Value> MaybeResult = Fn->Call(Ctx, Ctx->Global(), 1, Args);
    
    // 解析返回值...
    v8::Local<v8::Value> Result;
    if (MaybeResult.ToLocal(&Result)) {
        return FV8Utils::GetUObject<AActor>(Isolate, Result);  // Puerts helper
    }
    return nullptr;
}
```

**每次调用都经过 HandleScope 和 ContextScope**——v8 的规范调用方式。`FV8Utils` 是 Puerts 提供的类型转换 helper。

---

## FEffectSystemCSharpBridge(C# 实现)

```cpp
class FEffectSystemCSharpBridge
{
    // 30 个 FKuroEffectXxx Dynamic Delegate (不是 v8::Global!)
    FSkeletalMeshSpecOnBodyEffectChangeRetVal       SkeletalMeshSpec_OnBodyEffectChange_Internal;
    FSkeletalMeshSpecCreateRenderCompRetVal         SkeletalMeshSpec_CreateRenderingComponent_Internal;
    // ... (30 个)
    
public:
    void Initialize();
    void Clear(bool IsForce = false);
    void RegisterCSharpFunction(
        const FSkeletalMeshSpecOnBodyEffectChangeRetVal& In..., /* 30 个 */ );
    
    /* 29 个 Is<X>Bound() 查询 + 29 个 Call X 方法 */
    
    // Holder
    TArray<TSharedPtr<ICSharpFunctionHolder>> CSharpFunctionHolders;
    TSharedPtr<FCSharpEffectSpawnCallbackHolder> CreateSpawnCallbackHolder(...);
    TSharedPtr<FCSharpEffectFinishCallbackHolder> CreateFinishCallbackHolder(...);
    TSharedPtr<FCSharpEffectDynamicInitCallbackHolder> CreateDynamicInitCallbackHolder(...);
    void OnPostUpdateWorkTick(float DeltaTime);
};
```

### 30 个 UE Dynamic Delegate 字段

和 JS Bridge **对称**,但**每个类型都是 UE Dynamic Delegate**:
```cpp
FSkeletalMeshSpecOnBodyEffectChangeRetVal SkeletalMeshSpec_OnBodyEffectChange_Internal;
```

类型定义在 [[entities/project-game/kuro-effect-system-new/scripting-bridge-architecture|Reflection/KuroEffectDefine.h]]:
```cpp
DECLARE_DYNAMIC_DELEGATE_RetVal_FourParams(UActorComponent*, FSkeletalMeshSpecOnBodyEffectChangeRetVal,
    float, Opacity, UActorComponent*, CharRenderingComponent, 
    USkeletalMeshComponent*, SkeletalMeshComponent, AActor*, Owner);
```

### Is<X>Bound() 每个都有!

```cpp
bool IsSkeletalMeshSpec_OnBodyEffectChangeBound() const;
bool IsEffectHandle_GetEntityOwnerActorBound() const;
// ...(29 个 Is*Bound())
```

**为什么 JS 没有 IsBound**?
- `v8::Global<v8::Function>` 可以 `IsEmpty()` 判断,但判断逻辑在调用处
- UE Dynamic Delegate 有 `.IsBound()` 成员方法,但 C# Bridge **外部显式暴露出来**——可能为了门面层预检查

### 调用机制(推断)

```cpp
// 伪代码
AActor* FEffectSystemCSharpBridge::EffectHandle_GetEntityOwnerActor(int32 EntityId) const
{
    if (!EffectHandle_GetEntityOwnerActor_Internal.IsBound()) return nullptr;
    return EffectHandle_GetEntityOwnerActor_Internal.Execute(EntityId);
    // UE 反射自动 marshal 到 C#
}
```

**UE Dynamic Delegate 的 `Execute` 直接调**——没有 HandleScope / ContextScope 的开销。性能可能比 JS 调用好,但跨语言 marshaling 成本在 UHT 那一层(通常是每个参数做类型转换)。

---

## 两路 Bridge 并排对照

| 维度 | FEffectSystemJsBridge | FEffectSystemCSharpBridge |
|---|---|---|
| **回调类型** | `v8::Global<v8::Function>` × 30 | `FKuroEffectXxxCallback` (UE Dynamic Delegate) × 30 |
| **注册方式** | `RegisterJsFunction(30 个 v8::Local)` | `RegisterCSharpFunction(30 个 FKuroEffectXxxCallback)` |
| **调用前检查** | 隐式(HandleScope 内 Fn.IsEmpty) | 显式 `Is*Bound()` 方法 |
| **调用开销** | v8 HandleScope + ContextScope + Call | `Delegate.Execute` + UE 反射 marshaling |
| **Holder 基类** | `IJsFunctionHolder` | `ICSharpFunctionHolder` |
| **3 种 Holder** | `FEffectSpawnCallbackHolder` / `FEffectFinishCallbackHolder` / `FEffectDynamicInitCallbackHolder` | `FCSharpEffect*CallbackHolder`(相同 3 种,C 前缀) |
| **Isolate 依赖** | `MainIsolate` + `DefaultContextGlobal` | 无 |

### 为什么 Holder 要分两种

业务侧注册回调时,C++ 要把 `TSharedPtr<FEffectInitCallback>` 这种 **C++ 原生类型** 暴露给脚本。但:
- **JS 侧**:业务写的是 `(result, effectId) => void` TS 函数,转换成 `v8::Global<v8::Function>`
- **C# 侧**:业务写的是 `(uint8 result, int32 handle) => void` C# 方法,转换成 `FKuroEffectInitCallback`(UE Dynamic Delegate)

Holder 把各语言的原生 callback 表达**包装成统一的 `TSharedPtr<FEffectInitCallback>`**(C++ 原生 UE Delegate)暴露给内部使用。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::EffectSystemScriptBridge]] **持有**(static)
- **使用的脚本函数签名**:见 [[entities/project-game/kuro-effect-system-new/scripting-bridge-architecture|Reflection/KuroEffectDefine.h]] 的 30 个 DECLARE_DYNAMIC_DELEGATE
- **产生的 Holder**:进入 [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel]] 作为 `TSharedPtr<FEffectInitCallback>` 等
- **调用者**:各种 Spec([[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec]] `RegisterBodyEffect` 等都走 `FEffectSystem::EffectSystemScriptBridge.*`)

## Twin(Old 版对应)

**Old 没有分层 Bridge**。回调路径是:
- 业务调 `RegisterEffectCommonJsFunction(Isolate, PreStop, UpdateLifeCycle, EnterStopping)` 
- `FEffectHandle` 转发给 `FEffectHandleInfo::RegisterCommonFunction`
- 存 `v8::Global<v8::Function>` 成员
- Spec/LifeTime Tick 时直接调 `EffectHandleInfo->PreStop(Isolate)` 等

**没有抽象层次**——JS 耦合贯穿 Handle/Info/LifeTime/Spec 全链。

## 引用来源

- 原始代码:
  - `F:\...\ScriptBridge\EffectSystemScriptBridge.h`(~60 行)
  - `F:\...\ScriptBridge\EffectSystemJsBridge.h`(~143 行)
  - `F:\...\ScriptBridge\EffectSystemCSharpBridge.h`(~216 行)
- Holders:见 [[entities/project-game/kuro-effect-system-new/effect-function-holders|EffectFunctionHolders]]

## 开放问题

- Bridge 路由(JS vs C#)的实际策略(并行/优先级/fallback)在 .cpp
- `OnPostUpdateWorkTick` 每帧扫描 Holder 列表的具体清理逻辑
- C# Bridge 在运行中动态添加/移除回调的支持(Register 是一次性全注册,但能否 re-Register?)
- v8 Bridge 如何处理 Isolate 未就绪的情况(`bEffectSystemJsBridgeValid` 何时 true)
- 某些方法在两 Bridge 里都有但非全部对称?具体哪些只有一版?(需逐方法核对 .cpp)
