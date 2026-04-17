---
type: concept
created: 2026-04-17
updated: 2026-04-17
tags: [concept, architecture, scripting, cross-language, pattern]
sources: 2
aliases: [Scripting Bridge, Language Bridge Pattern, FFI Holder]
---

# 脚本桥模式(Scripting Bridge Pattern)

> 当 C++ 主系统和脚本语言(JS/TS/C#/Python/Lua)互相调用时,**把跨语言边界集中到一个专门的 Bridge 层**,让主系统内部只和原生 C++ 类型打交道。从 [[syntheses/kuro-effect-system-old-vs-new|KuroEffectSystem Old→New]] 的重构里总结出来。

## 一句话定义

**脚本桥** = 在"主系统"和"脚本宿主"之间插入一层,对上暴露 C++ 原生类型,对下吸收各种脚本边界特殊性(v8::Global, UE Dynamic Delegate, Python C API 等)。

## 为什么需要

### 反面教材:无 Bridge 的架构

[[entities/project-game/kuro-effect-system/effect-handle|Kuro Old]] 就是典型:
```cpp
class FEffectHandle {
    void Tick(v8::Isolate* Isolate, float DeltaSeconds);
    void SeekTo(v8::Isolate* Isolate, float Time, bool AutoLoop);
    void RegisterJsObject(v8::Isolate* Isolate, v8::Local<v8::Object>);
    // ... v8 贯穿所有公开 API
};
```

**症状**:
1. **脚本宿主类型渗透到核心**(`v8::Isolate*` 作为参数)
2. **切换脚本宿主几乎不可能**(签名全是 v8 专用)
3. **单元测试必须启脚本环境**(v8 Isolate 依赖)
4. **GC 和 Closure 语义泄漏**(`v8::Global<T>` 生命周期规则感染主系统)

### Bridge 的解法

```
┌─ 脚本业务 ────────────┐
│ TS / C# / Python ...  │
└──────────┬────────────┘
           │
           ▼
┌─ Bridge 层 ──────────────────────────────┐
│                                          │
│ 持脚本原生 Callback:                     │
│   JsBridge:  v8::Global<Function> × N     │
│   CsBridge:  FDynamicDelegate × N          │
│   LuaBridge: lua_ref × N                   │
│                                          │
│ 提供一致的 C++ 原生接口给主系统           │
│                                          │
└──────────┬───────────────────────────────┘
           │
           ▼
┌─ 主系统(核心业务逻辑) ────────────────┐
│                                         │
│ 类型纯 C++:                             │
│   TSharedPtr<FEffectInitCallback>       │
│   TFunction<void(int)>                  │
│                                         │
│ 不知道脚本宿主是什么                     │
└─────────────────────────────────────────┘
```

## 核心组件

### 1. Bridge 门面(Facade)

**职责**:暴露"上行 API"(主系统要的功能,需要脚本回答),dispatch 到具体 Bridge 实现。

Kuro 例子:[[entities/project-game/kuro-effect-system-new/effect-system-script-bridge|FEffectSystemScriptBridge]]
```cpp
class FEffectSystemScriptBridge {
    FEffectSystemJsBridge     JsBridge;
    FEffectSystemCSharpBridge CSharpBridge;
public:
    // 门面方法,内部 dispatch
    AActor* EffectHandle_GetEntityOwnerActor(int32 EntityId) {
        if (CSharpBridge.IsBound_GetEntityOwnerActor())
            return CSharpBridge.GetEntityOwnerActor(EntityId);
        if (JsBridge.IsValid())
            return JsBridge.GetEntityOwnerActor(EntityId);
        return nullptr;
    }
};
```

### 2. 脚本侧 Bridge 实现

**每种脚本宿主一个**。存它自己的原生 callback 类型,暴露和门面对应的方法:
- **JsBridge**:`v8::Global<v8::Function>` + `HandleScope` + `Call` 模式
- **CSharpBridge**:UE Dynamic Delegate(`DECLARE_DYNAMIC_DELEGATE_*`)+ `.Execute` 模式
- **LuaBridge**:`luaL_ref` 保存函数,`lua_call` 调用

### 3. Callback Holder(回调持有者)

**职责**:把脚本侧的 lambda/function 包装成 C++ 原生 Delegate,业务可以像用普通 C++ 回调一样消费。

Kuro 例子:[[entities/project-game/kuro-effect-system-new/effect-function-holders|FEffectSpawnCallbackHolder]]
```cpp
class FEffectSpawnCallbackHolder {
    v8::Global<v8::Function> BeforeInitGlobal;
    v8::Global<v8::Function> InitGlobal;
    // ...
    v8::Global<v8::Object> JsObjectGlobal;   // this 绑定
    
    // thunk 方法,作为 Delegate 的实际执行
    void OnBeforeInitThunk(int32 EffectId) {
        // HandleScope + Context + Call
        v8::Local<v8::Function> Fn = BeforeInitGlobal.Get(Isolate);
        Fn->Call(Ctx, Self, 1, args);
    }
    
public:
    // 暴露 C++ 原生 Delegate(主系统只看到这个)
    TSharedPtr<FEffectBeforeInitCallback> GetBeforeInitCallback() {
        auto Cb = MakeShared<FEffectBeforeInitCallback>();
        Cb->BindRaw(this, &FEffectSpawnCallbackHolder::OnBeforeInitThunk);
        return Cb;
    }
};
```

### 4. 类型反射 / 双版本

**脚本友好类型**(USTRUCT / 反射版)**≠ 运行时类型**(纯 C++)。边界处 ctor 一次性转换:
```cpp
// USTRUCT 版:脚本可见
USTRUCT(BlueprintType) struct FKuroEffectContext {
    GENERATED_BODY()
    UPROPERTY() int32 EntityId;
    // ...
};

// 原生版:运行时用
class FEffectContext {
    int32 EntityId;
    // ...
    FEffectContext() = default;
    FEffectContext(const FKuroEffectContext& Reflected) {    // 转换 ctor
        EntityId = Reflected.EntityId;
    }
};
```

**为什么**:USTRUCT 反射每次访问要走 UPROPERTY 元数据,慢。热路径用纯 C++ 对象零开销。跨边界**一次性转换**替代**每次访问**。

参考 [[entities/project-game/kuro-effect-system/effect-parameters|Kuro Old 的 FKuroEffectNiagaraParameters]] 顶部注释——作者原话:

> "作为UObject在Ts函数传参时的效率是比Struct高的,不过在Ts中new一个新的UObject走反射消耗在四五十微妙,如果new得多可以考虑作为一个纯C++类,然后不走反射,走模板绑定"

## 关键设计决策

### 所有权方向:Holder 弱,Model 强

业务注册回调的所有权图:
```
FEffectInitModel   (TSharedPtr<Callback>)   <─── 强持
    │
    │ 注册进
    ▼
FEffect*FunctionHolder   (TWeakPtr<Callback>)  <─── 弱持
```

原因:
- Model 代表业务意图,**业务决定 Callback 存多久**
- Holder 是"桥",**不该延长 Callback 生命**
- Model 析构 → SharedPtr 释放 → Holder 的 WeakPtr 自动失效 → 下次 IsValid 返回 false,Holder 自动清理

### 双语言并存 vs 择一

**策略 A**:编译期 `#ifdef` 择一——简单但不灵活
**策略 B**:运行时并存,门面 dispatch——灵活但复杂

Kuro 选 B。代价:
- Bridge 门面需要"谁绑定了就用谁"的逻辑
- 两种 Holder 类型翻倍(6 个 Holder 类)
- 每个方法都要检查 `IsBound()` / `IsValid()`

收益:
- 未来新加 Lua/Python 不用改主系统
- 同一系统不同场景用不同脚本(编辑器用 C# 工具链,游戏用 JS 业务逻辑)

### 上行 vs 下行分离

- **下行(脚本调 C++)**:通过 BP Function Library / Puerts 绑定,**方法签名固定**,一次导出
- **上行(C++ 调脚本)**:通过 Bridge 的已注册 callback,**运行时 dispatch**

这两个方向**复杂度完全不同**——别放在同一条路径。

## 与其他概念的关系

- **类似**:Adapter Pattern、Facade Pattern、Command Pattern、Bridge Pattern(GoF)
- **相邻**:FFI(Foreign Function Interface)、IPC 序列化
- **依赖**:**UE 反射系统**(USTRUCT/UCLASS/Dynamic Delegate)—— 本模式在 UE 里特别好用,因为 UE 已有反射 infra

## 实现此模式的步骤(从 Old 改造到有 Bridge 的推荐路径)

1. **识别污染面**:搜索 `v8::`、`lua_`、`python::` 等脚本宿主类型在你的主系统里的出现点
2. **圈定"上行 API"**:列出所有 C++ 调脚本的函数,它们就是 Bridge 门面的方法集
3. **创建 Bridge 门面**:先创建一个空的 Bridge 类,把"上行 API"暂时全部标为 `return {};`
4. **创建 Holder 类**:每种业务回调场景(Init/Finish/Custom)一个 Holder,包装脚本原生 callback → `TSharedPtr<C++ Delegate>`
5. **替换签名**:主系统里的 `void Tick(v8::Isolate*, float)` → `void Tick(float)`,内部不再需要 Isolate
6. **脚本侧注册**:脚本启动时 `Bridge.Register(callback1, callback2, ...)` 一次性注册所有上行函数
7. **测试独立性**:主系统的单元测试**不启脚本环境**也能跑——替代品是 C++ 原生 callback bind 到测试函数

## 适用场景

适合:
- 游戏引擎 + 脚本语言(UE + Puerts / Unity + C# + IL2CPP)
- C++ 后端 + Python 算法(AI / 数据分析)
- C++ Native app + Lua 配置 / 扩展(Neovim, Redis, ...)

不适合:
- 单一语言的项目(没有边界何来桥)
- **小项目**:Bridge 的基础设施成本高,只写几个跨语言调用不值得

## 引用来源

- [[wiki/sources/project-game/kuro-effect-system/overview|Kuro Old:无 Bridge 的反面教材]]
- [[wiki/sources/project-game/kuro-effect-system-new/overview|Kuro New:完整 Bridge 实现]]
- [[entities/project-game/kuro-effect-system-new/scripting-bridge-architecture|Bridge 架构总览]]
- [[entities/project-game/kuro-effect-system-new/effect-function-holders|Callback Holder 实现]]
- [[syntheses/kuro-effect-system-old-vs-new|Old→New 综合(包含 Bridge 作为结构性改变之一)]]

## 开放问题 / 观察

- 这个模式在**非 UE 项目**里(纯 C++ 程序 + Lua/Python)的变体?(没有 USTRUCT 反射 infra 时怎么做 USTRUCT 双版本?)
- 多脚本并存时,如果两边都绑定同一个上行 API,优先级/合并策略怎么定?
- **Go / Rust 等新语言**的 FFI 和这个模式的契合度?
