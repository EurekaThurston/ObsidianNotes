---
type: entity
category: implementation-notes
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, implementation, private]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Private/KuroEffectSystemNew
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [Private Implementation Insights, Cpp Details]
---

# New 系统 Private 实现洞察

> Batch 6 精读:选读 `EffectSpecFactory.cpp` + `EffectSystem.cpp`(前 400 行)+ `EffectLifeTime.cpp`(前 200 行),填上前 5 批的**开放问题**和**关键实现细节**。New 系统 Private/ 共 23 个 .cpp 13k 行,本页只覆盖骨架。

## 填回前批的开放问题

### Q(来自 Batch 1): LRU 池默认容量?
```cpp
// EffectSystem.cpp:75
int32 FEffectSystem::EFFECT_LRU_CAPACITY = 100;
```
**100**。可通过 `FEffectSystem::SetEffectLruCapacity(int32)` 运行时调整。

### Q(Batch 1): Handle Id 位编码?
```cpp
// EffectSystem.cpp:82-93(含 #if WITH_EDITOR 分叉!)

#if WITH_EDITOR
uint8 Digit = 32;
uint8 IndexDigit = 15;     // → MaxIndex = 32,767
uint8 VersionDigit = 17;   // → MaxVersion = 131,071
#else  // Shipping
uint8 Digit = 32;
uint8 IndexDigit = 12;     // → MaxIndex = 4,095
uint8 VersionDigit = 20;   // → MaxVersion = 1,048,575
#endif
```

**关键洞察**:**编辑器 vs Shipping 策略不同**:
- **编辑器**:允许**同时存在 32k 特效**,slot 复用次数只 128k(编辑 Scene 可能瞬间创建大量特效)
- **Shipping**:只允许**同时 4k 特效**,但 slot 复用次数 1M(实际游戏每个特效反复 spawn/destroy,slot 复用频繁)

Handle Id 的位布局推测(`.cpp` 别处完整算法,本文件未全读):
```
Version (high bits) | Index (low bits)
```
`Version` 随 slot 复用递增,防止 ABA 问题——Handle A 被回池后,同 slot 被 Handle B 重用,Old Handle 仍持 Id 却已失效,Version 升级一次就能判断"这个 Id 过期"。

### Q(Batch 2): `FEffectHandle::AfterConstruct(FEffectSpecData)` 做什么?
```cpp
// EffectSystem.cpp:373-383
TSharedPtr<FEffectHandle> FEffectSystem::LruCreator(const FName& Key, void* Parameter)
{
    FEffectSpecData* DataPtr = static_cast<FEffectSpecData*>(Parameter);
    if (DataPtr)
    {
        TSharedPtr<FEffectHandle> Handle = MakeShared<FEffectHandle>(Key);  // ctor(Path)
        Handle->AfterConstruct(*DataPtr);   // ← 带着 SpecData 完成二阶段
        return Handle;
    }
    return nullptr;
}
```

**`AfterConstruct` 在 LRU 创建 Handle 时被调**:
- `MakeShared<FEffectHandle>(Key)` 是**第一阶段**(ctor(FName Path))——只存 Path,没 Spec
- `AfterConstruct(SpecData)` 是**第二阶段**——应该调 [[entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory::CreateEffectSpec(SpecData)]] 构造对应 Spec 类型,然后 `SetHandle`、`InitLifeTime` 等。

两阶段的意义:LRU 接口 `TLru<FName, T>` 只能 key-value;`AfterConstruct` 让 Handle 在 LRU 外也能用 ctor 构造(方便测试/临时对象)。

### Q(Batch 3): `FEffectSpecFactory::CreateEffectSpec` 实际实现
```cpp
TSharedPtr<IEffectSpecBase> FEffectSpecFactory::CreateEffectSpec(const FEffectSpecData& SpecData)
{
    TSharedPtr<IEffectSpecBase> EffectSpec;
    switch (SpecData.SpecType)
    {
    case EEffectSpecDataType_Group:
        EffectSpec = MakeShared<FEffectModelGroupSpec>(); break;
    case EEffectSpecDataType_Niagara:
    case EEffectSpecDataType_PointCloudNiagara:                 // ← 两种走同一 Spec!
        EffectSpec = MakeShared<FEffectModelNiagaraSpec>(); break;
    // ... 15 个 case
    //待兼容
    case EEffectSpecDataType_MaterialController:
        EffectSpec = MakeShared<FEffectModelMaterialControllerSpec>(); break;
    case EEffectSpecDataType_SequencePose:
        EffectSpec = MakeShared<FEffectModelSequencePoseSpec>(); break;
    // ...
    default:
        EFFECT_SYSTEM_LOG(LogKuroEffectSystem, Error, TEXT("[特效框架]EffectType没有对应的EffectSpec %d"), SpecData.SpecType)
        break;
    }
    return EffectSpec;
}
```

关键洞察:
- **`PointCloudNiagara` 和 `Niagara` 共享 `FEffectModelNiagaraSpec`**——只是 SpecData.SpecType 不同,运行时都用同一个 Spec 类(Spec 内部可能检查 DA 的 `bPointCloudNiagara` 走不同逻辑)
- **`MaterialController` / `SequencePose` / `NDC` / `CurveTrailDecal` 标注 "待兼容"**——说明 New 里这些 Spec 的支持还不完全!
- 没匹配就 log error,返回 null

### Q(Batch 1/2): `FEffectSystem::Initialize` 都做什么?

按顺序:
1. 检查 `GKuroCppEffectSystemEnable` CVar——总开关
2. 已初始化过 → 先 `Clear()`(支持重初始化)
3. `EffectSystemScriptBridge.Initialize(Isolate)` — 初始化 JS Bridge
4. 运行时创建 `HoldPreloadObject = NewObject<UHoldPreloadObject>(GameInstance)`——**以 GameInstance 为 Outer**,保证随 GameInstance 生命周期
5. 若 IsResetSpecData,刷新 SpecData Map
6. **构造 LRU**:
   ```cpp
   LruPool = TLru<FName, FEffectHandle>(EFFECT_LRU_CAPACITY, &LruCreator, &LruClearer);
   ```
7. `PlayerEffectContainer.Initialize()`
8. `VisibilityOptimizeController.Init(各种剔除阈值)`
9. `IsMobile = FeatureLevel == ES3_1`
10. 保存 BP View Class
11. **注册 3 条核心 delegate**:
    - `IJsEnvModule::Get().JsEnvClean` → JS 环境销毁时 `OnJsEnvClean()`
    - `FCoreUObjectDelegates::PostLoadMapWithWorld` → 切场景后 `OnPostLoadLevel`
12. **(WITH_EDITOR)注册 3 条编辑器 delegate,只注册一次**:
    - `FEditorDelegates::BeginPIE` / `EndPIE`
    - `LevelEditor.OnCurrentMapFinishExit`
13. **注册 3 段 Tick**:
    ```cpp
    PrePhysicsTickFunction(TG_PrePhysics)
        .BindStatic(&Tick).RegisterTickFunction(World->PersistentLevel);
    PostPhysicsTickFunction(TG_PostPhysics)
        .BindStatic(&PostPhysicsTick).RegisterTickFunction(...);
    PostUpdateWorkTickTickFunction(TG_PostUpdateWork)
        .BindStatic(&PostUpdateWorkTick).RegisterTickFunction(...);
    ```

### Q(Batch 2): `EffectForSpecChildMap` 为什么只缓存某些?
```cpp
// EffectSystem.cpp:313-342(RefreshEffectForSpecData 内)
// 出于内存考虑，暂时只缓存带有PointCloudNiagara的映射关系
for (const FEffectSpecChildData& ChildData : SpecChildData) {
    bool bTarget = false;
    if (EffectForSpecMap.Contains(ChildData.Id)) {
        for (const int32 ChildId : ChildData.Children) {
            if (EffectForSpecMap[ChildId].SpecType == EEffectSpecDataType_PointCloudNiagara) {
                bTarget = true;
                break;
            }
        }
    }
    if (bTarget) {
        EffectForSpecChildMap.Add(ChildData.Id, ChildData.Children);
    }
}
```

**惊奇**:**`EffectForSpecChildMap` 只存"含 PointCloud 子特效的 Group"**!内存考虑——绝大多数 Group 不需要运行时展开 child 列表(`FEffectModelGroup` 已含 `TMap<UEffectModelBase*, float>` 数据),**只有 PointCloud 场景需要 runtime 查 child 关系**。

### Q(Batch 2/3): `FEffectLifeTime::SetTime` 怎么判断循环?
```cpp
// EffectLifeTime.cpp:181-205
void FEffectLifeTime::SetTime(float InStartTime, float InLoopTime, float InEndTime, bool InContinuousLoop)
{
    IsInitTime = true;
    StartTime = InStartTime;
    LoopTime = InLoopTime;
    EndTime = InEndTime;
    LoopTimeStamp = StartTime + LoopTime;
    LifeTimeStamp = StartTime + LoopTime + EndTime;
    
    IsLoopInternal = StartTime < 0 || LoopTime > 0;           // ← 判循环
    WillEverPlay = IsLoopInternal || LifeTimeStamp <= 0;      // ← 判会播
    
    if (InContinuousLoop)
        ContinuousLoop = IsLoopInternal;
    
    if (!IsLoopInternal) {
        // 非循环特效:只能设置一次生命周期
        if (!WaitPlayFinishedTimerHandle.IsValid()) {
            SetLifeCycle(LifeTimeStamp);   // 直接注册定时器
        }
    }
    // ...
}
```

关键:
- `IsLoopInternal = StartTime < 0 || LoopTime > 0`——**特效是循环**的判定:
  - `StartTime < 0`(等同"-1" = 手动触发/永久) 
  - 或 `LoopTime > 0`(有明确循环段)
- `WillEverPlay = IsLoopInternal || LifeTimeStamp <= 0`——**会不会播**:
  - 循环特效会播
  - 或总时长 ≤ 0(推测是"立即完成")也算"会播"(只是马上结束)
- **非循环 + TimerHandle 未绑定 → 立即 `SetLifeCycle(LifeTimeStamp)` 启动定时器**
- 循环特效的"播放结束"需要外部调 Stop,Timer 不自动触发

### Q(Batch 4): MiniTimeScale 保底的具体逻辑
```cpp
// EffectLifeTime.cpp:86-102
void FEffectLifeTime::WaitMiniTimeScaleFinish()
{
    WaitMiniTimeScaleTimerHandle.Reset();
    TWeakPtr<FEffectHandle> Handle = EffectSpec.Pin()->GetHandle();
    if (!Handle.IsValid()) return;
    
    Handle.Pin()->SetTimeScale(1);                   // ← 强制复原 Handle 的 TimeScale
    SetTimeScale(1);                                  // ← 再复原 LifeTime 自己的 TimeScale
    
    if (FEffectSystem::UseLog) {
        EFFECT_SYSTEM_LOG(LogKuroEffectSystem, Warning,
            TEXT("[特效框架]特效设置TimeScale为极小值，没有及时置回，造成泄漏，句柄Id: %d Path: %s CreateReason: %s"),
            Handle.Pin()->Id, *Handle.Pin()->Path.ToString(), *Handle.Pin()->CreateReason.ToString());
    }
}
```

**触发条件**(回顾 [[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec::SetTimeScale]]):
- TimeScale 从 ≥0.1 降到 <0.1 → 注册 `MAX_WAIT_TIME_SCALE_ZERO_TIME = 10` 秒后的定时器
- 10 秒内没改回来,系统自己复原 + **打 Warning 日志**,含 `CreateReason`
- 业务侧在 log 里看到就能追查"谁 set 了 mini TimeScale 没清"

### Q(Batch 3): `PlayFinished` 的分支逻辑
```cpp
// EffectLifeTime.cpp:29-84
void FEffectLifeTime::PlayFinished()
{
    IsPendingPlayFinish = false;
    IEffectSpecBase* Spec = EffectSpec.Pin().Get();
    TSharedPtr<FEffectHandle> Handle = Spec->GetHandle().Pin();
    WaitPlayFinishedTimerHandle.Reset();
    
    // 分支 1:子特效 → 直接 Stop
    if (!Handle->IsRoot()) {
        Handle->Stop(PlayFinishStopEffectReason, true);
        return;
    }
    
    // 分支 2:Spec 不能停 → 延迟 1 秒重试
    if (!Spec->CanStop()) {
        FKuroTimerSystem::Delay(WaitPlayFinishedTimerHandle, 1.0f);
        return;
    }
    
    // 分支 3:运行时根特效
    if (FEffectSystem::IsGameRunning) {
        AActor* EffectActor = Handle->GetSureEffectActor();
        if (EffectActor && !Handle->IsExternalActor) {
            EffectActor->K2_DetachFromActor();        // 先解绑
            Handle->SetHidden(true, PlayFinishStopEffectReason);
        }
        Handle->UnregisterTick();                     // 反注册 tick
        FEffectSystem::AddRemoveHandle(Handle, PlayFinishStopEffectReason);  // 加入删除队列
    }
    // 分支 4:编辑器预览
    else {
        FEffectSystem::StopEffect(Handle, PlayFinishStopEffectReason, true);
    }
}
```

—— **非常清晰的决策树**:
1. 子特效 → 直接杀
2. 根特效但 `CanStop == false`(Spec 说还在忙)→ 延迟 1 秒重试
3. 运行时根特效 → detach + hide + unregister tick + 加删除队列(**延迟删除**,下帧清理)
4. 编辑器预览 → 立刻 Stop

### Q(Batch 3): `MakeLoop` 的真实逻辑
```cpp
// EffectLifeTime.cpp:104-123
void FEffectLifeTime::MakeLoop()
{
    if (LoopTime <= 0.001f) {
        PassTime = StartTime;   // LoopTime 极小,卡在 StartTime 末端
        return;
    }
    
    if (PassTime >= LoopTimeStamp + LoopTime) {
        // 过了**两倍** LoopTime——例如卡顿导致一帧 DeltaTime 太大
        float HasLoopTime = PassTime - StartTime;
        float ModTime = 0;
        UKismetMathLibrary::FMod(HasLoopTime, LoopTime, ModTime);
        PassTime = StartTime + ModTime;   // 用 FMod 跳过多 Loop,保持 0~LoopTime
    } else {
        PassTime -= LoopTime;              // 正常一次 Loop
    }
}
```

卡顿 / DeltaTime 激增防御:当 PassTime 一下子超过 LoopTimeStamp + LoopTime(一轮循环),不做多次 Loop,直接 FMod 跳过。

## 新的重要 CVars(这次读到)

```cpp
Kuro.CppEffectSystem.Enable                          // 整个 Cpp 特效系统总开关
Kuro.CppEffectSystem.RemoveHandleForceDelete         // RemoveHandle 支持 ForceDelete
Kuro.CppEffectSystem.RemoveHandleImmediately         // RemoveHandle 是否立即执行
Kuro.CppEffectSystem.PreviewWithoutConfig (WITH_EDITOR)   // 编辑器 Preview 不需导出配置
Kuro.CppEffectSystem.NotifyTickSystemPausedChange   // TickSystemPaused 变化立即广播
Kuro.CppEffectSystem.DebugTotalEffectHandle         // 触发 DebugTotalEffectHandle 的命令
```

加上前几批读到的 10+ 个 CVar,**New 系统运行时可调参数总数 15+**——线上运维友好度极高。

## 新常量

```cpp
FEffectSystem::EFFECT_REASON_LENGTH_LIMIT = 4;         // Reason 名最大长度?推测为 debug log 截断用
FEffectSystem::EFFECT_LIFETIME_FLOAT_TO_INT = 10000;   // 时间 float→int 乘数(毫秒×10 或 微秒级)
FEffectSystem::EFFECT_SPEC_DATA_PATH = "../Config/Client/EffectData/";  // 配置加载路径
FEffectSystem::EFFECT_LRU_CAPACITY = 100;              // LRU 池默认容量
FEffectSystem::PERCENT = 100;                          // 百分比常量(业务意图不明)
FEffectSystem::LruFolderPath = "LruActorPool";         // LRU 池的 Actor 组织文件夹名
FEffectSystem::CHECK_EFFECT_OWNER_INTERVAL = 60000;    // 60 秒检查一次 Owner(内存检测)
FEffectSystem::ClearRemoveHandleReason = "EffectSystem::Clear";  // Clear 时用的 Reason
FEffectLifeTime::PlayFinishStopEffectReason = "EffectLifeTime.PlayFinished";
```

**`EFFECT_SPEC_DATA_PATH`** 是一个业务信号:**配置文件从磁盘路径加载**。实际路径 `../Config/Client/EffectData/`——项目打包时作为 staged content 存在。但 [[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]] 也通过 `Initialize(..., SpecData, SpecChildData, ...)` 从业务侧传入——**两路加载并存**。可能:
- 脚本**先从磁盘读** `EffectData/` 下的 json / 自定义格式
- 解析后**通过 Initialize 传入** `TArray<FKuroEffectSpecData>`
- C++ 不直接读文件,文件路径只是常量文档性质

## 三段式 Tick 的实际绑定

```cpp
FKuroTickNativeFunction FEffectSystem::PrePhysicsTickFunction(TG_PrePhysics);
FKuroTickNativeFunction FEffectSystem::PostPhysicsTickFunction(TG_PostPhysics);
FKuroTickNativeFunction FEffectSystem::PostUpdateWorkTickTickFunction(TG_PostUpdateWork);
```

Initialize 时绑定 3 个静态方法:
- `TG_PrePhysics` → `FEffectSystem::Tick(DeltaTime)` — 主 Tick,特效核心逻辑在这
- `TG_PostPhysics` → `FEffectSystem::PostPhysicsTick(DeltaTime)` — 物理后处理(Attach 刚骨骼特效的 location 更新)
- `TG_PostUpdateWork` → `FEffectSystem::PostUpdateWorkTick(DeltaTime)` — 最末端帧末清理

**Old 只在 TG_PostUpdateWork 一个 tick group**,New 在 3 个 tick group 都跑——**粒度细化**,特效能精确选择在 UE 帧内何时推进。

### `OnPostLoadLevel` — 切场景时重挂 tick
```cpp
void FEffectSystem::OnPostLoadLevel(UWorld* World)
{
    MainWorld = World;
    // 先反注册旧 tick
    PrePhysicsTickFunction.UnRegisterTickFunction();
    PostPhysicsTickFunction.UnRegisterTickFunction();
    PostUpdateWorkTickTickFunction.UnRegisterTickFunction();
    
    // 再在新 World 上注册
    if (World) {
        PrePhysicsTickFunction.RegisterTickFunction(World->PersistentLevel);
        // ...
    }
}
```

**切 Level 后 tick 必须重挂**——UE 的 tick 绑定是挂在 Level 上的,旧 Level 销毁后 TickFunction 失效。

## PIE 处理(编辑器里玩游戏)

```cpp
// WITH_EDITOR
OnEndPIEDelegate = FEditorDelegates::EndPIE.AddStatic(&FEffectSystem::OnEndPIE);
OnBeginPIEDelegate = FEditorDelegates::BeginPIE.AddStatic(&FEffectSystem::OnBeginPIE);

void OnEndPIE(const bool bIsSimulating)
{
    Clear();
    VisibilityOptimizeController.SetActive(false);   // 非 PIE 时关剔除
}

void OnBeginPIE(const bool bIsSimulating)
{
    HasJsEnv = false;
    Clear();
    VisibilityOptimizeController.SetActive(true);    // 只有运行时开剔除
}
```

**清理时机**:
- BeginPIE:Clear + 开启剔除(运行时才需要)
- EndPIE:Clear + 关闭剔除(编辑器不需要剔除干扰)
- EditorCurrentMapFinishExit:`HasJsEnv = false; Clear();`
- JsEnvClean 触发:同上

**Clear 被调用的 4 种场景** — 编辑器场景切换 / PIE 进出 / JsEnv 销毁 / 业务显式调。

## 这次揭示的整体架构地图

```
[业务: TS/C#/BP]
      ├─→ FEffectSystem::Initialize(Isolate, GI, SpecData[], SpecChildData[], ...)
      │         → 构造 LRU(LruCreator/LruClearer 回调)
      │         → 初始化 Bridge/Container/Visibility
      │         → 注册 3 段 Tick + 4 种 Editor Hooks
      │
      └─→ FEffectSystem::SpawnEffect(WorldCtx, Transform, Path, Reason, Callbacks, Context, ...)
                │
                ▼
        LRU.Get(Path, &SpecData) → 命中则复用 Handle;否则 LruCreator 触发
                │ (未命中)
                ▼
        MakeShared<FEffectHandle>(Path)  (ctor 只存 Path)
                ▼
        Handle->AfterConstruct(SpecData)
                │    (内部) FEffectSpecFactory::CreateEffectSpec(SpecData)
                │                   → switch on SpecType → MakeShared<FEffectModelXxxSpec>()
                │          Handle->EffectSpec = Factory 产出的 Spec
                │          Spec->SetHandle(Handle)
                │          Spec->InitLifeTime(Spec)  (Spec 持 LifeTime)
                ▼
        Handle->PendingInit(...) (进 Pending 态)
                ▼
        [Async] LoadObject<UEffectModelBase>(Path) 完成
                ▼
        Handle->Init(EffectData, InitModel)
                │   Spec->Init(EffectData, InitModel)
                │       EffectModel = Cast<T>(EffectData)
                │       Spec::OnInit(InitModel)  (各子类专属初始化)
                │       LifeTime.SetTime(StartTime, LoopTime, EndTime)
                │           → 非循环:立即 SetLifeCycle(LifeTimeStamp) 注册 Timer
                ▼
        Handle->Play(Reason)
                ▼
        [Tick 期间] TG_PrePhysics → FEffectSystem::Tick(dt)
                      → 遍历 TickHandleMap,每个 Handle->Tick(dt)
                         → Spec::Tick(dt)
                            - OnTick(dt) 子类专属工作
                            - LifeTime.Tick(dt)  推进时间
                               → MakeLoop() / 触发 Timer
                ▼
        [特效结束] WaitPlayFinishedTimer 触发
                ▼
        LifeTime::PlayFinished
                │
          ├─→ 子特效:Handle->Stop(true)
          ├─→ Spec !CanStop:延迟 1 秒重试
          ├─→ 运行时:Detach + Hide + UnregisterTick + AddRemoveHandle
          └─→ 编辑器:StopEffect(true)
```

## 与其他实体的关系

- **补充/修正开放问题** 到:
  - [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]](Initialize, Id 编码, LRU 容量)
  - [[entities/project-game/kuro-effect-system-new/effect-spec-factory|FEffectSpecFactory]](实际实现)
  - [[entities/project-game/kuro-effect-system-new/effect-life-time|FEffectLifeTime]](PlayFinished, MakeLoop, MiniTimeScale 保底)
  - [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]](AfterConstruct 语义)
  - [[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]](ChildMap 内存优化,EFFECT_SPEC_DATA_PATH)

## 未读内容清单(下次可补)

主 .cpp 文件**都未读完**:
- `EffectSystem.cpp`:3849 行,读了前 400 行(仅 Initialize 周边)
  - 未读:SpawnEffect 实现、Tick 实现、LRU 管理、RemoveHandle 队列处理、PIE 钩子细节
- `EffectHandle.cpp`:1926 行,**完全未读**
  - PendingInit 完整流程、Init 实现、Play/Stop 细节、OwnerEffect 同步、BodyEffect 实现
- `EffectActorHandle.cpp`:501 行,**未读**(InitEffectActor flush 动作队列的细节)
- `EffectSpecFactory.cpp` ✓(本批读了,小文件全读)
- `EffectLifeTime.cpp`:507 行,前 200 行(后面:Tick 实现、SeekTo 实现)
- `NiagaraComponentHandle.cpp`:342 行,**未读**(Init 时 flush 参数的具体逻辑)
- `PlayerEffectContainer.cpp`:287 行,**未读**(分池细节)
- `ContinuousEffectController.cpp`:未统计,**未读**(AnsSlot 决策)
- Bridge/Holder .cpp:**未读**(v8 thunk 实现)
- Reflection/.cpp:**未读**

如需后续深入这些文件,建议"按需问"——某个具体问题出现时,回到相关 .cpp 找答案。

## 引用来源

- 已读 .cpp:
  - `F:\...\Private\KuroEffectSystemNew\EffectSpec\EffectSpecFactory.cpp`(163 行,全读)
  - `F:\...\Private\KuroEffectSystemNew\EffectSystem.cpp`(3849 行,读前 400 行)
  - `F:\...\Private\KuroEffectSystemNew\EffectLifeTime.cpp`(507 行,读前 200 行)
- 回答的开放问题来自:Batch 1、2、3、4 各页的"开放问题"节

## 仍然开放的问题(留给未来)

- `EFFECT_REASON_LENGTH_LIMIT = 4` 的用处(4 字节限制?)
- `EFFECT_LIFETIME_FLOAT_TO_INT = 10000` 为什么 ×10000?毫秒 × 10 = 100μs 级?
- `CHECK_EFFECT_OWNER_INTERVAL = 60000`—— 60 秒定时器做什么检查?
- `Version` 具体怎么递增的(每次 slot 复用 +1)?
- SpawnEffect 的完整代码路径(EffectSystem.cpp 剩余 3400 行)
- Handle->Init 的具体实现(EffectHandle.cpp)
- Group 的 DelayPlay 协调(部分在 .cpp)
- 哪些 "待兼容" Spec(MaterialController/SequencePose/NDC/CurveTrailDecal)实际运行时会怎样
