---
type: entity
category: uobject-class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, actor, uobject]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSystemActor.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [AEffectSystemActor]
---

# AEffectSystemActor(特效系统 Actor)

> New 独有。**承载特效的专用 UE Actor 类**——把"把特效塞到场景里"所需的 Actor 层能力集中到一个 `AActor` 派生类。Old 用通用 `AActor`,New 有这个专用类。

## 为什么要专门搞个 Actor 类

在 UE 里,Niagara/静态网格/光源等都得挂在一个 `AActor` 下(具体是挂在 Actor 的 SceneComponent 上)。Old 里这个 Actor 要么复用业务的 Actor,要么是个无名 `AActor` 实例——**没有特效系统自己的扩展机会**。

New 引入 `AEffectSystemActor`:
- 自带 **HandleId / Path / EffectType / TimeScale / OwnerEntityId** 字段,Actor 自己就能回答"我是谁的特效"
- 重写 `SetActorHiddenInGame` 让 hidden 逻辑经过特效系统的管线
- 重写 `EndPlay` 自动通知系统 "Actor 不见了"
- 给蓝图暴露一组 `UFUNCTION(BlueprintCallable)`,让蓝图直接操作特效

## 完整定义

```cpp
UCLASS()
class KUROGAMEPLAY_API AEffectSystemActor : public AActor
{
    GENERATED_BODY()
    
    int32 HandleId = 0;
    FName EffectPath;
    EEffectType EffectType = EEffectType_Scene;
    float TimeScale = 1.0f;
    bool HasRegisterOnEndPlay = false;
    int32 OwnerEntityId = 0;

public:
    AEffectSystemActor(const FObjectInitializer& ObjectInitializer);
    
    EEffectActorPoolEnum InPool = EEffectActorPoolEnum_None;
    
    // 生命周期 API
    void SetEffectHandle(int32 InHandleId = 0, const FName& Path = NAME_None, EEffectType Type = EEffectType_Scene);
    void SetTimeScale(float InTimeScale);
    
    // 蓝图可调
    UFUNCTION(BlueprintCallable)
    void StopEffect(const FName& Reason, bool Immediately = false, bool DestroyActor = false) const;
    
    UFUNCTION(BlueprintCallable)
    float GetTimeScale() const;
    
    UFUNCTION(BlueprintCallable)
    int32 GetHandle() const;
    
    FName GetEffectPath() const;
    
    UFUNCTION(BlueprintCallable)
    int32 GetEffectType() const;
    
    UFUNCTION(BlueprintCallable)
    void SetOwnerEntityId(int32 EntityId);
    
    UFUNCTION(BlueprintCallable)
    int32 GetOwnerEntityId() const;
    
    // UE 覆写
    virtual void SetActorHiddenInGame(bool bNewHidden) override;
    virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
};
```

## 关键字段详解

### `int32 HandleId`
绑定的 [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 的 Id。Actor 可以通过 `FEffectSystem::StopEffectById(HandleId, ...)` 反查对应 Handle。

### `FName EffectPath`
特效 DA 路径。用于 debug 显示、LRU 池 key。

### `EEffectType EffectType`
特效类型(Fight/UI/Scene/UiScene3D)。[[entities/project-game/kuro-effect-system/effect-define|见 EffectDefine]]。

### `float TimeScale`
**Actor 级别的 TimeScale**——Actor 作为 UE Actor 有 `CustomTimeDilation` 字段,这个 TimeScale 同步过去。在 [[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec::SetTimeScale]] 里可以看到:

```cpp
// 摘自 FEffectSpec<T>::SetTimeScale
AActor* EffectActor = SharedHandle->GetSureEffectActor();
if (EffectActor) {
    EffectActor->CustomTimeDilation = Scale;
    AEffectSystemActor* EffectSystemActor = Cast<AEffectSystemActor>(EffectActor);
    if (EffectSystemActor) {
        EffectSystemActor->SetTimeScale(Scale);   // ← 调用本类
    }
}
```

**两层 TimeDilation 下发**:UE 原生 `CustomTimeDilation`(影响 UE 的 tick 时间缩放)+ 本类的 `TimeScale`(特效逻辑自己用)。

### `bool HasRegisterOnEndPlay`
Actor 是否已注册 EndPlay 回调(防止重复注册)。

### `int32 OwnerEntityId`
Actor 属于哪个业务 Entity(角色/怪物/玩家 Id)。

### `EEffectActorPoolEnum InPool`
当前 Actor 在哪个池(见 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine]]):
- `None` — 不在池里,活跃使用中
- `ActorSystem` — 在 Kuro ActorSystem 全局池
- `LRU` — 在 EffectSystem 自己的 LRU 池

## API 语义

### SetEffectHandle
```cpp
void SetEffectHandle(int32 HandleId, const FName& Path, EEffectType Type);
```
**Actor 被创建分配给某 Handle 时调用**,写入身份信息。

### StopEffect(蓝图可调!)
```cpp
UFUNCTION(BlueprintCallable)
void StopEffect(const FName& Reason, bool Immediately = false, bool DestroyActor = false) const;
```

**给蓝图用的便捷 API**——蓝图里拿到这个 Actor 就能直接停对应特效,不需要通过 `FEffectSystem::StopEffectById`。内部应该是:
```cpp
void AEffectSystemActor::StopEffect(...) const {
    FEffectSystem::StopEffectById(HandleId, Reason, Immediately, DestroyActor);
}
```

### 重写的两个 UE 方法

#### SetActorHiddenInGame
```cpp
virtual void SetActorHiddenInGame(bool bNewHidden) override;
```
UE 的 Actor 自带 `SetActorHiddenInGame` 只改可见性。**Kuro 在这里接管**——可能:
- 通知 `FEffectSpec::OnHiddenInGameChange` 扩散到 Spec 逻辑
- 处理 BodyEffect 的 Hidden 分离 (`GKuroCppEffectSpecSeparateBodyEffectHidden`)
- 调用 ContinuousEffectController 处理 pending stops

具体在 `.cpp`(Batch 6)。

#### EndPlay
```cpp
virtual void EndPlay(const EEndPlayReason::Type EndPlayReason) override;
```
Actor 被销毁/World 结束时 UE 调 EndPlay。本类重写来**通知 EffectSystem 清理对应 Handle**——调用 `FEffectSystem::OnEffectActorEndPlay(Id)`(Batch 1 见过的入口)。

`HasRegisterOnEndPlay` 大概率是为了配合业务 Actor 也绑了自己的 EndPlay 回调时避免重复触发。

## 蓝图集成

`UFUNCTION(BlueprintCallable)` 的 6 个方法:
```cpp
StopEffect / GetTimeScale / GetHandle / GetEffectType / SetOwnerEntityId / GetOwnerEntityId
```

—— **蓝图能直接操作特效 Actor**。典型使用场景:
- 业务蓝图拿到一个 EffectActor(比如 SpawnEffect 返回的 Actor)
- 挂个 `StopEffect` 节点,提供 Reason 参数
- 不需要写任何 C++

## 与其他实体的关系

- **承载者**:`TStrongObjectPtr<AActor> EffectActor` in [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]——Handle 强持有 Actor,Actor 通过 HandleId 反查 Handle
- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::CreateEffectActor]] **创建**
- **被** [[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec::SetTimeScale]] **下发 TimeScale**
- **宿主** Niagara/StaticMesh/Light 等组件(Spec 创建时添加到 Actor)

## Twin(Old 版对应)

Old **没有专用 Actor 类**。EffectActor 要么是通用 `AActor`,要么是业务自定义的 `BP_EffectActor`(蓝图类,`FEffectContext::CreateFromBpEffectActor` 字段纪录)。New 专用类化之后:
- 统一行为(Hidden/EndPlay/TimeScale 都一致)
- 蓝图 API 统一(不需要每个 BP_EffectActor 自己实现 StopEffect 节点)
- 但仍兼容旧的 BP_EffectActor(Context 里有 flag 区分)

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectSystemActor.h`(~39 行)
- 实现:`.cpp`(Batch 6)
- 使用者:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 CreateEffectActor、[[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec]] 的 SetTimeScale

## 开放问题

- `FEffectSystem::CreateEffectActor` 何时创建 `AEffectSystemActor` vs 其他 AActor?(`bool IsPreview` 参数)
- `SetActorHiddenInGame` 的具体扩展行为(.cpp 答)
- 蓝图里 BP_EffectActor 如何继承或替代这个类?
- `EEffectActorPoolEnum InPool` 写入时机—— Actor 回池/出池时谁负责更新?
- 构造函数里的 `FObjectInitializer` 用途(默认就够吗?需要 SetRootComponent?)
