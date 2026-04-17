---
type: entity
category: catalog
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, spec, catalog]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSpec
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
twin: [[entities/project-game/kuro-effect-system/effect-spec-subclasses]]
aliases: [New Spec Subclasses]
---

# New Spec 子类目录

> [[entities/project-game/kuro-effect-system-new/effect-spec-base|KuroEffect::FEffectSpec<T>]] 的 17 个具体子类。比 Old 多 5 个(Audio/CurveTrailDecal/MaterialController/NDC/SequencePose)——这些是 New 扩展支持的特效类型。

## 总览

New 的 `EffectSpec/` 目录有 **20 个文件**:
- `EffectSpec.hpp`(模板基类)
- `IEffectSpecBase.h`(接口)
- `EffectSpecFactory.h`(工厂)
- **17 个 `EffectModelXxxSpec.hpp`**(具体 Spec 类型)

## 子类列表

| Spec 文件 | DA 类型 | 新增? | 核心职责 |
|---|---|---|---|
| `EffectModelNiagaraSpec.hpp` | `UEffectModelNiagara` | 重写 | 持 `TSharedPtr<FNiagaraComponentHandle>`,包装 `UNiagaraComponent`;PendingPause 机制;Body/Skeletal/Spline 集成 |
| `EffectModelGroupSpec.hpp` | `UEffectModelGroup` | 重写 | 递归 SpawnChildEffect;DelayPlay 列表;完整 Spec 接口向子特效 propagate |
| `EffectModelMultiEffectSpec.hpp` | `UEffectModelMultiEffect` | 重写 | 自管子特效(直接 SpawnEffect/StopEffectById),无需 TS 回调 |
| `EffectModelStaticMeshSpec.hpp` | `UEffectModelStaticMesh` | 重写 | Material 参数下发,ScreenSize 剔除集成 |
| `EffectModelSkeletalMeshSpec.hpp` | `UEffectModelSkeletalMesh` | 重写 | SkeletalMesh + 动画 + 材质替换 |
| `EffectModelDecalSpec.hpp` | `UEffectModelDecal` | 重写 | 贴花材质 + Z-fading + 强制渲染模式 |
| `EffectModelBillboardSpec.hpp` | `UEffectModelBillboard` | 重写 | 朝向跟踪 + 距离缩放 |
| `EffectModelGhostSpec.hpp` | `UEffectModelGhost` | 重写 | 骨骼残影生成 + Alpha 衰减;用 `FEffectRuntimeGhostEffectContext` |
| `EffectModelLightSpec.hpp` | `UEffectModelLight` | 重写 | 点光/聚光灯 + 卡通专用参数;融合 `UKuroCustomLight` |
| `EffectModelPostProcessSpec.hpp` | `UEffectModelPostProcess` | 重写 | 700+ 字段下发到后处理体积 |
| `EffectModelTrailSpec.hpp` | `UEffectModelTrail` | 重写 | 贝塞尔拖尾 + 骨骼绑定 |
| `EffectModelGpuParticleSpec.hpp` | `UEffectModelGpuParticle` | 重写 | GPU 粒子 Loop/PingPong/ReversePlay |
| **`EffectModelAudioSpec.hpp`** | `UEffectModelAudio` | **新!** | Wwise `UAkAudioEvent` 播放;P1 Bus 路由用 `FEffectAudioContext` |
| **`EffectModelCurveTrailDecalSpec.hpp`** | `UEffectModelCurveTrailDecal` | **新!** | 曲线轨迹贴花(车辙、拖印) |
| **`EffectModelMaterialControllerSpec.hpp`** | `UEffectModelMaterialController` | **新!** | 独立 MaterialController 资产的驱动 |
| **`EffectModelNDCSpec.hpp`** | `UEffectModelNDC` | **新!** | Niagara Data Channel(非常规,不支持暂停/显隐) |
| **`EffectModelSequencePoseSpec.hpp`** | `UEffectModelSequencePose` | **新!** | Pose 历史渲染(挥刀拖尾的硬件实现) |

## 几个关键子类详解

### `FEffectModelNiagaraSpec`(New)— 最重要的子类

`~986 行`。用到了 **New 架构的全套新概念**:

```cpp
class FEffectModelNiagaraSpec : public FEffectSpec<UEffectModelNiagara>
{
    TWeakObjectPtr<UNiagaraComponent> NiagaraComponent;
    TSharedPtr<FNiagaraComponentHandle> NiagaraComponentHandle;   // ← 新增门面
    
    // WaitPostTick 机制:所有 SetPaused 推迟到帧末执行
    bool WaitPostTick = false;
    bool IsPendingPause = false;
    bool HasSetPendingPause = false;
    bool IsForceSetPaused = false;
    bool CacheFirstPaused = false;
    
    // 各种 Niagara 操作状态
    bool IsResetImmediate = false;
    bool IsDeactivate = false;
    bool IsDeactivateImmediate = false;
    bool IsTickWhenPauesd = false;       // 原注释保留 typo "Pauesd"
    bool ComponentTickOriginalState = false;
    
    /* ... 详见后文 ... */
};
```

**文件顶部 CVars**:
```cpp
"r.Kuro.EffectSystemNiagaraUseLocalTime"         = true  // 用 NiagaraLocalTime
"Kuro.CppEffectSystem.GetNiagaraComponentWithoutHandle" = true  // 不需要 Handle 直接获取 Component
```

#### OnInit:创建 NiagaraComponent
1. 查 Context,若 `PlayFlag & NoNiagara` 直接返回 Success(特效被要求无 Niagara)
2. 从 DA 取 `NiagaraRef` → `NiagaraSystem` 弱引用
3. 找 Parent SceneComponent(若有父特效)或用 EffectActor 的 RootComponent
4. `UKuroEffectLibrary::AddSceneComponentWithTransform(EffectActor, UNiagaraComponent::StaticClass(), Parent, ...)` 动态创建组件
5. 配置组件:EffectType、ReceiveDecal、TranslucencySortPriority、CastShadow、LocalTime 等
6. `FinishAddComponent` 激活组件

#### OnPlay:重置 Niagara 状态
```cpp
NiagaraComponent->ResetOverrideParametersAndActivate();  // 清旧参数,激活
// 设置 NiagaraEffectType 枚举:1=Fight, 2=Scene, 3=UI
switch (GetEffectType()) {
  case Fight:     NiagaraComponent->NiagaraEffectType = 1; break;
  case UiScene3D:
  case UI:        NiagaraComponent->NiagaraEffectType = 3; break;
  case Scene:     NiagaraComponent->NiagaraEffectType = 2; break;
}
```

#### OnTick:更新 + 下发参数
```cpp
if (NiagaraComponent invalid || VisibleCull) return;
if (!IgnoreTimeScale || HasAnyAdditionTimeScale()) UpdateNiagara(DeltaTime);
UpdateParameter(false);  // 下发 DA 里的 Parameters 曲线
```

#### **WaitPostTick / SetPaused 机制**(最精巧的点!)

Niagara 的 Pause 调用有个问题:**在 Tick 链中途 Pause 可能导致本帧 tick 不一致**。解决办法是**延后到帧末 PostTick 再执行**:

```cpp
void SetPaused(bool IsPaused, bool IsForce = false)
{
    if (IsPaused != IsPendingPause || !HasSetPendingPause) {
        HasSetPendingPause = true;
        IsPendingPause = IsPaused;
        IsForceSetPaused = IsForce;
        WaitPostTick = true;      // ← 标记等 PostTick
    }
}

virtual bool NeedPostTick() override
{
    return WaitPostTick || FEffectSpec::NeedPostTick();
}

virtual void OnPostTick(float DeltaTime) override
{
    WaitPostTick = false;
    if (IsDeactivateImmediate) { 
        NiagaraComponent->DeactivateImmediate();
        return;
    }
    if (IsResetImmediate) { ResetNiagaraComponent(); ... }
    if (IsDeactivate) return;
    if (!HasSetPendingPause) return;
    
    if (CacheFirstPaused) {  // 首帧特殊处理
        CacheFirstPaused = false;
        FEffectSystem::SetNiagaraComponentPaused(NiagaraComponent, IsPendingPause);
    } else {
        if (NiagaraComponent->IsPaused() != IsPendingPause || IsForceSetPaused) {
            FEffectSystem::SetNiagaraComponentPaused(NiagaraComponent, IsPendingPause);
        }
    }
    IsPendingPause = NiagaraComponent->IsPaused();
    IsForceSetPaused = false;
}
```

—— 中间任何代码调 `SetPaused(true)`,都只是**记录意图**,真正执行在 `OnPostTick`。

### `FEffectModelGroupSpec`(New)— 递归 Spawn 设计

完全重写了 Old 的"子 Spec 裸指针列表"模式。**New 用 `TMap<int32, TWeakPtr<FEffectHandle>> EffectSpecMap`**:

```cpp
virtual EEffectSpecInitState OnInit(const TSharedPtr<FEffectInitModel>& EffectInitModel) override
{
    /* 创建 GroupComponent */
    
    WaitInitChildCount = EffectModel->EffectData.Num();
    if (WaitInitChildCount == 0) return Success;
    
    WaitInitModel = EffectInitModel;
    
    // 对每个 child DA,调 SpawnChildEffect 递归走完整 Spawn 管线
    ChildEffectInitCallback = MakeShared<FEffectInitCallback>();
    ChildEffectInitCallback->BindRaw(this, &FEffectModelGroupSpec::OnSpawnChildEffectFinish);
    
    for (auto& KeyValue : EffectModel->EffectData) {
        FName EffectPath(KeyValue.Key->GetPathName());
        TSharedPtr<FEffectHandle> EffectHandle = FEffectSystem::SpawnChildEffect(
            EffectActor, Handle, EffectActor, EffectPath, Handle->CreateReason,
            ChildEffectInitCallback, false);
        
        if (EffectHandle.IsValid() && IsEffectValid(EffectHandle->Id)) {
            EffectSpecMap.Add(EffectHandle->Id, EffectHandle);
        }
    }
    
    return EEffectSpecInitState_Initializing;   // 等所有 child 完成才真正成功
}

void OnSpawnChildEffectFinish(ELoadEffectResult Result, int32 EffectId)
{
    WaitInitChildCount--;
    switch (Result) {
        case Fail:   HasInitFail = true; report fail; return;
        case Success: {
            // 从 DA.EffectData[ChildModel] 读 DelayTime(!),>0 则加到 DelayPlayList
            float DelayTime = EffectModel->EffectData[ChildModel];
            if (DelayTime > 0) {
                DelayPlayList.Add(FEffectDelayPlayInfo(EffectId, DelayTime));
            }
            break;
        }
        /* Stop/EffectActorError/CanNotPlay/Cancel:remove from map */
    }
    
    if (WaitInitChildCount == 0) {
        WaitInitModel->DoHandleInitCallback(Handle, ELoadEffectResult_Success, ...);
    }
}
```

**发现!**:[[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelGroup]]`::EffectData` 的 `TMap<UEffectModelBase*, float>` 里,`float` 值就是 **DelayTime**(子特效延迟播放的秒数)。Batch 2 的"待 Batch 3"开放问题在这里答了。

### `FEffectModelMultiEffectSpec`(New)— 自管子特效

和 Old 不同:**不再通过 TS 回调加减子特效,直接用 `FEffectSystem::SpawnEffect/StopEffectById`**:

```cpp
void AdjustNumber(AActor* EffectActor, int32 DesiredNum)
{
    int32 CurrentNum = MultiEffectArray.Num();
    if (CurrentNum < DesiredNum) {
        // 加一个(一次只加一个,性能考虑)
        FName EffectPath = EffectModel->EffectData->GetPathName();
        TUniquePtr<FEffectContext> NewContext;
        FEffectContextHelper::CreateUniqueContext(Handle->GetContext(), NewContext);  // ← 复制父 Context
        int32 EffectId = FEffectSystem::SpawnEffect(
            EffectActor->GetOuter(), EffectActor->D_GetTransform(), EffectPath,
            ChildAdjustReason, nullptr, nullptr, nullptr, NewContext);
        
        MultiEffectArray.Add(EffectId);
        FEffectSystem::AddFinishCallback(EffectId, FinishCallback);
        FEffectSystem::AttachToActor(EffectId, EffectActor, NAME_None, EAttachmentRule::KeepWorld, false);
    }
    else if (CurrentNum > DesiredNum) {
        // 删一个(一次只删一个)
        int32 EffectId = MultiEffectArray.Last();
        MultiEffectArray.RemoveAt(...);
        FEffectSystem::StopEffectById(EffectId, ChildAdjustReason, true);
    }
}
```

**`FinishCallback`** 绑定到 `OnChildPlayFinish(EffectId)`——子特效结束时自动从数组 remove,避免留悬空 Id。

## 对比:Old vs New 子类的架构差异

| 维度 | Old | New |
|---|---|---|
| 子类数 | 12 | 17(新增 5) |
| Ctor 参数 | 按类型专用(Niagara 传 Component,Group 传 children ids) | **统一无参 ctor**(由 Factory MakeShared) |
| Spec 初始化 | ctor 内一次性完成 | 两阶段:`Factory new` + `Init(EffectData, InitModel)` |
| 异步 Init | 不支持 | `OnInit` 返回 `Initializing` 状态让 Spec 异步完成 |
| Group 子特效 | 裸 `FEffectSpecBase*` 数组,依赖注册顺序 | `TMap<Id, TWeakPtr<FEffectHandle>>`,每个 child 走完整 Spawn 管线 |
| MultiEffect 子管理 | 调 TS `MultiEffectAdjustNumber`,TS 负责 | **Spec 自己调 `FEffectSystem::SpawnEffect`**,C++ 自管 |
| Niagara Pause | 立即执行 | `WaitPostTick` 延后到帧末 |
| 专属数据访问 | 直接裸指针 | 通过 Handle.GetContext() 取 `FEffectContext*` 再 static_cast |

## 与其他实体的关系

- **基类**:[[entities/project-game/kuro-effect-system-new/effect-spec-base|IEffectSpecBase + FEffectSpec<T>]]
- **工厂**:[[entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory]]
- **数据源**:[[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelXxx 家族]]
- **Context 依赖**:[[entities/project-game/kuro-effect-system-new/effect-context|FEffectContext 家族]](不同 Spec 读对应 Context 子类)
- **辅助**:`FNiagaraComponentHandle`(Batch 4)、`UKuroEffectLibrary`(工具库)
- **脚本桥**:`FEffectSystem::EffectSystemScriptBridge`(BodyEffect / QualityBias / EntityModelConfigId)

## Twin(Old 版对应)

- [[entities/project-game/kuro-effect-system/effect-spec-subclasses|Old Spec 子类目录]]

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectSpec\*.hpp`(20 文件)
- 具体子类:详见各 .hpp 文件(~200-1000 行/个)

## 开放问题

- `FEffectModelGhostSpec` 如何与 `FEffectRuntimeGhostEffectContext` 的 SpawnRate/Interval/GhostLifeTime 对接?(本批未读其 .hpp)
- `FEffectModelPostProcessSpec` 怎么管 700+ 字段的**哪些需要每帧下发 vs 一次性配置**?(本批未读)
- Group 的 `OnSpawnChildEffectFinish` 里如果 child 的 `EffectSpecMap.Contains(EffectId) == false` 但 Result == Success,会进 `PendingExecInitCallback` 分支——这是什么边角情况?
- `NeedTick()` 在各 Spec 子类的实现差异——哪些特效真正需要每帧 tick?(Light 类特效可能大部分帧不需要)
