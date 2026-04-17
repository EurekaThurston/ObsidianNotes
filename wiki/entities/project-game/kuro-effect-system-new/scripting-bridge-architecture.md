---
type: entity
category: architecture
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, script-bridge, architecture]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [Scripting Bridge Layer, Puerts Integration]
---

# 脚本桥架构(Scripting Bridge 总览)

> New 系统把 Old 硬塞进 Handle 的 "v8 回调" 重构成了一层**脚本桥架构**,能同时适配 **Puerts (JS/TS)** 和 **C#** 两种脚本宿主。本页是这层的**总览**——介绍涉及的 ~10 个类如何协作。

## 为什么需要这层

回忆 Old 的做法:
- [[entities/project-game/kuro-effect-system/effect-handle-info|FEffectHandleInfo]] 持 5 个 `v8::Global<v8::Function>` 作为 JS 回调
- [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 方法签名里埋 `v8::Isolate*`
- **只支持 Puerts(JS/TS)一种脚本**

问题:
1. **无法添加 C# 支持** —— 整个系统被 v8 类型污染
2. **耦合**:改脚本接口(加一个回调类型)要改 Handle/Info 两处
3. **测试困难**:C++ 单元测试必须启 v8 环境

New 的解决:
1. 抽象出 **`FEffectSystemScriptBridge`** 作为统一门面
2. 内部分 **`FEffectSystemJsBridge`**(Puerts)+ **`FEffectSystemCSharpBridge`**(C#)两条独立路径
3. 用 **Holder 类** 包装不同脚本环境的回调函数(`v8::Global` vs `FKuroEffectXxx` UE Dynamic Delegate)
4. 所有脚本可见的类型用 UE **USTRUCT/UCLASS** 包装(反射系统自动跨语言)

## 架构分层

```
┌─ 脚本层 (Puerts TS / C# / Blueprint) ────────────────────────────┐
│                                                                  │
│  业务代码调用:                                                    │
│    KuroEffectSystem.SpawnEffect(WorldCtx, Transform, "FX/Fire", ...)
│                                                                  │
└──────────────┬──────────────────────┬───────────────────────────┘
               │                      │
               ▼ (TS via Puerts)      ▼ (C# via UCLASS FunctionLib)
        EffectSystemForPuerts     UKuroEffectSystemFunctionLibrary
        (UsingCppType 模板绑定)    (UBlueprintFunctionLibrary)
               │                      │
               └──────────┬───────────┘
                          ▼
┌─ Reflection 层 ─────────────────────────────────────────────────┐
│                                                                  │
│   KuroEffectDefine.h:                                            │
│     - USTRUCT 版 Context: FKuroEffectContext 等 4 种              │
│     - USTRUCT 版 Parameters: FKuroEffectNiagaraParametersStruct  │
│     - USTRUCT 版 SpecData: FKuroEffectSpecData                   │
│     - DECLARE_DYNAMIC_DELEGATE_* × 30 (Spawn/Init/Finish/Bridge)  │
│                                                                  │
└──────────────┬──────────────────────────────────────────────────┘
               │(脚本调 C++ 主系统)
               ▼
┌─ 主系统入口 ─────────────────────────────────────────────────────┐
│                                                                  │
│   KuroEffect::FEffectSystem 主 static 类(Spawn/Stop/Tick)         │
│                                                                  │
└──────────────┬──────────────────────────────────────────────────┘
               │(系统回调脚本:生命周期钩子 / 业务查询)
               ▼
┌─ Bridge 层(核心) ────────────────────────────────────────────────┐
│                                                                  │
│   FEffectSystemScriptBridge (门面)                                │
│        ├── FEffectSystemJsBridge (持 30 个 v8::Global<Function>)   │
│        └── FEffectSystemCSharpBridge (持 30 个 FKuroXxxCallback)   │
│   |                                                              │
│   |-- Holder 类 (回调持有者):                                      │
│   |     JS: FEffectSpawnCallbackHolder, FEffectFinishCallbackHolder│
│   |         FEffectDynamicInitCallbackHolder (v8::Global 版)       │
│   |    C#: FCSharpEffectSpawnCallbackHolder, ... (UE Delegate 版) │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 两套语言,双倍 Bridge

New 支持两种脚本宿主:
- **Puerts JS/TS** — 成熟,业务主力
- **C#** — 新引入,可能为了某些边缘场景(编辑器工具?分布式服务?)

为了同时支持:
1. `FEffectSystemScriptBridge` 门面里**同时持有** `FEffectSystemJsBridge` 和 `FEffectSystemCSharpBridge`
2. **Bridge 层**分 `JsBridge` 和 `CSharpBridge`,接口相同但实现不同(v8::Global vs FKuroCallback)
3. **Holder 类**也分 JS 版和 C# 版(3 对,6 个 Holder 类)
4. **调用分发**:通过 `bEffectSystemJsBridgeValid` 标志决定优先走哪条路径

## 两种脚本调用方向

脚本和 C++ 的交互是**双向**的:

### 方向 1:脚本调 C++("**下行**")
- TS/C# 调 `SpawnEffect` → 经过 BP 函数库或 Puerts 绑定 → 走进 `FEffectSystem::SpawnEffect` C++ 实现
- 简单直接,用 UE 标准反射/模板绑定

### 方向 2:C++ 调脚本("**上行**")
- C++ 在某个决策点需要问脚本:**"这个 Entity 是联机玩家吗?"**
- 走 `FEffectSystemScriptBridge::EffectSystem_CheckIsNetPlayer(EntityId)` 
- Bridge 里有已注册的脚本函数(v8::Global 或 FKuroCallback)
- 调用它,拿返回值

上行调用才是 Bridge 层最复杂的部分——脚本必须**预先注册**这些函数,等 C++ 需要时才能调。

## Bridge 需要的 29 个"上行"函数

完整列表(`FEffectSystemScriptBridge` 暴露):

| 分类 | 函数名 | 职责 |
|---|---|---|
| **SkeletalMesh Spec** | `OnBodyEffectChange` / `CreateRenderingComponent` / `DestroyRenderingComponent` | 骨骼特效的渲染组件管理 |
| **EffectHandle 查询** | `GetEntityOwnerActor` / `GetEntityModelConfigId` / `GetOrAddEffectDynamicGroup` | Entity → Actor / Config / Dynamic Group 的反查 |
| **AudioSystem** | `GetAkComponent` / `ExecuteActionStop` / `PostEventTransform` / `PostEventAkComponent` | Wwise 音效播放 |
| **Spec 查询** | `NiagaraSpec_IsNeedQualityBias` / `PostProcessSpec_IsNeedPostEffect` / `PostProcessSpec_IsDisableInUltraSkill` | 特效类型特定的条件判断 |
| **ActorSystem** | `ActorSystem_Get` / `ActorSystem_Put` | Kuro 全局 Actor 池 |
| **EffectSystem** | `SetEffectView` / `CheckIsNetPlayer` / `CheckMobileBlackEffect` | 业务查询 |
| **BodyEffect** | `EffectSpec_RegisterBodyEffect` / `EffectSpec_UnregisterBodyEffect` | 身上光效注册 |
| **MaterialSpec** | `GetRenderingComponentByContext/BySkeletal/ByRenderActor`, `SpawnRenderActor`, `AddMaterialControllerData`, `RemoveMaterialControllerData`, `DestroyRenderingComponent` | 材质控制器 |
| **AudioController** | `AddPlayEffectAudio` / `AddPlayEffectAudioPriority` / `OnStopEffectAudio` | 高级音效控制 |

每个函数都有**两套实现**(v8 + C#),通过 Bridge 统一暴露。

## 29 个 UE Dynamic Delegate 类型

`Reflection/KuroEffectDefine.h` 里定义了对应 C# 侧的 UE Dynamic Delegate 类型:

```cpp
DECLARE_DYNAMIC_DELEGATE_RetVal_FourParams(UActorComponent*, FSkeletalMeshSpecOnBodyEffectChangeRetVal, 
    float, Opacity, UActorComponent*, CharRenderingComponent, USkeletalMeshComponent*, SkeletalMeshComponent, AActor*, Owner);

DECLARE_DYNAMIC_DELEGATE_RetVal_OneParam(AActor*, FEffectHandleGetEntityOwnerActorRetVal, int32, EntityId);

// ... 27 个类似的
```

**UE 的 Dynamic Delegate**:
- `DECLARE_DYNAMIC_DELEGATE_*` 在 UHT(Unreal Header Tool)里生成反射元数据
- 可以在**蓝图/C#** 里被绑定
- `RetVal_*` 系列**有返回值**,`_NoRetVal` 系列无
- 后面 N 个 params 指参数数量

所以 **C# 通过这些 Delegate 注册回调到 C++**,实际调用时 UE 反射系统做 marshaling。

## 与 Puerts 的模板绑定

文件 [[#EffectSystemForPuerts|`EffectSystemForPuerts.h`]]:
```cpp
#include "Binding.hpp"
#include "UEDataBinding.hpp"
#include "KuroEffectSystemNew/EffectSystem.h"

UsingCppTypeWithoutNamespace(KuroEffect, FEffectSpecData)
UsingCppTypeWithoutNamespace(KuroEffect, FEffectContext)
UsingCppTypeWithoutNamespace(KuroEffect, FSkeletalMeshEffectContext)
UsingCppTypeWithoutNamespace(KuroEffect, FEffectAudioContext)
UsingCppTypeWithoutNamespace(KuroEffect, FEffectRuntimeGhostEffectContext)
UsingCppTypeWithoutNamespace(KuroEffect, FSceneTeamItem)
UsingCppType(FKuroEffectNiagaraParameters)
```

`UsingCppType*` 是 **Puerts 提供的模板宏**——
- **把 C++ 类型注册到 Puerts 的类型系统**
- TS 侧 `import { FEffectContext } from 'cpp-types'` 就能拿到
- 比 USTRUCT 快:**不走 UE 反射**,走 Puerts 自己的模板元编程,**零开销的 passthrough**

注意:
- `FEffectContext` / `FEffectSpecData` / `FSceneTeamItem` 都在 `KuroEffect` 命名空间
- `UsingCppTypeWithoutNamespace(KuroEffect, ...)` 去掉命名空间暴露给 TS

## 为什么有 2 个 FEffectContext 版本

从 Batch 2 的 [[entities/project-game/kuro-effect-system-new/effect-context|FEffectContext]] 可以看到:
- `FEffectContext`(pure C++ class)在 `EffectContext.h`
- `FKuroEffectContext`(USTRUCT)在 `Reflection/KuroEffectDefine.h`

**两者互相转换**:
```cpp
FEffectContext(const FKuroEffectContext& Context);   // ctor 接受 USTRUCT 版
```

### 双版本的意义

| 版本 | 特点 | 用途 |
|---|---|---|
| `FKuroEffectContext` (USTRUCT) | 跨 UE 反射,可在 BP/C# 声明、序列化 | **脚本和 BP 层的入参**;UHT 自动 marshaling |
| `FEffectContext` (C++ class) | 虚继承,零反射开销,含 ctor 接受各种 Context | **运行时内部状态** |

调用路径:
```
TS 代码:       FKuroEffectContext ctx = { entityId: 42, ... };
                  ↓ Puerts UsingCppType 模板绑定
C++ 参数接收:  const FKuroEffectContext& Context
                  ↓ 内部 FEffectContext ctor
运行时对象:    TUniquePtr<FEffectContext> RuntimeCtx
                  ↓
Spec 使用:     Spec->SetContext(RuntimeCtx)
```

**一次拷贝穿过语言边界,运行时纯 C++**——这种模式在 [[entities/project-game/kuro-effect-system/effect-parameters|Old FKuroEffectNiagaraParameters 的顶部注释]] 里就有解释(避免每次走反射)。New 系统把它**系统化**地应用到 Context 和其他多个类型。

## 另一个 BP 函数库:UKuroEffectSystemHandleHelperLibrary

`Reflection/KuroEffectSystemHandleHelperLibrary.h` 是 [[entities/project-game/kuro-effect-system-new/effect-actor-handle|FEffectActorHandle]] 的脚本侧门面:
- 16 个 `ActorHandle_*` 方法——通过 int32 Id 间接调 EffectActor 操作
- 14 个 `NiagaraComponentHandle_*` 方法——通过 Id 间接调 Niagara 组件操作

**为什么不直接暴露 `FEffectActorHandle*`?**

回顾 [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 的注释:
> **"这些Handle得避免被Ts的堆内存持有,不然会有崩溃的风险"**

脚本语言的 GC 行为不可预测,若 TS 持 `FEffectActorHandle*` 裸指针,C++ 析构后 TS 还以为对象活着 → UAF。**改成 int32 Id 间接调用**,每次调都查表,永远安全。

## 分工速记

| 文件 | 角色 |
|---|---|
| `FEffectSystemScriptBridge.h` | 门面:暴露 29 个上行 API |
| `FEffectSystemJsBridge.h` | JS 实现:30 个 v8::Global + 30 个 Call* 方法 |
| `FEffectSystemCSharpBridge.h` | C# 实现:30 个 Callback + Is*Bound 查询 |
| `EffectJsFunctionHolder.h` | 3 种 JS Holder(Spawn/Finish/DynamicInit) |
| `EffectCSharpFunctionHolder.h` | 3 种 C# Holder(同上) |
| `Reflection/KuroEffectDefine.h` | 29 个 UE Dynamic Delegate 类型 + 4 种 USTRUCT Context + 4 种 USTRUCT Parameter + SpecData USTRUCT |
| `Reflection/KuroEffectSystemFunctionLibrary.h` | UBlueprintFunctionLibrary(~500 行,100+ UFUNCTION) |
| `Reflection/KuroEffectSystemHandleHelperLibrary.h` | ActorHandle/NiagaraHandle 的 BP Library |
| `EffectSystemHandleHelper.h` | 纯 C++ 的 static helper(Batch 1 已读) |
| `EffectSystemForPuerts.h` | Puerts 模板绑定声明 |

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] **持有**(静态)
- **使用各种** [[entities/project-game/kuro-effect-system-new/effect-context|Context]] **/** [[entities/project-game/kuro-effect-system/effect-model-base|DA]] **做参数**
- **生产** `TSharedPtr<FEffectInitCallback>` / `TSharedPtr<FEffectHandleFinishDelegate>` 等([[entities/project-game/kuro-effect-system/effect-define|DECLARE_DELEGATE 的 C++ 原生版]])

## Twin(Old 版对应)

**Old 没有分层的 Bridge**:
- 只支持 Puerts(v8)
- JS 回调直接存在 `FEffectHandleInfo` 里
- Handle/System 方法签名都带 `v8::Isolate*`
- 无 C# 支持路径

**New 把脚本集成完整分层化,这是架构最根本的进步**——未来加新脚本宿主(Lua、Python)也不会动核心代码。

## 引用来源

- `ScriptBridge/` 目录 5 个文件
- `Reflection/` 目录 3 个文件
- `EffectSystemForPuerts.h`(~15 行模板绑定声明)
- 消费者:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]]、[[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec<T>]]

## 开放问题

- **C# 集成的实际使用**:游戏业务代码还是 TS 主导,C# 路径的真实消费者是谁?(可能是编辑器工具?或某个边缘 mod 系统?)
- `bEffectSystemJsBridgeValid` 的作用时机—— JS 层未就绪时 `IsValid == false`?
- **双 Bridge 并行调用**:如果同一个事件 JS 和 C# 都注册了回调,会两个都触发吗?顺序?
- `UsingCppType` 模板在 Puerts 内部的代码生成机制(`Binding.hpp`/`UEDataBinding.hpp` 是 Puerts 私有)
- `FKuroEffectBeforeInitCallback` 等 Dynamic Delegate 在 C# 侧的语法体验(`+= lambda` 还是 `BindUFunction`?)
