---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, constants, namespaces, attributes]
sources: 1
aliases: [NiagaraConstants.h, Niagara 常量源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraConstants.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraConstants.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/NiagaraConstants.h`
- **快照**: commit `b6ab0dee9`
- **文件规模**: 209 行
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 4 — 数据模型 4.3

## 职责

定义 Niagara 脚本能**开箱即用**的所有 Engine/System/Emitter/Particles 级变量(约 70+ 个),外加**参数命名空间**的 `PARAM_MAP_*` 前缀常量。这是 Niagara 脚本里你看到的 `Engine.DeltaTime` / `Particles.Position` / `Emitter.Age` 等名字的**注册中心**。

## 参数命名空间(PARAM_MAP_*)

从头部宏一眼看清 Niagara 的参数域划分:

| 前缀 | 常量 | 含义 |
|---|---|---|
| `User.` | `PARAM_MAP_USER_STR` | 用户暴露给 Component 的参数(Phase 2 `OverrideParameters`) |
| `Engine.` | `PARAM_MAP_ENGINE_STR` | 引擎驱动(Time/Delta/QualityLevel...) |
| `Engine.Owner.` | `PARAM_MAP_ENGINE_OWNER_STR` | Component 的 Position/Velocity/Rotation... |
| `Engine.System.` | — | System 级引擎值(Age/TickCount) |
| `Engine.Emitter.` | — | Emitter 级引擎值(NumParticles) |
| `System.` | `PARAM_MAP_SYSTEM_STR` | System 脚本可写的 System-scope 变量 |
| `Emitter.` | `PARAM_MAP_EMITTER_STR` | Emitter 脚本可写的 Emitter-scope 变量 |
| `Particles.` | `PARAM_MAP_ATTRIBUTE_STR` | **粒子级属性**(Position/Color/Age...) |
| `Module.` | `PARAM_MAP_MODULE_STR` | Module 本地(其他 Module 看不到) |
| `Local.Module.` / `Output.Module.` | — | Module scope |
| `Initial.` / `Previous.` | `PARAM_MAP_INITIAL_STR` / `PREVIOUS_BASE_STR` | 粒子初始值 / 上帧值 |
| `Constants.` | `PARAM_MAP_RAPID_ITERATION_STR` | RapidIteration 参数(调试/编辑期间) |
| `Array.` | `PARAM_MAP_INDICES_STR` | 数组索引 |
| `Transient.` / `Intermedate.` | — | 脚本执行内部临时 |
| `ScriptTransient.` / `ScriptPersistent.` | — | 跨脚本但同实例 |
| `NPC.` | `PARAM_MAP_NPC_STR` | Niagara Parameter Collection |

## Engine Constants 宏族

大量 `SYS_PARAM_ENGINE_*` / `SYS_PARAM_PARTICLES_*` / `SYS_PARAM_EMITTER_*` 宏,每个展开为 `INiagaraModule::GetVar_*()` 返回全局注册的 `FNiagaraVariable`。代表性组:

### Engine 全局时间
- `SYS_PARAM_ENGINE_DELTA_TIME` / `INV_DELTA_TIME` / `TIME` / `REAL_TIME`
- `SYS_PARAM_ENGINE_QUALITY_LEVEL` / `GLOBAL_SPAWN_COUNT_SCALE` / `GLOBAL_SYSTEM_COUNT_SCALE`

### Engine Owner(Component 级)
- `POSITION` / `VELOCITY` / `X/Y/Z_AXIS` / `ROTATION` / `SCALE`
- `LOCAL_TO_WORLD` / `WORLD_TO_LOCAL`(+ `_TRANSPOSED` / `_NO_SCALE` 变体)
- `TIME_SINCE_RENDERED` / `LOD_DISTANCE` / `LOD_DISTANCE_FRACTION`
- `EXECUTION_STATE`

### Engine System
- `SYSTEM_AGE` / `SYSTEM_TICK_COUNT` / `SYSTEM_NUM_EMITTERS` / `SYSTEM_NUM_EMITTERS_ALIVE` / `SYSTEM_SIGNIFICANCE_INDEX`
- `NUM_SYSTEM_INSTANCES`(场景中本 Asset 总数,scalability 用)

### Engine Emitter
- `EMITTER_NUM_PARTICLES` / `EMITTER_TOTAL_SPAWNED_PARTICLES` / `EMITTER_SPAWN_COUNT_SCALE` / `EMITTER_INSTANCE_SEED`
- `EXEC_COUNT`(本次脚本执行的 count)

### Emitter 作用域
- `EMITTER_AGE` / `LOCALSPACE` / `DETERMINISM` / `RANDOM_SEED` / `SPAWNRATE` / `SPAWN_INTERVAL` / `SIMULATION_TARGET` / `INTERP_SPAWN_START_DT` / `SPAWN_GROUP`
- `OVERRIDE_GLOBAL_SPAWN_COUNT_SCALE`

### Particles(粒子属性)

最丰富:
- 基础:`ID` / `UNIQUE_ID` / `POSITION` / `VELOCITY` / `COLOR` / `LIFETIME` / `NORMALIZED_AGE` / `SCALE`
- Sprite:`SPRITE_ROTATION` / `SPRITE_SIZE` / `SPRITE_FACING` / `SPRITE_ALIGNMENT` / `SUB_IMAGE_INDEX`
- Mesh:`MESH_ORIENTATION`
- Dynamic Material:`DYNAMIC_MATERIAL_PARAM` / `_1` / `_2` / `_3`
- 光源:`LIGHT_RADIUS` / `LIGHT_EXPONENT` / `LIGHT_ENABLED` / `LIGHT_VOLUMETRIC_SCATTERING`
- 其他:`CAMERA_OFFSET` / `UV_SCALE` / `MATERIAL_RANDOM` / `VISIBILITY_TAG` / `COMPONENTS_ENABLED`
- Ribbon 专用:`RIBBONID` / `RIBBONWIDTH` / `RIBBONTWIST` / `RIBBONFACING` / `RIBBONLINKORDER` / `RIBBONU0OVERRIDE` / `RIBBONV0RANGEOVERRIDE` / `RIBBONU1OVERRIDE` / `RIBBONV1RANGEOVERRIDE`

### 其他
- `INSTANCE_ALIVE`(DataInstance 存活标记)
- `SCRIPT_USAGE` / `SCRIPT_CONTEXT`
- `TRANSLATOR_*_CALL_ID` / `BEGIN_DEFAULTS`(translator 内部)

## 主类:`FNiagaraConstants`(L134)

静态方法工厂,查询各种常量:

```cpp
static void Init();                                                 // 初始化静态表
static const TArray<FNiagaraVariable>& GetEngineConstants();
static const TArray<FNiagaraVariable>& GetTranslatorConstants();
static const TArray<FNiagaraVariable>& GetStaticSwitchConstants();

static const FNiagaraVariable* FindEngineConstant(const FNiagaraVariable&);
static FText GetEngineConstantDescription(const FNiagaraVariable&);

static const TArray<FNiagaraVariable>& GetCommonParticleAttributes();
static FText GetAttributeDescription(const FNiagaraVariable&);
static FString GetAttributeDefaultValue(const FNiagaraVariable&);
static FNiagaraVariable GetAttributeWithDefaultValue(const FNiagaraVariable&);
static FNiagaraVariable GetAttributeAsParticleDataSetKey(const FNiagaraVariable&);
static FNiagaraVariable GetAttributeAsEmitterDataSetKey(const FNiagaraVariable&);
static FNiagaraVariableAttributeBinding GetAttributeDefaultBinding(const FNiagaraVariable&);

static bool IsNiagaraConstant(const FNiagaraVariable&);
static const FNiagaraVariableMetaData* GetConstantMetaData(const FNiagaraVariable&);

static bool IsEngineManagedAttribute(const FNiagaraVariable&);
```

## 命名空间 FName 常量(L165-L193)

`FNiagaraConstants` 还公开一组 `FName` 静态常量作为命名空间标识符(`UserNamespace / EngineNamespace / SystemNamespace / EmitterNamespace / ParticleAttributeNamespace / ModuleNamespace / ...`)和作用域名(`EngineOwnerScopeName / EngineSystemScopeName / EngineEmitterScopeName / ScriptTransientScopeName / ScriptPersistentScopeName / InputScopeName / OutputScopeName / UniqueOutputScopeName / CustomScopeName`)——编辑器 UI 和编译器用这些分组显示与校验。

## 私有静态表

- `SystemParameters` / `TranslatorParameters` / `SwitchParameters`(L196-L198):常量数据表
- `UpdatedSystemParameters`(L199):per-version 的更新映射
- `Attributes`(L201):粒子属性常量表
- 各种 metadata map(描述、默认值、UI data)

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/FNiagaraConstants]]
- [[Wiki/Entities/Stock/FNiagaraVariable]] — 所有常量都是这个类型
- Phase 2 `UNiagaraComponent::SetVariableXxx` 的 `FName InVariableName` 最终就是查这里的命名空间注册

## 开放问题

- `Particles.VISIBILITY_TAG` 作用?→ 渲染 culling,Phase 6
- `Particles.COMPONENTS_ENABLED` 位用法?→ Phase 6 Component Renderer
- RapidIteration 参数(`Constants.*`)的 editor vs cooked 差异 → Phase 5 `NiagaraScriptExecutionParameterStore`
