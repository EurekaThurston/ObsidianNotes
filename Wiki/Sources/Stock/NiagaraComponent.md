---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, component, runtime]
sources: 1
aliases: [NiagaraComponent.h, UNiagaraComponent 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponent.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraComponent.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraComponent.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 741 行(本 Phase 最大的头文件)
- **同仓协作文件**: `NiagaraComponent.cpp`、`NiagaraSystemInstance.h`(运行时)、`NiagaraComponentPool.h`、`NiagaraUserRedirectionParameterStore.h`、`NiagaraScalabilityManager.h`、`Particles/ParticleSystemComponent.h`(父类链)
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 2 — 场景入口 · Component 层 2.1

## 职责 / 这个文件干什么的

定义 `UNiagaraComponent`——**把 `UNiagaraSystem` 资产挂到游戏世界里的"桥"**。它是整个 Phase 2 的主角:Actor / BP / C++ 三种生成路径最终都指向一个 `UNiagaraComponent` 实例。

Component 的职责可以按五类拆分,每一类对应一组字段和方法:

1. **Asset 持有者**——引用磁盘上的 `UNiagaraSystem` 资产
2. **Instance 管理者**——独占持有一个 `FNiagaraSystemInstance`(运行时对象,Phase 3)
3. **参数覆盖层**——在不改 Asset 的前提下为本实例覆盖 User 参数
4. **场景集成点**——继承 UPrimitiveComponent 链,提供 Bounds/SceneProxy/AutoAttachment
5. **生命周期调度**——Activate / Deactivate / Tick / Pool / Scalability 的接线

文件末尾还顺带定义了 `FNiagaraSceneProxy`(L650-L734),即 Component 在渲染线程上的"替身"。严格来说它属于 Phase 6 渲染层,只是物理上写在这里,本页不展开。

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraComponent : public UFXSystemComponent`(L53):Niagara 在场景里的承载者
  - `UFXSystemComponent` 是 UE 通用的特效组件基类,Cascade 的 `UParticleSystemComponent` 同样继承它——又一个与 [[Wiki/Entities/Stock/UNiagaraSystem]] `UFXSystemAsset` 对应的"历史兼容面"
  - `UCLASS(BlueprintSpawnableComponent, DisplayName = "Niagara Particle System")`——可以在 BP 的 AddComponent 菜单里直接加
  - `friend struct FNiagaraScalabilityManager`(L55):可扩展性管理器直接访问私有字段

### 配套结构体

- `FNiagaraMaterialOverride`(L33):每个渲染器的材质覆盖条目(`Material + MaterialSubIndex + EmitterRendererProperty`),存放在 `TArray<FNiagaraMaterialOverride> EmitterMaterials`(L184)。用于在 Component 级别改材质而不改 Asset
- `FOnNiagaraSystemFinished` delegate(L28):`DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam`,特效播完时广播 `UNiagaraComponent*`;BP 可绑定,也是 `ANiagaraActor::OnNiagaraSystemFinished` 的来源

### 五大职责 × 关键字段

#### 职责 1:Asset 持有者

- `UNiagaraSystem* Asset`(L82,`UPROPERTY(EditAnywhere)`):引用磁盘上的 Niagara System 资产。Component 生命全程持有
- `UFXSystemAsset* GetFXSystemAsset() const override`(L73):父类接口,返回 `Asset`——把 Niagara 统一接入 UE 通用 FX 系统

#### 职责 2:Instance 管理者

- `TUniquePtr<FNiagaraSystemInstance> SystemInstance`(L116):**核心状态**。注意是 `TUniquePtr`——独占所有权,Component 死则 Instance 死,不可共享
  - 这个字段是私有,外界通过 `GetSystemInstance()`(L307)读取,返回裸指针
- `bool InitializeSystem()`(L240):在 Activate 流程里创建 SystemInstance
- `void DestroyInstance()`(L241):销毁 SystemInstance(Deactivate / OnUnregister / BeginDestroy 时调用)
- `virtual void ActivateSystem(bool bFlagAsJustAttached = false) override`(L77):父类接口,内部会走 `ActivateInternal`
- `virtual void Activate(bool bReset = false) override` / `Deactivate() override` / `DeactivateImmediate()`(L189-L191):UActorComponent 标准生命周期的 Niagara 版
- `ActivateInternal / DeactivateInternal / DeactivateImmediateInternal`(L203-L205):私有内部版,含 `bIsScalabilityCull` 参数——scalability manager 可以走这条路径 kill/revive Component 而不触发用户语义
- `FORCEINLINE ENiagaraExecutionState GetRequestedExecutionState() / GetExecutionState()`(L193-L194):委托给 SystemInstance

#### 职责 3:参数覆盖层

- `FNiagaraUserRedirectionParameterStore OverrideParameters`(L89):本实例特有的参数覆盖存储。`UserRedirection` 表示这个 store 会自动为参数加 `User.` 前缀以便在 UI 中分组显示
  - 两个 getter:`GetOverrideParameters()`(L498 / L500)同时给出可写和只读重载
- `SynchronizeWithSourceSystem()`(L515,私有):对比 Asset 的参数列表,删除本地那些类型不匹配或已不存在的覆盖
- `CopyParametersFromAsset()`(L519,私有):从 Asset 复制参数到 OverrideParameters(初次绑定 Asset 时)
- `AssetExposedParametersChanged()`(L517,私有):Asset 参数变化的 delegate 回调;`AssetExposedParametersChangedHandle`(L630)保存句柄

**对外 API:18 个 SetVariable 函数**(L316-L397)——每种类型两个版本(`FName` vs `FString`):

| 类型 | FName 版 | FString 版(带 "By String") |
|---|---|---|
| `FLinearColor` | `SetVariableLinearColor` | `SetNiagaraVariableLinearColor` |
| `FVector4` | `SetVariableVec4` | `SetNiagaraVariableVec4` |
| `FQuat` | `SetVariableQuat` | `SetNiagaraVariableQuat` |
| `FVector` | `SetVariableVec3` | `SetNiagaraVariableVec3` |
| `FVector2D` | `SetVariableVec2` | `SetNiagaraVariableVec2` |
| `float` | `SetVariableFloat` | `SetNiagaraVariableFloat` |
| `int32` | `SetVariableInt` | `SetNiagaraVariableInt` |
| `bool` | `SetVariableBool` | `SetNiagaraVariableBool` |
| `AActor*` | `SetVariableActor` | `SetNiagaraVariableActor` |
| `UObject*` | `SetVariableObject` | `SetNiagaraVariableObject` |
| `UMaterialInterface*` | `SetVariableMaterial` | *(无 String 版)* |
| `UTextureRenderTarget*` | `SetVariableTextureRenderTarget` | *(无 String 版)* |

FString 版给 BP 图里直接用字符串拖线的场景,FName 版性能更好(免 FName 构造)。12 种类型 × 2 版 - 2(Material/TextureRenderTarget 无 String 版) = **18 个 setter**。

- 编辑器专用(L460-L469):`FindParameterOverride` / `HasParameterOverride` / `SetParameterOverride` / `RemoveParameterOverride`——支持"Template"和"Instance"两层覆盖(`TemplateParameterOverrides` L96 / `InstanceParameterOverrides` L99)
- `SetUserParametersToDefaultValues()`(L511):清除本地覆盖,恢复 Asset 默认——pool 归还时的关键动作

#### 职责 4:场景集成点(UPrimitiveComponent 接口)

- `virtual FBoxSphereBounds CalcBounds(const FTransform& LocalToWorld) const override`(L226):包围盒计算;若 Asset 有 FixedBounds 则用固定值,否则向 SystemInstance 查询
- `virtual FPrimitiveSceneProxy* CreateSceneProxy() override`(L227):创建 `FNiagaraSceneProxy`——游戏线程把渲染必需数据"快照"过去
- `virtual void CreateRenderState_Concurrent(...)` / `SendRenderDynamicData_Concurrent()`(L151-L152):渲染线程之间同步动态数据
- `virtual void GetUsedMaterials(...)` / `GetNumMaterials()`(L225, L228):材质统计,被 Editor / Profiler 用
- `virtual void OnAttachmentChanged() override`(L230):变换父节点时被调,Niagara 用来更新 LocalToWorld 缓存
- `virtual void OnChildAttached / OnChildDetached`(L234-L235):USceneComponent 接口

**Auto-Attachment 子系统**(L528-L573):
- `bAutoManageAttachment`(L166):是否自动挂/解父节点(`SpawnSystemAttached` 激活它)
- `AutoAttachParent`(L533,`TWeakObjectPtr`)、`AutoAttachSocketName`(L540)
- `AutoAttachLocationRule / RotationRule / ScaleRule`(L547-L561,各自是 `EAttachmentRule`)
- `bAutoAttachWeldSimulatedBodies`(L173)
- `SetAutoAttachmentParameters(...)`(L573):一次设五个参数
- `CancelAutoAttachment(bool bDetachFromParent)`(L622,私有):deactivate 时还原
- `SavedAutoAttachRelativeLocation/Rotation/Scale3D`(L626-L628):还原用的快照
- `bDidAutoAttach`(L604):标记是否经过自动挂载,避免在没自动挂载的情况下瞎解绑

#### 职责 5:生命周期调度

- `virtual void OnRegister() override` / `OnUnregister() override`(L148-L149):UActorComponent 标准时机——注册/反注册到世界
- `virtual void BeginDestroy() override`(L153):析构准备
- `virtual void TickComponent(float DeltaTime, ELevelTick, FActorComponentTickFunction*) override`(L218):**每帧入口**,驱动 SystemInstance tick
- `virtual void OnEndOfFrameUpdateDuringTick() override`(L150):帧末期收尾
- `virtual void OnComponentDestroyed(bool bDestroyingHierarchy) override`(L221)
- `virtual bool IsReadyForOwnerToAutoDestroy() const override`(L220):Actor 的 AutoDestroy 会问这个
- `void PostSystemTick_GameThread()`(L211,私有):每帧 tick 完回调,处理完成/池化
- `void OnSystemComplete(bool bExternalCompletion)`(L212,私有):特效播完的集中处理点
- `void OnPooledReuse(UWorld* NewWorld)`(L243):池里被复用时调,重置状态到可重用

**池化字段**:
- `ENCPoolMethod PoolingMethod`(L187):当前池化策略(`None / AutoRelease / ManualRelease / ManualRelease_OnComplete / FreeInPool` 等,见 `NiagaraComponentPool.h`)
- `bAutoDestroy`(L140):播完是否自销毁
- `virtual void ReleaseToPool() override`(L75):主动归还池子

**Scalability 字段**:
- `bAllowScalability`(L607):全局开关
- `bIsCulledByScalability`(L610):当前是否被 cull
- `int32 ScalabilityManagerHandle`(L632):在 `FNiagaraScalabilityManager` 里的索引,`INDEX_NONE` 表示未注册
- `SetAllowScalability(bool bAllow)`(L589):BP 可调
- `IsRegisteredWithScalabilityManager() / GetScalabilityManagerHandle()`(L591-L592)
- `RegisterWithScalabilityManager / UnregisterWithScalabilityManager / ShouldPreCull`(L207-L209,私有)

### DesiredAge / AgeUpdateMode 子系统(Sequencer 支持)

这一整套是为 Sequencer/Cinematic **时间轴回放** 设计的——允许把特效 seek 到任意时刻,可选"不渲染中间过程"。

- `ENiagaraAgeUpdateMode AgeUpdateMode`(L119):`TickDeltaTime`(正常)/ `DesiredAge` / `DesiredAgeNoSeek` 三种
- `float DesiredAge`(L122):目标时间
- `float LastHandledDesiredAge`(L125):NoSeek 模式用
- `bool bCanRenderWhileSeeking`(L128):是否在 seek 期间渲染
- `float SeekDelta`(L131):seek 时每步的 DeltaTime
- `float MaxSimTime`(L134):单帧最多 seek 多少秒
- `bool bIsSeeking`(L137):当前是否在 seek

BP 接口(L260-L302):`GetAgeUpdateMode / SetAgeUpdateMode / GetDesiredAge / SetDesiredAge / SeekToDesiredAge / SetCanRenderWhileSeeking / GetSeekDelta / SetSeekDelta / GetMaxSimTime / SetMaxSimTime` —— 共 10 个。

### 其他 BP 接口

- `SetAsset / GetAsset`(L246-L249)
- `SetForceSolo / GetForceSolo`(L251-L255)——⚠️ 见下
- `SetGpuComputeDebug`(L258)
- `SetAutoDestroy`(L304)
- `SetTickBehavior / GetTickBehavior`(L309-L312):控制如何选择 TickGroup
- `SetRenderingEnabled / GetRenderingEnabled`(L420-L424)
- `SetPaused / IsPaused`(L434-L438)
- `SetEmitterEnable(FName, bool) override`(L74):在 Component 级别启/禁某个 Emitter
- `GetDataInterface(const FString& Name)`(L440):按名取 DI,Phase 7 会深入
- `GetNiagaraParticlePositions_DebugOnly` / `GetNiagaraParticleValues_DebugOnly` / `GetNiagaraParticleValueVec3_DebugOnly`(L400-L409):**已全部标 `DeprecatedFunction`**,注释建议用"particle export DI"代替
- `ResetSystem / ReinitializeSystem`(L412-L417):前者保留参数重置模拟,后者从 Asset 重新初始化
- `AdvanceSimulation(int32 TickCount, float TickDeltaSeconds)` / `AdvanceSimulationByTime(float SimulateTime, float TickDeltaSeconds)`(L427-L432):手动推进模拟(测试/烘焙用)

### 开发辅助字段

- `bForceSolo : 1`(L110):**性能陷阱**——见下陷阱区
- `bEnableGpuComputeDebug : 1`(L114):GPU 模拟调试可视化开关
- `bRenderingEnabled : 1`(L143):渲染开关(独立于 Tick)
- `float MaxTimeBeforeForceUpdateTransform`(L180):动态包围盒多久强制收缩一次
- `WITH_NIAGARA_COMPONENT_PREVIEW_DATA`(L30,`!UE_BUILD_SHIPPING`):编辑器预览数据
- `PreviewLODDistance / bEnablePreviewLODDistance`(L578-L579,仅 preview build):预览 LOD 距离
- `SetPreviewLODDistance / GetPreviewLODDistanceEnabled / GetPreviewLODDistance`(L473-L480)
- `SetLODDistance`(L483):正式 LOD 距离,转发给 SystemInstance
- `bWaitForCompilationOnActivate`(L584,仅 editor):Asset 还在编译时 Activate,是否等待

### Editor-Only 参数覆盖层(双层)

`#if WITH_EDITORONLY_DATA`(L91-L104):
- `EditorOverridesValue_DEPRECATED`(L93):旧版覆盖数据,升级时用
- `TMap<FNiagaraVariableBase, FNiagaraVariant> TemplateParameterOverrides`(L96):Template 层覆盖(Instance 会继承)
- `TMap<FNiagaraVariableBase, FNiagaraVariant> InstanceParameterOverrides`(L99):Instance 层覆盖(覆盖 Template)
- `FOnSystemInstanceChanged` / `FOnSynchronizedWithAssetParameters` delegate

设计意图:在蓝图里继承一个 `ANiagaraActor` 蓝图时,父子蓝图可以分别覆盖——Template 是父,Instance 是子。

### 委托 / 事件

- `OnSystemFinished`(L508,`UPROPERTY(BlueprintAssignable)`):特效播完的 multicast delegate——BP 可绑
- `OnSystemInstanceChanged()` / `OnSynchronizedWithAssetParameters()`(L493-L495,仅 editor):editor 内部用

### `FNiagaraSceneProxy`(L650-L734)

Component 的渲染线程替身,继承 `FPrimitiveSceneProxy`。本 Phase 不展开,关键字段作为 Phase 6 前置指针:

- `TArray<FNiagaraRenderer*> EmitterRenderers`(L714):每 Emitter 一个 Renderer
- `TArray<int32> RendererDrawOrder`(L717):排序索引
- `NiagaraEmitterInstanceBatcher* Batcher`(L720):GPU 任务批处理器(Phase 8)
- 支持 RayTracing(L672-L675,`RHI_RAYTRACING` 宏)
- `GatherSimpleLights(...)`(L703):把 `FNiagaraRendererLights` 生成的粒子光源合进场景灯光系统

### 末尾全局变量

- `extern float GLastRenderTimeSafetyBias`(L736):用于 `GetSafeTimeSinceRendered` 的容差

## 依赖与被依赖

**上游(此文件 include / 使用):**
- `NiagaraCommon.h`(类型系统、枚举如 `ENiagaraExecutionState`)
- `NiagaraComponentPool.h`(`ENCPoolMethod`)
- `NiagaraSystemInstance.h`(`FNiagaraSystemInstance`——独占成员)
- `NiagaraUserRedirectionParameterStore.h`(`OverrideParameters` 类型)
- `NiagaraVariant.h`(Editor 覆盖层值类型)
- `PrimitiveSceneProxy.h` / `PrimitiveViewRelevance.h`(SceneProxy 基类)
- `Particles/ParticleSystemComponent.h`(父类链)

**下游(谁用它):**
- `ANiagaraActor`:把 `UNiagaraComponent` 作为 RootComponent 封装
- `UNiagaraFunctionLibrary`:`SpawnSystemAt*` 返回 `UNiagaraComponent*`
- `FNiagaraScalabilityManager`:管理所有 Component 的剔除/恢复(friend 访问)
- `UNiagaraComponentPool`:池化复用
- `FNiagaraSystemInstance`:反向持有 Component 指针
- `FNiagaraSceneProxy`:渲染线程替身
- 游戏代码 / BP:最终用户

## 关键代码片段

> `NiagaraComponent.h:L47-L56` @ `b6ab0dee9` — 类声明与核心定位
> ```cpp
> /**
> * UNiagaraComponent is the primitive component for a Niagara System.
> * @see ANiagaraActor
> * @see UNiagaraSystem
> */
> UCLASS(ClassGroup = (Rendering, Common), hidecategories = Object, hidecategories = Physics, ..., meta = (BlueprintSpawnableComponent, DisplayName = "Niagara Particle System"))
> class NIAGARA_API UNiagaraComponent : public UFXSystemComponent
> {
>     friend struct FNiagaraScalabilityManager;
>     GENERATED_UCLASS_BODY()
> ```

> `NiagaraComponent.h:L81-L116` @ `b6ab0dee9` — 五职责里最关键的三个字段
> ```cpp
> UPROPERTY(EditAnywhere, Category="Niagara", meta = (DisplayName = "Niagara System Asset"))
> UNiagaraSystem* Asset;                                   // 职责 1:Asset 持有
>
> UPROPERTY()
> FNiagaraUserRedirectionParameterStore OverrideParameters;// 职责 3:参数覆盖
>
> // ...(中间省略 editor-only 字段)
>
> TUniquePtr<FNiagaraSystemInstance> SystemInstance;       // 职责 2:Instance 管理(独占)
> ```

> `NiagaraComponent.h:L106-L110` @ `b6ab0dee9` — bForceSolo 的自白
> ```cpp
> /**
> When true, this component's system will be force to update via a slower "solo" path
> rather than the more optimal batched path with other instances of the same system.
> */
> UPROPERTY(EditAnywhere, Category = Parameters)
> uint32 bForceSolo : 1;
> ```

> `NiagaraComponent.h:L506-L508` @ `b6ab0dee9` — BP 可绑的"播完"事件
> ```cpp
> // Called when the particle system is done
> UPROPERTY(BlueprintAssignable)
> FOnNiagaraSystemFinished OnSystemFinished;
> ```

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/UNiagaraComponent]] — 本文件定义的主类
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Component 引用的 Asset
- [[Wiki/Entities/Stock/ANiagaraActor]] — Component 的 Actor 封装
- [[Wiki/Entities/Stock/UNiagaraFunctionLibrary]] — BP 入口
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — Component 是 Asset→Instance 转换的承载者
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — UCLASS / UPROPERTY / UFUNCTION 大量使用

## 与既有 wiki 的关系

- 延伸了 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 的三层架构图:Asset(Phase 1)与 Instance(Phase 3)之间的**承载层**,此前一笔带过
- 印证了 [[Wiki/Concepts/UE4/UE4-资产与实例]] 中 "同一 Asset 可对应多个运行时对象":一个 `UNiagaraSystem` 可被多个 `UNiagaraComponent` 同时引用,每个 Component 独占一个 `FNiagaraSystemInstance`
- 新增视角:"参数覆盖层" 概念——Component 持有一份 `OverrideParameters` 独立于 Asset 的 ExposedParameters,18 个 `SetVariableXxx` 函数共同服务这层

## 开放问题 / 待深入

> [!warning] `bForceSolo` 陷阱
> 这个字段设 true 会绕开 `FNiagaraSystemSimulation` 的批量 Tick(Phase 3 深入),Component 独自走"solo"路径——对于"一个场景里同 Asset 的 N 个实例"这种情况,性能差距可能几十倍。仅用于调试或极特殊的"我就要独立时序"场景。

- `bForceSolo` 究竟绕开了多少优化?批量 tick 共享哪些东西?→ 看 [[Wiki/Sources/Stock/NiagaraSystemSimulation]](Phase 3)
- `FNiagaraScalabilityManager` 被 friend 访问的是哪些字段?设计权衡?→ 看 [[Wiki/Sources/Stock/NiagaraScalabilityManager]](Phase 9)
- `ENCPoolMethod` 五种取值的决策表是什么?何时哪种?→ 看 [[Wiki/Sources/Stock/NiagaraComponentPool]](Phase 9)
- `ENiagaraTickBehavior::UsePrereqs`(L86 默认)的 prereq 计算逻辑?→ Phase 3 `SystemInstance`
- `AgeUpdateMode = DesiredAgeNoSeek` vs `DesiredAge` 的语义差别?`LastHandledDesiredAge` 如何参与计算?→ 留给 Sequencer 专题(本学习路径不必覆盖)
- `FNiagaraSceneProxy::CreateRenderers` 与 `UNiagaraComponent::UpdateEmitterMaterials` 的协同?→ Phase 6 渲染
- `TemplateParameterOverrides` 与 `InstanceParameterOverrides` 的两层覆盖在 Blueprint 继承里如何工作?→ 本 Phase 不展开,必要时专门开一篇 editor 专题
