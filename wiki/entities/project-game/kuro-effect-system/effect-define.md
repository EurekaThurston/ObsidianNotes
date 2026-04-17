---
type: entity
category: enums-and-delegates
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, enums, reference]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectDefine.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
aliases: [EffectDefine, EEffectFlag, EEffectType, EEffectSpecType]
---

# EffectDefine(枚举与 Delegate 参考)

> 整个 Kuro EffectSystem 共享的枚举和 Delegate 定义集合;**New 和 Old 都 include 这个文件**(`#include "KuroEffectSystem/EffectDefine.h"`),是两套系统的公共契约层。

## 概览

这是一个"constants + vocabulary"文件。当你看代码里出现 `EEffectFlag_Play` 或 `ELoadEffectResult_Success` 时,定义都在这里。**维护时注意:这个文件一改,两套系统的行为可能同时变。**

## 构建选项

```cpp
#define KURO_ENABLE_EFFECT_SYSTEM_DEBUG_DRAW !UE_BUILD_SHIPPING && !UE_BUILD_TEST
```
—— 非 shipping/test 构建中启用 debug draw。`FEffectHandle` 和 `FEffectSystem` 里带这个宏保护的代码块都是调试可视化相关。

## 日志宏
```cpp
EFFECT_SYSTEM_LOG(CategoryName, Verbosity, Format, ...)
// 实际展开:UE_LOG(CategoryName, Verbosity, "[HH:MM:SS:ss][FrameCount]" Format, ...)
```
自带时间戳和帧号前缀,便于时序回溯。

```cpp
DECLARE_LOG_CATEGORY_EXTERN(LogKuroEffectSystem, Display, All);
```

## 核心枚举

### `EEffectFlag` — 句柄生命周期位掩码
这是最关键的一个枚举。每个位代表 Handle 在生命周期里经过的一个阶段,**位会累积而不是覆盖**。一个"正在播"的 handle 的 Flag 通常会是 `Init | Start | Play`。

```cpp
enum EEffectFlag
{
    EEffectFlag_None,           // 0x00  初始化状态
    EEffectFlag_Init    = 0b1,  // 0x01  已执行过 Init
    EEffectFlag_Start   = 0b10, // 0x02  已执行过 Start
    EEffectFlag_Play    = 0b100,// 0x04  已播放
    EEffectFlag_PreStop = 0b1000,
    EEffectFlag_Stop    = 0b10000,
    EEffectFlag_End     = 0b100000,
    EEffectFlag_Clear   = 0b1000000,
    EEffectFlag_Destroy = 0b10000000,  // 最终态
};
```

**生命周期线性走向**:`None → Init → Start → Play → (PreStop) → Stop → End → Clear → Destroy`。

配套 `FEffectDefine` 类里有状态重置的 mask(具体值在 .cpp):
```cpp
class FEffectDefine {
public:
    static int32 ALL_EFFECT_FLAG_MASK;
    static int32 RESET_PLAY_FLAG;
    static int32 RESET_STOP_FLAG;
    static int32 RESET_PRESTOP_FLAG;
    static int32 NEED_CHECK_DISABLE_MASK;
};
```
—— Replay 时用 `RESET_PLAY_FLAG` AND 当前 Flag,把"播过"的位清掉。

### `EEffectType` — 特效类型(用于剔除策略)
```cpp
enum EEffectType
{
    EEffectType_Fight,        // 战斗
    EEffectType_UiScene3D,    // UI 里的 3D 场景特效(如角色展示柜)
    EEffectType_UI,           // 纯 UI 特效
    EEffectType_Scene,        // 场景默认
};
```
不同类型**走不同的降频/剔除策略**——UI 特效不会被相机距离剔除,Fight 特效优先级高等。`FEffectSystem::SpawnEffect` 的 `EffectType` 参数走这个枚举。

### `ELoadEffectResult` — Spawn 回调结果
```cpp
enum ELoadEffectResult
{
    ELoadEffectResult_Fail,              // 失败
    ELoadEffectResult_CanNotPlay,        // 不能播放(条件不满足)
    ELoadEffectResult_Cancel,            // 加载中被取消
    ELoadEffectResult_Stop,              // 已停止
    ELoadEffectResult_EffectActorError,  // EffectActor 已无效(!IsValid / World 变了)
    ELoadEffectResult_Success,           // 成功
};
```
业务在 `FEffectInitCallback` 回调里收到这个结果码。只有 `Success` 时 Handle 可用。

### `EEffectSpecType` — 运行时 Spec 类型(18 种)
```cpp
enum EEffectSpecType
{
    EEffectSpecType_Base, EEffectSpecType_Audio, EEffectSpecType_Billboard,
    EEffectSpecType_Decal, EEffectSpecType_Ghost, EEffectSpecType_GpuParticle,
    EEffectSpecType_Group, EEffectSpecType_Light, EEffectSpecType_MaterialController,
    EEffectSpecType_MultiEffect, EEffectSpecType_Niagara, EEffectSpecType_PostProcess,
    EEffectSpecType_SkeletalMesh, EEffectSpecType_StaticMesh, EEffectSpecType_Trail,
    EEffectSpecType_SequencePose, EEffectSpecType_NDC, EEffectSpecType_CurveTrailDecal,
};
```
和 DataAsset 家族 [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelXxx]] 一一对应。`FEffectSpec*` 系列(Batch 3)用它做 RTTI。

### `EEffectSpecDataType` — 数据层类型(19 种,**New 用**)
```cpp
enum EEffectSpecDataType {
    EEffectSpecDataType_Group = 0, EEffectSpecDataType_Niagara, EEffectSpecDataType_Audio,
    EEffectSpecDataType_Billboard, EEffectSpecDataType_Decal, EEffectSpecDataType_Ghost,
    EEffectSpecDataType_Light, EEffectSpecDataType_GPUParticle, EEffectSpecDataType_PostProcess,
    EEffectSpecDataType_SkeletalMesh, EEffectSpecDataType_StaticMesh, EEffectSpecDataType_Trail,
    EEffectSpecDataType_MultiEffect, EEffectSpecDataType_MaterialController,
    EEffectSpecDataType_SequencePose, EEffectSpecDataType_NDC, EEffectSpecDataType_CurveTrailDecal,
    EEffectSpecDataType_PointCloudNiagara,  // ← 比 EEffectSpecType 多一个!
    EEffectSpecDataType_Max,
};
```

**`EEffectSpecType` 和 `EEffectSpecDataType` 的区别**:前者是运行时 Spec 类型(Old 主用),后者是 New 的 `FEffectSpecData.SpecType` 字段值。`PointCloudNiagara` 是 New 从 `UEffectModelNiagara::bPointCloudNiagara` 细化出来的子类。

### `EEffectErrorCode`
```cpp
enum EEffectErrorCode
{
    EEffectErrorCode_Success,
    EEffectErrorCode_WithoutSpec,
    EEffectErrorCode_NiagaraComponentError,
    EEffectErrorCode_NiagaraComponentPausedError,
};
```
见 `FEffectHandle::GetDebugErrorCode()`(New)。

### `EEffectPlayFlag`
```cpp
enum EEffectPlayFlag
{
    EEffectPlayFlag_Default = 0,
    EEffectPlayFlag_NoNiagara = 0b1,  // 不播 Niagara 子特效
};
```
存在 `FEffectContext::PlayFlag`,用于"禁用某类子表现"。

### `EEffectHandleCreateSource`(New 用)
```cpp
enum EEffectHandleCreateSource
{
    EEffectHandleCreateSource_None,
    EEffectHandleCreateSource_Lru,                      // 从 LRU 池取出
    EEffectHandleCreateSource_PlayerEffectPool,         // 玩家池
    EEffectHandleCreateSource_PlayerEffectPool1,
    EEffectHandleCreateSource_PlayerEffectPool2,
    EEffectHandleCreateSource_PlayerEffectPool3,
};
```
Handle 诞生时记录来源——供 debug 和 [[entities/project-game/kuro-effect-system-new/player-effect-container|FPlayerEffectContainer]](Batch 4) 使用。

### `EEffectSpecInitState`
```cpp
enum EEffectSpecInitState
{
    EEffectSpecInitState_Fail,
    EEffectSpecInitState_Success,
    EEffectSpecInitState_Initializing,  // 还在初始化
};
```
Spec 的三态初始化。

### `EEffectActorPoolEnum`
```cpp
enum EEffectActorPoolEnum {
    EEffectActorPoolEnum_None,
    EEffectActorPoolEnum_ActorSystem,  // 在 Kuro ActorSystem 池中
    EEffectActorPoolEnum_LRU,          // 在特效自己的 LRU 池中
};
```
**两种不同的对象池**:UE-ActorSystem 全局池 vs EffectSystem 自己的 LRU。

### `EEffectCreateFromType`(**位掩码**)
```cpp
enum EEffectCreateFromType
{
    EEffectCreateFromType_None,
    EEffectCreateFromType_An    = 0b1,
    EEffectCreateFromType_Ans   = 0b10,
};
```
`An`/`Ans` 是**动画事件 / 动画 State** 的缩写(AnimNotify / AnimState?)——`FEffectContext::CreateFromType` 记录这个特效是从哪种动画资产触发的。

### `EEffectTimeScaleSourceType`
```cpp
enum EEffectTimeScaleSourceType
{
    EEffectTimeScaleSourceType_SelfCentered,
    EEffectTimeScaleSourceType_Max,
};
```
只有一个值 + Max 哨兵,未来可能扩展。见 `FEffectSystem::SetAdditionTimeScale(SourceType, ...)`(Batch 1 已介绍)。

## Delegate 定义

```cpp
// Spawn 流程回调(4 段,New 四段式的核心契约)
DECLARE_DELEGATE_OneParam(FEffectBeforeInitCallback, int)                    // (EffectId)
DECLARE_DELEGATE_TwoParams(FEffectInitCallback, ELoadEffectResult, int)      // (Result, EffectId)
DECLARE_DELEGATE_OneParam(FEffectBeforePlayCallback, int)                    // (EffectId)
DECLARE_DELEGATE_OneParam(FEffectFinishCallback, int)                        // (EffectId)

// 完成 Delegate 的两种形态
DECLARE_DELEGATE_OneParam(FEffectHandleFinishDelegate, int);
DECLARE_MULTICAST_DELEGATE_OneParam(FEffectHandleFinishMulticastDelegate, int)
```

### UE Delegate 小科普
- `DECLARE_DELEGATE_*`:**单播**,只能 `Bind` 一个,`Execute` 调用。适合"完成回调"场景。
- `DECLARE_MULTICAST_DELEGATE_*`:**多播**,可以 `AddLambda` / `AddUObject` 挂多个,`Broadcast` 一起调。
- 本文件同时声明两种 Finish delegate:单播的 `FEffectHandleFinishDelegate` 绑到 Handle 的 `FinishDelegate` 字段(见 Batch 1),多播的 `FEffectHandleFinishMulticastDelegate` 暂未见使用位置(Batch 6 /.cpp 再看)。

## 相关

- **用户**:全部 `FEffectHandle`、`FEffectSystem`、`FEffectContext` 等都 `#include "EffectDefine.h"`
- 直接引用它的实体页:
  - [[entities/project-game/kuro-effect-system/kuro-effect-system|FKuroEffectSystem]]
  - [[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle (Old)]]
  - [[entities/project-game/kuro-effect-system-new/effect-system|KuroEffect::FEffectSystem]]
  - [[entities/project-game/kuro-effect-system-new/effect-handle|KuroEffect::FEffectHandle (New)]]
  - [[entities/project-game/kuro-effect-system-new/effect-context|FEffectContext (New)]]

## 开放问题

- **`FEffectDefine` 静态字段的实际位值**(`ALL_EFFECT_FLAG_MASK` / `RESET_PLAY_FLAG` 等)在 .cpp 里,Batch 6 回答
- 为什么 `EEffectSpecType` 有 18 个值而 `EEffectSpecDataType` 有 19 个(多了 `PointCloudNiagara`)—— 是 New 把 `EEffectSpecType_Niagara` 按 `bPointCloudNiagara` 细分为两种吗?
