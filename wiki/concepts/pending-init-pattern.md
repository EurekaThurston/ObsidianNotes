---
type: concept
created: 2026-04-17
updated: 2026-04-17
tags: [concept, architecture, async, pattern, command-pattern]
sources: 1
aliases: [Pending State Pattern, Async Init Pattern, Command Queue for Pending Objects]
---

# Pending Init 模式

> 当一个对象的真正可用状态需要**异步初始化**(资源加载、网络握手、子系统启动),**立即返回一个 Pending 状态的对象**,让业务可以马上操作,操作被缓存成命令队列,对象 Ready 时 flush 执行。

## 为什么需要

### 反面教材:同步等待

传统写法:
```cpp
// 业务代码
FEffectHandle* h = SpawnEffect(path);   // 内部同步 LoadObject,可能卡 1-2 帧
h->SetHidden(true);                      // 现在 h 肯定 valid,可以调
```

**问题**:
- 冷加载大资源时主线程阻塞
- 业务代码不能做"spawn 后立即隐藏"的意图表达(因为 spawn 本身就卡)
- UE 异步加载框架没被利用

### 同步替代:拒绝在未完成时操作

也有人这么做:
```cpp
int32 handleId = SpawnEffect(path);   // 立即返回,但资源异步加载
// ...帧过
if (IsReady(handleId)) {
    SetHidden(handleId, true);         // 只有 ready 才能调
}
```

**问题**:
- 业务代码变成一堆 `IsReady` 检查 + 状态机
- 无法表达"创建后立即隐藏"的意图(必须等 Ready,状态可能已变)

### Pending Init 的解法

```cpp
// 业务代码,几乎和同步版一样
int32 handleId = SpawnEffect(path);    // 立即返回(Pending 态)
SetHidden(handleId, true);             // 不管是不是 Pending,直接调——系统自动处理
AttachToActor(handleId, myActor);      // 同上,累积到 pending 队列
// ... 几帧后资源加载完成,pending 操作一次性 flush
```

**关键**:业务代码**写法和同步版基本一致**——差别只在语义:Pending 态下的操作**保证最终会生效**,但不是立即生效。

## 核心组件

### 1. 状态标识

对象必须能**自报告** Pending 状态:
```cpp
class FEffectHandle {
    bool IsPendingInit() const;   // 对外 API
    bool IsInitializing = false;   // 内部 flag
};
```

### 2. 命令队列(Command Queue)

Pending 期间收到的**有状态改变效果的调用**被转成命令对象:

```cpp
// 命令基类
class FEffectActorAction {
public:
    virtual void DoAction(AActor* ActualActor, const TSharedPtr<FEffectHandle>& Handle) = 0;
};

// 具体命令
class FEffectActor_K2_AttachToComponent : public FEffectActorAction {
    TWeakObjectPtr<USceneComponent> Parent;
    FName SocketName;
    EAttachmentRule Rules;
    // ...所有 Attach 需要的参数
    virtual void DoAction(AActor* Actor, const TSharedPtr<FEffectHandle>& Handle) override {
        Actor->K2_AttachToComponent(Parent.Get(), SocketName, ..., ..., ...);
    }
};
```

参考 [[entities/project-game/kuro-effect-system-new/effect-actor-handle|Kuro 的 FEffectActorAction 家族]]。

### 3. 命令缓存容器

Pending 对象持有一个**累积的命令集合**:
```cpp
class FEffectActorHandle {
    TUniquePtr<FEffectActor_K2_AttachToComponent> AttachActionInternal;  // 单个 action(最后一次覆盖)
    TArray<FEffectActorBeAttachedAction> BeAttachedActions;              // 多 action 累积
    bool HiddenInGame = false;                                            // 或者直接存 flag
};
```

**选 single vs array 取决于语义**:
- **幂等/覆盖式**(Attach、Hide)—— 只记最后一次 → `TUniquePtr<Action>`
- **累加式**(Add Observer)—— 全记 → `TArray<Action>`
- **简单 flag**(Hidden)—— 直接存 bool

### 4. Flush 时机

对象变 Ready 时调用 flush:
```cpp
void FEffectActorHandle::InitEffectActor(AActor* EffectActor, TSharedPtr<FEffectHandle> Handle)
{
    // 按次序执行 pending 命令
    for (auto& Action : BeAttachedActions) {
        Action.DoAction(EffectActor, Handle);
    }
    BeAttachedActions.Empty();
    
    if (AttachActionInternal) {
        AttachActionInternal->DoAction(EffectActor, Handle);
        AttachActionInternal.Reset();
    }
    
    EffectActor->SetActorHiddenInGame(HiddenInGame);
}
```

## 公共 API 设计原则

Pending 模式要求**公共 API 对 Pending 态和 Ready 态都工作**:

```cpp
void FEffectHandle::SetHidden(bool Hidden, FName Reason) {
    if (IsPendingInit()) {
        InitCache->EffectActorHandle.HiddenInGame = Hidden;   // Pending:写缓存
        return;
    }
    EffectActor->SetActorHiddenInGame(Hidden);                 // Ready:直接调
}
```

**业务代码无感**——调用方式一样,幕后走不同路径。

## 边界情况

### Pending 期间多次调用
```cpp
int32 h = SpawnEffect(path);
SetHidden(h, true);    // 缓存:Hidden = true
SetHidden(h, false);   // 缓存覆盖:Hidden = false
AttachToActor(h, a1);   // 缓存:Attach 到 a1
AttachToActor(h, a2);   // 缓存覆盖:Attach 到 a2
// ...flush 时:最终 Hidden=false + Attach 到 a2
```

**覆盖式命令**(单 `TUniquePtr`):自动处理多次调用,只保留最后一次。
**累加式**(`TArray`):可能需要去重或合并逻辑。

### Pending 期间对象被取消
```cpp
int32 h = SpawnEffect(path);
SetHidden(h, true);
StopEffect(h);   // 业务决定不要了
```

Pending 态下的 Stop 需要:
- 标记"**取消加载**"(异步加载如果还在队列,要能中断)
- Clear pending 命令队列
- 触发 `FEffectInitCallback` 回调 `ELoadEffectResult_Cancel`

### Pending 永远不完成
资源加载失败、文件不存在、网络超时。
- 定时器兜底(见 `FEffectLifeTime::WaitMiniTimeScaleTimerHandle` 10 秒保底)
- 回调 `ELoadEffectResult_Fail`
- 释放 pending 资源(Handle / InitCache / Pending 命令队列都要清)

### 多层 Pending 嵌套
对象 A Pending 期间创建对象 B(B 也是 Pending)。B ready 后又创建 C ...
- Kuro 用 `WaitInitChildCount` 计数(见 [[entities/project-game/kuro-effect-system-new/effect-spec-subclasses|Group Spec::OnInit]])
- 所有 child 都 ready 才触发父的 InitFinished

## 与其他相关模式

### Command Pattern(GoF)
Pending Init 是 Command 模式的特定应用——**命令对象具备延后执行的能力**。

### Promise/Future
Pending Handle 概念上类似 Promise:
- Handle Id = Promise
- `FEffectInitCallback` = `.then()` / `.catch()`
- Resolve = Load 完成 → Init → Callback
- Reject = ELoadEffectResult_Fail

**差别**:Promise 通常单次 resolve 后结束;Pending Handle resolve 后**变成持久对象**继续被业务持有。

### Optimistic UI(前端开发)
前端的"立即响应,后台同步"也是这个思路——UI 假装操作立即完成,后台失败才回滚。Pending Init 是其后端版。

## 何时不用这个模式

- **同步加载成本低**(配置小、hot cached)——简单同步 API 就够
- **业务逻辑无法接受"最终一致"**(实时 PvP 的关键操作)
- **代码复杂度限制**——团队能力/项目规模不支持维护命令队列基础设施

## 使用时的坑

1. **Pending 期间的 Get 返回什么**?`GetEffectActor()` — 返 null 还是"将来会有的 Actor 代理对象"?Kuro 选代理门面 [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]]。
2. **命令执行失败**:flush 时某条命令失败(比如 AttachToActor 的 Target 已死)怎么办?通常 log + 跳过 + 不中断后续。
3. **时间敏感操作**:Pending 期间设了 `TimeScale = 0.3`,过了 1 秒 ready,这 1 秒的 PassTime 该怎么算?(Kuro 用 `FEffectInitHandle::StartTime + TimeDiff` 追溯)
4. **业务的并发 Spawn**:业务一秒内 SpawnEffect 100 次同路径,LRU 和 Pending 的交互?

## 引用来源

- **主要案例**:[[entities/project-game/kuro-effect-system-new/effect-init-pipeline|Kuro New 的 FEffectInitModel + FEffectInitHandle]]
- **命令对象实现**:[[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] 的 Action 家族
- **Handle 状态机**:[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 的 IsPendingInit / PendingInitTimeScale / InitCache
- **缓冲层**:[[entities/project-game/kuro-effect-system-new/niagara-component-handle|FNiagaraComponentHandle]](Niagara 参数 Pending 缓存)
- **反面**:[[entities/project-game/kuro-effect-system/effect-handle|Kuro Old]]——无 Pending 概念,同步 LoadObject
- **综合对比**:[[syntheses/kuro-effect-system-old-vs-new|Old vs New]] 的"改变 4"节

## 开放问题

- Pending 期间的 Handle 能作为**另一个 Pending Handle 的 parent**吗?(嵌套 Pending)
- 如果游戏引擎没有 async load 原生支持,如何在没 future/promise 的环境实现?
- 这个模式对**网络游戏的延迟掩盖**(predict + reconcile)的契合度?
