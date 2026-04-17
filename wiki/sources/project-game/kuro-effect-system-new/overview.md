---
type: source
source_kind: code-module-overview
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current]
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[sources/project-game/kuro-effect-system/overview]]
---

# KuroEffectSystemNew — 模块总览

- **Repo**:`project-game`
- **路径**:`Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/`(+ 同目录 `Private/KuroEffectSystemNew/`)
- **快照**:p4 CL `7086024` (2026-04-17)
- **规模**:42 个 Public 头 / 23 个 Private 实现 / ~23k 行(New 的 Private 比 Public 还大,说明逻辑往 `.cpp` 里沉)
- **状态**:**当前版本**,正在演进
- **Ingest 日期**:2026-04-17(Batch 1 扫顶层)

## 定位

Aki 特效运行时系统"**第二代**",正在替代 [[sources/project-game/kuro-effect-system/overview|Old 系统]]。不是简单重构——是**架构重写**:解耦 JS、引入池化、引入多玩家作用域、把业务规模化所需的骨架全装上。

代码文件里明确标注这是"TsEffectHandle 完全迁移"(见 [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]] 类顶部的 `#pragma region` 注释)——暗示 **部分逻辑以前在 TS 侧,这次下沉到了 C++**。

## 模块结构

```
KuroEffectSystemNew/                        ← Public/
├── EffectSystem.h                          ← 入口类 KuroEffect::FEffectSystem(全 static)
├── EffectSystemActor.h                     ← AEffectSystemActor(新增,特效 Actor 的宿主)
├── EffectHandle.h                          ← FEffectHandle(TSharedFromThis,~265 行,核心)
├── EffectInitHandle.h                      ← FEffectInitHandle(初始化缓存,Pending 态持有)
├── EffectInitModel.h                       ← FEffectInitModel(初始化参数模型)
├── EffectContext.h                         ← FEffectContext(创建/调用上下文)
├── EffectActorHandle.h                     ← FEffectActorHandle(EffectActor 的句柄包装)
├── EffectSpecData.h                        ← FEffectSpecData(Spec 的配置数据层)
├── EffectLifeTime.h                        ← FEffectLifeTime(与 Timer System 集成)
├── NiagaraComponentHandle.h                ← FNiagaraComponentHandle
├── PlayerEffectContainer.h                 ← FPlayerEffectContainer(玩家作用域容器)
├── ContinuousEffectController.h            ← FContinuousEffectController(连续特效)
├── EffectSystemHandleHelper.h              ← 脚本用 ID-based 门面(Batch 5)
├── EffectSystemForPuerts.h                 ← Puerts 专用导出(Batch 5)
├── EffectSpec/                             ← Spec 家族,含 `IEffectSpecBase` + `FEffectSpecFactory`
├── Reflection/                             ← Puerts 反射层(Batch 5)
└── ScriptBridge/                           ← JS/C# 函数回调持有层(Batch 5)
```

## 关键架构特征

- **全 static 模块单例**:`KuroEffect::FEffectSystem` 所有成员/方法都是 `static`,没有实例。每个 `static` 成员是"全局特效系统状态的一部分"。参见 [[entities/project-game/kuro-effect-system-new/effect-system]]。
- **namespace `KuroEffect`**:避免与 Old 的全局同名类冲突;代码里引用 Old 的东西时写 `KuroEffectSystem/XXX`(见 EffectSystem.h L9:`#include "KuroEffectSystem/KuroEffectVisibilityOptimizeController.h"`——New 复用了 Old 的 Visibility 剔除逻辑)。
- **智能指针所有权**:
  - `TArray<TSharedPtr<FEffectHandle>> Effects` —— 强持有
  - `TMap<int32, TSharedPtr<FEffectHandle>> TickHandleMap` —— 需要 tick 的子集
  - `TLru<FName, FEffectHandle> LruPool` —— 按 Path 索引的 LRU 池(Old 无此概念)
  - `FEffectHandle : TSharedFromThis<FEffectHandle>` —— handle 自身能拿到自己的共享所有权
- **零 v8 泄漏到核心**:整个 `FEffectSystem` 和 `FEffectHandle` 类体内**几乎看不到 `v8::Isolate*`**——唯一出现是 `Initialize(v8::Isolate* Isolate, ...)` 的入参(用于初始化 JsEnv 清理回调)。其余全部通过 `FEffectSystemScriptBridge` 抽一层(Batch 5 细读)。
- **四段式生命周期回调**:`FEffectBeforeInitCallback` → `FEffectInitCallback` → `FEffectBeforePlayCallback` → `FEffectHandleFinishDelegate`。每阶段都能注入业务逻辑,`BeforeInit` 还能改写 handle 参数。
- **数据驱动分离**:
  - `FEffectSpecData`(配置数据,来自 db)→ `FEffectInitModel`(每次创建的参数) → `FEffectContext`(运行时上下文)→ `IEffectSpecBase`(运行时 Spec 对象)
  - 每一层都能独立测试、替换、序列化
- **明确的编辑器/PIE 钩子**:`OnBeginPIE` / `OnEndPIE` / `OnEditorCurrentMapFinishExit` —— Old 完全没有,导致编辑器里预览特效逻辑混乱的问题在 New 里有解。
- **丰富的 doxygen 注释**:Public API 几乎每个函数都有中文说明,是 Old 没有的。

## 关键子系统(待 Batch 2-5 深入)

| 子系统 | 位置 | 负责 |
|---|---|---|
| **Handle 核心** | `EffectHandle.h` (~265 行) | 特效实例;含 Pending Init / OwnerEffect / BodyEffect 多个 region |
| **Init Pipeline** | `EffectInitHandle.h` + `EffectInitModel.h` | 特效初始化缓存 + 参数;Pending 态特效的持有 |
| **Spec 家族** | `EffectSpec/` | `IEffectSpecBase` 接口 + `FEffectSpecFactory` + 类型化子类 |
| **Actor 层** | `EffectSystemActor.h` + `EffectActorHandle.h` | `AEffectSystemActor`(Old 无);EffectActor 的脚本侧句柄 |
| **Data 层** | `EffectSpecData.h` | 来自 db 的配置数据模型 |
| **Player Container** | `PlayerEffectContainer.h` | 玩家/队伍作用域的特效池(Old 无) |
| **Continuous** | `ContinuousEffectController.h` | 连续特效控制器(Old 无) |
| **Script Bridge** | `ScriptBridge/` + `Reflection/` + `EffectSystemForPuerts.h` + `EffectSystemHandleHelper.h` | 脚本调用的统一门面;Batch 5 专项 |
| **Visibility 剔除** | `KuroEffectSystem/KuroEffectVisibilityOptimizeController.h`(**复用 Old**) | — |

## 涉及实体(本批已建页)

- [[entities/project-game/kuro-effect-system-new/effect-system|KuroEffect::FEffectSystem]] — 入口静态类
- [[entities/project-game/kuro-effect-system-new/effect-handle|KuroEffect::FEffectHandle]] — 句柄核心

`FEffectLifeTime`(new)、`FEffectSystemHandleHelper`、其他句柄辅助类本批扫过,独立页延后到对应专项 batch。

## 与 Old 的关系

见孪生页 [[sources/project-game/kuro-effect-system/overview|KuroEffectSystem overview]]。完整 diff 延后到 **Batch 7 synthesis**。

## 开放问题 / 待读

- **Static 单例的初始化时机**:`FEffectSystem::Initialize` 在哪里被调用?是 `UKuroGameInstance` 还是 `KuroGameplayModule`?(Batch 6 会读 Module.cpp 确认)
- **LRU 容量**:`EFFECT_LRU_CAPACITY` 的默认值?(定义是 `static int32`,赋值在 .cpp,Batch 6)
- **Old 还在运行吗**:New 已经是主力,Old 为什么还没删?是否只保留某些特殊特效类型(如 GpuParticle、SequencePose、NDC 这些 New 里没看到对应 Spec 的)走 Old?Batch 2 看 DataAsset 分布时会有答案。
- **`HoldPreloadObject`**:`TStrongObjectPtr<UHoldPreloadObject> HoldPreloadObject` 是什么?和预加载特效相关但没见过 Old 里有对应物。
