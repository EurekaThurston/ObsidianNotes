---
type: synthesis
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, learning-path, phase-9, reader, world-manager, scalability]
sources: 6
aliases: [Phase 9 读本, Niagara 世界管理读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 9 读本 — Niagara 的世界管理与可扩展性

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 9 的**主题读本**。一次读完掌握 Niagara 的 world 级架构 + scalability 决策 + pool + platform 分支。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 9 要回答的问题

> [!question] Phase 9 要回答
> - Niagara 的"全局状态"住在哪?
> - Scalability(距离 cull / 可见性 cull / 实例数限制 / significance 排序 cull)是怎么工作的?
> - Component Pool 的 5 种方法具体行为?
> - 跨平台(PC/PS4/Mobile)+ 多质量等级 × CVar 条件的配置怎么组织?

6 个文件(1365 行):

| # | 文件 | 行 | 角色 |
|---|---|---|---|
| 9.1 | `NiagaraWorldManager.h` | 286 | **核心** — 每 World 的中央管理器 |
| 9.2 | `NiagaraScalabilityManager.h` | 140 | 每 EffectType 的 cull 管理器 |
| 9.3 | `NiagaraComponentPool.h` | 149 | 池化 |
| 9.4 | `NiagaraSettings.h` | 75 | 项目设置 |
| 9.5 | `NiagaraEffectType.h` | 337 | **核心** — 特效分类 + scalability 预算 |
| 9.6 | `NiagaraPlatformSet.h` | 378 | 跨平台 × quality 决策 |

---

## 1. 全局状态的层次

```
UNiagaraSettings(项目级,UDeveloperSettings)
    ↓ DefaultEffectType / QualityLevels / 默认 RT/Grid format
UNiagaraEffectType(Asset 级,按分类共享预算)
    ↓ SystemScalabilitySettings / EmitterScalabilitySettings / SignificanceHandler / CullReaction
FNiagaraWorldManager(每 World 一个,runtime)
    ├─ SystemSimulations × TickGroup × Asset
    ├─ ScalabilityManagers × EffectType
    ├─ UNiagaraComponentPool
    ├─ FNDI_SkeletalMesh_GeneratedData(Phase 7 共享 skinning)
    └─ CachedPlayerViewLocations

GNiagaraViewDataManager(全局单例,RT 驻留)
    ← PostOpaqueRender 写入 scene textures
```

## 2. `FNiagaraWorldManager` — 中央管理器

### 2.1 为什么 per-World

Niagara 大量状态和 `UWorld` 生命周期绑定:打开新关卡 → old world destroy → Niagara state 清;Editor PIE multi-world → 每 world 独立状态。`FNiagaraWorldManager` 把这些统一到一个**按 World 索引**的全局 TMap。

### 2.2 继承 `FGCObject`

非 UObject,但持有大量 UObject 引用(SystemSimulations 里的 Asset、ParameterCollections、ScalabilityManagers 里的 EffectType / Components)。`AddReferencedObjects` 告诉 UE GC 保护。

### 2.3 生命周期钩子

```cpp
OnWorldInit / OnWorldCleanup / OnPreWorldFinishDestroy / OnWorldBeginTearDown / TickWorld
OnPreGarbageCollect / OnPostGarbageCollect / OnPostReachabilityAnalysis / OnPreGarbageCollectBeginDestroy
```

全部 static callback,注册到 UE 的全局 world delegate。New world → create manager。World die → delete。

### 2.4 `TickFunctions[NiagaraNumTickGroups]`

每 TickGroup(TG_PrePhysics 到 TG_LastDemotable)一个 `FNiagaraWorldManagerTickFunction`(继承 UE 的 `FTickFunction`)。每个 tick group 到了 → 对应 `TickFunction.ExecuteTick` 调 `Manager.Tick(TickGroup, ...)` → 遍历 `SystemSimulations[TG]` 逐 Simulation 触发 Tick_GameThread。

### 2.5 `SystemSimulations[NiagaraNumTickGroups]`

```cpp
TMap<UNiagaraSystem*, TSharedRef<FNiagaraSystemSimulation, ThreadSafe>> SystemSimulations[NiagaraNumTickGroups];
```

Phase 3 `FNiagaraSystemSimulation` 的身份 = (Asset × World × TickGroup)——这里就是实体化。`GetSystemSimulation(TickGroup, Asset)` 不存在就 create。

### 2.6 `DeferredDeletionQueue`

`TArray<TUniquePtr<FNiagaraSystemInstance>>`。Complete 的 instance 先进队列,`PostActorTick` 统一释放——避免 tick 中途的指针失效。

### 2.7 `CachedPlayerViewLocations`(最多 8)

每 tick 开头从 PlayerController 收集,给 scalability 的距离 cull 用。`TInlineAllocator<8>` 够覆盖多玩家(split screen 典型 4,保留余量)。

---

## 3. Scalability — Niagara 的 cull 决策引擎

### 3.1 四层模型

```
UNiagaraSettings::QualityLevels(Low/Medium/High/...)
    ↓
UNiagaraEffectType(按分类共享预算)
    ├─ SystemScalabilitySettings(FNiagaraPlatformSet 过滤)
    ├─ EmitterScalabilitySettings
    ├─ SignificanceHandler
    └─ CullReaction(Deactivate/Asleep/...)
    ↓
FNiagaraScalabilityManager(每 EffectType 一个,住 WorldManager)
    ├─ ManagedComponents
    ├─ State(每 Component 一个 FNiagaraScalabilityState)
    └─ Update flow(见下)
    ↓
UNiagaraComponent::bIsCulledByScalability → ActivateInternal(bIsScalabilityCull=true/false)
```

### 3.2 `FNiagaraSystemScalabilitySettings` 的 4 类 cull

```cpp
bCullByDistance + MaxDistance
bCullMaxInstanceCount + MaxInstances              // EffectType 级总数
bCullPerSystemMaxInstanceCount + MaxSystemInstances  // System 级总数
bCullByMaxTimeWithoutRender + MaxTimeWithoutRender  // 可见性(没被 render 多久)
```

每 flag 独立 toggle。Override 子类让 System 覆盖 EffectType 默认。

### 3.3 Significance 排序 cull

```cpp
virtual void UNiagaraSignificanceHandler::CalculateSignificance(Components, OutState);
```

两个默认实现:
- `UNiagaraSignificanceHandlerDistance`:离相机越近 significance 越高
- `UNiagaraSignificanceHandlerAge`:越新 significance 越高

自定义 handler 常见场景:"玩家相关特效更重要"。

### 3.4 Update 流程(FNiagaraScalabilityManager)

```
1. 对每 ManagedComponents[i]:
    - DistanceCull(Component, State[i])
    - VisibilityCull(Component, State[i])
    - InstanceCountCull(EffectType, System, State[i])

2. 如 SignificanceHandler 非 null:
    - CalculateSignificance → State[i].Significance
    - Sort SignificanceSortedIndices by Significance desc

3. SortedSignificanceCull(EffectType, ScalabilitySettings, ...):
    - 从低 significance 开始标 bCulled=true 直到 count <= MaxInstances

4. 对每 Component:
    - 如 State[i].bCulled 变化 → 通过 friend 访问调 Component::ActivateInternal / DeactivateInternal(bIsScalabilityCull=true)
```

**关键**:走 `bIsScalabilityCull=true` 路径,**不触发 `OnSystemFinished`** delegate——这是"系统决策"不是"用户决策"。

### 3.5 `UpdateFrequency` 控制频率

```
SpawnOnly    → 只 spawn 时检查
Low → Medium → High → Continuous
```

不同 EffectType 可独立——Combat VFX 用 Continuous,环境 VFX 用 Low。

### 3.6 `ENiagaraCullReaction`

```
Deactivate                   "Kill"
DeactivateImmediate          "Kill and Clear"
DeactivateResume             "Asleep"
DeactivateImmediateResume    "Asleep and Clear"
```

关键区别:**是否自动 reactivate**。Asleep 类会持续评估 cull 条件,通过就 resume。Kill 类一旦 cull 就永远不恢复(直到下次 spawn)。

### 3.7 `NumInstances` 原子计数

```cpp
int32 NumInstances;   // 跨 world 总数,instance count cull 基础
```

Phase 3 `FNiagaraSystemSimulation` 里缓存了 EffectType 指针(`WeakSystem` 被 GC 后仍能保 count 不归零)——就是为这个字段服务。

---

## 4. ComponentPool — 5 种方法的取舍

```
ENCPoolMethod:
├─ None                       不进池(高级/自管)
├─ AutoRelease                fire-and-forget,最简
├─ ManualRelease              用户必须 ReleaseToPool(否则泄露)
├─ ManualRelease_OnComplete   手动释放但等 complete 才真回池
└─ FreeInPool                 标记在池中(内部用)
```

**选型**:

| 场景 | 方法 | 理由 |
|---|---|---|
| 爆炸、命中、子弹拖尾 | `AutoRelease` | 触发频繁,无需后续管理 |
| 持续 Buff、法术特效 | `ManualRelease` | 需要配置、修改参数 |
| 玩家可能主动放弃控制 | `ManualRelease_OnComplete` | 折中,complete 才回池 |
| 环境永久特效 | `None` + `bAutoDestroy=false` | 不需要池化 |

## 4.2 单 Asset 一个 FNCPool

```cpp
UCLASS UNiagaraComponentPool {
    TMap<UNiagaraSystem*, FNCPool> WorldParticleSystemPools;
};
```

每个 Asset 独立 FNCPool:`FreeElements / InUseComponents_Auto / InUseComponents_Manual / MaxUsed`。

TODO 注释提到改 FIFO queue 避免 LIFO 集中用最后一个元素(可能导致热/冷 Component 不均)。

## 4.3 `PrimePool` 预热

```cpp
void UNiagaraWorldManager::PrimePool(UNiagaraSystem*);   // 单 System
void PrimePoolForAllWorlds(UNiagaraSystem*);              // 静态,全 world
void PrimePoolForAllSystems();                             // 本 world 所有 system
```

关卡加载时预分配,避免战斗首次 spawn 卡顿。典型触发点:`OnWorldInit` 回调后。

---

## 5. Platform Set + CVar Condition

### 5.1 三态压双 mask

`FNiagaraDeviceProfileStateEntry` 对每 quality level 有三态:

```cpp
uint32 QualityLevelMask;      // 启用 bit
uint32 SetQualityLevelMask;   // 显式设置 bit
```

- `Set & QualityMask` = **Enabled**
- `Set & ~QualityMask` = **Disabled**
- `~Set` = **Default**(按 PlatformSet 其他规则)

双 mask 用 2 bit 表达三态。

### 5.2 CVar 条件

```cpp
FNiagaraPlatformSetCVarCondition {
    FName CVarName;
    bool Value;                        // bool CVar 相等
    int32 MinInt / MaxInt;              // int range
    float MinFloat / MaxFloat;
    uint32 bUseMinInt/MaxInt/MinFloat/MaxFloat : 1;
};
```

**除了 device profile,还能按 CVar 值过滤**——"r.Shadow.MaxResolution >= 1024 才启用这个 scalability 条目"。

### 5.3 使用点

- `UNiagaraEffectType::SystemScalabilitySettings[i].Platforms`(多 entry 按 Platforms 选第一个匹配)
- `UNiagaraRendererProperties::Platforms`(Phase 6)
- `UNiagaraEmitter` 的 Platforms

### 5.4 Conflict 检测

两个 PlatformSet 对同一 (platform, quality) 都 Enabled → 冲突。Editor 用 `FNiagaraPlatformSetConflictInfo` 数据警告作者。

---

## 6. `UNiagaraSettings`

项目级配置,5 个关键字段:

| 字段 | 作用 |
|---|---|
| `DefaultEffectType` | Asset 没配 EffectType 时的 fallback |
| `QualityLevels: TArray<FText>` | 名字数组(Low/Medium/High/Epic/Cinematic...) |
| `DefaultRenderTargetFormat` | RT DI 默认(Phase 7) |
| `DefaultGridFormat` | Grid DI 默认(Phase 10) |
| `ComponentRendererWarningsPerClass` | Component Renderer 警告(Phase 3 遇到) |

---

## 7 条关键洞察

1. **`FNiagaraWorldManager` 是 Niagara 的 "世界状态中心"**。FGCObject + 全局 TMap<UWorld*, Manager*>。通过 world delegate 钩子自动创建/销毁
2. **SystemSimulations 按 TickGroup 分数组**,`[NiagaraNumTickGroups]`—— Phase 3 `FNiagaraSystemSimulation` 的 (Asset × World × TickGroup) 身份在此实体化
3. **Scalability 四层叠加**:Settings(项目)→ EffectType(分类)→ System(override)→ Platforms(匹配)
4. **Cull 与 OnSystemFinished 解耦**:`bIsScalabilityCull=true` 路径**不触发**用户 delegate,scalability 决策对用户透明
5. **Significance Handler 是扩展点**,默认 Distance/Age 之外可自定义——gameplay-driven 排序
6. **Pool 方法 5 选 1** 是性能关键:高频 spawn 不用池 = GC 压力灾难
7. **PlatformSet 三态双 mask + CVar 条件** = 极精细平台分支能力,但配置复杂,editor 冲突检测必要

---

## 自检问题(读完回答)

下面这些题需要把"4 层 scalability 叠加 + WorldManager 中心 + Pool 5 法 + PlatformSet 三态"综合起来回答。

1. **`FNiagaraWorldManager` 不做 UCLASS 的取舍**:它持有大量 UObject 引用,看上去像该用 UObject 管理。为什么设计成"非 UObject + FGCObject"?(per-World 实例化、TMap<UWorld*, Manager*> 索引、UObject GC 扫描成本——把这三点串起来)
2. **Scalability cull 不发 OnSystemFinished 的语义边界**:如果一个游戏 BP 用 `OnSystemFinished` 来"特效结束后开门",Scalability cull 会让门永远不开吗?Niagara 的"系统决策 vs 用户决策"语义如何避免这个 bug?如果你写 BP 时确实想"任何结束都触发开门",该用什么 delegate 替代?
3. **`NumInstances` 是"全局"的边界在哪**:PIE 同时打开 3 个 world,某 EffectType 三个 world 各 spawn 100 个,`MaxInstances=200` 会触发 cull 吗?——这取决于 NumInstances 是 per-EffectType-跨 world、per-EffectType-per-world、还是 per-EffectType-per-Asset。Phase 9 的设计是哪种?为什么这种选择反映了 EffectType 的"分类预算"语义?
4. **`ManualRelease` vs `ManualRelease_OnComplete` 的选错代价**:这两种都是"用户调 ReleaseToPool",但 OnComplete 版多一层"等 complete 才真回池"。如果你给"持续 30 秒的 Buff 特效"选了普通 ManualRelease,玩家中途取消 Buff 立即调 Release,会出现什么问题?反之 OnComplete 选错会怎样?
5. **PlatformSet 三态双 mask 的"必需性"**:为什么不用单 bit + "默认值反转"?——双 mask(QualityLevelMask + SetQualityLevelMask)给你什么 single bit 不能表达的能力?(提示: 默认 vs 显式 disable 在多层叠加时是不同语义)
6. **PrimePool 没做的代价**:你忘了 prime,首次战斗的 spawn 会经历什么 cost(allocation / register / RHI buffer 初始化 / shader bind)?为什么后续 spawn 不会有这些 cost?能不能在游戏代码里晚一点 prime(战斗前 1 秒)?会有什么副作用?
7. **Significance Handler 的扩展点价值**:默认 Distance/Age,自定义 handler 典型是"玩家相关特效更重要"。如果你写一个 "队友的 Buff 比敌人的 Debuff 更重要" 的 handler,需要从哪些 World/Game state 取数据?这个 handler 在 worker 线程跑还是 GT 跑?跨线程访问 GameState 安全吗?

---

## Phase 9 留下的问题

- `FNiagaraPlatformSet` 后 180 行(实际 IsActive/IsEnabled 逻辑)→ 按需 offset 读
- 动态 perf-based budgeting(TODO 注释)未实现 → 未来版本
- `FNiagaraWorldManagerTickFunction::ExecuteTick` 完整实现 → cpp
- ScalabilityManager 的 swap-pop 复杂度与并发问题 → cpp
- Pool FIFO 改造(TODO)→ 现状 LIFO

## 下一步预告

**Phase 10**(最后一个 Phase!):高级特性——SimStages 与 Grid 流体模拟。6 文件(NiagaraSimulationStageBase + NiagaraDataInterfaceRW / Grid2DCollection / Grid2DCollectionReader / Grid3DCollection / NeighborGrid3D)。Fluids/Smoke 的底层。

---

## 深入阅读

### 本议题的原子页

- 源摘要(Source × 6)
  - 全局四层:[[Wiki/Sources/Stock/NiagaraWorldManager]] / [[Wiki/Sources/Stock/NiagaraScalabilityManager]] / [[Wiki/Sources/Stock/NiagaraComponentPool]]
  - 配置家族:[[Wiki/Sources/Stock/NiagaraSettings]] / [[Wiki/Sources/Stock/NiagaraEffectType]] / [[Wiki/Sources/Stock/NiagaraPlatformSet]]
- 实体(Entity × 6)
  - [[Wiki/Entities/Stock/FNiagaraWorldManager]]
  - [[Wiki/Entities/Stock/FNiagaraScalabilityManager]]
  - [[Wiki/Entities/Stock/UNiagaraComponentPool]]
  - [[Wiki/Entities/Stock/UNiagaraSettings]]
  - [[Wiki/Entities/Stock/UNiagaraEffectType]]
  - [[Wiki/Entities/Stock/FNiagaraPlatformSet]]

### 前置议题

- [[Readers/Niagara/Phase2-component-layer-读本]] — Component 的 `PoolingMethod` + Scalability 接线 + AutoDestroy 三方决策
- [[Readers/Niagara/Phase3-runtime-instance-读本]] — `FNiagaraSystemSimulation` 的 (Asset × World × TickGroup) 身份由 WorldManager 作 key
- [[Readers/Niagara/Phase7-data-interface-读本]] — SkeletalMesh DI 共享数据由 WorldManager 持有

### 相关概念

- [[Wiki/Concepts/UE4/UE4-资产与实例]] — ComponentPool 正是通过复用"Instance"绕过 GC 的典型

### 下一步 / 导航

- 选修终点:[[Readers/Niagara/Phase10-advanced-features-读本]] — SimStages + Grid 流体模拟
- 学习路径总图:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 仓综合视图:[[Wiki/Overview]]

---

*本读本由 [[Claudian]] 基于 Phase 9 的 6 个头文件(合计 1365 行)综合生成,2026-04-20。commit `b6ab0dee9`。*
