---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, world-manager, world, runtime]
sources: 1
aliases: [NiagaraWorldManager.h]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraWorldManager.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraWorldManager.h

- **Repo**: stock · **路径**: `Public/NiagaraWorldManager.h` · **规模**: 286 行
- **Ingest**: 2026-04-20 · **Phase**: 9.1

## 职责

定义 **`FNiagaraWorldManager`**——**每个 `UWorld` 一个**的 Niagara 中央管理器。持有该 world 下所有 `FNiagaraSystemSimulation`(按 TickGroup × Asset)、`UNiagaraComponentPool`、`FNDI_SkeletalMesh_GeneratedData`、per-EffectType 的 `FNiagaraScalabilityManager`、`FNiagaraViewDataMgr` view 数据。绝大多数 Niagara 全局状态都在本类。

## 主类(L97)

```cpp
class FNiagaraWorldManager : public FGCObject
{
    static FNiagaraWorldManager* Get(const UWorld*);
    static void OnStartup() / OnShutdown();
    static void DestroyAllSystemSimulations(UNiagaraSystem*);
    static void OnBatcherDestroyed(NiagaraEmitterInstanceBatcher*);

    virtual void AddReferencedObjects(FReferenceCollector&) override;

    void Init(UWorld*);

    UNiagaraParameterCollectionInstance* GetParameterCollection(UNiagaraParameterCollection*);
    TSharedRef<FNiagaraSystemSimulation> GetSystemSimulation(ETickingGroup, UNiagaraSystem*);
    void DestroySystemSimulation(UNiagaraSystem*);
    void DestroySystemInstance(TUniquePtr<FNiagaraSystemInstance>&);

    void Tick(ETickingGroup, float, ELevelTick, ENamedThreads::Type, const FGraphEventRef&);
    void PostActorTick(float);

    UNiagaraComponentPool* GetComponentPool();
    FNDI_SkeletalMesh_GeneratedData& GetSkeletalMeshGeneratedData();
    TArrayView<const FVector> GetCachedPlayerViewLocations() const;

    // Scalability API
    void RegisterWithScalabilityManager(UNiagaraComponent*);
    void UnregisterWithScalabilityManager(UNiagaraComponent*);
    bool ShouldPreCull(UNiagaraSystem*, UNiagaraComponent*);
    bool ShouldPreCull(UNiagaraSystem*, FVector Location);
    void CalculateScalabilityState(...);
    void SortedSignificanceCull(...);

    void UpdateScalabilityManagers(bool bNewSpawnsOnly);

    // Pool 管理
    static void PrimePoolForAllWorlds(UNiagaraSystem*);
    void PrimePoolForAllSystems();
    void PrimePool(UNiagaraSystem*);
};
```

## 核心字段

```cpp
static TMap<UWorld*, FNiagaraWorldManager*> WorldManagers;   // 全局 world→manager 映射

UWorld* World;
FNiagaraWorldManagerTickFunction TickFunctions[NiagaraNumTickGroups];   // 每 TickGroup 一个 tick function

TMap<UNiagaraParameterCollection*, UNiagaraParameterCollectionInstance*> ParameterCollections;

// 每 (TickGroup, Asset) 对应一个 Simulation(Phase 3)
TMap<UNiagaraSystem*, TSharedRef<FNiagaraSystemSimulation>> SystemSimulations[NiagaraNumTickGroups];
TArray<TSharedRef<FNiagaraSystemSimulation>> SimulationsWithPostActorWork;

int32 CachedEffectsQuality;
bool bCachedPlayerViewLocationsValid;
TArray<FVector, TInlineAllocator<8>> CachedPlayerViewLocations;   // 至多 8 视点缓存

UNiagaraComponentPool* ComponentPool;
bool bPoolIsPrimed;

FNDI_SkeletalMesh_GeneratedData SkeletalMeshGeneratedData;  // Phase 7 共享 skinning 数据

TArray<TUniquePtr<FNiagaraSystemInstance>> DeferredDeletionQueue;  // PostActorTick 清

TMap<UNiagaraEffectType*, FNiagaraScalabilityManager> ScalabilityManagers;   // 按 EffectType

bool bAppHasFocus;   // 没 focus 时部分 culling 禁用
```

## World Lifecycle Hooks(静态)

```cpp
static void OnWorldInit(UWorld*, FWorld::InitializationValues);
static void OnWorldCleanup(UWorld*, bool bSessionEnded, bool bCleanupResources);
static void OnPreWorldFinishDestroy(UWorld*);
static void OnWorldBeginTearDown(UWorld*);
static void TickWorld(UWorld*, ELevelTick, float);
// GC 钩子:
static void OnPreGarbageCollect / OnPostGarbageCollect / OnPostReachabilityAnalysis / OnPreGarbageCollectBeginDestroy;
```

通过 `FDelegateHandle` 注册到 UE 全局 world delegates。

## 私有 Scalability helpers

```cpp
bool CanPreCull(UNiagaraEffectType*);
void DistanceCull(... FVector/UNiagaraComponent, OutState);
void VisibilityCull(...);
void InstanceCountCull(...);
```

## `FNiagaraViewDataMgr`(L31,全局 RT 资源)

```cpp
class FNiagaraViewDataMgr : public FRenderResource {
    // PostOpaqueRender 回调写入 scene textures 供 DI 读(Camera DI / CollisionQuery DI)
    FRHITexture2D* SceneDepthTexture;
    FRHITexture2D* SceneNormalTexture;
    FRHITexture2D* SceneVelocityTexture;
    FRHIUniformBuffer* ViewUniformBuffer;
    TUniformBufferRef<FSceneTextureUniformParameters> SceneTexturesUniformParams;
};
extern TGlobalResource<FNiagaraViewDataMgr> GNiagaraViewDataManager;
```

全局单例,RT 驻留。

## 模板辅助

```cpp
template<typename TAction> void ForAllSystemSimulations(TAction);
template<typename TAction> static void ForAllWorldManagers(TAction);
```

## 涉及实体

- [[Wiki/Entities/Stock/Niagara/FNiagaraWorldManager]]
- [[Wiki/Entities/Stock/Niagara/FNiagaraSystemSimulation]](Phase 3)
- [[Wiki/Entities/Stock/Niagara/FNiagaraScalabilityManager]]
- [[Wiki/Entities/Stock/Niagara/UNiagaraComponentPool]]
