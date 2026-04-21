---
type: entity
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, class, world-manager, runtime, gc-object]
sources: 1
aliases: [FNiagaraWorldManager, FNiagaraViewDataMgr]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraWorldManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# FNiagaraWorldManager

> **每 `UWorld` 一个**的 Niagara 中央管理器。Niagara 绝大多数 world 级全局状态集中在这。继承 `FGCObject`(非 UObject 但持 UObject 引用)。

## 一句话角色

`class FNiagaraWorldManager : public FGCObject`。按 (TickGroup × Asset) 持 `FNiagaraSystemSimulation`,按 EffectType 持 `FNiagaraScalabilityManager`,持 `UNiagaraComponentPool`、`FNDI_SkeletalMesh_GeneratedData`、`CachedPlayerViewLocations`。

## 核心字段

| 字段 | 作用 |
|---|---|
| `static TMap<UWorld*, FNiagaraWorldManager*> WorldManagers` | 全局 world→manager |
| `TickFunctions[NiagaraNumTickGroups]` | 每 TickGroup 一个 tick function |
| `SystemSimulations[NiagaraNumTickGroups]` | `TMap<Asset, SharedRef<Simulation>>` × TickGroups |
| `ScalabilityManagers` | `TMap<EffectType, FNiagaraScalabilityManager>` |
| `ComponentPool` | `UNiagaraComponentPool*` |
| `SkeletalMeshGeneratedData` | Phase 7 共享 skinning |
| `CachedPlayerViewLocations` | 至多 8 视点,scalability 用 |
| `DeferredDeletionQueue` | `TArray<TUniquePtr<FNiagaraSystemInstance>>` |

## 关键 API

```cpp
static FNiagaraWorldManager* Get(const UWorld*);
TSharedRef<FNiagaraSystemSimulation> GetSystemSimulation(ETickingGroup, UNiagaraSystem*);
UNiagaraComponentPool* GetComponentPool();
bool ShouldPreCull(UNiagaraSystem*, UNiagaraComponent*);
void CalculateScalabilityState(...);
void UpdateScalabilityManagers(bool bNewSpawnsOnly);
void RegisterWithScalabilityManager(UNiagaraComponent*);
void UnregisterWithScalabilityManager(UNiagaraComponent*);
static void PrimePoolForAllWorlds(UNiagaraSystem*);
```

## World Lifecycle 钩子

`OnWorldInit / OnWorldCleanup / OnPreWorldFinishDestroy / OnWorldBeginTearDown / TickWorld / PreGC / PostReachabilityAnalysis / PostGC / PreGCBeginDestroy` —— 注册到 UE 全局 world delegates。

## `FNiagaraViewDataMgr`(全局 RT 资源)

```cpp
class FNiagaraViewDataMgr : public FRenderResource {
    FRHITexture2D* SceneDepthTexture / SceneNormalTexture / SceneVelocityTexture;
    FRHIUniformBuffer* ViewUniformBuffer;
    TUniformBufferRef<FSceneTextureUniformParameters> SceneTexturesUniformParams;
};
extern TGlobalResource<FNiagaraViewDataMgr> GNiagaraViewDataManager;
```

全局单例,RT 驻留,`PostOpaqueRender` 回调填数据。Camera / CollisionQuery DI 从这读。

## 相关

- [[Wiki/Entities/Stock/FNiagaraSystemSimulation]] — 持有
- [[Wiki/Entities/Stock/FNiagaraScalabilityManager]] — 持有
- [[Wiki/Entities/Stock/UNiagaraComponentPool]] — 持有
- [[Wiki/Entities/Stock/UNiagaraEffectType]] — key

## 深入阅读

- 源:[[Wiki/Sources/Stock/NiagaraWorldManager]]
- 读本:[[Readers/Niagara/Phase 9 - Niagara 的世界管理与可扩展性]]
