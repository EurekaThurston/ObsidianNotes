---
type: entity
category: class
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, pool, player]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/PlayerEffectContainer.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FPlayerEffectContainer, PlayerEffectPool, FSceneTeamItem]
---

# FPlayerEffectContainer(玩家作用域特效池)

> New 独有。给"**当前队伍/玩家**"相关的特效配的分池 LRU 容器——多个玩家各自一个小池,避免不同玩家的特效互相挤占池容量,也便于切队伍时批量清理。

## 为什么需要独立池

Kuro 是联机动作游戏,一个场景里可能有:
- **玩家自己**(主控角色)+ 她的武器、宠物、Buff 球
- **队友 1/2/3**(远程联机,由联机层同步)
- 每个角色身上的特效有"**我独占**"的特性(比如主角技能的粒子,别人看不到)

**问题**:如果所有特效共用一个 LRU 池:
- 主角技能特效把池挤爆后,队友的特效就得实时创建(卡)
- 切队时主角特效可能被别人的特效挤出池

**解决方案**:**每个玩家一个独立小池**。主角有自己的 pool,队友 1/2/3 各有自己的 pool,互不干扰。

## 配套类型

### `FSceneTeamItem`
```cpp
struct FSceneTeamItem
{
    int32 EntityId;
    bool IsMyRole;    // 是"我"吗
};
```

轻量的"场景队伍成员"标识:一个 EntityId + 是不是我。`FEffectSystem::OnPlayerEffectContainerFormationLoaded(TArray<FSceneTeamItem>&)` 接收这个列表——**联机场景切队时会广播**。

### `FPlayerEffectContainer`

```cpp
class FPlayerEffectContainer
{
private:
    static int32 SCENE_TEAM_MAX_NUM;                        // 队伍最大人数
    static int32 PLAYER_EFFECT_LRU_CAPACITY;                // 每个玩家池容量
    static int32 LOW_MEMORY_PLAYER_EFFECT_LRU_CAPACITY;     // 内存紧张时降级容量
    
    TArray<TUniquePtr<TLru<FName, FEffectHandle>>> PlayerEffectPoolArray;  // N 个池
    TArray<int32> EffectPoolOriginIndexArray;                              // 池编号映射
    TArray<int32> SourceInPositionArray;                                   // 谁占哪个位置
    TArray<int32> PositionFilledWithSourceArray;                           // 反向查找
    TArray<int32> PoolFormationEntities;                                   // 当前编队成员
    TArray<int32> OldPoolFormationEntities;                                // 上次编队
    TArray<bool>  HoldPoolIndexArray;                                      // 哪些位是保留的
    
    int32 GetIndexOfFormation(int32 EntityId);
    int32 GetIndexOfFormationWithContext(TUniquePtr<FEffectContext>& Context);

public:
    FPlayerEffectContainer() {}
    ~FPlayerEffectContainer() {}
    
    void Initialize();
    void Clear();
    void ClearPool();
    bool CheckGetCondition(TUniquePtr<FEffectContext>& Context);
    int32 GetPlayerEffectPoolSize(int32 Position);
    TSharedPtr<FEffectHandle> CreateEffectHandleFromPool(const FName& Path, TUniquePtr<FEffectContext>& Context, void* Parameter = nullptr);
    TSharedPtr<FEffectHandle> GetEffectHandleFromPool(const FName& Path, TUniquePtr<FEffectContext>& Context);
    bool PutEffectHandleToPool(const TSharedPtr<FEffectHandle>& Handle);
    bool LruRemoveExternal(const TSharedPtr<FEffectHandle>& Handle);
    void OnFormationLoaded(TArray<FSceneTeamItem>& SceneTeamItems);
};
```

## 核心数据结构拆解

### `PlayerEffectPoolArray`(**关键**)
`TArray<TUniquePtr<TLru<FName, FEffectHandle>>>` —— **外层数组**,元素是**"按 Path 组织的 LRU 池"**。

- 数组索引 = **队伍位置**(0=我,1=队友 1,2=队友 2,3=队友 3 ...)
- 每个位置的 LRU 池按 FName(特效 Path)索引 FEffectHandle
- `TUniquePtr` 独占所有权

### `EffectPoolOriginIndexArray` + 4 个位置追踪数组
这几个数组一起维护"**EntityId ↔ Position 的动态映射**":
- 当玩家 X 加入队伍进入位置 P:`SourceInPosition[P] = X`,`PositionFilledWithSource[X] = P`
- 当玩家 Y 替换 X 离开:清空 X 的池(OldPoolFormation)、把 Y 的数据装进该位置的池
- **Pool 本身不变**(避免重分配,复用容量);只是"拥有者"身份变了

`HoldPoolIndexArray`——**"保留位"**。可能用于"某玩家暂时掉线但别立刻清池,等他回来复用"的场景。

## 关键方法语义(基于签名推断)

### Initialize / Clear / ClearPool
- `Initialize` 启动时配池数量 = `SCENE_TEAM_MAX_NUM`(应为 4),每个池容量 = `PLAYER_EFFECT_LRU_CAPACITY`
- `Clear` 彻底清空所有数据结构
- `ClearPool` 只清池内容,不动映射表(用于"所有玩家战斗状态切换时清理特效"等场景)

### CheckGetCondition
```cpp
bool CheckGetCondition(TUniquePtr<FEffectContext>& Context);
```
—— 快速判断"**这个 Context 应该走玩家池吗**"。
看 `FEffectContext::EntityId` 是否属于当前 Formation(队伍),是就 true。

### GetEffectHandleFromPool / CreateEffectHandleFromPool
```cpp
TSharedPtr<FEffectHandle> GetEffectHandleFromPool(Path, Context);       // 池里有就取,没有返 nullptr
TSharedPtr<FEffectHandle> CreateEffectHandleFromPool(Path, Context, ...); // 池里有就取,没有 MakeShared
```

**典型调用**:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem::TryCreateFromContainer]] 先问这个容器有没有;有 → 从玩家池取(带 [[entities/project-game/kuro-effect-system/effect-define|CreateSource = EEffectHandleCreateSource_PlayerEffectPool1/2/3]] 标记),没有 → 再走全局 LRU → 最后才新建。

### PutEffectHandleToPool
回池。Handle Stop 完调这个接口试图放回。

### LruRemoveExternal
强制从 LRU 剔除某 Handle(外部操作,比如业务逻辑显式销毁特效)。

### OnFormationLoaded(**最重要的外部接口**)
```cpp
void OnFormationLoaded(TArray<FSceneTeamItem>& SceneTeamItems);
```

联机系统**队伍构成变化**时调用——比如:
- 副本开始,队伍成员确定下来
- 副本中有人退出或加入
- 切换角色

实现层会:
1. 比较 `PoolFormationEntities`(现在)和 `OldPoolFormationEntities`(之前)
2. 离队的玩家 → 清掉对应池(或标 Hold 保留)
3. 新成员进队 → 给他分配一个空闲池位置
4. 位置不变的 → 啥也不做(池原地保留,无缝切换)

## 使用场景(从代码和命名推断)

**场景 1:角色切换**
- 玩家从 A 角色切到 B 角色,场景队伍变化
- A 的特效预加载池被清空 → 释放内存
- B 的池被激活或 primed

**场景 2:联机同步**
- 队友玩家上线,分配池位置
- 每个队友的独占特效独立管理,不互相影响

**场景 3:内存紧张降级**
- 内存压力大时,`PLAYER_EFFECT_LRU_CAPACITY → LOW_MEMORY_PLAYER_EFFECT_LRU_CAPACITY`
- 各玩家池容量变小,但不全丢

## EEffectHandleCreateSource 的关联

回顾 [[entities/project-game/kuro-effect-system/effect-define|EffectDefine]] 的 `EEffectHandleCreateSource`:

```cpp
enum EEffectHandleCreateSource
{
    EEffectHandleCreateSource_None,
    EEffectHandleCreateSource_Lru,              // 全局 LRU
    EEffectHandleCreateSource_PlayerEffectPool,  // 玩家池通用
    EEffectHandleCreateSource_PlayerEffectPool1, // 位置 1 (队友 1?)
    EEffectHandleCreateSource_PlayerEffectPool2,
    EEffectHandleCreateSource_PlayerEffectPool3,
};
```

**4 个 PlayerEffectPool 枚举值对应 4 个池位置**——验证了 `SCENE_TEAM_MAX_NUM = 4`。

## 与其他实体的关系

- **被** [[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]] **持有**:`static FPlayerEffectContainer PlayerEffectContainer;`
- **调用者**:
  - `FEffectSystem::TryCreateFromContainer` / `TryRecycleToContainer`
  - `FEffectSystem::OnPlayerEffectContainerFormationLoaded(TArray<FSceneTeamItem>&)` 入口
- **依赖**:
  - [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle]]
  - [[entities/project-game/kuro-effect-system-new/effect-context|FEffectContext]](查 EntityId)
  - `TLru<FName, FEffectHandle>` — Kuro 自定义 LRU
- **外部输入**:联机系统的队伍状态变化事件

## Twin(Old 版对应)

**Old 没有玩家作用域池**。所有特效同一个池(甚至没有池——就是 TMap\<int, FEffectHandle*\>)。联机场景下主角/队友特效相互挤占、切角色泄漏都是 Old 的痛点。

**New 的这个容器是"联机 + 多玩家"架构演进的产物**。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\PlayerEffectContainer.h`(~51 行)
- 实现:`.cpp`(Batch 6)
- 使用者:[[entities/project-game/kuro-effect-system-new/effect-system|FEffectSystem]]、联机层

## 开放问题

- `SCENE_TEAM_MAX_NUM` 的默认值?(推测 4,看 EEffectHandleCreateSource 枚举数量)
- `GetIndexOfFormationWithContext` 怎么从 Context 取 EntityId 映射到位置?(Batch 6 .cpp)
- `HoldPoolIndexArray`:保留位的具体触发条件(玩家切换角色 vs 真正离队)
- `CheckGetCondition`:**主角的特效要走池吗**?还是只有队友相关的走?主角的资源一般本地常驻,可能不需要池化
- 低内存模式怎么触发?(`FGenericPlatformMisc::GetFreeMemoryMB` 监控?)
- 和 `FEffectSystem::LruPool`(全局 LRU)的优先级关系——推测顺序:先查 PlayerEffectContainer,没有再查 LruPool,再没有才 new
