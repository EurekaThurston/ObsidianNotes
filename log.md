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
