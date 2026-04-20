# Index — Wiki 目录

> 本文件是整个 wiki 的内容目录。LLM 每次 ingest 都会更新。
> 查询时先读这里,再深入相关页面。

最后更新:2026-04-21（+ Multi-agent 对话 ingest:1 Raw + 1 Source + 1 Concept,subagent 架构从"Ai-agent 一小节"升级为独立概念页）

---

## Overview
- [[Wiki/Overview]] — 当前主题:4 大主题并行（方法论自举 / Niagara / AI 美术 / AI 应用生态）

## Entities(实体)
*人、组织、地点、产品、项目。*

### 方法论相关
- [[Wiki/Entities/Methodology/Karpathy|Andrej Karpathy]] — AI 研究者;LLM Wiki 方法论提出者、Context Engineering 倡导者、Vibe Coding 命名者 (来源:2)
- [[Wiki/Entities/Claudian|Claudian]] — 本仓 LLM 作者身份;按 [[CLAUDE]] 规程运行,负责 ingest/query/lint/synthesis 全部 wiki 操作 (来源:1)

### AI 美术（工具与基座）
- [[Wiki/Entities/AIArt/Illustrious-XL|Illustrious XL]] — SDXL 架构二次元基座,2026 初二次元 SOTA 之一 (来源:1)
- [[Wiki/Entities/AIArt/NoobAI-XL|NoobAI XL]] — Illustrious 的社区继续训练版,许可更友好 (来源:1)
- [[Wiki/Entities/AIArt/Flux|Flux.1]] — BFL 的 12B 写实 SOTA,Dev 版非商用 (来源:1)
- [[Wiki/Entities/AIArt/Kohya-ss|Kohya-ss]] — 主流 LoRA 训练工具,Windows 首选 (来源:1)
- [[Wiki/Entities/AIArt/ComfyUI|ComfyUI]] — 节点式推理平台,生产管线首选 (来源:1)

### AI 应用生态（产品 / 项目）
- [[Wiki/Entities/AIApps/OpenClaw|OpenClaw（小龙虾）]] — 开源的本地部署个人 AI Agent 系统，记忆+主动+行动三件套 (来源:1)

### Niagara 代码实体（stock / UE 4.26）
- [[Wiki/Entities/Stock/UNiagaraSystem|UNiagaraSystem]] — Niagara 特效顶层资产;EmitterHandles + System 脚本 + 编译产物 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraEmitter|UNiagaraEmitter]] — Emitter 资产;脚本/渲染器/模拟参数/继承-merge 机制 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraEmitterHandle|FNiagaraEmitterHandle]] — System 引用 Emitter 的 USTRUCT 包装器;Id + Name + enabled + Instance (来源:1)
- [[Wiki/Entities/Stock/UNiagaraScript|UNiagaraScript]] — 编译后脚本资产;承载 VectorVM 字节码 + GPU shader 双形态 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase|UNiagaraScriptSourceBase]] — 图源抽象基类(editor-only 实现在 NiagaraEditor 模块) (来源:1)
- [[Wiki/Entities/Stock/UNiagaraComponent|UNiagaraComponent]] — Niagara 承载层组件;Asset 持有 + Instance 管理 + 参数覆盖 + 场景集成 + 生命周期调度 五职责 (来源:1)
- [[Wiki/Entities/Stock/ANiagaraActor|ANiagaraActor]] — 纯 ComponentWrapperClass;仅作 Component 挂载壳 + "播完自销毁" observer (来源:1)
- [[Wiki/Entities/Stock/UNiagaraFunctionLibrary|UNiagaraFunctionLibrary]] — Niagara BP 静态工具集;Spawn / 重型 DI 覆盖 / ParameterCollection / VM FastPath 四组 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraSystemInstance|FNiagaraSystemInstance]] — Niagara 运行时心脏;双状态机 + 三阶段 Tick + Emitter 容器;非 UObject (来源:1)
- [[Wiki/Entities/Stock/FNiagaraEmitterInstance|FNiagaraEmitterInstance]] — Emitter 运行时;持粒子 DataSet + Spawn/Update ExecContext(或 GPU Context) (来源:1)
- [[Wiki/Entities/Stock/FNiagaraSystemSimulation|FNiagaraSystemSimulation]] — 同 Asset 多实例批量 Tick 调度器;身份 (Asset, World, TickGroup);TickBatch=4 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraTypeDefinition|FNiagaraTypeDefinition]] — Niagara 类型系统身份;对应 UStruct/UEnum/UClass/内置基础类型;静态 Getter 单例 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraVariable|FNiagaraVariable]] — 命名的数据(TypeDef + Name + VarData);Niagara 世界里所有参数/属性的统一表示 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraTypeLayoutInfo|FNiagaraTypeLayoutInfo]] — 把任何类型拍成 Float/Int32/Half 三路 component offset;SoA 拍平的桥梁 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraConstants|FNiagaraConstants]] — 约 70+ 预定义 Engine/System/Emitter/Particle 变量注册中心 + 命名空间 FName 常量 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraDataSet|FNiagaraDataSet]] — 粒子数据 SoA 存储;CurrentData/DestinationData 双 buffer + 2-3 buffer 池 + Persistent ID 系统 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraDataSetAccessor|FNiagaraDataSetAccessor]] — 类型安全模板家族;Float/Int32/Struct 三家族 + Half 自动降级 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraParameterStore|FNiagaraParameterStore]] — 运行时参数存储;参数 byte + DI + UObject 三类 + 绑定图 + 三路 dirty (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceBase|UNiagaraDataInterfaceBase]] — DI 抽象基类;住在 NiagaraCore 模块方便 Shader 依赖;CS 参数绑定 non-virtual + LAYOUT_FIELD (来源:1)
- [[Wiki/Entities/Stock/FNiagaraScriptExecutionContext|FNiagaraScriptExecutionContext]] — CPU VM 执行上下文;Script + FunctionTable + Parameters + DataSetInfo;System 版带 PerInstanceFunctionHook (来源:1)
- [[Wiki/Entities/Stock/FNiagaraComputeExecutionContext|FNiagaraComputeExecutionContext]] — GPU 执行上下文(Phase 8 详);MainDataSet + GPUScript_RT + CombinedParamStore + SimStageInfo (来源:1)
- [[Wiki/Entities/Stock/FNiagaraGPUSystemTick|FNiagaraGPUSystemTick]] — GT→RT 的 GPU tick 打包;InstanceData_ParamData_Packed + 5 种 UniformBuffer (来源:1)
- [[Wiki/Entities/Stock/NiagaraEmitterInstanceBatcher|NiagaraEmitterInstanceBatcher]] — RT 驻留的 GPU 总调度器;三 stage dispatch + InstanceCountMgr + SortMgr (来源:1)
- [[Wiki/Entities/Stock/UNiagaraRendererProperties|UNiagaraRendererProperties]] — Renderer Asset 基类(abstract);Platforms/SortOrderHint/AttributeBindings + 4 纯虚工厂 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraRenderer|FNiagaraRenderer]] — Renderer 运行时基类;GenerateDynamicData→SetDynamicData_RT→GetDynamicMeshElements 三阶段流程 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraSpriteRendererProperties|UNiagaraSpriteRendererProperties]] — Sprite(每粒子 billboard quad);17 VF slot/5 facing/Cutout/SubUV;CPU+GPU (来源:1)
- [[Wiki/Entities/Stock/UNiagaraRibbonRendererProperties|UNiagaraRibbonRendererProperties]] — Ribbon(粒子连成带);CPU only;RibbonId 多带/自适应 tessellation/双 UV channel (来源:1)
- [[Wiki/Entities/Stock/UNiagaraMeshRendererProperties|UNiagaraMeshRendererProperties]] — Mesh(每粒子 StaticMesh 实例化);14 VF slot;CPU+GPU;全粒子共享 LOD (来源:1)
- [[Wiki/Entities/Stock/UNiagaraLightRendererProperties|UNiagaraLightRendererProperties]] — Light(每粒子 SimpleLight);CPU only;不产 mesh batch,走 GatherSimpleLights (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterface|UNiagaraDataInterface]] — DI 主基类(Niagara 主模块);三路代码生成(VM/C++ lambda/HLSL)+ 绑定模板族 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceCurve|UNiagaraDataInterfaceCurve]] — 最简单 DI 示例;LUT 烘焙 + 可 Expose 为 UTexture 给材质共享 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceCamera|UNiagaraDataInterfaceCamera]] — 相机查询 + 距离排序;TickGroup prereqs + bRequireCurrentFrameData 性能开关 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceCollisionQuery|UNiagaraDataInterfaceCollisionQuery]] — 4 种查询(Sync/Async CPU + SceneDepth/DistanceField GPU) (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceStaticMesh|UNiagaraDataInterfaceStaticMesh]] — StaticMesh 采样;面积加权 + section 过滤;4 种 SourceMode (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceSkeletalMesh|UNiagaraDataInterfaceSkeletalMesh]] — 最复杂 DI;共享 SkinningData(引用计数+RWLock)+ 3 种 SkinningMode (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceTexture|UNiagaraDataInterfaceTexture]] — GPU only 2D 纹理采样;最小 DI 模板 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRenderTarget2D|UNiagaraDataInterfaceRenderTarget2D]] — RW DI 第一个示范;继承 UNiagaraDataInterfaceRWBase;Phase 10 基础 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraShader|FNiagaraShader 家族]] — Niagara GPU compute shader 体系;ShaderMapId / ShaderMap / ShaderType / DI 参数绑定 (来源:1)
- [[Wiki/Entities/Stock/FNiagaraGPUInstanceCountManager|FNiagaraGPUInstanceCountManager]] — 全局 GPU 粒子计数管理器;共享 CountBuffer + DrawIndirect 解决 CPU 不知 GPU count (来源:1)
- [[Wiki/Entities/Stock/FNiagaraGPUSort|FNiagaraGPUSort 家族]] — GPU 粒子排序;SortInfo + SortKeyGenCS(4 permutation)接入 FGPUSortManager (来源:1)
- [[Wiki/Entities/Stock/FNiagaraVertexFactory|FNiagaraVertexFactory 家族]] — 3 种 VF(Sprite/Ribbon/Mesh);粒子到像素的桥;Mesh 独占 tessellation (来源:1)
- [[Wiki/Entities/Stock/FNiagaraDrawIndirect|FNiagaraDrawIndirect 家族]] — Draw indirect args GPU 生成;TaskInfo 双用(gen + clear) (来源:1)
- [[Wiki/Entities/Stock/FNiagaraWorldManager|FNiagaraWorldManager]] — 每 World 一个;SystemSimulations × TickGroup + ScalabilityManagers × EffectType + Pool + GC hooks (来源:1)
- [[Wiki/Entities/Stock/FNiagaraScalabilityManager|FNiagaraScalabilityManager]] — 按 EffectType 的 cull 决策;4 类 cull + significance 排序;`bIsScalabilityCull=true` 不触发用户 delegate (来源:1)
- [[Wiki/Entities/Stock/UNiagaraComponentPool|UNiagaraComponentPool]] — Component 池化;5 种 ENCPoolMethod;按 Asset 分 FNCPool + PrimePool 预热 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraSettings|UNiagaraSettings]] — 项目级 UDeveloperSettings;DefaultEffectType / QualityLevels / 默认 RT/Grid format (来源:1)
- [[Wiki/Entities/Stock/UNiagaraEffectType|UNiagaraEffectType]] — 特效分类 + scalability 预算;4 类 ScalabilitySettings + SignificanceHandler + CullReaction (来源:1)
- [[Wiki/Entities/Stock/FNiagaraPlatformSet|FNiagaraPlatformSet]] — 跨平台 × quality × CVar 决策;双 mask 三态;conflict 检测 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraSimulationStage|UNiagaraSimulationStage]] — SimStages 多 pass 架构;IterationSource(Particles/DataInterface)+ Iterations 迭代数 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceRWBase|UNiagaraDataInterfaceRWBase]] — RW DI 基础;含 Grid2D/Grid3D abstract 基类 + ProxyRW(GetElementCount + 4 SimStage 钩子) (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid2DCollection|UNiagaraDataInterfaceGrid2DCollection]] — 2D 场网格(Texture2DArray);含 Reader 只读版跨 emitter 交互 (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceGrid3DCollection|UNiagaraDataInterfaceGrid3DCollection]] — 3D 体积场(RWTexture3D + tile 打包) (来源:1)
- [[Wiki/Entities/Stock/UNiagaraDataInterfaceNeighborGrid3D|UNiagaraDataInterfaceNeighborGrid3D]] — 空间哈希,SPH/邻域查询基础;MaxNeighborsPerCell lossy (来源:1)

## Concepts(概念)
*想法、理论、方法、术语。*

### 方法论
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 让 LLM 增量构建并维护持久化 wiki 的范式 (来源:1)
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 查询时检索文档片段的主流 LLM+文档范式；底层靠 Embedding 向量嵌入 (来源:2)
- [[Wiki/Concepts/Methodology/Memex|Memex]] — Vannevar Bush 1945 提出的个人关联式知识存储愿景 (来源:1)
- [[Wiki/Concepts/Methodology/Vibe-coding|Vibe Coding]] — Karpathy 2025-02 命名;AI 编程三阶段(Vibe/Spec/Harness)的起点 (来源:1)

### UE4 基础
- [[Wiki/Concepts/UE4/UE4-uobject-系统|UE4 UObject 系统]] — UCLASS/USTRUCT/UPROPERTY 宏、GC、反射、序列化
- [[Wiki/Concepts/UE4/UE4-资产与实例|UE4 资产与实例]] — Asset（磁盘，只读模板）vs Instance（运行时，有状态）二元模型
- [[Wiki/Concepts/UE4/UE4-ddc|UE4 DDC]] — Derived Data Cache,机器/团队级编译产物缓存;Niagara 编译身份证直接作 key (来源:1)

### Niagara 基础
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade|Niagara vs Cascade]] — 设计哲学对比：黑盒模块 vs 完全可编程数据流
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟|Niagara CPU vs GPU 模拟]] — VectorVM(CPU) 与 Compute Shader(GPU) 的分工、能力边界与源码分叉点

### AI 美术
- [[Wiki/Concepts/AIArt/Lora|LoRA]] — 冻结主模型,只训低秩矩阵;游戏美术 DNA 固化的核心技术 (来源:1)
- [[Wiki/Concepts/AIArt/Base-model-selection|基座模型选型]] — Illustrious/NoobAI/Flux 对比,选错全白干 (来源:1)
- [[Wiki/Concepts/AIArt/Caption-strategy|Caption 策略]] — 想让 LoRA 永远带的不写,按需触发的写清楚（反常识） (来源:1)
- [[Wiki/Concepts/AIArt/Trigger-word|Trigger Word]] — 激活 LoRA 的独特词,命名约定 (来源:1)
- [[Wiki/Concepts/AIArt/Multi-lora-composition|Multi-LoRA 组合]] — ComfyUI 真正威力,MJ 做不到的模块化风格控件 (来源:1)

### AI 应用生态
- [[Wiki/Concepts/AIApps/Llm|LLM]] — 大语言模型,本质是"词语接龙引擎";三步训练(Pretrain/SFT/RLHF) (来源:1)
- [[Wiki/Concepts/AIApps/Embedding|Embedding 向量嵌入]] — 把语义变成几何距离;RAG / 语义检索 / 多模态对齐的数学底座 (来源:1)
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]] — AI 一本正经胡说的结构性毛病;用户第一纪律:具体事实必核验 (来源:1)
- [[Wiki/Concepts/AIApps/Context-window|上下文窗口 & Context Rot]] — AI 的视力范围,塞满反而变笨;Skills 渐进式披露的根本动机 (来源:1)
- [[Wiki/Concepts/AIApps/Reasoning-model|推理模型]] — o1/R1/Extended Thinking,先打草稿再答题 (来源:1)
- [[Wiki/Concepts/AIApps/Ai-agent|AI Agent]] — 给 LLM 装手脚,思考→行动→观察的循环 (来源:2)
- [[Wiki/Concepts/AIApps/Multi-agent|Multi-agent / Subagent 架构]] — 主 Agent 派活给一次性子 Agent;第一性原理是上下文隔离,不是分工并行 (来源:1)
- [[Wiki/Concepts/AIApps/Mcp|MCP]] — Model Context Protocol,AI 世界的 USB-C 协议 (来源:1)
- [[Wiki/Concepts/AIApps/Harness-engineering|Harness Engineering]] — AI 的操作系统;四大支柱:上下文/约束/反馈/熵管理 (来源:1)
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — 专业知识打包为可复用模块,渐进式披露三层加载 (来源:1)

## Sources(源摘要)
*每个 raw 文件或代码源对应一页摘要。按时间倒序。*

### 文章 / 笔记
- [[Wiki/Sources/AIApps/Multi-agent-conversation]] — Multi-agent 对话,Claudian 解释自己的多 agent 工作机制 (2026-04,note)
- [[Wiki/Sources/AIApps/AI-primer-v2]] — AI 应用技术发展脉络与核心概念扫盲手册 v2 (2026-04,article)
- [[Wiki/Sources/Methodology/Karpathy-llm-wiki]] — Karpathy 的 LLM Wiki idea file (2026-04,note)
- [[Wiki/Sources/AIArt/Lora-deep-dive]] — LoRA 深度指南：给鸣潮美术向流水线的落地方案 (2026-04,note)

### 代码源摘要 — stock（UE 4.26 Niagara）
- [[Wiki/Sources/Stock/NiagaraSystem]] — NiagaraSystem.h @ b6ab0dee9 (Phase 1.1,Asset 层顶点)
- [[Wiki/Sources/Stock/NiagaraEmitter]] — NiagaraEmitter.h @ b6ab0dee9 (Phase 1.2,Emitter 资产)
- [[Wiki/Sources/Stock/NiagaraEmitterHandle]] — NiagaraEmitterHandle.h @ b6ab0dee9 (Phase 1.3,System→Emitter 引用包装)
- [[Wiki/Sources/Stock/NiagaraScript]] — NiagaraScript.h @ b6ab0dee9 (Phase 1.4,编译后脚本 + VMExecutableData)
- [[Wiki/Sources/Stock/NiagaraScriptSourceBase]] — NiagaraScriptSourceBase.h @ b6ab0dee9 (Phase 1.5,图源抽象基类)
- [[Wiki/Sources/Stock/NiagaraComponent]] — NiagaraComponent.h @ b6ab0dee9 (Phase 2.1,承载层主角,741 行)
- [[Wiki/Sources/Stock/NiagaraActor]] — NiagaraActor.h @ b6ab0dee9 (Phase 2.2,ComponentWrapperClass,66 行)
- [[Wiki/Sources/Stock/NiagaraFunctionLibrary]] — NiagaraFunctionLibrary.h @ b6ab0dee9 (Phase 2.3,BP 静态工具集)
- [[Wiki/Sources/Stock/NiagaraSystemInstance]] — NiagaraSystemInstance.h @ b6ab0dee9 (Phase 3.1,单实例心脏,574 行)
- [[Wiki/Sources/Stock/NiagaraEmitterInstance]] — NiagaraEmitterInstance.h @ b6ab0dee9 (Phase 3.2,Emitter 运行时,239 行)
- [[Wiki/Sources/Stock/NiagaraSystemSimulation]] — NiagaraSystemSimulation.h @ b6ab0dee9 (Phase 3.3,批量 Tick 调度器,429 行)
- [[Wiki/Sources/Stock/NiagaraTypes]] — NiagaraTypes.h @ b6ab0dee9 (Phase 4.1,类型系统,1739 行)
- [[Wiki/Sources/Stock/NiagaraCommon]] — NiagaraCommon.h @ b6ab0dee9 (Phase 4.2,共享枚举/结构,1200 行)
- [[Wiki/Sources/Stock/NiagaraConstants]] — NiagaraConstants.h @ b6ab0dee9 (Phase 4.3,常量注册中心,209 行)
- [[Wiki/Sources/Stock/NiagaraDataSet]] — NiagaraDataSet.h @ b6ab0dee9 (Phase 4.4,粒子 SoA 存储,554 行)
- [[Wiki/Sources/Stock/NiagaraDataSetAccessor]] — NiagaraDataSetAccessor.h @ b6ab0dee9 (Phase 4.5,类型安全 accessor,619 行)
- [[Wiki/Sources/Stock/NiagaraParameters]] — NiagaraParameters.h @ b6ab0dee9 (Phase 4.6,editor-only 参数列表,82 行)
- [[Wiki/Sources/Stock/NiagaraParameterStore]] — NiagaraParameterStore.h @ b6ab0dee9 (Phase 4.7,运行时参数存储,1260 行)
- [[Wiki/Sources/Stock/NiagaraCore]] — NiagaraCore.h @ b6ab0dee9 (Phase 5.1,模块入口,6 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceBase]] — NiagaraDataInterfaceBase.h @ b6ab0dee9 (Phase 5.2,DI 抽象基类,136 行)
- [[Wiki/Sources/Stock/NiagaraScriptExecutionContext]] — NiagaraScriptExecutionContext.h @ b6ab0dee9 (Phase 5.3,CPU VM + GPU 上下文,531 行)
- [[Wiki/Sources/Stock/NiagaraScriptExecutionParameterStore]] — NiagaraScriptExecutionParameterStore.h @ b6ab0dee9 (Phase 5.4,VM 参数 padding,195 行)
- [[Wiki/Sources/Stock/NiagaraEmitterInstanceBatcher]] — NiagaraEmitterInstanceBatcher.h @ b6ab0dee9 (Phase 5.5 / 实际 GPU Batcher,317 行)
- [[Wiki/Sources/Stock/NiagaraRendererProperties]] — NiagaraRendererProperties.h @ b6ab0dee9 (Phase 6.1,Renderer Asset 基类,259 行)
- [[Wiki/Sources/Stock/NiagaraRenderer]] — NiagaraRenderer.h @ b6ab0dee9 (Phase 6.2,Renderer 运行时基类,144 行)
- [[Wiki/Sources/Stock/NiagaraSpriteRendererProperties]] — NiagaraSpriteRendererProperties.h @ b6ab0dee9 (Phase 6.3,316 行)
- [[Wiki/Sources/Stock/NiagaraRendererSprites]] — NiagaraRendererSprites.h @ b6ab0dee9 (Phase 6.4,102 行)
- [[Wiki/Sources/Stock/NiagaraRibbonRendererProperties]] — NiagaraRibbonRendererProperties.h @ b6ab0dee9 (Phase 6.5,361 行)
- [[Wiki/Sources/Stock/NiagaraRendererRibbons]] — NiagaraRendererRibbons.h @ b6ab0dee9 (Phase 6.6,103 行)
- [[Wiki/Sources/Stock/NiagaraMeshRendererProperties]] — NiagaraMeshRendererProperties.h @ b6ab0dee9 (Phase 6.7,288 行)
- [[Wiki/Sources/Stock/NiagaraRendererMeshes]] — NiagaraRendererMeshes.h @ b6ab0dee9 (Phase 6.8,74 行)
- [[Wiki/Sources/Stock/NiagaraLightRendererProperties]] — NiagaraLightRendererProperties.h @ b6ab0dee9 (Phase 6.9,98 行)
- [[Wiki/Sources/Stock/NiagaraRendererLights]] — NiagaraRendererLights.h @ b6ab0dee9 (Phase 6.10,31 行)
- [[Wiki/Sources/Stock/NiagaraDataInterface]] — NiagaraDataInterface.h @ b6ab0dee9 (Phase 7.2,DI 主基类,890 行,扒前 400)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceCurve]] — NiagaraDataInterfaceCurve.h @ b6ab0dee9 (Phase 7.3,58 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceCurveBase]] — NiagaraDataInterfaceCurveBase.h @ b6ab0dee9 (Phase 7.4,Curve 基类,251 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceCamera]] — NiagaraDataInterfaceCamera.h @ b6ab0dee9 (Phase 7.5,107 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceCollisionQuery]] — NiagaraDataInterfaceCollisionQuery.h @ b6ab0dee9 (Phase 7.6,93 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceStaticMesh]] — NiagaraDataInterfaceStaticMesh.h @ b6ab0dee9 (Phase 7.7,491 行,扒前 250)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceSkeletalMesh]] — NiagaraDataInterfaceSkeletalMesh.h @ b6ab0dee9 (Phase 7.8,976 行,扒前 300)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceTexture]] — NiagaraDataInterfaceTexture.h @ b6ab0dee9 (Phase 7.9,63 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceRenderTarget2D]] — NiagaraDataInterfaceRenderTarget2D.h @ b6ab0dee9 (Phase 7.10,138 行,RW DI)
- [[Wiki/Sources/Stock/NiagaraShared]] — NiagaraShared.h @ b6ab0dee9 (Phase 8.1,Shader 共享基础,902 行,扒前 400)
- [[Wiki/Sources/Stock/NiagaraShader]] — NiagaraShader.h @ b6ab0dee9 (Phase 8.2,FNiagaraShader compute shader,168 行)
- [[Wiki/Sources/Stock/NiagaraShaderType]] — NiagaraShaderType.h @ b6ab0dee9 (Phase 8.3,160 行)
- [[Wiki/Sources/Stock/NiagaraShaderMap]] — NiagaraShaderMap.h @ b6ab0dee9 (Phase 8.4,stub 文件,8 行)
- [[Wiki/Sources/Stock/NiagaraScriptBase]] — NiagaraScriptBase.h @ b6ab0dee9 (Phase 8.5,UNiagaraScriptBase + SimStageMetaData,53 行)
- [[Wiki/Sources/Stock/NiagaraGPUInstanceCountManager]] — NiagaraGPUInstanceCountManager.h @ b6ab0dee9 (Phase 8.6,127 行)
- [[Wiki/Sources/Stock/NiagaraGPUSortInfo]] — NiagaraGPUSortInfo.h @ b6ab0dee9 (Phase 8.7,76 行)
- [[Wiki/Sources/Stock/NiagaraVertexFactory]] — NiagaraVertexFactory.h @ b6ab0dee9 (Phase 8.9,VF 基类,135 行)
- [[Wiki/Sources/Stock/NiagaraSpriteVertexFactory]] — NiagaraSpriteVertexFactory.h @ b6ab0dee9 (Phase 8.10,223 行)
- [[Wiki/Sources/Stock/NiagaraRibbonVertexFactory]] — NiagaraRibbonVertexFactory.h @ b6ab0dee9 (Phase 8.11,239 行)
- [[Wiki/Sources/Stock/NiagaraMeshVertexFactory]] — NiagaraMeshVertexFactory.h @ b6ab0dee9 (Phase 8.12,198 行)
- [[Wiki/Sources/Stock/NiagaraSortingGPU]] — NiagaraSortingGPU.h @ b6ab0dee9 (Phase 8.13,SortKeyGenCS,81 行)
- [[Wiki/Sources/Stock/NiagaraDrawIndirect]] — NiagaraDrawIndirect.h @ b6ab0dee9 (Phase 8.14,DrawIndirectArgsGenCS,130 行)
- [[Wiki/Sources/Stock/NiagaraWorldManager]] — NiagaraWorldManager.h @ b6ab0dee9 (Phase 9.1,286 行)
- [[Wiki/Sources/Stock/NiagaraScalabilityManager]] — NiagaraScalabilityManager.h @ b6ab0dee9 (Phase 9.2,140 行)
- [[Wiki/Sources/Stock/NiagaraComponentPool]] — NiagaraComponentPool.h @ b6ab0dee9 (Phase 9.3,149 行)
- [[Wiki/Sources/Stock/NiagaraSettings]] — NiagaraSettings.h @ b6ab0dee9 (Phase 9.4,75 行)
- [[Wiki/Sources/Stock/NiagaraEffectType]] — NiagaraEffectType.h @ b6ab0dee9 (Phase 9.5,337 行)
- [[Wiki/Sources/Stock/NiagaraPlatformSet]] — NiagaraPlatformSet.h @ b6ab0dee9 (Phase 9.6,378 行,扒前 200)
- [[Wiki/Sources/Stock/NiagaraSimulationStageBase]] — NiagaraSimulationStageBase.h @ b6ab0dee9 (Phase 10.1,78 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceRW]] — NiagaraDataInterfaceRW.h @ b6ab0dee9 (Phase 10.2,RW DI 基础,246 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollection]] — NiagaraDataInterfaceGrid2DCollection.h @ b6ab0dee9 (Phase 10.3,268 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid2DCollectionReader]] — NiagaraDataInterfaceGrid2DCollectionReader.h @ b6ab0dee9 (Phase 10.4,95 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceGrid3DCollection]] — NiagaraDataInterfaceGrid3DCollection.h @ b6ab0dee9 (Phase 10.5,158 行)
- [[Wiki/Sources/Stock/NiagaraDataInterfaceNeighborGrid3D]] — NiagaraDataInterfaceNeighborGrid3D.h @ b6ab0dee9 (Phase 10.6,99 行)

## Readers(主题读本)
*每个议题"一次读完即完整掌握"的线性读物。人类阅读的首选入口,详见 [[CLAUDE]] §3.4。*

### 方法论
- [[Readers/Methodology/Llm-wiki-方法论-读本|方法论读本 — 本仓库为什么存在]] — LLM Wiki 方法论主题读本,详解三层架构/三操作/两文件/Memex-RAG-Wiki 脉络/Karpathy 三里程碑/本仓库如何具体化 (2026-04-20)

### AI 美术
- [[Readers/AIArt/Lora-深度指南-读本|LoRA 深度指南读本]] — 鸣潮美术向 LoRA 落地方案主题读本,从战略(离开 MJ)到技术(LoRA/caption/trigger)到工具(kohya+ComfyUI)到落地路线 (2026-04-20)

### AI 应用生态
- [[Readers/AIApps/AI-primer-v2-读本|AI 应用生态读本]] — AI Primer v2 主题读本,从 LLM 本质到 2026 技术栈三层全景,一次读完掌握整条主线 (2026-04-20)

### Niagara 源码学习
- [[Readers/Niagara/Phase0-心智模型-读本|Phase 0 读本 — 上阵前的四层脑内地图]] — 把 UObject / Asset-Instance / Niagara-vs-Cascade / CPU-vs-GPU 四概念编成自下而上一条叙事链,一次读完掌握 Phase 1+ 所需全部前置 (2026-04-19)
- [[Readers/Niagara/Phase1-asset-layer-读本|Phase 1 读本 — Niagara 的资产层]] — 把 5 个 header 讲成一个连贯故事,从 System 到图源抽象基类,一次读完掌握 Asset 层全部心智模型 (2026-04-19)
- [[Readers/Niagara/Phase2-component-layer-读本|Phase 2 读本 — Niagara 的 Component 层]] — 把 Component/Actor/FunctionLibrary 三文件讲成从"资产到场景里跑着"的完整控制流,涵盖三入口/五职责/四生命周期源/Pool-Scalability-AutoDestroy 三方决策 (2026-04-20)
- [[Readers/Niagara/Phase3-runtime-instance-读本|Phase 3 读本 — Niagara 的心脏]] — 运行时实例层:三阶段 Tick + 双状态机 + 4-instance TickBatch + 参数双缓冲 + ParameterStore↔DataSet 绑定,含 `bForceSolo` 5-20× 退化定量估算 (2026-04-20)
- [[Readers/Niagara/Phase4-data-model-读本|Phase 4 读本 — Niagara 的数据语言]] — 类型系统(TypeDef/Variable/LayoutInfo)+ SoA 布局 + Double Buffer + Persistent ID 双整数 + 命名空间系统 + ParameterStore 三路 dirty + 绑定图,一次读完掌握 Niagara "数据"全貌 (2026-04-20)
- [[Readers/Niagara/Phase5-cpu-script-execution-读本|Phase 5 读本 — Niagara 脚本如何"跑起来"]] — CPU VM 上下文 + PerInstanceFunctionHook + CPU/GPU 参数 padding + GPU Tick 打包 + FFXSystemInterface 三阶段 stage,为 Phase 7(DI)/Phase 8(GPU)搭脚手架 (2026-04-20)
- [[Readers/Niagara/Phase6-rendering-读本|Phase 6 读本 — Niagara 粒子如何变成屏幕像素]] — Properties/Renderer 对偶 + GT→RT 三阶段数据流 + 4 种类型能力边界(Sprite/Ribbon/Mesh/Light)+ Cutout/Tessellation/SimpleLight 特殊路径 (2026-04-20)
- [[Readers/Niagara/Phase7-data-interface-读本|Phase 7 读本 — Niagara 的最强扩展点]] — DI 三路代码生成(VM/C++ lambda/HLSL)+ Per-Instance Data 生命周期 + 7 种典型 DI 能力矩阵 + SkeletalMesh 共享 skinning 缓存 + RW DI 预告 (2026-04-20)
- [[Readers/Niagara/Phase8-gpu-simulation-读本|Phase 8 读本 — Niagara 的 GPU 模拟管线]] — Shader 编译 + 共享 GPU Instance Count + DrawIndirect 生成 + GPU Sort(FGPUSortManager + SortKeyGenCS 4 permutation)+ 3 种 VertexFactory 能力矩阵 (2026-04-20)
- [[Readers/Niagara/Phase9-world-management-读本|Phase 9 读本 — Niagara 的世界管理与可扩展性]] — 全局状态四层模型 + Scalability 决策引擎(4 类 cull + significance)+ Pool 5 方法取舍 + PlatformSet 三态双 mask 配置 (2026-04-20)
- [[Readers/Niagara/Phase10-advanced-features-读本|Phase 10 读本 — Niagara 的高级特性]](**学习路径终点**)— SimStages 多 pass + IterationSource 二选一 + RW DI 四钩子 + Grid 2D/3D 存储策略差异(Texture2DArray vs RWTexture3D tile 打包)+ NeighborGrid 空间哈希 + Reader 跨 emitter 交互 (2026-04-20)

## Syntheses(综合/专题)
*跨源分析、对比、专题报告、好答案的沉淀(非读本)。读本见上方 [[#Readers(主题读本)]] 分区。*

### 方法论
- [[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat|如何向 AI 提问（详细版）]] — Chat 场景 prompt 指南，含心智模型、5 要素、8 技巧、反模式、团队推广经验

### AI 应用生态
- [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]] — AI 工程方法论 2022-2026 演进主线叙事

### Niagara 源码学习
- [[Wiki/Syntheses/Niagara/Niagara-learning-path|Niagara 源码学习路径]] — UE 4.26 Niagara 插件 10 阶段学习路线图，含 ~69 个文件待 ingest (stock)

---

## 快速导航

- 方法论源文档:[[Raw/Notes/Karpathy Wiki 方法论]]
- AI 扫盲手册 v2：[[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]
- 操作规程:[[CLAUDE]]
- 时间线日志:[[log]]
