# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

## [2026-04-17] bootstrap | 仓库初始化
- 按 Karpathy LLM Wiki 方法论搭建三层架构
- 新建目录:`raw/{articles,papers,books,notes,assets}`、`wiki/{entities,concepts,sources,syntheses}`
- 新建:[[CLAUDE]](schema)、[[index]](目录)、[[log]](本文件)、[[wiki/overview]](占位)
- 源文档:`Karpathy Wiki 方法论.md`(暂放根目录)
- 下一步:等待第一个 raw source。

## [2026-04-17] ingest | Karpathy — LLM Wiki
- source: [[raw/notes/Karpathy Wiki 方法论]](从根目录迁入 raw/notes/)
- 新建:
  - [[wiki/sources/karpathy-llm-wiki]]
  - [[wiki/concepts/llm-wiki-方法论]]
  - [[wiki/concepts/rag]]
  - [[wiki/concepts/memex]]
  - [[wiki/entities/karpathy]]
- 更新:[[index]](登记 5 个新页)、[[wiki/overview]](首次有实质内容,确立"当前主题=方法论自举")、[[CLAUDE]](修正 wikilink 指向 raw/notes)
- 要点:自举式 ingest——用这套方法论本身 ingest 这套方法论。wiki 现在可以正式运转。
- 下一步:扔进来任意一个真实 source,验证流程。

## [2026-04-17] ingest | KuroEffectSystem — Batch 1(骨架 + 入口类)
- source: 项目游戏代码 `project-game`,p4 CL 6985991 (Old) / 7086024 (New)
- 路径:
  - `Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/` (Old)
  - `Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/` (New)
- 读入:8 个 Public 头(入口类 ×2、Handle ×2、HandleInfo ×1、LifeTime ×2、HandleHelper ×1)
- 新建:
  - [[wiki/sources/project-game/kuro-effect-system/overview]]
  - [[wiki/sources/project-game/kuro-effect-system-new/overview]]
  - [[wiki/entities/project-game/kuro-effect-system/kuro-effect-system]](FKuroEffectSystem)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-system]](KuroEffect::FEffectSystem)
  - [[wiki/entities/project-game/kuro-effect-system/effect-handle]](Old FEffectHandle)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-handle]](New FEffectHandle)
- 更新:[[index]](登记 6 个代码页 + 2 个 source 分类重排)、[[log]]
- 4 对 `twin:` 孪生链:2 个 source overview 对、2 对 entity(入口类 + Handle)
- 关键发现(仅列出本批已证实的,未推测):
  - **架构级重写**:Old instance / 裸指针 / v8 耦合 → New 全 static / `TSharedPtr`+LRU / ScriptBridge 抽象
  - **DataAsset 接入方式变化**:Old 7 个类型化 `Init` 重载 → New 单个 `ctor(FName Path)` + `AfterConstruct(FEffectSpecData)` 两阶段
  - **生命周期扩展**:Old `Init/Play/Stop` → New `Init/Start/End/Clear/Destroy/Play/PlayEffect/PreStop/Stop/Replay/AfterLeavePool/OnEnabledChange` 更细粒度
  - **New 独有新概念**:`FEffectContext` / `FEffectInitModel` / `FEffectInitHandle` (Pending) / `FPlayerEffectContainer` / `FContinuousEffectController` / `EEffectHandleCreateSource` / `SourceEntityId` / OwnerEffect / AdditionTimeScale / BodyEffect / PIE 钩子 / GameBudget 集成
  - **New 代码注释提示**:`"TsEffectHandle 完全迁移"` → 部分业务逻辑以前在 TS 侧,此次下沉 C++
- 下一步:Batch 2 DataAsset 家族(18 个 `UEffectModelXxx` + `EffectParameters/` + `EffectLifeTime` 两版独立实体页 + `FEffectSpecData`)
- 注:所有 `source_commit` 用 p4 CL 号;因 p4 无等价 `git rev-parse`,按文件路径下最新变更 CL 取。

## [2026-04-17] ingest | KuroEffectSystem — Batch 2(数据模型 & 初始化管线)
- source: 同 Batch 1
- 读入:28 个 Public 头
  - Old: EffectDefine.h、EffectDataAsset/ 13 个核心 DA(Niagara/Group/MultiEffect/Ghost/MaterialController/PostProcess/Trail/StaticMesh/Decal/Audio/Billboard/Light/SkeletalMesh/SequencePose/CurveTrailDecal/GpuParticle/NDC)、EffectParameters/ 4 个
  - New: EffectSpecData.h、EffectInitModel.h、EffectInitHandle.h、EffectContext.h、EffectActorHandle.h(EffectLifeTime.h 已 Batch 1 读)
- 新建 10 页:
  - [[wiki/entities/project-game/kuro-effect-system/effect-define]] — EffectDefine 枚举/Delegate 参考(两版共享)
  - [[wiki/entities/project-game/kuro-effect-system/effect-model-base]] — 18 DataAsset 分类表
  - [[wiki/entities/project-game/kuro-effect-system/effect-parameters]] — Old Parameters 4 类
  - [[wiki/entities/project-game/kuro-effect-system/effect-handle-info]] — Old JS 伴生体
  - [[wiki/entities/project-game/kuro-effect-system/effect-life-time]] — Old 时间状态机
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-data]] — New 数据索引层(FEffectSpecData / FEffectSpecChildData)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-init-pipeline]] — FEffectInitModel + FEffectInitHandle + 4 阶段 Spawn 流程
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-context]] — FEffectContext 4 层继承 + Helper
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-actor-handle]] — FEffectActorHandle + FEffectActorAction 命令模式
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-life-time]] — New 时间状态(Timer 驱动,去 v8)
- 更新:[[index]](+10 条)、[[log]]
- 1 对新增 twin 链:LifeTime Old/New
- 关键发现:
  - **3 段时间模型**(StartTime + LoopTime + EndTime)是整套系统的节奏轴,围绕它构建所有状态切换
  - **Old 的 Parameters 一分为三**(裸 struct + Base class + Niagara 独立派);Niagara 独立是为了脚本传参性能(注释明示:反射 40-50μs)
  - **New 数据层四层分离**:FEffectSpecData(db 元)→ FEffectInitModel(意图)→ FEffectInitHandle(进度)→ FEffectContext(运行上下文)→ IEffectSpecBase(运行 Spec)
  - **Pending Init 机制**的核心基础设施:FEffectInitHandle 持 FEffectActorHandle,FEffectActorHandle 内部用命令模式延后执行 Attach/Hidden 等操作,Actor 创建完成时 flush
  - **Old 大量 v8 渗透**(Info/LifeTime),New 全部抽走到 ScriptBridge
  - **双版本 Context**:FEffectContext(C++ 运行时)/ FKuroEffectContext(USTRUCT,脚本可见),ctor 做一次性转换
  - 非常大的 DA:`UEffectModelPostProcess` 700+ 行,涵盖天气/色调/灰度/VHS/BurstDissolve 等十几项后处理子效果
- 下一步:Batch 3 — Spec 系统(`FEffectSpecBase` + 各类型 hpp,`IEffectSpecBase` + `FEffectSpecFactory`)

## [2026-04-17] ingest | KuroEffectSystem — Batch 3(Spec 运行时系统)
- source: 同 Batch 1
- 读入:13 个文件(Old EffectSpec.hpp + 5 个 Spec 子类;New IEffectSpecBase.h + EffectSpec.hpp + EffectSpecFactory.h + 3 个 Spec 子类)
- 新建 5 页:
  - [[wiki/entities/project-game/kuro-effect-system/effect-spec-base]] — Old FEffectSpecBase + FEffectSpec<T> CRTP 模板 + Template Method 模式讲解 + 3 子类 exemplar
  - [[wiki/entities/project-game/kuro-effect-system/effect-spec-subclasses]] — Old 12 子类目录表
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-base]] — New IEffectSpecBase 80+ 虚接口 + FEffectSpec<T> 1150 行模板 + AdditionTimeScale / BodyEffect 机制
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-factory]] — FEffectSpecFactory 工厂
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-spec-subclasses]] — New 17 子类目录(新增 Audio/CurveTrailDecal/MaterialController/NDC/SequencePose)
- 更新:[[index]](+5 条)、[[log]]
- 1 对新增 twin 链:Spec 基类 Old/New
- 关键发现:
  - **架构模式**:两版都用"抽象(Base/Interface)+ CRTP 模板 + 子类"三层。Old 的 Base 是 `FEffectSpecBase`(含 LifeTime 字段),New 的是纯接口 `IEffectSpecBase`
  - **接口规模爆炸**:Old 6 纯虚 → New 80+ 纯虚。New 把之前散落在 System/Handle 的方法(Init/Start/End/Clear/Destroy/Play/PreStop/Stop/Replay/PreFirstPlay/OnParentInit/OnBeginDelayPlay 等)全部拉到 Spec 接口
  - **AdditionTimeScale 机制**(New 独有):`TMap<int32, float> AdditionTimeScaleMap`,多来源乘法叠加,帧内缓存(`LastUpdateTotalAdditionTimeScale == GFrameCounter`)
  - **Factory 集中化**:`FEffectSpecFactory::CreateEffectSpec(FEffectSpecData)` 替代 Old 里散落在 RegisterEffectHandle 7 重载的 `new` 调用——加新类型从"改 5 处"降到"改 2 处"
  - **异步 Init 状态**:`EEffectSpecInitState_Fail/Success/Initializing` 三态;Group::OnInit 返回 Initializing 并等所有 child SpawnChildEffect 完成
  - **DelayPlay 的 float 值**:回答 Batch 2 开放问题——`UEffectModelGroup::EffectData` map 的 float 值就是 **child 延迟播放秒数**
  - **MultiEffect 管控权转移**:Old 靠 TS 回调 (MultiEffectAdjustNumber) 增删子特效,New 的 Spec 直接 SpawnEffect/StopEffectById 自管
  - **WaitPostTick 机制**(Niagara):所有 SetPaused 延后到帧末 OnPostTick,避免 tick 中途状态不一致
  - **CVars**:New 引入 `Kuro.CppEffectSystem.*` 系列 6+ 个运行时开关,线上可控
  - **ScriptBridge 调用**:BodyEffect 注册/反注册经过 `FEffectSystem::EffectSystemScriptBridge`(Batch 5 细读)
- 下一步:Batch 4 — New 独有新概念(FPlayerEffectContainer / FContinuousEffectController / AEffectSystemActor / FNiagaraComponentHandle)

## [2026-04-17] ingest | KuroEffectSystem — Batch 4(New 独有新概念)
- source: 同 Batch 1
- 读入 4 个新概念 header:PlayerEffectContainer.h、ContinuousEffectController.h、EffectSystemActor.h、NiagaraComponentHandle.h
- 新建 4 页(**全部 New 独有,无 Old twin**):
  - [[wiki/entities/project-game/kuro-effect-system-new/player-effect-container]] — 玩家作用域 LRU 分池(队伍 N 人各自一池)+ FSceneTeamItem + OnFormationLoaded
  - [[wiki/entities/project-game/kuro-effect-system-new/continuous-effect-controller]] — AnsSlot 维度的连续特效过渡(前特效柔停 + Pending 等帧数)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-system-actor]] — AActor 派生,含 HandleId/Path/EffectType/TimeScale/OwnerEntityId,6 个 UFUNCTION(BlueprintCallable)
  - [[wiki/entities/project-game/kuro-effect-system-new/niagara-component-handle]] — Niagara 参数缓存门面(8 种 Map + 3 种 Emitter Array),Pending 期间延迟下发
- 更新:[[index]](+4 条)、[[log]]
- 关键发现(全部 New 独有能力):
  - **玩家分池**:每个玩家(主角/队友 ×3)一个独立 LRU 池(`TArray<TUniquePtr<TLru<FName, FEffectHandle>>>`),队伍变化时 OnFormationLoaded 重组
  - **AnsSlot 连续特效**:`TMap<SkeletalMesh, TMap<SlotName, HandleId>>` 维护当前活跃特效;新特效 Spawn 时柔停旧的;`MaxWaitContinuousEffectFrame` 超时兜底
  - **AEffectSystemActor 蓝图可调**:6 个 UFUNCTION 直接暴露给蓝图,兼容旧 BP_EffectActor 模式
  - **双层 TimeDilation 下发**:UE 原生 CustomTimeDilation + AEffectSystemActor::TimeScale(特效逻辑用)
  - **FNiagaraComponentHandle 解答 Pending Init 的参数问题**:Component 未创建前,参数先缓存到 TMap,Component Init 时一次性 flush
  - **双参数命名空间**:Variable(FString Key,对应 Niagara User.XXX)vs Parameter(FName Key,对应渲染参数 override)
  - **TUniquePtr\<TMap\> 懒分配**:大部分特效只用 1-2 种参数,空 Map 不分配内存
  - **GlobalAlphaBodyOpacity**:Niagara Handle 的字段,配合 BodyEffect 做身上特效 Alpha 混合
- 下一步:Batch 5 — ScriptBridge & Reflection(Puerts 集成)—— 读 ScriptBridge/、Reflection/、EffectSystemHandleHelper、EffectSystemForPuerts

## [2026-04-17] ingest | KuroEffectSystem — Batch 5(ScriptBridge & Reflection)
- source: 同 Batch 1
- 读入 9 个文件:ScriptBridge/ 5 个、Reflection/ 3 个、EffectSystemForPuerts.h
- 新建 5 页(**全部 New 独有**):
  - [[wiki/entities/project-game/kuro-effect-system-new/scripting-bridge-architecture]] — 脚本桥架构总览(~10 类协作)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-system-script-bridge]] — 核心三件套(门面 + JsBridge + CSharpBridge)
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-function-holders]] — 6 类 Callback Holder(3 业务场景 × 2 语言)
  - [[wiki/entities/project-game/kuro-effect-system-new/kuro-effect-reflection]] — 29 Dynamic Delegate + USTRUCT Context / SpecData / Parameters
  - [[wiki/entities/project-game/kuro-effect-system-new/effect-bp-libraries]] — BP 函数库(~130 UFUNCTION)
- 更新:[[index]](+5 条)、[[log]]
- 关键发现:
  - **分层架构**:BP/脚本层 → BP FunctionLib(反射/marshal)→ ScriptBridge 门面 → JsBridge/CSharpBridge → Spec/Handle/System 原生 C++
  - **双脚本宿主**:Puerts(v8)+ C#,各 30 个回调注册点,门面 dispatch
  - **29 个上行 API**:脚本提前注册,C++ 在运行时查询(IsNetPlayer / GetEntityOwnerActor / IsNeedQualityBias 等)
  - **6 个 Holder 类**:每业务场景(Spawn/Finish/DynamicInit)× 每语言(JS/C#)一个;JS 版多一个 `JsObjectGlobal` 字段作为 this
  - **TWeakPtr 打破 Callback 循环所有权**:Model 强持 Callback,Callback bind 到 Holder thunk,Holder 持 Weak 指回 Callback
  - **USTRUCT 双版本**:FEffectContext(C++ 原生)↔ FKuroEffectContext(USTRUCT);Parameter 和 SpecData 同样双版本,ctor 做一次性 marshal
  - **BP 友好类型转换**:FString→FName、enum→uint8、TUniquePtr→值 USTRUCT、UClass*→TSubclassOf;反射层做脏活
  - **12 个 SpawnEffect 重载**:4 种 Context × 3 种 Spawn(Unlooped/Normal/WithActor)——蓝图不支持结构体多态
  - **Puerts 绑定**:`UsingCppType` 模板宏,C++ 类型零开销 passthrough 到 TS(vs UE 反射的 marshal 开销)
  - **为什么分层**:核心热路径纯 C++(性能),脚本友好层做 marshal(易用性),两者并存不互相污染
- 下一步:Batch 6 — Private 实现精读(选 6-8 个关键 cpp 读,补充 "关键代码片段")
