---
type: entity
category: class-hierarchy
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, context]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectContext.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectContext, FSkeletalMeshEffectContext, FEffectAudioContext, FEffectRuntimeGhostEffectContext, FEffectContextHelper]
---

# FEffectContext 家族(新版创建上下文)

> 一次 `SpawnEffect` 调用**附带的业务侧运行时信息**——谁创建的、从哪个动画触发、对应哪个实体、要走什么回放规则。Old 没有对应概念,这些信息过去散落在 Handle/Info/DataAsset 的若干字段里,New 抽成独立继承层次。

## 类结构

```
                     FEffectContext (base)        ← 最常用
                       ▲       ▲       ▲
                       │       │       │
                       │       │       └── FEffectAudioContext     (音效)
                       │       │
                       │       └── FSkeletalMeshEffectContext       (绑骨骼)
                       │               ▲
                       │               └── FEffectRuntimeGhostEffectContext  (残影)
                       │
                       (FEffectContextHelper 静态辅助)
```

分类:
- **Base**:所有特效都有的基础上下文
- **Skeletal**:附着到骨骼网格的特效(Trail / Ghost 这类)
- **Audio**:音效(需要 P1 Bus 路由等)
- **RuntimeGhost**:骨骼残影(继承 Skeletal,加 ghost 专属字段)

## FEffectContext(基类)

```cpp
enum EEffectContextType {
    EEffectContextType_Base,
    EEffectContextType_Skeletal,
    EEffectContextType_Audio,
    EEffectContextType_Ghost,
};

class FEffectContext
{
public:
    int32 EntityId = 0;                        // 创建者的 Entity Id(角色、怪物...)
    TWeakObjectPtr<UObject> SourceObject = nullptr;  // 创建者对象(Actor、Component)
    bool DisablePostProcess = false;           // 禁用后处理子效果
    uint16 CreateFromType = 0;                 // 来源类型(EEffectCreateFromType,An/Ans 位掩码)
    uint16 PlayFlag = 0;                       // 播放标志(EEffectPlayFlag,如 NoNiagara)
    bool CreateFromBpEffectActor = false;      // 是否来自 BP 特效 Actor
    EEffectContextType ContextType = EEffectContextType_Base;
    uint8 HitEffectType = 0;                   // 击中特效类型
    FName AnsSlotName;                         // AnimNotify State 的 slot 名

    FEffectContext() { ContextType = EEffectContextType_Base; }
    FEffectContext(const FKuroEffectContext& Context);  // 从 USTRUCT 版本构造
    virtual ~FEffectContext() = default;
};
```

### 字段分组解读

#### 溯源字段(给 debug / 查 leak 用)
- `EntityId`:这个特效是谁(角色、怪物)创造的。配合 `FEffectSystem::StopEffectFromEntity(EntityId, FilterName, Immediately)` 可以"**杀死该实体的所有特效**"。
- `SourceObject`:更细的创建者对象(可能是某个 Component)。弱引用,UObject 死了自动无效。

#### 配置字段(影响表现)
- `DisablePostProcess`:屏蔽此特效内的 PostProcess 子效果——典型"性能分级":低端机关掉某些特效的 Bloom/Fog 但保留主体
- `CreateFromType`(位掩码):特效是从哪种动画资产触发的
  - `_An = 0b1` 动画 AnimNotify
  - `_Ans = 0b10` 动画 AnimNotifyState
- `PlayFlag`(位掩码):播放选项
  - `_NoNiagara = 0b1` 不播 Niagara 子特效(轻量模式)
- `CreateFromBpEffectActor`:这个特效是否由一个 `BP_EffectActor`(业务自定义的特效蓝图 Actor)触发

#### 动画集成字段
- `AnsSlotName`:如果 `CreateFromType & _Ans`,这是动画里 AnimNotifyState 的 slot 名

#### 类型标识
- `ContextType`:**运行时 RTTI 标识**——因为是虚继承,`dynamic_cast` 虽可行但慢。用 uint8 tag + `static_cast` 代替。

## FSkeletalMeshEffectContext(骨骼特效用)

```cpp
class FSkeletalMeshEffectContext : public FEffectContext
{
public:
    TWeakObjectPtr<USkeletalMeshComponent> SkeletalMeshComponent = nullptr;
    bool IsSyncTimeDilation = false;           // 跟随骨骼组件的时间缩放
    bool IsSyncEventTimeToEffectTime = false;  // 把 AnimEvent 时间同步到特效时间

    FSkeletalMeshEffectContext() { ContextType = EEffectContextType_Skeletal; }
    FSkeletalMeshEffectContext(const FKuroSkeletalMeshEffectContext& Context): FEffectContext(Context)
    {
        SkeletalMeshComponent = Context.SkeletalMeshComponent;
        IsSyncTimeDilation = Context.IsSyncTimeDilation;
        IsSyncEventTimeToEffectTime = Context.IsSyncEventTimeToEffectTime;
    }
};
```

**附加字段 3 个**,全与"如何与骨骼组件协作"有关:
- 直接持骨骼组件引用(避免后续再查)
- TimeDilation 同步:角色慢动作时,其身上特效也慢
- AnimEvent 时间同步:动画时间 = 特效时间,用于精确同步刀光(动画打到第 0.3s 时刀光到第 0.3s)

用于:`UEffectModelGhost`、`UEffectModelTrail`、各种骨骼绑定特效。

## FEffectAudioContext(音效用)

```cpp
class FEffectAudioContext : public FEffectContext
{
public:
    /** 当特效发生源为玩家主控角色或其召唤物时应置为 `true`, 此时音效通过 P1 Bus 输出 */
    bool FromPrimaryRole = false;
    /* ... */
};
```

**只加 1 个字段** `FromPrimaryRole`——但有 doxygen 注释说明:

- Wwise(AkAudio)里有"**总线(Bus)**"的概念,不同总线走不同混音策略
- **P1 Bus 是主角专用的总线**,音量/混响可能单独处理
- 主角的特效音(脚步声、技能音)走这个通道,其他 NPC 走默认 Bus
- 这种设计让玩家在嘈杂战斗中仍能听清自己的技能音效

用于 `UEffectModelAudio`。

## FEffectRuntimeGhostEffectContext(骨骼残影)

```cpp
class FEffectRuntimeGhostEffectContext : public FSkeletalMeshEffectContext
{
public:
    float SpawnRate = 1.0f;          // 每秒生成残影数
    bool  UseSpawnRate = false;       // 是否用 rate-based 生成
    float SpawnInterval = 1.0f;       // interval-based 生成间隔
    float GhostLifeTime = 1.0f;       // 单个残影存活时间
    bool  UseBaseColorTex = false;    // 用基础色纹理
    /* ... */
};
```

**双模式生成**:
- `UseSpawnRate = true`:用 SpawnRate (每秒 N 个)
- `UseSpawnRate = false`:用 SpawnInterval (每 N 秒 1 个)

两种模式应对不同视觉需求——高速冲刺可能要每帧都拉残影(rate-based),缓慢移动要固定间隔(interval-based)。

这是四层继承里最深的一个,说明 Ghost 的运行时行为**比单纯 Skeletal 特效更复杂**。

## FEffectContextHelper(工厂/辅助)

```cpp
class FEffectContextHelper
{
public:
    static void CreateUniqueContext(FEffectContext* ContextPtr, TUniquePtr<FEffectContext>& Context)
    {
        if (!ContextPtr) { Context = MakeUnique<FEffectContext>(); return; }
        if (ContextPtr->ContextType == EEffectContextType_Ghost) { 
            Context = MakeUnique<FEffectRuntimeGhostEffectContext>(*static_cast<FEffectRuntimeGhostEffectContext*>(ContextPtr)); 
            return; 
        }
        if (ContextPtr->ContextType == EEffectContextType_Audio) { 
            Context = MakeUnique<FEffectAudioContext>(*static_cast<FEffectAudioContext*>(ContextPtr)); 
            return; 
        }
        if (ContextPtr->ContextType == EEffectContextType_Skeletal) { 
            Context = MakeUnique<FSkeletalMeshEffectContext>(*static_cast<FSkeletalMeshEffectContext*>(ContextPtr)); 
            return; 
        }
        Context = MakeUnique<FEffectContext>(*ContextPtr);
    }
};
```

**Virtual clone 的替代**:C++ 没有"自动深克隆虚对象"。一种办法是每个类实现 `virtual Clone()`;Kuro 选了另一种——一个集中的工厂函数,**按 `ContextType` 标签 dispatch**。

优点:不用改现有类,新增类型只要在这里加一个分支。
缺点:这里成为**加法性约定**——添加新 ContextType 必须来改这个函数,否则会退化成 Base。

典型使用场景:业务调 `SpawnEffect` 时传一个 `FEffectContext*`(可能是派生类),系统不能持 raw 指针(业务可能马上释放),需要自己拷贝一份 `TUniquePtr`。通过这个 Helper 拷贝后,Handle 用 `TUniquePtr<FEffectContext> Context` 持有,类型信息保留在 ContextType 里。

## 双版本模式(非常重要!)

每个 Context 类都有两个构造器:
```cpp
FEffectContext();                                  // 空构造
FEffectContext(const FKuroEffectContext& Context); // 从 USTRUCT 构造
```

**两个并列存在的版本**:
| 类型 | 位置 | 用途 |
|---|---|---|
| `FEffectContext` (pure C++ class) | 本文件 | 运行时 hot path |
| `FKuroEffectContext` (USTRUCT) | `KuroEffectSystemNew/Reflection/KuroEffectDefine.h` | 脚本(TS/BP)可见 |

这呼应了 Old 的 [[entities/project-game/kuro-effect-system/effect-parameters|FKuroEffectNiagaraParameters]] 里那段注释——**"脚本层用 USTRUCT,运行时层用纯 C++"**。跨层边界用一次性转换(这里的 copy ctor)代替每次反射。

## 全家族使用矩阵

| 特效类型 | Context 类 |
|---|---|
| Niagara / StaticMesh / Decal / Light / PostProcess / Billboard 等常规 | `FEffectContext` |
| Audio | `FEffectAudioContext` |
| Trail / SkeletalMesh(非 Ghost)/ 骨骼 attach | `FSkeletalMeshEffectContext` |
| Ghost(骨骼残影) | `FEffectRuntimeGhostEffectContext` |

## 与其他实体的关系

- **被 Handle 持有**:`TUniquePtr<FEffectContext> Context` in [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]
- **Spawn 时创建**:`FEffectSystem::SpawnEffectWithoutActor` / `SpawnEffectWithActor` 的参数 `TUniquePtr<FEffectContext>& Context`
- **`FKuroEffectContext` 等反射版本** 在 Puerts Reflection 层定义,Batch 5 详读

## Twin(Old 版对应)

**Old 没有独立的 Context 类**。功能散落在:
- `FEffectHandle` 里的标志位(IsFreeze, SeekXxx)
- `FKuroEffectSystem::RegisterEffectHandle` 的过长参数列表(每种 DA 类型不同签名)
- TS 侧自己维护 "特效 Id → 创建者" 的映射

New 把这些信息**集中成一个继承对象**,好处:
1. Spawn 接口签名一致(都是 `TUniquePtr<FEffectContext>&`),不需要为每种 DA 写专用重载
2. Handle 有清晰的"创建意图档案",debug 工具可以可视化(`DebugPrintEffect` 等)
3. 扩展新类型只要加派生类,不改主系统 API

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectContext.h`(~145 行)
- 依赖:`Reflection/KuroEffectDefine.h`(`FKuroEffectContext` 等 USTRUCT),Batch 5 详读
- 使用者:Handle / System SpawnEffect

## 开放问题

- `HitEffectType` 具体值含义(可能是打击特效分类:切割/钝击/火/冰等)
- `CreateFromType` 是位掩码但只定义了 2 个值(An / Ans),预留的空间用于什么?
- `FEffectContextHelper::CreateUniqueContext` 的调用点在 .cpp 哪里(Spawn 路径),业务调 Spawn 时 Context 是否必须 UniquePtr<>(从签名看是引用,答案是 Yes)
- 为什么 Helper 是 `static` 函数而不是 `virtual Clone()` 在基类?(可能是为了保持 Context 类纯 POD 化,不引入 vtable 开销——虽然它们已经有 virtual dtor 所以已经有 vtable 了)
