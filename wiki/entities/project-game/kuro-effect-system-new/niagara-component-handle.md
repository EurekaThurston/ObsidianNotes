---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, niagara, handle]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/NiagaraComponentHandle.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FNiagaraComponentHandle]
---

# FNiagaraComponentHandle(Niagara 组件参数缓存门面)

> New 独有。**UNiagaraComponent 的参数写入缓存层**——Handle 在真正拿到 NiagaraComponent 前,业务可能已经要设参数了。这个类把所有参数先攒到 TMap 里,Component 可用时一次性 flush;并提供一组与 UNiagaraComponent 同名的方法让外部无感切换。

## 为什么需要这个类

Niagara 参数下发的时机问题:
- `FEffectHandle::PendingInit` 期间,`UNiagaraComponent` 还没创建
- 业务此时调 `SetEffectParameterNiagara(Params)` 或 `SetFloatParameter("Speed", 1.5f)`
- **Component 不存在,直接下发会失败**

解决方案:**双存储**——
- **有 Component**:直接 passthrough 调 `NiagaraComponent->SetFloatParameter(...)`
- **无 Component**:存到 Handle 里的 `TMap<FName, float>`;Component 创建时一次性 flush

`FNiagaraComponentHandle` 就是这个**缓存 + 统一接口层**。

## 完整字段

```cpp
class FNiagaraComponentHandle
{
    // === 8 种参数 Map ===(TUniquePtr,懒分配)
    TUniquePtr<TMap<FString, float>>         NiagaraVariableFloatMap;       // NiagaraVariable 命名空间
    TUniquePtr<TMap<FString, FVector>>       NiagaraVariableVec3Map;
    TUniquePtr<TMap<FString, FLinearColor>>  VariableLinearColorMap;
    
    TUniquePtr<TMap<FName, int32>>           IntParameterMap;               // Parameter 命名空间
    TUniquePtr<TMap<FName, float>>           FloatParameterMap;
    TUniquePtr<TMap<FName, FLinearColor>>    ColorParameterMap;
    TUniquePtr<TMap<FName, FVector>>         VectorParameterMap;
    TUniquePtr<TMap<FName, TArray<FVector>>> VectorArrayParameterMap;
    
    // === 3 种 Emitter 参数(需要 Emitter 名称的,用数组不用 map)===
    TUniquePtr<TArray<FEmitterFloatParam>>         EmitterFloatParams;
    TUniquePtr<TArray<FEmitterVector4Param>>       EmitterVector4Params;
    TUniquePtr<TArray<FEmitterCustomTextureParam>> EmitterCustomTextureParams;

    // === 非参数状态(延迟同步到 Component)===
    bool bCastShadowDirty = false;
    bool bCastShadow = false;
    TWeakObjectPtr<UKuroEnviInteractionComponent> EnviInteractionCompShiftColor;
    bool bEmitterQualityLevelBiasDirty = false;
    int32 EmitterQualityLevelBias = 0;
    float GlobalAlphaBodyOpacity = 1.f;

public:
    bool bForceSolo = false;
    /* ... 见下 ... */
};
```

### 两套"参数命名空间"(**关键**)

Niagara 有 **2 套不同的参数寻址**:

1. **Variable**(`SetNiagaraVariableFloat`):按 **FString 变量名**,写到 Niagara System 的 User Namespace 变量(`User.XXX`)
2. **Parameter**(`SetFloatParameter`):按 **FName 参数名**,写到渲染参数(比如材质参数 override)

两者在 Niagara 里有独立的查找路径。所以这里才有 Variable Map × 3(Float/Vec3/LinearColor) + Parameter Map × 5(Int/Float/Color/Vector/VectorArray)。

### 3 个 Emitter 级参数用 `TArray`,其余用 `TMap`

```cpp
TUniquePtr<TArray<FEmitterFloatParam>> EmitterFloatParams;
// struct FEmitterFloatParam { FString EmitterName; FString VariableName; float FloatValue; };
```

**为什么不用 Map**?
- Emitter 参数的 key 是 **(EmitterName, VariableName)** 二元组——要用 `TPair<FString, FString>` 做 Map key 要特殊处理
- 用小 struct + TArray,遍历性能足够(一般 <10 个)
- `FEmitterCustomTextureParam` 的 value 是 `TWeakObjectPtr<UTexture>`——Texture 是 UObject,弱引用避免持有不释放

### `TUniquePtr<TMap<...>>` 而不是直接 `TMap`

**懒分配内存**。大部分特效只用 1-2 种参数类型,让其他 Map 留为 `nullptr` 避免浪费 TMap 的桶数组空间。

具体实现应该是:
```cpp
void SetFloatParameter(const FName& Name, float Value) {
    if (NiagaraComponent valid) NiagaraComponent->SetFloatParameter(Name, Value);
    else {
        if (!FloatParameterMap.IsValid()) FloatParameterMap = MakeUnique<...>();
        (*FloatParameterMap)[Name] = Value;
    }
}
```

## 核心 API

### 参数下发(11 个同名方法,passthrough 或缓存)

```cpp
void SetNiagaraVariableFloat(const FString& VariableName, float Value);
void SetNiagaraVariableVec3(const FString& VariableName, const FVector& Value);
void SetNiagaraVariableLinearColor(const FString& VariableName, const FLinearColor& Value);

void SetIntParameter(const FName& ParameterName, int32 Param);
void SetFloatParameter(const FName& ParameterName, float Param);
void SetColorParameter(const FName& ParameterName, const FLinearColor& Param);
void SetVectorParameter(const FName& ParameterName, const FVector& Param);
void SetArrayVectorParameter(const FName& ParameterName, const TArray<FVector>& Param);

void SetKuroNiagaraEmitterFloatParam(const FString& EmitterName, const FString& VariableName, float Value);
void SetKuroNiagaraEmitterVectorParam(const FString& EmitterName, const FString& VariableName, const FVector4& Value);
void SetKuroNiagaraEmitterCustomTexture(const FString& EmitterName, const FString& VariableName, UTexture* Value);
```

**命名策略**:方法名**和 UNiagaraComponent 上的同名方法匹配**。这样业务代码:
```cpp
// Old
NiagaraComponent->SetFloatParameter(Name, 1.0f);

// New (几乎无改动)
NiagaraComponentHandle->SetFloatParameter(Name, 1.0f);
```

切换接口几乎不用改业务代码。

### 状态控制
```cpp
void SetCastShadow(bool);                                            // → bCastShadow + Dirty
void SetEnviInteractionComp(UKuroEnviInteractionComponent*);         // 持弱引用
void SetEmitterQualityLevelBias(int32);                              // → EmitterQualityLevelBias + Dirty
void SetGlobalAlphaBodyOpacity(float);                               // BodyEffect 用
```

带 `Dirty` 标志——等 Component 可用时才真下发(延迟 flush)。

### 生命周期钩子(Component 生灭时机)

```cpp
void PreResetNiagaraComponent(UNiagaraComponent* NiagaraComponent);  // Reset 之前
void InitNiagaraComponent(UNiagaraComponent* NiagaraComponent);       // Reset 之后
```

这两个是**关键**!

`InitNiagaraComponent` 应该实现:
```cpp
void InitNiagaraComponent(UNiagaraComponent* NC) {
    if (NiagaraVariableFloatMap.IsValid()) {
        for (auto& Pair : *NiagaraVariableFloatMap) {
            NC->SetNiagaraVariableFloat(Pair.Key, Pair.Value);
        }
        NiagaraVariableFloatMap.Reset();   // 清缓存
    }
    // ...类似地 flush 其他 10 种 Map/Array
    // ...处理 Dirty 标志(CastShadow / EmitterQualityLevelBias / GlobalAlphaBodyOpacity)
}
```

`PreResetNiagaraComponent` **在 ResetOverrideParametersAndActivate 之前**调用,用于保留一些需要先抓取的状态(例如可能要记录 Component 当前的 ExecutionState)。

### `bForceSolo`

```cpp
bool bForceSolo = false;
```

Public 字段!业务可直接写。对应 UNiagaraComponent 的 `SetForceSolo(bool)` API——Niagara Solo 模式影响 tick 并行性(Solo 时在当前 worker 线程 tick)。

### `IsHandleValid()` 和 ComponentHandle 可以没有 Component

```cpp
bool IsHandleValid() const;
```

Handle 存在 ≠ Niagara 存在。这个接口检查**"这个 Handle 本身是不是有效的数据载体"**,不是 Component 是否存在。

## 使用流程

### 场景:Pending Init 期间设参数

```
T=0 SpawnEffect(...)
    → Handle 创建,PendingInit 态
    → NiagaraComponentHandle 作为 Handle.NiagaraParameters 被创建(TUniquePtr)
    
T=1 业务:Handle->SetEffectParameterNiagara(Params)
    → Spec::SetEffectParameterNiagara 调 NiagaraComponentHandle::SetFloatParameter
    → 缓存到 FloatParameterMap

T=2 DA 加载完成,Niagara Component 创建好
    → Spec::ResetNiagaraComponent 调用 NiagaraComponentHandle::InitNiagaraComponent(NC)
    → flush 所有 Map 到真 NC
    → 缓存释放

T=3 业务:Handle->SetEffectParameterNiagara(NewParams)
    → Spec::SetEffectParameterNiagara 调 NiagaraComponentHandle::SetFloatParameter
    → 此时 NiagaraComponentHandle 内部已经没有 NiagaraComponent 引用(Reset 时丢了)
    → 等等!
```

**观察**:看 `FEffectModelNiagaraSpec::SetEffectParameterNiagara`:
```cpp
virtual void SetEffectParameterNiagara(const FKuroEffectNiagaraParameters& Parameters) override
{
    if (!IsPlaying()) return;
    
    if (NiagaraComponentHandle.IsValid()) {
        // 走缓存路径
        for (Param : Parameters) NiagaraComponentHandle->SetFloatParameter(...);
    }
    else if (NiagaraComponent.IsValid()) {
        // 直接 passthrough 到真 Component
        for (Param : Parameters) NiagaraComponent->SetFloatParameter(...);
    }
}
```

**`NiagaraComponentHandle` 的生命周期是 Pending 期间**:
- 创建时生(`MakeShared<FNiagaraComponentHandle>()`)
- `Reset` 后即死(`NiagaraComponentHandle.Reset()`)

Pending 之后参数走真 Component。

### 场景:只在某些条件下需要 Handle
看 `FEffectModelNiagaraSpec::OnPlay`:
```cpp
if (FEffectSystem::IsGameRunning && Handle.IsValid() && !Handle.Pin()->IsPreview)
{
    IsResetImmediate = true;
    if (!NiagaraComponentHandle.IsValid()) {
        NiagaraComponentHandle = MakeShared<FNiagaraComponentHandle>();
    }
    WaitPostTick = true;
}
else
{
    ResetNiagaraComponent();   // 编辑器预览直接走,不经缓存层
}
```

—— **编辑器预览时不走 Handle 缓存**(编辑器非 hot path,直接下发);游戏运行时走 Handle 做缓冲。

## 3 种 Emitter 参数 struct

```cpp
struct FEmitterFloatParam       { FString EmitterName; FString VariableName; float FloatValue = 0.0f; };
struct FEmitterVector4Param     { FString EmitterName; FString VariableName; FVector4 Vector4Value; };
struct FEmitterCustomTextureParam { FString EmitterName; FString VariableName; TWeakObjectPtr<UTexture> TextureValue; };
```

都是按"Emitter.Variable"的二元路径写参数——定向到某个 Niagara Emitter 的某个变量。

## 与其他实体的关系

- **被 Handle 持有**:`TSharedPtr<FNiagaraComponentHandle> NiagaraComponentHandle` in [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]
- **也被 ActorHandle 持有(两个字段!)**:`TUniquePtr<FNiagaraComponentHandle>` in [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] — 回答 Batch 2 `EffectActorHandle` 里双 Niagara 字段的疑问(**都是 `FNiagaraComponentHandle`,一个单数一个复数,作用在下面开放问题里**)
- **操作 UNiagaraComponent**:InitNiagaraComponent 把缓存 flush 到真 Niagara
- **暴露到脚本**:`FEffectSystemHandleHelper::NiagaraComponentHandle_*`(Batch 5)

## Twin(Old 版对应)

**Old 没有这层门面**。Niagara 参数下发:
- 要么直接操作 `UNiagaraComponent*`(必须等 Component 存在)
- 要么等 Play 之后再下发
- Pending 期间调用会丢失

New 的这层缓冲解决了 Pending Init 期间的参数下发问题——配合整个 PendingInit 管线成立。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\NiagaraComponentHandle.h`(~69 行)
- 实现:`.cpp`(Batch 6)
- 使用:[[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectModelNiagaraSpec]]、[[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]]、[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle::GetNiagaraComponentHandle]]

## 开放问题

- **`NiagaraComponentHandle` vs `NiagaraComponentHandles`** in [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]]:类型同样是 `TUniquePtr<FNiagaraComponentHandle>`。复数字段存在的原因?是保留向后兼容/两个独立的不同用途?(Batch 6 看 .cpp)
- `InitNiagaraComponent` 的完整 flush 流程(哪些字段一起下发,顺序)
- `PreResetNiagaraComponent` 在 Reset 之前做什么准备工作
- 参数缓存是"上次 set 覆盖"还是"累积"?(TMap 语义上是覆盖,TArray 则可能累积)
- `bForceSolo` 为什么是 public 字段而非 Setter?(大概率是临时调试用,运行时很少切)
- 参数缓冲清空的时机除了 `InitNiagaraComponent` 还有?(Destroy 时?Stop 时?)
