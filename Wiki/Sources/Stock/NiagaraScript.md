---
type: source
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, source-code, asset, script, compile]
sources: 1
aliases: [NiagaraScript.h, UNiagaraScript 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScript.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraScript.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraScript.h`
- **快照**: commit `b6ab0dee9`
- **同仓协作文件**: `NiagaraScript.cpp`、`NiagaraScriptBase.h`(基类)、`NiagaraScriptSourceBase.h`(editor 图源)、`NiagaraScriptExecutionContext.h`(运行时执行,Phase 5)、`NiagaraShared.h / NiagaraShader.h`(GPU,Phase 8)
- **Ingest 日期**: 2026-04-19
- **学习阶段**: Phase 1 — 资产层 1.4

## 职责 / 这个文件干什么的

定义 `UNiagaraScript`——**编译后的脚本资产**。头文件里还定义了两个关键 USTRUCT:

- `FNiagaraVMExecutableDataId`:一份编译产物的"身份证",用于 DDC(Derived Data Cache)查询、比较是否需要重新编译
- `FNiagaraVMExecutableData`:实际的编译产物内容(VectorVM 字节码、参数布局、DataInterface 信息等)

以及一个调试辅助 `FNiagaraScriptDebuggerInfo`、两个 UENUM(`EUnusedAttributeBehaviour`、`ENiagaraScriptLibraryVisibility`)、模块依赖 `FNiagaraModuleDependency` 等。

**一个 UNiagaraScript 不是"源代码",而是"编译好的可执行体 + 可以执行的 shell"**:
- 在 editor 里,它有一个指向图源的 `Source`(`UNiagaraScriptSourceBase*`)
- 在 cooked / runtime 里,只剩 `CachedScriptVM`(字节码)和 `ScriptResource`(GPU compute shader map)

`UNiagaraScript` 是 Niagara 的**核心执行单元**,也是连接编辑器(图)与运行时(VectorVM / Compute Shader)的桥梁。

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraScript : public UNiagaraScriptBase`(L373),`UCLASS(MinimalAPI)`
  - `UNiagaraScriptBase` 住在 `NiagaraShader` 模块(Phase 8),这种"基类拆到更底层模块"是为了让 Shader 模块不依赖完整 Niagara 模块

### 枚举 / 依赖

- `ENiagaraScriptUsage`(`NiagaraCommon.h` L715):**最重要的枚举**,决定此脚本的角色
  - `Function / Module / DynamicInput`:编辑器侧素材
  - `ParticleSpawnScript / ParticleSpawnScriptInterpolated / ParticleUpdateScript / ParticleEventScript / ParticleSimulationStageScript / ParticleGPUComputeScript`:粒子级
  - `EmitterSpawnScript / EmitterUpdateScript`:Emitter 级(**不可单独编译**,合并进 System)
  - `SystemSpawnScript / SystemUpdateScript`:System 级
- `EUnusedAttributeBehaviour`(L32):未使用属性的 4 种处理(Copy/Zero/None/MarkInvalid/PassThrough)
- `ENiagaraModuleDependencyType`(L47)、`ENiagaraModuleDependencyScriptConstraint`(L56):模块依赖约束
- `ENiagaraScriptLibraryVisibility`(L65):库面板可见性(Unexposed / Library / Hidden)

### 核心结构体

#### `FNiagaraVMExecutableDataId`(L131)
编译产物的"**身份证 + 重编译判定器**"。字段:
- `CompilerVersionID / ScriptUsageType / ScriptUsageTypeID`
- `bUsesRapidIterationParams / bInterpolatedSpawn / bRequiresPersistentIDs`:影响编译结果的开关
- `BaseScriptCompileHash`(`FNiagaraCompileHash`):此脚本所在子图的内容哈希
- `ReferencedCompileHashes`(editor-only):依赖的子图哈希,任何一个变了都得重编译
- `operator==` + `GetTypeHash`:用作 `TMap` / DDC 的 key

#### `FNiagaraVMExecutableData`(L241)
实际编译产物内容:
- `ByteCode` + `OptimizedByteCode`:VectorVM 字节码(OptimizedByteCode 是针对当前平台二次优化的,非 serialized)
- `NumTempRegisters / NumUserPtrs`
- `ScriptLiterals`:编译期烘死的常量
- `Attributes`(`TArray<FNiagaraVariable>`):此脚本读写的粒子属性清单
- `DataUsage`(`FNiagaraScriptDataUsageInfo`)
- `DataInterfaceInfo`(`TArray<FNiagaraScriptDataInterfaceCompileInfo>`):每个 DI 的编译信息
- `CalledVMExternalFunctions`(`TArray<FVMExternalFunctionBindingInfo>`):外部函数绑定表——DataInterface 函数或 C++ 辅助函数
- `CalledVMExternalFunctionBindings`(非 UPROPERTY,`TArray<FVMExternalFunction>`):运行时填好的函数指针
- `ReadDataSets / WriteDataSets`:事件/流数据的读写声明
- `LastHlslTranslation / LastHlslTranslationGPU / LastAssemblyTranslation`(editor-only):编译中间产物,调试用
- `DIParamInfo`(`TArray<FNiagaraDataInterfaceGPUParamInfo>`):GPU DI 参数信息(**注释作者吐槽**:"GPU Param info should not be in the 'VM executable data'" —— 技术债标记)
- `LastCompileStatus / ErrorMsg / CompileTime / LastCompileEvents`:编译结果状态
- `SimulationStageMetaData`:Simulation Stage 的元数据

#### `FNiagaraScriptDebuggerInfo`(L107)
Debug 用,记录一次执行的 `FNiagaraDataSet Frame` 快照 + 参数快照 + 使用信息。编辑器 Debug 面板读这个。

#### `FNiagaraModuleDependency`(L79)
模块脚本的依赖声明:`Id`(要求提供的能力名,如 `ProvidesNormalizedAge`)+ `Type`(Pre/Post) + `ScriptConstraint`(SameScript / AllScripts)。Stack UI 根据这个做"模块必须在 X 之前/之后"的校验。

### `UNiagaraScript` 字段

**Usage 与依赖**
- `Usage`(`ENiagaraScriptUsage`,`UPROPERTY(AssetRegistrySearchable)`,L379):此脚本的角色
- `UsageId`(`FGuid`,L393):同 usage 多个脚本时区分(典型:多个 EventHandler)
- `ModuleUsageBitmask`(editor-only,L399):此 Module 脚本可在哪些 Usage 上下文使用
- `ProvidedDependencies / RequiredDependencies`(editor-only,L407/411)
- `Category / bDeprecated / DeprecationMessage / DeprecationRecommendation / ConversionUtility / bExperimental / ExperimentalMessage / LibraryVisibility`:编辑器元数据

**编译产物 / 执行**
- `RapidIterationParameters`(`FNiagaraParameterStore`,L448):用户可在编辑器里快速调的参数(通常不触发重编译)
- `CachedScriptVM`(`FNiagaraVMExecutableData`,L800):**运行时执行靠这个**
- `CachedScriptVMId`(`FNiagaraVMExecutableDataId`,L762):CachedScriptVM 对应的身份
- `CachedParameterCollectionReferences`(L803):Parameter Collection 引用缓存
- `CachedDefaultDataInterfaces`(`TArray<FNiagaraScriptDataInterfaceInfo>`,L806)
- `ScriptExecutionParamStore`(`FNiagaraScriptExecutionParameterStore`,L736)+ `ScriptExecutionBoundParameters`(L739):cooked 平台的绑定数据
- (editor-only)`ScriptExecutionParamStoreCPU / ScriptExecutionParamStoreGPU`(L728-L731):未 cook 时的 transient 版本

**GPU 相关(Phase 8 再深入)**
- `ScriptResource`(`TUniquePtr<FNiagaraShaderScript>`,L770):RenderThread 端的 shader script 资源
- `ScriptShader`(`FComputeShaderRHIRef`,L778):RHI 句柄
- (editor-only)`LoadedScriptResources / ScriptResourcesByFeatureLevel / CachedScriptResourcesForCooking`

**Source(editor-only)**
- `Source`(`UNiagaraScriptSourceBase*`,L748):图源

### 关键方法

**Usage 辅助(大量静态/实例布尔查询)**
`IsParticleSpawnScript / IsParticleUpdateScript / IsModuleScript / IsFunctionScript / IsDynamicInputScript / IsParticleEventScript / IsParticleScript / IsNonParticleScript / IsSystemSpawnScript / IsSystemUpdateScript / IsEmitterSpawnScript / IsEmitterUpdateScript / IsStandaloneScript / IsSpawnScript / IsCompilable / IsGPUScript / IsInterpolatedParticleSpawnScript`

- `ConvertUsageToGroup(Usage, out Group)`(L545):Usage → ENiagaraScriptGroup 聚合
- `IsUsageDependentOn(A, B)`(L498):A usage 是否依赖 B usage(用于 tick 顺序)
- `IsEquivalentUsage`(L495-L496):把 `ParticleSpawnScript` 与 `ParticleSpawnScriptInterpolated` 视为等价

**编译(editor-only)**
- `ComputeVMCompilationId(Id)` → 基于当前图源生成 VMID
- `GetComputedVMCompilationId()` → cooked 时返 `CachedScriptVMId`,editor 时返 `LastGeneratedVMId`
- `AreScriptAndSourceSynchronized / MarkScriptAndSourceDesynchronized`
- `RequestCompile(bForceCompile)` / `RequestExternallyManagedAsyncCompile(RequestData, out CompileId, out AsyncHandle)`
- `SetVMCompilationResults(InCompileId, InScriptVM, InRequestData)` — 编译完成回填点
- DDC:`BuildNiagaraDDCKeyString(CompileId)` / `GetNiagaraDDCKeyString()`
- `BinaryToExecData / ExecToBinaryData` — DDC 二进制 ↔ 结构体(注释警告仅 GameThread 能调)
- 委托:`OnVMScriptCompiled / OnGPUScriptCompiled / OnPropertyChanged`

**GPU(editor)**
- `CacheResourceShadersForCooking / CacheResourceShadersForRendering`
- `BeginCacheForCookedPlatformData / IsCachedCookedPlatformDataLoaded`
- `CacheShadersForResources`

**运行时**
- `GetVMExecutableData()` → `CachedScriptVM`
- `GetExecutionReadyParameterStore(SimTarget)` / `InvalidateExecutionReadyParameterStores`
- `IsReadyToRun(SimTarget)` / `CanBeRunOnGpu()`

### 私有辅助

- `OwnerCanBeRunOnGpu()` / `LegacyCanBeRunOnGpu()`
- `ProcessSerializedShaderMaps` / `SerializeNiagaraShaderMaps`
- `GetSimTarget()`:推测脚本的目标平台
- `AsyncOptimizeByteCode()`:启动异步字节码优化
- `GenerateDefaultFunctionBindings()`:生成不需要用户数据的 DI 函数绑定
- `HasValidParameterBindings()`
- `CopyDataInterface(Src, Owner)`:DI 拷贝辅助(静态)

## 关键代码片段

> `NiagaraScript.h:L371-L380` @ `b6ab0dee9` — 类声明 + Usage 字段
> ```cpp
> /** Runtime script for a Niagara system */
> UCLASS(MinimalAPI)
> class UNiagaraScript : public UNiagaraScriptBase
> {
>     GENERATED_UCLASS_BODY()
> public:
>     UPROPERTY(AssetRegistrySearchable)
>     ENiagaraScriptUsage Usage;
> ```

> `NiagaraScript.h:L239-L249` @ `b6ab0dee9` — VM 可执行数据入口
> ```cpp
> /** Struct containing all of the data needed to run a Niagara VM executable script.*/
> USTRUCT()
> struct NIAGARA_API FNiagaraVMExecutableData
> {
>     GENERATED_USTRUCT_BODY()
> public:
>     FNiagaraVMExecutableData();
>
>     /** Byte code to execute for this system */
>     UPROPERTY()
>     TArray<uint8> ByteCode;
> ```

> `NiagaraScript.h:L519` @ `b6ab0dee9` — EmitterSpawn/Update 不可单独编译
> ```cpp
> bool IsCompilable() const { return !IsEmitterSpawnScript() && !IsEmitterUpdateScript(); }
> ```
> 印证了 Phase 1 的关键结论:Emitter 级脚本被合并进 System 脚本,不独立存在字节码。

## 依赖与被依赖

**上游:**
- `NiagaraCommon.h`(`ENiagaraScriptUsage` 等核心枚举、`FNiagaraVariable` 等)
- `NiagaraScriptBase.h`(基类,NiagaraShader 模块)
- `NiagaraShared.h / NiagaraShader.h`(`FNiagaraShaderScript`,Phase 8)
- `NiagaraParameters.h / NiagaraParameterStore.h`(`FNiagaraParameterStore`)
- `NiagaraDataSet.h`(`FNiagaraDataSet` 用在 Debug)
- `NiagaraScriptExecutionParameterStore.h`(cooked 绑定)
- `NiagaraScriptHighlight.h`(编辑器语法高亮元数据)

**下游:**
- `UNiagaraSystem`:`SystemSpawnScript / SystemUpdateScript` + 间接通过 Emitter 引用所有粒子脚本
- `UNiagaraEmitter`:`SpawnScriptProps / UpdateScriptProps / GPUComputeScript / EventHandlerScriptProps`
- `FNiagaraScriptExecutionContext`(Phase 5):运行时执行入口,持有 `UNiagaraScript*`
- `FNiagaraEmitterInstance`(Phase 3):执行 Emitter 的粒子脚本
- `FNiagaraSystemSimulation`(Phase 3):执行 System 脚本
- 编辑器大半个 Niagara:Stack、图、Compile Button…

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/UNiagaraScript]] — 本文件定义的主类
- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] — editor 下 Source 字段的类型
- [[Wiki/Entities/Stock/UNiagaraSystem]] — 持有 System 脚本
- [[Wiki/Entities/Stock/UNiagaraEmitter]] — 持有 Emitter/Particle/GPU 脚本
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — `CanBeRunOnGpu / GetSimTarget / ScriptResource / ScriptShader` 是 GPU 分支的入口
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — "显式编译的字节码 + DDC 缓存" 是 Niagara 相对 Cascade 的质变

## 与既有 wiki 的关系

- 印证了 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] 的 "`UNiagaraScript` 是编译后的字节码容器,区分 Spawn/Update/Event 等脚本类型"
- 印证 [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] 的 "Niagara 的脚本编译成字节码由 VectorVM / GPU Compute Shader 执行"
- 补完 [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]:同一 `UNiagaraScript` 类承载 CPU 和 GPU 两种执行方式,通过 `ScriptResource` vs `CachedScriptVM` 区分

## 开放问题 / 待深入

- `CachedScriptVM.OptimizedByteCode` 非 serialized,来自 `AsyncOptimizeByteCode()`。具体"针对平台优化"做了什么?Phase 5 看 VectorVM 执行路径时对照
- `RapidIterationParameters` 存储在脚本上,但语义上是"用户暴露的参数"——它们和 `UNiagaraSystem::ExposedParameters` 的关系?后者是 User Parameter,前者是 Module 暴露的 input,二者命名空间不同(`User.*` vs `Module.*`)。Phase 5 读 `NiagaraScriptExecutionParameterStore` 时对照
- `FNiagaraVMExecutableData::DIParamInfo` 的 "技术债"注释 —— 被作者自己认为"不该在这里"。Phase 7/8 看 DI GPU 路径时再看它是否有更合适的位置
- `SynchronizeExecutablesWithMaster`(L667):merge 时不走 DDC 直接把编译产物拷过来的快路径
- `ReleasedByRT`(L811):RenderThread 端释放前的线程安全标记,与 `ScriptResource` 生命周期配合;Phase 8 时验证
