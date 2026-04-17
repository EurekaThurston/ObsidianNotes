---
type: source
source_kind: code-module-overview
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy]
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
twin: [[sources/project-game/kuro-effect-system-new/overview]]
---

# KuroEffectSystem(老版)— 模块总览

- **Repo**:`project-game`
- **路径**:`Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/`(+ 同目录 `Private/KuroEffectSystem/`)
- **快照**:p4 CL `6985991` (2026-04-10)
- **规模**:43 个 Public 头 / 13 个 Private 实现 / ~9k 行
- **状态**:**基本冻结**——最新修改为 2026-04-10;团队已迁往 [[sources/project-game/kuro-effect-system-new/overview|New 系统]]
- **Ingest 日期**:2026-04-17(Batch 1 扫顶层)

## 定位

Aki 游戏项目里的特效运行时系统"**第一代**"。所有特效(Niagara 粒子、粒子 + Decal 组合、幽灵模型、后处理、骨骼网格、贝塞尔轨迹、静态网格、GPU 粒子等)的 spawn → tick → 表现参数同步 → stop 全生命周期,都从这里走。

业务 TS 脚本(Puerts)通过 v8 直接调用 C++ 接口,因此代码里大量方法签名直接嵌 `v8::Isolate*`。

## 模块结构

```
KuroEffectSystem/                           ← Public/
├── KuroEffectSystem.h                      ← 入口类 FKuroEffectSystem(instance-based)
├── KuroEffectLibrary.h                     ← BP/库式函数入口(本批未读)
├── KuroEffectSystemInterafece.h            ← 接口定义(本批未读,注意拼写)
├── EffectHandle.h                          ← FEffectHandle(每个特效实例一份)
├── EffectHandleInfo.h                      ← FEffectHandleInfo(Handle 的 JS 侧伴生体)
├── EffectLifeTime.h                        ← FEffectLifeTime(时间/播放状态管理)
├── EffectDefine.h                          ← 宏/枚举定义(本批未读)
├── KuroEffectVisibilityOptimizeController.h ← 可视性剔除(本批未读)
├── EffectDataAsset/                        ← DataAsset 家族(18 个 UEffectModelXxx,Batch 2)
├── EffectParameters/                       ← 参数结构体(Batch 2/3)
└── EffectSpec/                             ← 类型化 Spec(12+ 个 .hpp,Batch 3)
```

## 关键架构特征(从入口 + Handle 读到的)

- **Instance-based**:`FKuroEffectSystem` 是普通类,每个 World 一个实例;不是 static module。参见 [[entities/project-game/kuro-effect-system/kuro-effect-system]]。
- **裸指针所有权**:`TMap<int HandleId, FEffectHandle*>` 存句柄;`FEffectSpecBase*`、`FEffectHandleInfo*` 也都是裸指针。
- **类型化 Register**:`RegisterEffectHandle` 有 7 个重载,每种主要 DataAsset 类型(Base/Group/Ghost/PostProcess/StaticMesh/Trail)有专属签名——组件参数在编译期就确定。参见 [[entities/project-game/kuro-effect-system/effect-handle]]。
- **直接 v8 耦合**:`Tick(v8::Isolate*, ...)`、`SeekTo(v8::Isolate*, ...)`、`Register*JsFunction` 都直接接受 `v8::Isolate*`;Handle 持 `v8::Global<v8::Object>` 和 5 个 `v8::Global<v8::Function>`(见 [[entities/project-game/kuro-effect-system/effect-handle]] 的"JS 伴生"一节)。
- **无命名空间**:类名平铺,`FEffectHandle`、`FEffectSpecBase` 等占全局名字空间——和 New 系统同名类冲突,必须靠目录隔离。
- **浅层 Wrapper 风格**:入口类里大量 `FORCEINLINE` 方法,体内只是 `ProxyTickHandleMap[Id]->DoXxx()`,把调用直接转发给 Handle。

## 关键子系统(待 Batch 2-3 深入)

| 子系统 | 位置 | 负责 |
|---|---|---|
| **DataAsset 家族** | `EffectDataAsset/` | 18 个 `UEffectModelXxx`,美术/策划配置的特效蓝图数据 |
| **Parameters** | `EffectParameters/` | `FKuroEffectParameters` / `FKuroEffectMaterialParameters` / `FKuroEffectNiagaraParameters` — 运行时参数包 |
| **Spec 家族** | `EffectSpec/` | `FEffectSpecBase` + 12 个类型化子类 `FEffectModelXxxSpec` — 运行时状态机 |
| **Visibility 剔除** | `KuroEffectVisibilityOptimizeController.h` | 远距/不可见特效的 CPU Tick 降频 |
| **MultiEffect** | 散落在 Handle/Spec | 一个句柄可以持有多个子特效(见 `AddMultiEffect`/`RemoveMultiEffect`) |
| **Ghost** | Ghost 相关 DA + Spec | 骨骼残影效果(Ghost 特有的生命周期管控) |

## 涉及实体(本批已建页)

- [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]] — 入口类
- [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] — 句柄类

其余顶层类(`FEffectHandleInfo`、`FEffectLifeTime`、`KuroEffectLibrary`、`KuroEffectSystemInterafece`、`KuroEffectVisibilityOptimizeController`)**本批已扫过**,但独立实体页等到 Batch 2-3 再建,避免骨架过早膨胀。

## 与 New 系统的关系

见孪生页 [[sources/project-game/kuro-effect-system-new/overview|KuroEffectSystemNew overview]]。最顶层的 delta:

| 维度 | Old | New |
|---|---|---|
| 模式 | instance | 全 `static` + `KuroEffect::` 命名空间 |
| 所有权 | 裸指针 | `TSharedPtr<FEffectHandle>` + `TLru<FName, FEffectHandle>` 池 |
| JS 耦合 | 方法签名里埋 `v8::Isolate*` | 完全抽到 `ScriptBridge/` 一层 |
| Spec | `FEffectSpecBase*` struct + 继承 | `IEffectSpecBase` + `FEffectSpecFactory`(interface + factory) |
| 生命周期 | 2 段(Play/Stop) | 4 段(BeforeInit → Init → BeforePlay → Finish) |
| 数据 | 直接持 `UEffectModelBase*` | `FEffectSpecData` + `FEffectInitModel`(数据与运行时分离) |
| 新增能力 | — | `FPlayerEffectContainer`、`FContinuousEffectController`、`AEffectSystemActor`、LRU、PIE 钩子 |

完整对比延后到 **Batch 7 synthesis**(读完全部细节后才写,避免推测)。

## 开放问题 / 待读

- `KuroEffectLibrary.h` 作用(BP 导出 API 还是 C++ util?)
- `KuroEffectSystemInterafece.h` — 它是纯接口抽象还是别的?拼写疑似笔误(应为 Interface)。
- 13 个 Private/ 文件包括哪些(Batch 6 精读)。
- `FKuroEffectSystem::OverriderTick(HandleId, FName GroupTag, EffectToken Token)` 这个"Override Tick"是何时触发?单字未读,疑似 MultiEffect 的定向 tick 机制。
