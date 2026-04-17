---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, entry-point]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/KuroEffectSystem.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
twin: [[entities/project-game/kuro-effect-system-new/effect-system]]
aliases: [FKuroEffectSystem]
---

# FKuroEffectSystem(老版入口类)

> Old KuroEffectSystem 的入口类;**实例化** class,每个游戏 World 一个实例;所有特效 spawn/tick/参数下发都从这里走。

## 概览

`FKuroEffectSystem` 是老版特效系统的顶层对象。**注意它不是 `U` 前缀 UObject,也不是 `A` 前缀 Actor,是裸 C++ 类**——通过 `Init/Destroy Environment` 显式生命周期管理,把自己绑在一个 `UWorld` 上。

对外暴露的职责可以拆成几类:

1. **特效句柄注册**:`RegisterEffectHandle()` 7 个重载,每种主要 DataAsset 类型一套签名。
2. **句柄状态控制**:`SetEffectFlag` / `SetIsPlaying` / `SetIsStopping` / `SetEffectTimeScale` / `SetEffectStoppingTime` / `SetEffectIsFreeze` / `EffectSeekTo` / `SetEffectSeekToTimeWithProcessInfo` ——全是 `FORCEINLINE` 的"查 Map + 转发给 Handle"式 wrapper。
3. **JS 回调注册**:`RegisterEffectJsObject` / `RegisterEffectCommonJsFunction` / `RegisterEffectGroupPlayJsFunction` ——把 TS 侧的 `v8::Local<v8::Function>` 注册进 Handle,用于 PreStop / UpdateLifeCycle / EnterStopping / GroupDelayPlay 几个事件回调。
4. **全局状态**:`SetEffectGlobalTimeScale`、`SetEffectGlobalStoppingTime` ——通过静态成员 `FEffectLifeTime::GlobalTimeScale` / `GlobalStoppingTime` / `GlobalStoppingPlayTime` 下发。
5. **MultiEffect**:`AddMultiEffect` / `RemoveMultiEffect` / `RegisterMultiEffectAdjustNumJsFunction` ——一个父句柄下挂子特效的调节接口。
6. **Ghost / PostProcess / Niagara / Material**:各类型特效特有的分支,例如 `GetGhostEffectCanStop` / `SetNiagaraEffectExtraState` / `CollectEffectFloatCurve`。
7. **Visibility 剔除**:`IgnoreVisibilityOptimize` / `SetNiagaraComponentPaused` / `SetNiagaraComponentStarting` / `GetNiagaraComponentIsLogicPaused` ——把 Niagara 组件的可视性状态转交给 [[entities/project-game/kuro-effect-system/kuro-effect-visibility-optimize-controller|VisibilityOptimizeController]](Batch 待建)。

## 关键字段

```cpp
bool VisibilityOptimize = false;
bool TickOptimize = false;
TWeakObjectPtr<UWorld> MainWorld = nullptr;
FDelegateHandle PostLoadLevelDelegate;
FKuroEffectVisibilityOptimizeController VisibilityOptimizeController;
TMap<int, FEffectHandle*> ProxyTickHandleMap;   // ← 核心:HandleId → Handle*
FKuroTickNativeFunction* PostUpdateWorkTickPtr = nullptr;
```

**观察**:
- `TMap<int, FEffectHandle*>` 用**裸指针**;所有权并不明确在 SystemCls 这里——`FEffectHandle` 的 delete 时机得去 `.cpp` 找(Batch 6)。
- `FriendClass` 只有 `FEffectSystemDebugDraw`。
- `MainWorld` 是 `TWeakObjectPtr`,但 Map 里的 Handle 是裸 ptr——**生命周期风险**的典型姿态。

## 核心 API(按职责分组)

### 生命周期
```cpp
void InitializeEnvironment(UWorld* World, bool InVisibilityOptimize, bool InTickOptimize, ...);
void DestroyEnvironment();
```

### 注册(7 种重载)
```cpp
// 基础类型
void RegisterEffectHandle(int HandleId, int ParentId, UEffectModelBase* DataAsset, AActor* EffectActor);
void RegisterEffectHandle(..., UActorComponent* ActorComponent);

// 类型化特化
void RegisterEffectHandle(..., UEffectModelGroup*, ..., USceneComponent*, TArray<int> Children);
void RegisterEffectHandle(..., UEffectModelGhost*, ..., USkeletalMeshComponent*, float GhostLifeTime, float SpawnInterval);
void RegisterEffectHandle(..., UEffectModelPostProcess*, ..., UKuroPostProcessComponent*, ...);
void RegisterEffectHandle(..., UEffectModelStaticMesh*, ..., UStaticMeshComponent*, TArray<UMaterialInstanceDynamic*>);
void RegisterEffectHandle(..., UEffectModelTrail*, ..., UKuroBezierMeshComponent*, USkeletalMeshComponent*, UMaterialInstanceDynamic*);

void UnregisterEffectHandle(int HandleId);
void OverriderTick(int HandleId, FName GroupTag, EffectToken Token);  // 意图待 Batch 6 确认
```

### 下发(典型样式)
```cpp
FORCEINLINE void SetEffectFlag(int HandleId, int InHandleFlag) const
{
    if (ProxyTickHandleMap.Contains(HandleId))
    {
        ProxyTickHandleMap[HandleId]->SetFlag(InHandleFlag);
    }
}
```
——**几乎每个 Setter 都是这个模式**。入口类本身不存业务状态,只做"按 Id 查表 + 转发"。

## 与其他实体的关系

- **持有**:`FKuroEffectVisibilityOptimizeController`(值成员)、`TMap<int, FEffectHandle*>`、`UWorld` 弱引用。
- **依赖**:[[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]](绝大多数 API 都是向它转发)、`FKuroTickManager`(注册 `FKuroTickNativeFunction*`)。
- **不依赖**:**没有任何 static 成员**,与 static 模块化的 New 形成鲜明对比。

## JS 交互约定

入口类上带 `v8::Isolate*` 的 API:

- `SetIsStopping(v8::Isolate*, HandleId, IsPlaying)` ——转发给 `Handle::SetIsStopping(Isolate, ...)`;需要 Isolate 是因为 `IsStopping` 状态切换时会回调 TS 的 `EnterStopping` 函数。
- `RegisterEffectJsObject` / `RegisterEffectCommonJsFunction` / `RegisterEffectGroupPlayJsFunction`  / `RegisterMultiEffectAdjustNumJsFunction` ——把 TS 侧的对象/函数用 `v8::Global<>` 存到 Handle 的 `FEffectHandleInfo` 里(见 [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]])。
- `EffectSeekTo(v8::Isolate*, ...)` ——Seek 过程需要回 JS 侧通知。

这正是 New 系统要解掉的**核心债**:入口类完全不该知道 v8。

## Twin(New 版对应)

- [[entities/project-game/kuro-effect-system-new/effect-system|KuroEffect::FEffectSystem]] — New 版全 static 单例

### Delta 速记
- **实例化 → 静态单例**:实例数据变 static 成员。
- **`TMap<int, FEffectHandle*>` → `TMap<int32, TSharedPtr<FEffectHandle>>`**:裸指针 → 共享智能指针。
- **分散的 Register 重载 → 统一 Path-based Spawn**:New 用 `FName Path` 载入 DataAsset,不是编译期专用签名(详见 [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] 的 `SpawnEffect` / `SpawnUnloopedEffect`)。
- **v8 直接耦合 → ScriptBridge**:入口类几乎不再含 v8(Batch 5 验证)。

## 引用来源

- [[sources/project-game/kuro-effect-system/overview|Old 系统 overview]]
- 原始代码:`F:\Aki\dev\Source\Client\Plugins\Kuro\KuroGameplay\Source\KuroGameplay\Public\KuroEffectSystem\KuroEffectSystem.h`

## 开放问题

- `OverriderTick(HandleId, GroupTag, EffectToken)` 的触发时机?`EffectToken` 是什么?
- `FKuroTickNativeFunction* PostUpdateWorkTickPtr` —— `FKuroTickManager` 里的 tick 注册机制是什么?(外部概念,可能在 Batch 5/6 触及)
- `FEffectHandle*` 裸指针的释放时机:Map erase 时是否同时 delete?(读 .cpp 回答)
- 入口类有没有 instance accessor 的全局入口(如 `FKuroGameGlobal::GetEffectSystem()`)?本文件未体现,推测在 `UKuroGameInstance` 或 `KuroGameplayModule`(Batch 6 回答)。
