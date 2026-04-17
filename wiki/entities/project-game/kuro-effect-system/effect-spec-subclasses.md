---
type: entity
category: catalog
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, spec, catalog]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectSpec
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
aliases: [Old Spec Subclasses]
---

# Old Spec 子类目录

> [[entities/project-game/kuro-effect-system/effect-spec-base|FEffectSpec<T>]] 的 12 个具体子类,每个对应一种 [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelXxx]] DataAsset。

## 总览

Old 的 `EffectSpec/` 目录有 13 个 .hpp 文件:`EffectSpec.hpp`(模板基类)+ 12 个 `EffectModelXxxSpec.hpp`。**比 DataAsset 少 6 个**——Audio/CurveTrailDecal/MaterialController/NDC/SequencePose 等 DA 在 Old **没有对应 Spec**,说明 Old 根本不支持这些类型(都是 New 新增的)。

## 子类列表

| Spec 文件 | DA 类型 | 核心职责 |
|---|---|---|
| `EffectModelNiagaraSpec.hpp` | `UEffectModelNiagara` | 持 `UNiagaraComponent`,Tick 下发参数曲线、暂停/启动控制 |
| `EffectModelGroupSpec.hpp` | `UEffectModelGroup` | 管 `TArray<FEffectSpecBase*> Children` 裸指针,propagate 所有操作 |
| `EffectModelMultiEffectSpec.hpp` | `UEffectModelMultiEffect` | 用 `FMultiEffectBase` 策略 + TS 回调调整子特效数量 |
| `EffectModelStaticMeshSpec.hpp` | `UEffectModelStaticMesh` | 持 `UStaticMeshComponent`,下发 Material 动态参数、可见性 |
| `EffectModelSkeletalMeshSpec.hpp` | `UEffectModelSkeletalMesh` | 持 `USkeletalMeshComponent`,播动画、换材质 |
| `EffectModelDecalSpec.hpp` | `UEffectModelDecal` | 持 `UDecalComponent`,下发材质参数 |
| `EffectModelBillboardSpec.hpp` | `UEffectModelBillboard` | 持 `UKuroBillboardComponent`,每帧更新相机朝向 |
| `EffectModelGhostSpec.hpp` | `UEffectModelGhost` | 骨骼残影生成 + alpha 衰减 |
| `EffectModelLightSpec.hpp` | `UEffectModelLight` | 持 `UPointLightComponent` / spot,下发曲线强度/颜色/半径 |
| `EffectModelPostProcessSpec.hpp` | `UEffectModelPostProcess` | 下发 700+ 字段到 `UKuroPostProcessComponent` |
| `EffectModelTrailSpec.hpp` | `UEffectModelTrail` | 贝塞尔拖尾更新(绑骨,unit length,消散速度) |
| `EffectModelGpuParticleSpec.hpp` | `UEffectModelGpuParticle` | Kuro 自研 GPU 粒子 loop/pingpong 控制 |

**Old 没有独立 Spec 的 DA**:
- `UEffectModelAudio` — 音效
- `UEffectModelCurveTrailDecal` — 曲线拖尾贴花
- `UEffectModelMaterialController` — 材质控制器
- `UEffectModelNDC` — Niagara Data Channel
- `UEffectModelSequencePose` — 残影序列
- `UEffectModelMaterialController` 的 Group 版

这些类型在 **New 里才有对应 Spec**(见 [[entities/project-game/kuro-effect-system-new/effect-spec-subclasses|New Spec 子类目录]])——是 New 在 Old 基础上**扩展支持**的特效种类。

## 公共约定(所有 Spec 都遵守)

### 1. 构造函数签名不一致(类型专用)
每个 Spec 的 ctor 参数和它对应的 DA 类型匹配:
```cpp
// Niagara 需要传 NiagaraComponent
FEffectModelNiagaraSpec(UEffectModelNiagara*, FEffectHandleInfo*, UNiagaraComponent*);

// Group 需要传 SceneComponent + child HandleId[]
FEffectModelGroupSpec(UEffectModelGroup*, FEffectHandleInfo*, USceneComponent*, TArray<int> Children);

// MultiEffect 只需基础参数
FEffectModelMultiEffectSpec(UEffectModelMultiEffect*, FEffectHandleInfo*);
```

—— 这和 [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem::RegisterEffectHandle]] 的 7 个重载直接对应!每个重载内部 `new` 对应的 Spec,传入对应签名。

### 2. 几乎都走 `OnTick` 钩子
大多数 Spec 只重写 `OnTick(Isolate, DeltaTime)` 做自己的工作。`FEffectModelGroupSpec` 是例外,它直接重写 `Tick()` 以便在父类 Tick 之后还能 tick 子特效。

### 3. `UKuroEffectLibrary` 做 UE API 的薄 wrapper
频繁调用 `UKuroEffectLibrary::UpdateEffectModelNiagaraSpec(...)` / `SetNiagaraFrameDeltaTime(...)` 等——这是一个"UE API 的 Kuro 封装层",把底层 API 包装成 Kuro 期望的语义(比如处理特殊的可视性剔除状态)。

### 4. 访问 `FKuroEffectSystemInterface` 做系统级查询
如 Group::FindEffectSpecBase(Child)、MultiEffect::FindHandleEffectActor(Id)——这是一个**全局静态门面**,暴露"根据 Id 查 Spec 裸指针"等危险接口(用完不能持有)。

## 两个"非标准"子类的特殊处理

### `FEffectModelGroupSpec` 的递归 Tick
```cpp
virtual void Tick(v8::Isolate* Isolate, float DeltaSeconds) override  // 注意是 Tick 不是 OnTick!
{
    FEffectSpec::Tick(Isolate, DeltaSeconds);  // 先走父类流程
    for (FEffectSpecBase* Child : Children)    // 再 tick 每个 child
    {
        if (有"Stop/Destroy/Clear"标志 || 未 Start) continue;
        Child->Tick(Isolate, DeltaSeconds);
    }
}
```

—— **GroupSpec.Tick 直接递归 child.Tick**。child 如果也是 Group,再递归下去。大型复杂特效可能有多层嵌套,全部同步 tick(不是并行)。

### `FEffectModelMultiEffectSpec` 的策略 + 数量调节
把"**数量控制**"(通过 TS 回调)和"**摆放计算**"(通过策略对象)分开:
```
  OnTick:
    DesiredNum = MultiEffect->GetDesireNum(PassTime)   # 应有多少子特效
    if (当前数量 != 期望) MultiEffectAdjustNumber(Isolate, DesiredNum)  # 通知 TS 加/减
    MultiEffect->Update(DeltaTime, PassTime, Handles)   # 摆放现有 handles
    SyncHiddenState(Handles)  # 同步父的 hidden 到所有子
```

—— TS 侧是"子特效控制器",C++ 侧只做位置更新。这种分工到 New 变了(New 的 MultiEffectSpec 直接通过 `FEffectSystem::SpawnEffect/StopEffectById` 自己管子特效)。

## 类型专用字段的多样性示例

| Spec | 专属字段 |
|---|---|
| `FEffectModelNiagaraSpec` | TWeakObjectPtr\<UNiagaraComponent\>, ExtraState, LogicIsPaused, RequestForceUpdate |
| `FEffectModelGroupSpec` | TArray\<FEffectSpecBase*\> Children, TWeakObjectPtr\<USceneComponent\> GroupComponent, HasTransformAnim |
| `FEffectModelMultiEffectSpec` | TArray\<int\> Handles, FMultiEffectBase* MultiEffect(策略), LastHiddenInGame |

## 共性:没有 Play/Stop/Init/Destroy API

Old 的 Spec **只有 Tick、SeekTo、PostTick**——没有 Play/Stop/Clear/Destroy 的显式 API。生命周期状态完全靠 `EffectFlag` 位掩码隐式表达:
- `EEffectFlag_Init` 被 RegisterEffectHandle 设置(暗含"Spec 已构造")
- `EEffectFlag_Play` 被业务 Setter 设置
- 无显式 "OnPlay" 钩子——所有初始化放在 ctor

**New 的 Spec 反而显式化了**:加了 `Init / Start / End / Clear / Destroy / Replay / Play / PreStop / Stop` 钩子,让 Spec 成为完整状态机。这是 New 扩展性提升的核心之一。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystem\EffectSpec\*.hpp`(13 文件)
- 基类:[[entities/project-game/kuro-effect-system/effect-spec-base|FEffectSpecBase + FEffectSpec<T>]]
- 使用者:[[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem::RegisterEffectHandle]] 7 个重载

## 开放问题

- Old 有没有为"业务临时扩展新 Spec 类型"留口子?(看起来没有——加类型要改 System + Handle + Spec 文件夹三处)
- Niagara 的 `RequestForceUpdate` 何时 reset 回 false?(每次 UpdateParameter 后自动?)
- Group 里 children 的**初始化顺序**是否重要?Parent Spec 构造时用 `FindEffectSpecBase` 去找已经注册过的 child——如果 child 还没注册就创建 Group,是否无法找到?
- 没独立 Spec 的 DA(Audio/NDC/SequencePose/MaterialController/CurveTrailDecal)——Old 里走什么路径播放?是不是**这些 DA 在 Old 压根没实际用过**,只是提前声明占位?
