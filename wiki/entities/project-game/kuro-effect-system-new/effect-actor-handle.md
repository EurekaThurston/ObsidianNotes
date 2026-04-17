---
type: entity
category: class-hierarchy
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, actor, command-pattern]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectActorHandle.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectActorHandle, FEffectActorAction]
---

# FEffectActorHandle + Action 家族

> New 独有的**EffectActor 门面类**——把"**特效的宿主 Actor**"(即承载 NiagaraComponent / 各类组件的 AActor)的所有操作集中,并借助**命令模式**支持"Actor 还没创建就下指令"的 Pending 流程。

## 为什么存在这个类

在 Old 系统里,EffectActor 的操作直接走 `AActor*` 调用:
```cpp
// Old 风格
AActor* actor = GetEffectActor();
actor->SetActorHiddenInGame(true);
actor->AttachToComponent(...);
```

**问题**:
1. Spawn 过程中 Actor 可能还没创建(异步 load 中),调用会 null check 失败
2. 业务想"攻击出手瞬间隐藏拖尾"这种操作时,如果 Handle 在 Pending,没法"记下命令等 Actor 起来再执行"
3. `AActor` 的 K2_* 函数在脚本/蓝图调用要跨 UObject 反射,慢

**解决方案**:
1. `FEffectActorHandle` 持有 Actor 的**弱引用 + 缓存**
2. Pending 态下 `Attach` / `Hide` 操作**转成命令对象存队列**,Actor 创建后回放
3. `K2_*` 接口同名,业务调用习惯不变

## Action 家族(命令模式)

### FEffectActorAction(虚基类)
```cpp
class FEffectActorAction
{
public:
    virtual ~FEffectActorAction() {}
    virtual void DoAction(AActor* EffectActor, const TSharedPtr<FEffectHandle>& EffectHandle) = 0;
};
```

一个"延后执行的动作"抽象。`DoAction` 被调时 Actor 一定存在。

### FEffectActor_K2_AttachToComponent(附着到组件)
```cpp
class FEffectActor_K2_AttachToComponent : public FEffectActorAction
{
    float CachedChildTransformTime = 0;
    bool HasChildTransformInit = false;
    FTransformDouble ChildTransform;

public:
    bool HasRelativeTransformInit = false;
    FTransformDouble RelativeTransform;
    
    FEffectActor_K2_AttachToComponent(USceneComponent* AttachComponent, const FName& SocketName,
                                      EAttachmentRule LocationRule, EAttachmentRule RotationRule,
                                      EAttachmentRule ScaleRule, bool WeldSimulatedBodies);

    TWeakObjectPtr<USceneComponent> Parent;
    FName SocketName;
    EAttachmentRule LocationRule = EAttachmentRule::KeepRelative;
    EAttachmentRule RotationRule = EAttachmentRule::KeepRelative;
    EAttachmentRule ScaleRule = EAttachmentRule::KeepRelative;
    bool WeldSimulatedBodies = false;

    FTransformDouble GetRelativeTransform() const;
    void SetRelativeTransform(const FTransformDouble&);
    virtual void DoAction(AActor*, const TSharedPtr<FEffectHandle>&) override;
    void UpdateChildTransform(bool ForceUpdate = false);
    FVectorDouble GetLocation(bool ForceUpdate = false);
    FRotator GetRotation();
    FVector GetScale();
    void InitRelativeTransform(bool HasRelativeTransformInit, const FTransformDouble& RelativeTransform, const FTransformDouble& Transform);
};
```

**做什么**:记录一次 AttachToComponent 调用的所有参数(Parent 组件/SocketName/规则),延迟到 `DoAction` 时调用 `EffectActor->AttachToComponent(Parent, ...)`。

**`Cached*` 字段**:缓存 child transform(位置/旋转/缩放),避免每帧重新计算——配合 `CachedChildTransformTime` 做 staleness 判断。

### FEffectActor_K2_AttachToActor(附着到另一个 Actor)
```cpp
class FEffectActor_K2_AttachToActor : public FEffectActor_K2_AttachToComponent
{
public:
    FEffectActor_K2_AttachToActor(AActor* ParentActor, USceneComponent* AttachComponent, ...);
    TWeakObjectPtr<AActor> ParentActor;
};
```

继承 AttachToComponent,多记一个 Parent Actor 引用——这样即使 AttachComponent 失效,也能靠 ParentActor 找到替代组件。

### FEffectActorBeAttachedAction(反向:让别人 attach 到自己)
```cpp
class FEffectActorBeAttachedAction : public FEffectActorAction
{
    TWeakObjectPtr<AActor> ActorInternal;
    FName SocketNameInternal;
    EAttachmentRule TransformRuleInternal = EAttachmentRule::KeepRelative;

public:
    FEffectActorBeAttachedAction(AActor* Actor, const FName& SocketName, EAttachmentRule TransformRule);
    virtual void DoAction(AActor*, const TSharedPtr<FEffectHandle>&) override;
};
```

"让别的 Actor 附到我这个特效 Actor 上"——用于 `AttachSkeletalMesh` 流程,Handle 里 `TArray<TWeakObjectPtr<AActor>> AttachToActors` 对应这种情况。

## FEffectActorHandle(主类)

```cpp
class FEffectActorHandle
{
    static int32 REFRESH_ACTOR_LOCATION_INTERVAL;
    FName Path;
    TUniquePtr<FEffectActor_K2_AttachToComponent> AttachActionInternal;  // 唯一!
    FEffectActor_K2_AttachToComponent* GetAttachAction();
    bool HiddenInGame = false;
    FVectorDouble CacheLocation = FVectorDouble::ZeroVector;
    bool LocationDirty = true;
    int32 GetLocationCountFromBudget = -1;
    void UpdateRelativeTransform();
    void UpdateTransform(bool IgnoreLocation = false, bool IgnoreRotation = false, bool IgnoreScale = false);

public:
    FEffectActorHandle() {}
    
    FTransformDouble Transform;
    TArray<FEffectActorBeAttachedAction> BeAttachedActions;   // 被附加的 action 队列
    TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandle;
    TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandles;  // 单复同名字段——Batch 4 看
    
    bool IsHandleValid() const;
    void Init(const FTransformDouble& Transform, FName Path);
    void InitEffectActor(AActor* EffectActor, const TSharedPtr<FEffectHandle>& EffectHandle);
    FNiagaraComponentHandle* GetNiagaraComponent() const;
    FNiagaraComponentHandle* GetNiagaraComponents() const;
    void SetBeAttached(AActor*, const FName& SocketName, EAttachmentRule);
    
    /* 16 个 K2 / D_K2 接口,见下 */
};
```

### 为什么 `AttachAction` 是**单个** TUniquePtr 而不是队列

看字段:`TUniquePtr<FEffectActor_K2_AttachToComponent> AttachActionInternal`——**只能持一个**。
**语义**:**最后一次调用的 Attach 会覆盖前面的**——这是合理的,"我先 Attach 到 A 再 Attach 到 B"时最终应该只附在 B。
Set → 直接替换 unique_ptr,旧的 action 丢弃。

但 `BeAttachedActions` 是 `TArray`——允许"多个 Actor 同时 attach 到我"。

### 两个 NiagaraComponentHandle 字段(单数 + 复数)?
```cpp
TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandle;
TUniquePtr<FNiagaraComponentHandle> NiagaraComponentHandles;
```
字段名 **一个单数,一个复数**,但类型相同,都是 `TUniquePtr<FNiagaraComponentHandle>`。**推测**:
- `NiagaraComponentHandle`:主 Niagara 组件
- `NiagaraComponentHandles`:聚合其他 Niagara 组件的"主句柄"(虽然用 UniquePtr 看起来也是单个,但内部可能是链/组?)

待 Batch 4 读 `NiagaraComponentHandle.h` 时解答。

### CacheLocation + GetLocationCountFromBudget(性能优化)

```cpp
FVectorDouble CacheLocation = FVectorDouble::ZeroVector;
bool LocationDirty = true;
int32 GetLocationCountFromBudget = -1;
static int32 REFRESH_ACTOR_LOCATION_INTERVAL;
```

**位置查询的 Tick-budget 节流**:
- 调 `AActor::GetActorLocation()` 实际上要走组件 transform,略贵
- 大量特效频繁查位置 → 主机 CPU 压力
- 解决:**缓存** + **计数器** + **每 N 次 budget 才真查一次**
  - `GetLocationCountFromBudget` 是"当前在 TickManager budget 里的计数",递减到 0 再真查
  - `REFRESH_ACTOR_LOCATION_INTERVAL` 是刷新间隔(static 常量)
  - `LocationDirty` 强制标记"必须真查"(如 Transform 被外部改动)

**配合** Handle 的 `LocationProxyFunction()`(Batch 1 提到过):该函数返回当前 CacheLocation,让 TickManager 用这个做距离判断 / 重要性评估——这样一次特效的位置只在"需要时"才真查。

### Init 两阶段

```cpp
void Init(const FTransformDouble& InTransform, FName InPath);             // Pending 态
void InitEffectActor(AActor* EffectActor, const TSharedPtr<FEffectHandle>&);  // Actor 来了
```

- **`Init`** 在 Pending 早期调用,只记录 Transform 和 Path
- **`InitEffectActor`** 在 Actor 真正创建后调用——**flush 所有 pending action**:
  1. 遍历 `BeAttachedActions`,每个 `DoAction(EffectActor, EffectHandle)` 执行
  2. 如果 `AttachActionInternal` 存在,`DoAction(EffectActor, EffectHandle)` 执行
  3. `EffectActor->SetActorHiddenInGame(HiddenInGame)`

这样 **业务在 Pending 态的所有 `SetHidden` / `AttachToComponent` 等调用都能"延迟生效"**——这是整个 Pending Init 机制能工作的关键基础设施。

### 16 个 K2 / D_K2 接口

```cpp
// 同名函数 Ts 调用
void SetActorHiddenInGame(bool InHiddenInGame);

void K2_AttachToActor(AActor* Parent, const FName& SocketName, EAttachmentRule LocationRule,
                      EAttachmentRule RotationRule, EAttachmentRule ScaleRule, bool bWeldSimulatedBodies);
void K2_AttachToComponent(USceneComponent*, const FName& SocketName, ...);
FVectorDouble GetActorLocation();
FVectorDouble D_K2_GetActorLocation();           // Double 精度
FRotator      K2_GetActorRotation();
FVectorDouble D_GetActorScale3D();

bool D_K2_SetActorLocation(const FVectorDouble&, bool bSweep, FHitResult&, bool bTeleport);
bool K2_SetActorRotation(const FRotator&, bool bTeleportPhysics);
void D_SetActorScale3D(const FVectorDouble&);
bool D_K2_SetActorLocationAndRotation(...);
void D_K2_AddActorWorldOffset(...);
bool D_K2_SetActorTransform(...);
void D_K2_SetActorRelativeLocation(...);
void K2_SetActorRelativeRotation(...);
void D_K2_SetActorRelativeTransform(...);
void K2_AddActorLocalTransform(...);
```

### 命名约定
- **`K2_` 前缀** = 来自 UE 的 Kismet(蓝图)API 命名约定。`AActor::K2_AttachToActor` 是 BlueprintCallable 版本。这里保留前缀让脚本习惯的调用者熟悉。
- **`D_` 前缀** = Double-precision(64-bit float)。Kuro 用 `FTransformDouble` / `FVectorDouble` 做世界坐标,因为游戏世界可能超出 UE 默认 float 精度(典型开放世界几十公里地图)。
- **`D_K2_*`** 同时有两个前缀:**K2 接口 + double 精度**。

### 注释说明
文件里有一句 "同名函数Ts调用" —— 意思是 TS 侧通过这些方法调 Actor,所以名字必须和 UE 原生 `K2_XXX` 保持一致,让脚本层的人写代码像写 UE 原生。

## 在 Pending Init 里的位置

[[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitHandle]] 持有:
```cpp
FEffectActorHandle EffectActorHandle;
```

Pending 期间,业务调 `FEffectSystemHandleHelper::ActorHandle_K2_AttachToActor(Id, ...)` 最终落到这个字段上。`SetHidden(Id, true)` 也是。Actor 创建完成时 `EffectHandle::InitEffectActorAfterPendingInit()` 调 `EffectActorHandle.InitEffectActor(Actor, Handle)` flush 掉所有累积操作。

## Handle 也持有自己的 EffectActorHandle

注意 Handle 已经持有 `TStrongObjectPtr<AActor> EffectActor`(真正的 Actor),而 `FEffectActorHandle` 是 Handle 的 `GetEffectActor() -> FEffectActorHandle*` 返回值。**Actor 对象是一个,Handle 是一层薄门面**。

```
 FEffectHandle
   ├── TStrongObjectPtr<AActor> EffectActor       // Actor 对象本身
   ├── FEffectActorHandle? (Get via GetEffectActor())   // 薄门面,含 action queue + cache
```

实际 Handle 里没直接 `FEffectActorHandle` 字段——推测门面类在 [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitHandle]]::EffectActorHandle 或者系统层别处分配。具体 .cpp 时确认(Batch 6)。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitHandle]] **值持有**(每次 Spawn 一个)
- **在** [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] **通过 `GetEffectActor() -> FEffectActorHandle*`** 暴露
- **被** `FEffectSystemHandleHelper` (Batch 5) **作为脚本 API 的后台**调用
- **封装** `AActor` + `USceneComponent` + `UNiagaraComponent` 相关操作,避免业务直接碰 UObject
- **持有** `FNiagaraComponentHandle`(Batch 4)
- **使用** `FTransformDouble` / `FVectorDouble` 扩展类型(Kuro 自定义)

## Twin(Old 版对应)

**Old 里没有这层门面**。Actor 操作直接调 `AActor::K2_AttachToActor`,位置查 `GetActorLocation()`。没有命令队列、没有缓存、没有双精度封装。

Old 遗迹痕迹:
- `FEffectHandle::GetEffectActor() -> TWeakObjectPtr<AActor>` 直接返回原生弱指针
- 业务需要自己做 null check 和 Pending 态判断

New 的这层额外抽象的成本(命令对象/缓存/门面)换来:
- Pending Init 可用
- 位置查询节流
- 脚本 API 不碰 Actor 裸指针(见 `FEffectSystemHandleHelper`,Batch 5)
- 坐标双精度支持

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectActorHandle.h`(~160 行)
- 配套:Private/EffectActorHandle.cpp(Batch 6)、`FEffectSystemHandleHelper.h`(Batch 5)
- 关联:[[entities/project-game/kuro-effect-system-new/effect-init-pipeline|Init Pipeline]]

## 开放问题

- `NiagaraComponentHandle` vs `NiagaraComponentHandles`:字段同类型,字段名一个单一个复,可能是拼写错误或者两者用途不同。Batch 4 读 `NiagaraComponentHandle.h` 确认
- `REFRESH_ACTOR_LOCATION_INTERVAL` 的默认值(.cpp)
- `GetLocationCountFromBudget` 的更新时机(每帧 tick manager 减 1?)
- `UpdateRelativeTransform` / `UpdateTransform(IgnoreLocation, IgnoreRotation, IgnoreScale)` 的调用者:是 Attach 完成后的回调,还是外部定时调?
- `FEffectActorHandle` 实例的生命周期:在 InitHandle 里一直持有,直到 Handle 初始化完成?还是持到 Handle 销毁?
