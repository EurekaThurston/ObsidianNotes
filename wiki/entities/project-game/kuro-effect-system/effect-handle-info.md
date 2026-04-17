---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, js-bridge]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectHandleInfo.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
aliases: [FEffectHandleInfo]
---

# FEffectHandleInfo(老版 Handle 的 JS 伴生体)

> Old [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 的贴身伴生对象,专门管**跨 C++/JS 边界的状态**:EffectActor 引用 + 5 个 JS 回调函数。Handle 里任何涉及 v8 的操作都是转发到这个对象。

## 为什么要有这个类

在 Old 系统里,特效的行为一半在 C++(Spec),一半在 TS(生命周期回调)。`FEffectHandle` 不想让 v8 的复杂类型(`v8::Global<T>`)污染自己的字段——毕竟这些只是"与 TS 通信用的粘合层"。

所以拆成:
- **`FEffectHandle`** 本体:持 Spec + 状态机(见 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle (Old)]])
- **`FEffectHandleInfo`** 伴生:持 EffectActor + 5 个 v8::Global<v8::Function>

职责分离是好的。**但这层分离是不彻底的**——Handle 的 API 签名里还是带 `v8::Isolate*`,因为它仍然要转发给 Info。New 彻底删掉了这个问题。

## 完整字段

```cpp
class FEffectHandleInfo
{
    TWeakObjectPtr<AActor>       EffectActor;                    // 特效的 Actor 宿主(弱引用)
    v8::Global<v8::Object>       GlobalJsHandle;                 // TS 侧的句柄镜像对象
    v8::Global<v8::Function>     GlobalGroupDelayPlayFunction;
    v8::Global<v8::Function>     GlobalPreStopFunction;
    v8::Global<v8::Function>     GlobalUpdateLifeCycleFunction;
    v8::Global<v8::Function>     GlobalEnterStoppingFunction;
    v8::Global<v8::Function>     GlobalMultiEffectAdjustNumberFunction;

public:
    bool IsFreeze = false;                                        // 冻结标志(和 Handle 共享)
    
    FEffectHandleInfo(AActor* Actor);
    virtual ~FEffectHandleInfo();
    
    /* 各种 Register / Clear / 调用方法,见下 */
};
```

## v8 简介(背景知识)

如果你不熟 v8 类型:
- `v8::Local<T>` —— **栈句柄**,`HandleScope` 内有效,离开 scope 失效。跨 C++ 函数边界不能长期存。
- `v8::Global<T>` —— **堆句柄**,参与 v8 GC。只要还持有,JS 侧对象就不会被 GC。跨线程、长期持有用这个。
- `Reset(Isolate, Local)` —— 把 Local 存成 Global,持有所有权。
- `v8::Isolate*` —— v8 引擎实例。Puerts 里基本只有一个主 Isolate。所有跨 JS 调用都要它。

Kuro 用 Puerts 做 TS ↔ C++ 桥,所有"长期持有的 JS 对象"必须转成 `v8::Global`。**这就是为什么 Info 里全是 `v8::Global`**。

## 5 个回调函数含义

它们对应 TS 侧特效行为中可以被 C++ 端调用的钩子:

| 函数槽位 | 调用时机 | 典型用途 |
|---|---|---|
| `GlobalJsHandle` | 不是函数,是 TS 侧的"句柄镜像对象"。TS 业务代码手里拿的就是它 | 让 TS 能反查某个 C++ Handle 对应的 TS 对象 |
| `GlobalPreStopFunction` | `PreStop(Isolate)` 调用,在 Stop 之前 | 播停止音效、刷新 UI |
| `GlobalUpdateLifeCycleFunction` | 每帧 Tick 里调,携带当前 DeltaTime | TS 侧跟随特效进度刷表现(如 HUD 跟随特效缩放) |
| `GlobalEnterStoppingFunction` | 从 Play 状态进入 Stopping 时 | 过渡动画、提前清理资源引用 |
| `GlobalGroupDelayPlayFunction` | Group 类型特效中的"延迟播放子特效" | 美术想让 Group 里某些子特效延迟 T 秒播,能通过 TS 控制 |
| `GlobalMultiEffectAdjustNumberFunction` | `MultiEffect` 类型上调节子特效数量时 | 玩家 Buff 球数量变化时 TS 通知 |

## 注册接口(Handle 转发的目标)

```cpp
FORCEINLINE void RegisterJsObject(v8::Isolate*, v8::Local<v8::Object>);
FORCEINLINE void RegisterGroupDelayPlayFunction(v8::Isolate*, v8::Local<v8::Function>);
FORCEINLINE void ClearGroupDelayPlayFunction();
FORCEINLINE void RegisterCommonFunction(v8::Isolate*, v8::Local<v8::Function> PreStop,
                                                       v8::Local<v8::Function> UpdateLifeCycle,
                                                       v8::Local<v8::Function> EnterStopping);
FORCEINLINE void RegisterMultiEffectAdjustNumFunction(v8::Isolate*, v8::Local<v8::Function>);
```

**Null-safe 实现**(举一例):
```cpp
FORCEINLINE void RegisterGroupDelayPlayFunction(v8::Isolate* Isolate, v8::Local<v8::Function> GroupDelayPlayFunction)
{
    if (!GroupDelayPlayFunction->IsNullOrUndefined())
    {
        GlobalGroupDelayPlayFunction.Reset(Isolate, GroupDelayPlayFunction);
    }
}
```
—— **允许 TS 侧传 undefined**,仅跳过存储,不报错。

## 调用接口(从 C++ 反向调 JS)

```cpp
void GroupDelayPlay(v8::Isolate*, float DeltaTime) const;
void PreStop(v8::Isolate*) const;
void UpdateLifeCycle(v8::Isolate*, float DeltaTime) const;
void EnterStopping(v8::Isolate*) const;
void MultiEffectAdjustNumber(v8::Isolate*, int DesiredNum) const;

// 内部辅助(FORCEINLINE 但有声明,实现在 .cpp)
FORCEINLINE void DoTickCallback(v8::Isolate*, const v8::Global<v8::Function>& Function) const;
FORCEINLINE void DoTickDeltaCallback(v8::Isolate*, float DeltaTime, const v8::Global<v8::Function>& Function) const;
```

具体 `.cpp` 里应该是走 `v8::Local::New(Isolate, Global)` → 获得 Local → 拼 argv → `Function->Call(Isolate, Receiver, argc, argv)`。这是 Puerts 调 JS 的标准姿势。

## `TWeakObjectPtr<AActor>` 小科普

- `TWeakObjectPtr` 是 UE 为 UObject 专门设计的弱引用。
- 不影响 GC,UObject 被回收时自动变 invalid。
- 访问前必须 `IsValid()` 检查(下面 `GetEffectActor()` 就是这个套路)。
- 对比 C++ 裸指针:裸 `UObject*` 不会被 GC 识别,成野指针 = UAF crash。

```cpp
FORCEINLINE AActor* GetEffectActor() const
{
    if (!EffectActor.IsValid()) { return nullptr; }
    AActor* Actor = EffectActor.Get();
    if (!IsValid(Actor))        { return nullptr; }
    return Actor;
}
```
—— 双重检查(`IsValid()` 和 `IsValid(Actor)`)——前者是 WeakObjectPtr API,后者是 UE 的全局 `IsValid()`(还会检查 `HasAnyFlags(RF_PendingKill)`)。**这种 paranoid 检查说明 Old 在某些时序下能拿到"将要死"的 Actor**,这也是 New 改成 `TStrongObjectPtr<AActor>` 的原因——避免这种歧义。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] **持有**:`FEffectHandleInfo* EffectHandleInfo`(裸指针!)
- **持有**:`TWeakObjectPtr<AActor>` + 5 个 v8::Global
- **依赖 v8.h**(THIRD_PARTY_INCLUDES_START/END 保护 include)
- **生命周期**:ctor 传 `AActor*`;dtor 虚函数,应该会 Reset 所有 Global(清理 JS 端引用)

## Twin(New 版对应)

**New 里没有对应的类**!职责被拆散:
- `AActor*` 持有 → [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle::EffectActor]] 直接持(`TStrongObjectPtr<AActor>`)
- 5 个 JS 回调 → 移到 `ScriptBridge/` 目录(`EffectJsFunctionHolder.h`, `EffectSystemScriptBridge.h`,Batch 5 读)
- `IsFreeze` → 合并到 Handle 的 `IsFreezeInternal`
- JS 镜像对象 `GlobalJsHandle` → New 里由 `FEffectSystemScriptBridge` 统一管理(推测,待 Batch 5 验证)

**这次合并是架构进步**:Old 把 JS 东西硬塞在 Handle 的伴生对象里,但 Handle API 仍然泄漏了 `v8::Isolate*`;New 把 JS 回调抽到专用 Bridge 类,Handle 本体完全不碰 v8。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystem\EffectHandleInfo.h`(~100 行)
- 配套:[[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle (Old)]] 里 `RegisterJsObject` / `RegisterCommonJsFunction` / `RegisterGroupDelayPlayJsFunction` / `RegisterMultiEffectAdjustNumJsFunction` 的目标都是这里

## 开放问题

- 5 个 v8::Global 对应的 JS 函数签名(`PreStop()` vs `UpdateLifeCycle(deltaTime)` vs `MultiEffectAdjustNumber(num)` 等)——应能从 .cpp 的 `Call(Isolate, Receiver, argc, argv)` 反推,Batch 6 确认
- `GlobalJsHandle` 是否就是 TS 侧 `EffectHandle` class 的实例镜像?
- `~FEffectHandleInfo` 里怎么处理 Reset——单独 Reset 每个 Global,还是全部的生命周期绑在 Isolate 上?
