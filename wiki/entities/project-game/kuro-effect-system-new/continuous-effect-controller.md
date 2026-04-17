---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, continuous]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/ContinuousEffectController.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FContinuousEffectController, AnsSlot]
---

# FContinuousEffectController(连续特效控制器)

> New 独有。处理**"同一个骨骼槽、同一个动画 slot 触发的特效**"在**新触发时的平滑过渡**。比如连续轻击时,每一击的刀光特效会互相覆盖,新特效播,旧特效被"柔和"关掉——这个类负责其中的 Stop 判断和帧间延迟。

## 完整定义(25 行)

```cpp
namespace KuroEffect
{
    class FContinuousEffectController
    {
        static int32 MaxWaitContinuousEffectFrame;
        static FName WaitTimeOverStopEffectReason;
        static FName ForceStopEffectWhenSpawnReason;
        
        // 关键数据:Skeletal → Slot → HandleId 三级映射
        TMap<TWeakObjectPtr<USkeletalMeshComponent>, TMap<FName, int32>> AnsSlotEffects;
        
        TMap<int32, int32> PendingRemoveEffectMap;
        TArray<int32> PendingRemoveEffects;

    public:
        void OnBeforeSpawnEffect(const TUniquePtr<FEffectContext>& Context);
        void OnAfterSpawnEffect(const TSharedPtr<FEffectHandle>& Handle);
        bool OnStopEffect(const TSharedPtr<FEffectHandle>& Handle);
        void OnPostTick(float DeltaTime);
        void Clear();
    };
}
```

很小的类,但职责清晰。

## 关键理解:AnsSlot 是什么

背景知识:**UE AnimNotifyState**(AnS)

UE 的 `AnimNotifyState` 是"**动画区间标记**":
- 在 AnimSequence 时间轴上标一段区间
- 动画播到区间开始 → 触发 `NotifyBegin`
- 期间 → 每帧 `NotifyTick`
- 区间结束 → `NotifyEnd`

Kuro 游戏中,**每个 AnS 关联一个 SlotName + 一条特效 Path**:
- 动画进入 AnS 区间 → 根据 Path Spawn 特效,记录到 SlotName 下
- 动画出 AnS 区间 → 找到该 SlotName 的特效并 Stop

**问题**:玩家**连续轻击 x5**,每一击的动画里都有"刀光"AnS,触发同一个 SlotName:

```
T=0.0 Attack1 开始 → Spawn 刀光 A (slot="Slash")
T=0.3 Attack1 结束 Attack2 开始
      - Attack1.AnimEnd 停 A
      - Attack2.Spawn 刀光 B (slot="Slash")
T=0.4 Attack2 的 A 还在 Stopping 阶段 (尾部淡出),B 已经播
T=0.6 Attack3 开始 → Spawn C...
```

**期望行为**:前一个刀光不要立刻硬切走,让它自然消散,但不要挡住新刀光。

`FContinuousEffectController` 就是**维护"当前 SlotName 下的活跃特效 HandleId"**——新特效 Spawn 时,先通知旧特效"该走了"。

## `AnsSlotEffects` 数据结构详解

```cpp
TMap<TWeakObjectPtr<USkeletalMeshComponent>, TMap<FName, int32>> AnsSlotEffects;
```

**三级映射**:
- **Key1**:`TWeakObjectPtr<USkeletalMeshComponent>` — 骨骼网格组件(每个角色自己一个)
- **Key2**:`FName` — AnS slot 名(如 `"Slash"`、`"WeaponTrail"`)
- **Value**:`int32` — 当前在该 slot 活跃的 Effect HandleId

**为什么 Key 要骨骼 Component 而非 Actor**?
- 一个 Actor 可能有多个骨骼组件(主体 + 武器分离)
- 主体的"Slash"和武器的"Slash"是不同的 slot,应当独立管理
- 用 SkeletalMeshComponent 做 Key 更精确

## 核心方法语义(基于签名推断)

### OnBeforeSpawnEffect(before-hook)
```cpp
void OnBeforeSpawnEffect(const TUniquePtr<FEffectContext>& Context);
```

Spawn 新特效**之前**调。从 Context 查:
- `Context->AnsSlotName` — AnS slot
- `FSkeletalMeshEffectContext->SkeletalMeshComponent` — 骨骼组件

如果当前 slot 下已有活跃特效,**发 Stop 命令**(带 `ForceStopEffectWhenSpawnReason` 原因码)给旧 Handle。

### OnAfterSpawnEffect(after-hook)
```cpp
void OnAfterSpawnEffect(const TSharedPtr<FEffectHandle>& Handle);
```

Spawn 完成后调。把新 Handle 登记到 `AnsSlotEffects[SkeletalComp][SlotName] = HandleId`。

### OnStopEffect(hook + veto 权)
```cpp
bool OnStopEffect(const TSharedPtr<FEffectHandle>& Handle);
```

Effect 被 Stop 时调。**返回值 bool 可能是"是否真正停止"**——即 Controller 有机会**延迟或阻止**立即 Stop,返回 false 则"再等一会"。

### OnPostTick(帧末处理 pending)
```cpp
void OnPostTick(float DeltaTime);
```

`PendingRemoveEffectMap` / `PendingRemoveEffects` —— **延迟删除列表**。每帧 PostTick 处理:

- 对每个 pending 的 effect,等帧数是否超过 `MaxWaitContinuousEffectFrame`
- 超过 → 真正 Stop(发 `WaitTimeOverStopEffectReason`)
- 未超过 → 继续等

这实现**"等 N 帧后才停"**的容错——避免立即 Stop 打断视觉效果。`PendingRemoveEffectMap<int32, int32>` 的 Value 可能是 framecount。

### Clear
一次性清所有数据。场景切换 / Actor 被销毁 / 玩家退出时使用。

## Static 常量

```cpp
static int32 MaxWaitContinuousEffectFrame;       // 最长等待帧数
static FName WaitTimeOverStopEffectReason;        // 超时强制停原因码
static FName ForceStopEffectWhenSpawnReason;     // 新特效 Spawn 时强制停旧的原因码
```

这些 FName 作为 Stop 的 Reason 码传给 Handle,便于 debug 和 log:
```
[2026-04-17 12:34:56] Stop Handle=42 Reason=ForceStopEffectWhenSpawn
[2026-04-17 12:34:58] Stop Handle=42 Reason=WaitTimeOverStop
```

## Controller 在 Spawn 流程里的位置

```
业务调 FEffectSystem::SpawnEffect(...)
          │
          ▼
    判断 Context 是否关联 AnsSlot
          │
          ▼
    ContinuousEffectController.OnBeforeSpawnEffect(Context)
        → 如果 slot 下有活跃 effect,给它发 Stop
          │
          ▼
    创建新 Handle,异步加载 DA
          │
          ▼
    ContinuousEffectController.OnAfterSpawnEffect(NewHandle)
        → 登记 AnsSlotEffects[...][SlotName] = NewHandle.Id
          │
          ▼
    返回 HandleId 给业务
```

`Stop` 分支:
```
业务调 FEffectSystem::StopEffectById(Id, Reason, Immediately)
          │
          ▼
    ContinuousEffectController.OnStopEffect(Handle)
        → 返回 true: 真正 Stop
        → 返回 false: 加入 PendingRemove,等若干帧
          │
          ▼
    PostTick 处理 Pending 列表
```

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] **持有**:`static FContinuousEffectController ContinuousEffectController;`
- **依赖**:
  - [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]
  - [[entities/project-game/kuro-effect-system-new/effect-context|FEffectContext]] 的 `AnsSlotName` 字段
  - [[entities/project-game/kuro-effect-system-new/effect-context|FSkeletalMeshEffectContext]] 的 `SkeletalMeshComponent`
  - `USkeletalMeshComponent`(UE)
- **与 Spawn/Stop 集成**:Spawn 和 Stop 入口必须调对应钩子

## Twin(Old 版对应)

**Old 没有对应机制**。连续技特效重叠是 Old 的常见视觉 bug——新刀光出现时旧刀光直接消失,没有过渡。New 的这个 Controller 就是专门解决这个问题。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\ContinuousEffectController.h`(~25 行)
- 实现:`.cpp`(Batch 6)
- 使用:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 SpawnEffect / StopEffect 路径
- 相关枚举:[[entities/project-game/kuro-effect-system/effect-define|EEffectCreateFromType_Ans]]

## 开放问题

- `MaxWaitContinuousEffectFrame` 的默认值(数字)
- `OnStopEffect` 返回 true/false 的具体策略:什么情况延迟、什么情况立即?
- `PendingRemoveEffectMap` 的 Value 确认(帧 counter?)
- 特效**完成自己生命周期**(自然结束)vs **被 Stop**(外部调用)是否走不同路径?
- Controller 自己的 `OnPostTick` 和 Spec 的 `OnPostTick` 哪个先执行?
