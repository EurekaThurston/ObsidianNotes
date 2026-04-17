---
type: entity
category: class-hierarchy
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, script-bridge, callback]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/ScriptBridge
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectSpawnCallbackHolder, FEffectFinishCallbackHolder, FCSharpEffectSpawnCallbackHolder, IJsFunctionHolder, ICSharpFunctionHolder]
---

# Function Holders(脚本回调持有者)

> 脚本语言和 C++ 主系统之间的**回调适配器**。一组把"JS 函数" / "C# UE Dynamic Delegate" 包装成"C++ 原生 UE Delegate"的 Holder 类。每种回调场景 3 类(Spawn / Finish / DynamicInit),每类都有 JS 和 C# 两个实现 → 共 6 个 Holder + 2 个接口。

## 为什么要 Holder 这一层

业务场景:TS 侧业务代码写:
```typescript
KuroEffectSystem.SpawnEffect(..., 
    beforeInit = (effectId) => { console.log("before init", effectId); },
    initCallback = (result, effectId) => { console.log("init", result); },
    beforePlay = (effectId) => { console.log("before play"); });
```

这些 lambda 变成 `v8::Local<v8::Function>` 跨过 C++ 边界。

但 **C++ 主系统内部想要的是** `TSharedPtr<FEffectInitCallback>`(UE C++ 原生 Delegate)——因为:
- **统一类型**:JS 和 C# 的 callback 最终都要用同样的内部类型
- **解耦语言**:Spec/Handle 内部不知道(也不该知道)回调源自哪种语言
- **生命周期**:`TSharedPtr` 让 C++ 侧的 callback 可以被 Spec 长期持有

Holder 做的事:
1. **持久化**:把 `v8::Local` 转成 `v8::Global` 存起来(否则 HandleScope 一退就失效)
2. **包装**:生成 `TSharedPtr<FEffectInitCallback>` 暴露给主系统
3. **桥接**:主系统调用 `FEffectInitCallback` 时,内部反过来调 `v8::Global<v8::Function>`

## 两个接口(JS / C#)

### `IJsFunctionHolder`
```cpp
class IJsFunctionHolder
{
public:
    virtual bool IsValid() const = 0;
    virtual void Clear(bool IsForce = false) = 0;
    virtual ~IJsFunctionHolder() {}
};
```

### `ICSharpFunctionHolder`
```cpp
class ICSharpFunctionHolder
{
public:
    virtual bool IsValid() const = 0;
    virtual void Clear(bool IsForce = false) = 0;
    virtual ~ICSharpFunctionHolder() {}
};
```

**两个接口结构一致**——`IsValid` 判断持的函数还有效吗,`Clear` 释放。

## 3 种 Holder × 2 语言 = 6 类

每种 Holder 对应一种**业务回调场景**,每种又有 JS/C# 两个实现。

## 1️⃣ SpawnCallbackHolder(四段 Spawn 回调)

封装 Spawn 流程的 **BeforeInit + Init + BeforePlay + OnClear + JsObject** 共 5 个函数。

### JS 版 `FEffectSpawnCallbackHolder`

```cpp
class FEffectSpawnCallbackHolder : public IJsFunctionHolder
{
    v8::Global<v8::Function> BeforeInitCallbackGlobal;
    v8::Global<v8::Function> EffectInitCallbackGlobal;
    v8::Global<v8::Function> BeforePlayCallbackGlobal;
    v8::Global<v8::Function> OnClearGlobal;
    v8::Global<v8::Object>   JsObjectGlobal;              // ← JS 侧的 this 对象

    TWeakPtr<FEffectBeforeInitCallback> BeforeInitCallback;
    TWeakPtr<FEffectInitCallback>       EffectInitCallback;
    TWeakPtr<FEffectBeforePlayCallback> BeforePlayCallback;

    // 内部 thunks:被 FEffect*Callback delegate 触发,转调 v8::Global 函数
    void OnBeforeInitCallback(int32 EffectId) const;
    void OnEffectInitCallback(ELoadEffectResult Result, int32 EffectId) const;
    void OnBeforePlayCallback(int32 EffectId) const;

public:
    void Init(v8::Isolate*, 
              v8::Local<v8::Function> BeforeInit,
              v8::Local<v8::Function> EffectInit,
              v8::Local<v8::Function> BeforePlay,
              v8::Local<v8::Function> OnClear,
              v8::Local<v8::Object>   JsObject);
    
    virtual void Clear(bool IsForce) override;
    virtual bool IsValid() const override;
    
    TSharedPtr<FEffectBeforeInitCallback> GetBeforeInitCallback();
    TSharedPtr<FEffectInitCallback>       GetEffectInitCallback();
    TSharedPtr<FEffectBeforePlayCallback> GetBeforePlayCallback();
};
```

### 两个关键细节

**1. `TWeakPtr<FEffectBeforeInitCallback>` 弱引用**

```cpp
TWeakPtr<FEffectBeforeInitCallback> BeforeInitCallback;
```

**为什么是 Weak**?
- 真正的 `TSharedPtr<FEffectBeforeInitCallback>` 是由 [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel]] 持有的
- Holder 只持弱引用——**Holder 不强制让 callback 活着**
- InitModel 析构后,Holder 这里的弱引用自动失效,下次 `IsValid()` 返回 false

**2. `v8::Global<v8::Object> JsObjectGlobal`**

第 5 个 Global 不是函数,而是 **JS 侧的 this 对象**。

TS 方法绑定:
```typescript
class MySpawner {
    beforeInit(id: number) { this.lastId = id; }
    spawn() {
        KuroEffectSystem.SpawnEffect(..., this.beforeInit.bind(this), ...);
    }
}
```

但 `bind(this)` 在 v8 是会丢的。**Kuro 选另一种方式**:把 `this` 对象单独传给 Holder。调用 `beforeInit` 时:
```cpp
// 伪代码
v8::Local<v8::Object> Self = JsObjectGlobal.Get(Isolate);
v8::Local<v8::Function> Fn = BeforeInitCallbackGlobal.Get(Isolate);
Fn->Call(Ctx, Self, argc, argv);    // ← Self 作为 this
```

—— **this 明确传入,不依赖 JS 侧的 bind**。

### C# 版 `FCSharpEffectSpawnCallbackHolder`

```cpp
class FCSharpEffectSpawnCallbackHolder : public ICSharpFunctionHolder
{
    // 4 个 UE Dynamic Delegate(不是 v8::Global)
    FKuroEffectBeforeInitCallback BeforeInitCallbackGlobal;
    FKuroEffectInitCallback       EffectInitCallbackGlobal;
    FKuroEffectBeforePlayCallback BeforePlayCallbackGlobal;
    FKuroEffectOnClearCallback    OnClearGlobal;
    // 注意:**没有 JsObject**——C# 不需要 this 隐式绑定
    
    TWeakPtr<FEffectBeforeInitCallback> BeforeInitCallback;
    TWeakPtr<FEffectInitCallback>       EffectInitCallback;
    TWeakPtr<FEffectBeforePlayCallback> BeforePlayCallback;

    void OnBeforeInitCallback(int32 EffectId) const;
    void OnEffectInitCallback(ELoadEffectResult Result, int32 EffectId) const;
    void OnBeforePlayCallback(int32 EffectId) const;

public:
    void Init(const FKuroEffectBeforeInitCallback& BeforeInit,
              const FKuroEffectInitCallback& EffectInit,
              const FKuroEffectBeforePlayCallback& BeforePlay,
              const FKuroEffectOnClearCallback& OnClear);
    
    virtual void Clear(bool IsForce) override;
    virtual bool IsValid() const override;
    
    /* 同 3 个 Get*Callback */
};
```

**结构对称**,但类型不同。C# 的 UE Dynamic Delegate 已包含 target 对象信息(UE Reflection 会绑定 `UObject*` this),不需要单独 JsObject 字段。

## 2️⃣ FinishCallbackHolder(完成回调)

只有 1 个业务函数(加 OnClear 和 JsObject)。

### JS 版 `FEffectFinishCallbackHolder`
```cpp
class FEffectFinishCallbackHolder : public IJsFunctionHolder
{
    v8::Global<v8::Function> EffectFinishCallbackGlobal;
    v8::Global<v8::Function> OnClearGlobal;
    v8::Global<v8::Object>   JsObjectGlobal;

    TWeakPtr<FEffectHandleFinishDelegate> EffectFinishCallback;

    void OnEffectFinishCallback(int32 EffectId);

public:
    void Init(v8::Isolate*, v8::Local<v8::Function> FinishCB, v8::Local<v8::Function> OnClear, v8::Local<v8::Object> JsObject);
    virtual void Clear(bool IsForce) override;
    virtual bool IsValid() const override;
    
    TSharedPtr<FEffectHandleFinishDelegate> GetEffectHandleFinishCallback();
};
```

**注意**:这里是 `FEffectHandleFinishDelegate`([[entities/project-game/kuro-effect-system/effect-define|EffectDefine]] 的单播 delegate),不是 `FEffectFinishCallback`。

为什么?回忆 EffectDefine:
```cpp
DECLARE_DELEGATE_OneParam(FEffectFinishCallback, int)              // 旧的
DECLARE_DELEGATE_OneParam(FEffectHandleFinishDelegate, int);       // 新的,Handle 持
```

**业务用的 Finish** 最终挂到 Handle 的 `TSharedPtr<FEffectHandleFinishDelegate> FinishDelegate` 字段上——不是作为"Spawn 的一个回调",而是"Handle 自己的完成监听"。Holder 的返回类型顺应 Handle 的字段类型。

### C# 版 `FCSharpEffectFinishCallbackHolder`
结构同 JS 版,用 `FKuroEffectFinishCallback` 替代 v8::Global。

## 3️⃣ DynamicInitCallbackHolder(动态注册的 Init 回调)

对应 `FEffectSystem::DynamicRegisterSpawnCallback(EffectId, Callback)`——业务侧**在 Spawn 之后**还能追加注册 init callback 的场景。

典型用途:BP_EffectActor 的 Play 里没有 Callback 参数,特效实际创建后业务如果想知道 Init 结果,调 DynamicRegister 追加。

### JS 版
```cpp
class FEffectDynamicInitCallbackHolder : public IJsFunctionHolder
{
    v8::Global<v8::Function> EffectDynamicInitCallbackGlobal;
    v8::Global<v8::Function> OnClearGlobal;
    v8::Global<v8::Object>   JsObjectGlobal;

    TWeakPtr<FEffectInitCallback> DynamicEffectInitCallback;

    void OnEffectDynamicInitCallback(ELoadEffectResult Result, int32 EffectId) const;

public:
    void Init(v8::Isolate*, v8::Local<v8::Function> Cb, v8::Local<v8::Function> OnClear, v8::Local<v8::Object> JsObject);
    virtual void Clear(bool IsForce) override;
    virtual bool IsValid() const override;
    
    TSharedPtr<FEffectInitCallback> GetEffectDynamicInitCallback();
};
```

`FEffectInitCallback`(非 `Handle`Finish 版)——因为这是"针对一次 init 的回调",语义上和 Spawn 回调同类。

### C# 版
结构同 JS 版,用 UE Dynamic Delegate。

## 生命周期(全部 Holder 共享)

```
1. [Init]   业务 Spawn/Register → JsBridge::CreateXxxHolder(...)
            ├─ Init() 存 v8::Global
            └─ 加入 JsFunctionHolders 数组

2. [Use]    业务/系统需要 TSharedPtr<FEffectInitCallback>
            ├─ Holder::GetEffectInitCallback() 
            └─ 返回强 Delegate,Holder 自持 TWeakPtr

3. [Invoke] 主系统调 Callback.Execute(...)
            ├─ Delegate 触发 Holder::OnEffectInitCallback(thunk)
            └─ Thunk 从 v8::Global 取 Local,调 JS 函数

4. [Clear]  Holder 每帧 PostTick 被扫描 IsValid()
            └─ 失效 → Clear() → Reset v8::Global(让 v8 GC JS 函数)

5. [Dtor]   ~FEffectSpawnCallbackHolder ≈ Clear(true)
```

## Thunk 方法(**核心细节**)

每个 Holder 里都有一个或多个**私有 thunk 方法**。它们是真正的"桥":
```cpp
void OnEffectInitCallback(ELoadEffectResult Result, int32 EffectId) const;
```

推断实现(.cpp 未读):
```cpp
void FEffectSpawnCallbackHolder::OnEffectInitCallback(ELoadEffectResult Result, int32 EffectId) const
{
    // 1. HandleScope + ContextScope
    v8::Isolate* Isolate = /* get */;
    v8::HandleScope HS(Isolate);
    v8::Local<v8::Context> Ctx = /* get default */;
    v8::Context::Scope CS(Ctx);
    
    // 2. 准备 this 和 function
    v8::Local<v8::Object> Self = JsObjectGlobal.Get(Isolate);
    v8::Local<v8::Function> Fn = EffectInitCallbackGlobal.Get(Isolate);
    if (Fn.IsEmpty()) return;
    
    // 3. 准备参数
    v8::Local<v8::Value> Args[] = {
        v8::Integer::New(Isolate, (int)Result),
        v8::Integer::New(Isolate, EffectId)
    };
    
    // 4. 调用 JS
    Fn->Call(Ctx, Self, 2, Args);
}
```

`Init` 里通过 `MakeShared<FEffectInitCallback>(...)` 创建 UE Delegate,让它绑 `OnEffectInitCallback`(`BindRaw(this, &...)`)。

## 业务使用 flow

```cpp
// 伪代码:业务 TS 调 SpawnEffect
// 1. TS 调用 EffectSystemForPuerts 暴露的 C++ 接口
EffectSystemForPuerts::SpawnEffect(ctx, transform, path, reason, 
    beforeInitFn, initFn, beforePlayFn, onClearFn, thisObj);

// 2. C++ 侧:
auto Holder = ScriptBridge.EffectSystemJsBridge.CreateSpawnCallbackHolder(
    Isolate, beforeInitFn, initFn, beforePlayFn, onClearFn, thisObj);

// 3. 从 Holder 抽 C++ 原生 Delegate
TSharedPtr<FEffectBeforeInitCallback> BI = Holder->GetBeforeInitCallback();
TSharedPtr<FEffectInitCallback>       I  = Holder->GetEffectInitCallback();
TSharedPtr<FEffectBeforePlayCallback> BP = Holder->GetBeforePlayCallback();

// 4. 走标准 SpawnEffect 管线
FEffectSystem::SpawnEffect(WorldCtx, Transform, Path, Reason,
                            BI, I, BP, Context);
```

**脚本回调和 C++ 原生回调完全解耦**——SpawnEffect 实现**不关心**回调来自哪,只看 `TSharedPtr<FEffectBeforeInitCallback>`。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system-script-bridge|FEffectSystemJsBridge / FEffectSystemCSharpBridge]] **创建 + 持有**(`TArray<TSharedPtr<IXxxFunctionHolder>>`)
- **生产** `TSharedPtr<FEffect*Callback>`,给 [[entities/project-game/kuro-effect-system-new/effect-init-pipeline|FEffectInitModel]]
- **使用类型** [[entities/project-game/kuro-effect-system/effect-define|DECLARE_DELEGATE 声明的 3 种 Callback]]
- **弱引用** `TWeakPtr<FEffect*Callback>` 保证不循环

## Twin(Old 版对应)

**Old 没有 Holder 抽象**。JS 回调直接存 `FEffectHandleInfo` 的字段,业务调用 Handle API 时直接传 `v8::Local<v8::Function>`,然后 Info 把它们 Reset 成 Global 存起来。**没有"统一的 C++ 原生类型"作为目标** —— Handle 本身调 `v8::Global->Call(Isolate, ...)` 触发回调,全流程 v8 污染。

## 引用来源

- 原始代码:
  - `F:\...\ScriptBridge\EffectJsFunctionHolder.h`(~85 行,3 个 JS Holder)
  - `F:\...\ScriptBridge\EffectCSharpFunctionHolder.h`(~78 行,3 个 C# Holder)
- 依赖:[[entities/project-game/kuro-effect-system/effect-define|EffectDefine 的 3 种 Delegate]]
- 使用者:[[entities/project-game/kuro-effect-system-new/effect-system-script-bridge|FEffectSystemJsBridge / FEffectSystemCSharpBridge]] 的 CreateXxxHolder

## 开放问题

- `Clear(bool IsForce)` 参数 `IsForce` 的语义(强制清 vs 条件清)
- `OnPostUpdateWorkTick` 里清理 Holder 的具体触发条件(根据 `IsValid() == false` 还是其他?)
- `TWeakPtr<FEffect*Callback>` 失效时机:`FEffectInitModel` 析构时 `Reset`,Holder 下次检查就无效
- Holder 和 InitModel 的所有权环:Model 持 `TSharedPtr<Callback>`,Callback 里 bind 到 Holder 的 thunk,Holder 持 `TWeakPtr<Callback>` —— **循环被 Weak 打破**,但 Holder 的清理依赖 Model 的 Weak 失效
- C# 的 `FCSharpEffect*CallbackHolder` 没有 `JsObjectGlobal` 因为不需要(Dynamic Delegate 带 Target);那如何表示"callback 的 this 上下文"?
