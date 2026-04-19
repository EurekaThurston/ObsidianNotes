---
type: synthesis
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, learning-path, phase-1, reader, asset]
sources: 5
aliases: [Phase 1 读本, Niagara Asset 层读本]
repo: stock
source_root: UE-4.26-Stock
source_ref: "4.26"
source_commit: b6ab0dee9
---

# Phase 1 读本 — Niagara 的资产层

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 1 的**主题读本**——详细、精确、满满当当,一次读完即完整掌握 Niagara Asset 层(System / Emitter / Script)的全部心智模型,不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 1 要回答的问题

Phase 0 建立了几个抽象概念:[[Wiki/Concepts/UE4/UE4-资产与实例|Asset vs Instance]]、[[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟|CPU 还是 GPU 模拟]]、[[Wiki/Concepts/Niagara/Niagara-vs-cascade|Niagara 为什么换掉 Cascade]]——都是**脑内地图**,没读一行代码。

Phase 1 回答一个非常具体的问题:

> 在你硬盘上的那个 `MyFX.uasset` 文件被 UE 反序列化进内存后,到底长什么样?

换句话说——**当一个 Niagara 特效处于"静止"状态(没有在任何场景里播放,只是躺在 Content Browser 里)时,它由哪些 C++ 类、哪些字段组成?**

策略是**自顶向下**:从你在 Content Browser 里右键能直接创建的那个东西(`UNiagaraSystem`)开始,往下剥到脚本的字节码。途经 5 个头文件,构成一条完整的叙事链:

```
UNiagaraSystem                 ← Content Browser 里那个资产
    ↓ 持有 TArray<
FNiagaraEmitterHandle          ← 引用 Emitter 的"带 Id 的指针"
    ↓ 指向
UNiagaraEmitter                ← 粒子行为的单元(一堆脚本 + 一堆渲染器)
    ↓ 持有若干
UNiagaraScript                 ← 编译后的字节码 / GPU shader
    ↓ editor 下还挂着
UNiagaraScriptSourceBase       ← 图源(runtime 只看到抽象基类)
```

下面这一整篇,就是这条链条的"步道说明牌"。读完你应该能回答:

1. 为什么 `UNiagaraSystem` 里不直接存 `TArray<UNiagaraEmitter*>`,要中间套一层 Handle?
2. Niagara 脚本为什么要有"Usage"这种东西?光靠类名不够吗?
3. 为什么 Script 的 `Source` 字段指向一个看起来啥也不干的抽象基类?

---

## 1. 从 Content Browser 说起:`UNiagaraSystem`

在 UE 编辑器里右键 → Create → FX → Niagara System,你得到一个 `.uasset` 文件。这个文件在内存里就是一个 `UNiagaraSystem` 对象:

```cpp
/** Container for multiple emitters that combine together to create a particle system effect.*/
UCLASS(BlueprintType)
class NIAGARA_API UNiagaraSystem : public UFXSystemAsset
```

注释写得很直白:**一个 System 是若干 Emitter 的容器**。`UFXSystemAsset` 是 UE 通用的特效资产基类,Cascade 的 `UParticleSystem` 也继承它——这是 Niagara 没完全抛开的 Cascade 历史遗产,不是 bug,是有意的兼容面。

### 1.1 最核心的那一行

打开 `NiagaraSystem.h`,如果只看一行,就是这个:

```cpp
/** Handles to the emitter this System will simulate. */
UPROPERTY()
TArray<FNiagaraEmitterHandle> EmitterHandles;
```

**System 的本质就是这个数组**。其他一切字段(脚本、参数、预热时间、scalability 归属……)都是围绕它构建的上下文。

### 1.2 System 自己也有脚本

一个常见误解是"只有 Emitter 有脚本,System 只是装 Emitter 的盒子"。错。System 有两个自己的脚本:

```cpp
UPROPERTY()
UNiagaraScript* SystemSpawnScript;   // System 首次激活时执行一次

UPROPERTY()
UNiagaraScript* SystemUpdateScript;  // 每帧 tick 执行
```

这两个脚本做什么?它们负责**整个 System 级别的状态**——比如"所有 Emitter 共享的一个全局计数器"、"根据整体能量动态调整 spawn rate"。

这里藏着一个**重要事实**,等到第 4 节讲 Script 时会再展开:**Emitter 级脚本(EmitterSpawnScript / EmitterUpdateScript)在编译时会被合并进 System 的这两个脚本**——从运行时视角看,"Emitter 级脚本"根本不独立存在,它们是 System 脚本的一部分。

### 1.3 暴露给外部的参数

```cpp
UPROPERTY()
FNiagaraUserRedirectionParameterStore ExposedParameters;
```

这是 BP / Sequencer 可以读写的"User Parameters"(在 Niagara 编辑器里就是 `User.*` 命名空间那些参数)。比如你给一个火焰特效暴露一个 `User.Intensity` float,在 BP 里 `SetFloatParameter("Intensity", 2.0)` 就是操纵这个东西。

### 1.4 编译产物:两块预计算的数据

```cpp
TArray<TSharedRef<const FNiagaraEmitterCompiledData>> EmitterCompiledData;  // 每个 Emitter 一份
FNiagaraSystemCompiledData SystemCompiledData;                              // System 级一份
```

这两个字段是 `UNiagaraSystem` 里"看起来很可怕但其实是性能关键"的部分。它们装的是**编译期预计算的数据**,核心是**参数名到内存偏移的映射表**。

回忆 Phase 0 里讲的 Cascade vs Niagara 的一个根本差异:**Niagara 的粒子属性是完全动态的**——脚本里用到 `Particle.Position`、`Particle.MyCustomFloat`、`Emitter.Age` 时,它们在 DataSet 里的偏移不是硬编码,是编译期算出来再缓存的。运行时脚本执行时**不做字符串查表**,直接按偏移拿数据——这才有 VectorVM 的 SIMD 性能。

所以 `FNiagaraSystemCompiledData` 里有一堆命名奇怪的字段:

```cpp
FNiagaraParameterDataSetBindingCollection SpawnInstanceGlobalBinding;
FNiagaraParameterDataSetBindingCollection SpawnInstanceSystemBinding;
FNiagaraParameterDataSetBindingCollection SpawnInstanceOwnerBinding;
TArray<FNiagaraParameterDataSetBindingCollection> SpawnInstanceEmitterBindings;
// ... 还有 Update 对应的四份
```

**你不需要现在理解每一条**,只需要记住一个模式:**每一条 Binding 都是"某个参数命名空间(Global/System/Owner/Emitter)在某个时机(Spawn/Update)下,参数到 DataSet 偏移的映射表"**。Phase 4(数据模型)和 Phase 5(脚本执行)会回头把这套映射讲透。

### 1.5 小结:System 是什么

- 一个 Emitter 列表容器(通过 Handle 间接持有)
- 两个自己的 System 级脚本
- 一份对外暴露的参数(User.*)
- 两块预计算的编译产物(核心是参数偏移映射)
- 以及一堆"配置型"字段:固定包围盒、预热、scalability 归属等

**走到这里,你已经能大致回答"一个 Niagara 特效资产在磁盘上是什么"的一半。** 下一步自然是问:`FNiagaraEmitterHandle` 这个中间层到底是什么?

---

## 2. 为什么要有 Handle?

打开 `NiagaraEmitterHandle.h`,整个文件才 114 行,非常短:

```cpp
USTRUCT()
struct NIAGARA_API FNiagaraEmitterHandle
{
private:
    UPROPERTY(VisibleAnywhere, Category="Emitter ID")
    FGuid Id;

    UPROPERTY(VisibleAnywhere, Category = "Emitter ID")
    FName IdName;

    UPROPERTY()
    bool bIsEnabled;

    UPROPERTY()
    FName Name;

    UPROPERTY()
    UNiagaraEmitter* Instance;
    // ... 还有几个 editor-only 和 deprecated 字段
};
```

换句话说,一个 Handle 里就这几样东西:**一个唯一 Id、一个显示名、一个启用开关、一个指向 Emitter 的指针**。

为什么不直接让 `UNiagaraSystem::EmitterHandles` 的类型是 `TArray<UNiagaraEmitter*>`?——**这就是 Phase 1 里最值得思考的一个设计问题**。答案分四点:

### 2.1 同一父 Emitter 可以被引用多次

想象一个爆炸特效,你想要"火花"和"大火花"两个 Emitter,但它们的行为基本一样——你会希望从同一个"父 Emitter 模板"衍生出两个副本。这时两个副本在 Content Browser 里可能指向**同一个 `UNiagaraEmitter` 资产**,但它们在 System 内显然是两个独立的单元。

需要一个**稳定的唯一标识**来区分这两个实例——这就是 `FGuid Id`。

### 2.2 "启用状态"属于 System 对 Emitter 的使用方式

`bIsEnabled` 如果挂在 `UNiagaraEmitter` 上,那么一个 Emitter 被两个 System 引用时,一边禁用就会影响另一边。这不合理。启用状态本质上是**"System 如何使用这个 Emitter"**,不是 Emitter 本身的属性——所以挂在 Handle 上。

### 2.3 显示名也是 System-specific

`Name` 字段决定了这个 Emitter 在 System 内的显示名(也是 `Particles.*` 命名空间的前缀 key)。同样的道理:一个 Emitter 被两个 System 引用时,两边可以起不同的显示名。

### 2.4 Solo / Isolation 模式

```cpp
#if WITH_EDITORONLY_DATA
    bool bIsolated;
#endif
```

编辑器里的"Solo"按钮——只模拟这一个 Emitter,其他禁用显示——这是纯 UI 状态,不该污染 Emitter 本身。挂在 Handle 上,而且不序列化(你会注意到 `bIsolated` 没有 `UPROPERTY` 标记,UE 反射完全看不到它)。

### 2.5 ⚠️ 命名陷阱

看这一行:

```cpp
UPROPERTY()
UNiagaraEmitter* Instance;
```

**这里的 `Instance` 不是**你学 Phase 0 时建立的那个 "Asset vs Instance 二元模型"里的 Instance(运行时对象)。**这个 `Instance` 仍然是 Asset 层**——它是一个 `UNiagaraEmitter` 资产的**副本**(System 在 `AddEmitterHandle` 时会 duplicate 源 Emitter)。

运行时真正的"Emitter Instance"叫做 `FNiagaraEmitterInstance`(注意是 `F` 前缀,非 UObject),住在 Phase 3。**Niagara 里有两层"Emitter 的 Instance"概念,一直要记得区分。**

这个命名冲突会贯穿后面所有 Phase,此处钉死。

### 2.6 小结:Handle 是什么

**Handle 是"System 对 Emitter 的引用包装"**。它是 Asset 层的一部分,不涉及运行时模拟状态。可以理解为:

> `UNiagaraSystem` 不直接持有 Emitter,而是持有"**我怎么使用这些 Emitter 的元数据**"——Id、显示名、启不启用。Emitter 本身只是被 Handle 指过去的那个资产。

---

## 3. Emitter:粒子行为的单元

现在进入 `NiagaraEmitter.h`——这是 Phase 1 里**最大的一个文件**(约 700 行),因为 Emitter 本身是 Niagara 里"干活的基本单位"。它头部注释非常坦率:

```cpp
/**
 *	UNiagaraEmitter stores the attributes of an FNiagaraEmitterInstance
 *	that need to be serialized and are used for its initialization
 */
UCLASS(MinimalAPI)
class UNiagaraEmitter : public UObject
{
    friend struct FNiagaraEmitterHandle;
```

**自述:"此类是运行时 `FNiagaraEmitterInstance` 需要序列化的那部分属性"**。这是纯粹的 "Asset describes Instance" 模型——Asset 是模板,Instance 是基于模板在 World 里实例化出来的活物。

此外,`friend FNiagaraEmitterHandle` 这行很重要:Handle 可以访问 Emitter 的私有成员,这在 merge / rename 等场景下会用到。

### 3.1 最关键的分叉字段:`SimTarget`

```cpp
UPROPERTY(EditAnywhere, Category = "Emitter")
ENiagaraSimTarget SimTarget;  // CPUSim 或 GPUComputeSim
```

这就是 [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] 里讲过的那个分叉点——**同一个 Niagara Emitter 资产,仅仅改这一个字段,走的就是两套完全不同的执行路径**:

- `CPUSim` → VectorVM 字节码 → CPU SIMD 批量执行(Phase 5)
- `GPUComputeSim` → HLSL Compute Shader → GPU Dispatch(Phase 8)

Phase 1 只需知道这个字段在这里,它决定了从 Phase 5 / Phase 8 看到的代码路径。

### 3.2 一堆模拟开关

```cpp
bool bLocalSpace;                   // 粒子坐标相对 Emitter 还是世界
bool bDeterminism;                  // 确定性随机
int32 RandomSeed;                   // 种子
uint32 bInterpolatedSpawning : 1;   // 插值 spawn(抗帧率抖动)
uint32 bRequiresPersistentIDs : 1;  // 每粒子一个持久 ID
uint32 bLimitDeltaTime : 1;         // 限制单 tick dt(防大帧破坏模拟)
float MaxDeltaTimePerTick;
EParticleAllocationMode AllocationMode;  // 预分配策略
int32 PreAllocationCount;
// ...
```

**不需要记住每一条**,这里的设计哲学是:**每一个"模拟行为选项"都对应一个明确的字段**,而不是像 Cascade 那样藏在"模块"里(模块是编辑器图里连线的那些节点,不是数据字段)。这是 Niagara 相对 Cascade 的"数据显式化"原则在实际代码里的体现。

`bInterpolatedSpawning` 特别值得一提——它会影响脚本的 Usage 枚举(见第 4.2 节),把 `ParticleSpawnScript` 变成 `ParticleSpawnScriptInterpolated`。

### 3.3 脚本集合:Emitter 的核心

一个 Emitter 里同时存在**四种**脚本字段:

```cpp
UPROPERTY() FNiagaraEmitterScriptProperties SpawnScriptProps;          // ParticleSpawn
UPROPERTY() FNiagaraEmitterScriptProperties UpdateScriptProps;         // ParticleUpdate
UPROPERTY() UNiagaraScript* GPUComputeScript;                          // GPU 计算脚本
UPROPERTY(..., meta=(NiagaraNoMerge))
TArray<FNiagaraEventScriptProperties> EventHandlerScriptProps;         // 事件处理器们
```

注意**不在**这里的:`EmitterSpawnScript` 和 `EmitterUpdateScript`(Emitter 级脚本)。它们只在 `#if WITH_EDITORONLY_DATA` 下存在——因为在最终产物里,它们已经被合并进 System 脚本。这是 Niagara 的一个隐藏设计:**Emitter 级的"Emitter Spawn/Update" UI 概念和运行时的"脚本对象"并不一一对应**。

再看 `ForEachScript` 的实现,一目了然:

```cpp
template<typename TAction>
void UNiagaraEmitter::ForEachScript(TAction Func) const
{
    Func(SpawnScriptProps.Script);
    Func(UpdateScriptProps.Script);
    Func(GPUComputeScript);
    for (auto& EventScriptProps : EventHandlerScriptProps)
    {
        Func(EventScriptProps.Script);
    }
}
```

一个 Emitter 的脚本树就这四类,没别的。

### 3.4 `FNiagaraEmitterScriptProperties`:脚本的包装器

脚本字段不是直接 `UNiagaraScript*`,而是包装在 `FNiagaraEmitterScriptProperties` 里:

```cpp
USTRUCT()
struct FNiagaraEmitterScriptProperties
{
    UPROPERTY() UNiagaraScript* Script;
    UPROPERTY() TArray<FNiagaraEventReceiverProperties> EventReceivers;
    UPROPERTY() TArray<FNiagaraEventGeneratorProperties> EventGenerators;
};
```

为什么要包装?因为**事件配置是挂在脚本上的**,不是挂在 Emitter 上的。同一 Emitter 的 SpawnScript 和 UpdateScript 可以有独立的事件收发定义——这是 Niagara 事件系统的一个精巧之处,但 Phase 1 知道"事件挂在脚本上"就够了,细节放到后面。

### 3.5 渲染器列表(Phase 6 的接口)

```cpp
UPROPERTY()
TArray<UNiagaraRendererProperties*> RendererProperties;
```

Sprite / Mesh / Ribbon / Light 等渲染器配置,Phase 6 会详细讲。这里只需要知道:**Emitter 持有渲染器的"配置对象"(Properties),真正的运行时渲染器会从这些配置生成出来**——又是一次 Asset/Instance 二元模式的应用。

### 3.6 Simulation Stages(Phase 10 的接口)

```cpp
UPROPERTY(meta = (NiagaraNoMerge))
TArray<UNiagaraSimulationStageBase*> SimulationStages;
```

GPU 多 pass 计算(流体模拟基础),Phase 10 才讲。此处只留个接口。

### 3.7 继承 / Merge 机制:Niagara 的架构升级

这是 Cascade 没有、Niagara 独创的特性:

```cpp
#if WITH_EDITORONLY_DATA
    UPROPERTY() UNiagaraEmitter* Parent;                  // 此 Emitter 的模板来源
    UPROPERTY() UNiagaraEmitter* ParentAtLastMerge;      // 上次 merge 时父的快照
#endif
```

**一句话概括**:Niagara 的 Emitter 可以从另一个 Emitter"继承"——父 Emitter 改了,子 Emitter 按三方 merge 算法把变化合进来,同时保留子 Emitter 自己的改动。Cascade 里 duplicate 一个模板就"断线"了,父改动再也不会传到子。

这让 Niagara 可以构建一套**模板库**——例如公司内部积累的"标准爆炸"、"标准烟雾"等 Emitter 模板,项目里的特效从这些模板派生,模板改进时收益可以传播下去。

具体 merge 能传播什么、怎么冲突处理,属于编辑器模块的大主题,Phase 1 不展开。

### 3.8 Emitter 归纳

一个 Emitter 是:

- **一堆开关字段**(SimTarget / 空间 / 确定性 / 插值 spawn / 分配策略 / ...)
- **四类脚本**(ParticleSpawn / ParticleUpdate / GPUCompute / 若干 EventHandler)
- **一组渲染器配置**
- **一组 SimulationStage**(可选)
- **scalability 规则**(Platforms + ScalabilityOverrides)
- **editor-only 的继承链**(Parent + LastMerge)+ **图源**(GraphSource)

现在我们该往更深一层走了——**`UNiagaraScript` 到底是什么?**

---

## 4. 脚本:编译后的执行体

打开 `NiagaraScript.h`,800 多行。看起来吓人,但大部分是"usage 分类的布尔方法"(如 `IsParticleSpawnScript`、`IsSystemUpdateScript` 等),真正的核心就三样:

```cpp
UCLASS(MinimalAPI)
class UNiagaraScript : public UNiagaraScriptBase
{
public:
    UPROPERTY(AssetRegistrySearchable)
    ENiagaraScriptUsage Usage;            // 我是什么角色的脚本

    UPROPERTY() FNiagaraParameterStore RapidIterationParameters;  // 用户可调参数

    // 在私有区:
    UPROPERTY() FNiagaraVMExecutableData CachedScriptVM;           // CPU 字节码
    UPROPERTY() FNiagaraVMExecutableDataId CachedScriptVMId;       // 字节码的身份证
    TUniquePtr<FNiagaraShaderScript> ScriptResource;              // GPU shader 资源
};
```

**三行关键字段,就构成了 Niagara 脚本的本质**:

1. `Usage` 告诉世界"我是哪种脚本"
2. `CachedScriptVM` 是 CPU 路径要跑的字节码
3. `ScriptResource` 是 GPU 路径要用的 compute shader 资源

**一个 `UNiagaraScript` 同时承载 CPU 和 GPU 两种执行形态**——Phase 1 里最需要记住的就是这一点。

### 4.1 关键领悟:脚本不是源代码

一个常见误解是把 `UNiagaraScript` 当成"源代码文件"——**错**。`UNiagaraScript` 是**编译后的产物**:

- 在 editor 下,它同时还持有 `Source`(指向图源 `UNiagaraScriptSourceBase`,见第 5 节)
- 在 cooked / shipping 包里,`Source` 被剥离,`UNiagaraScript` 只剩编译产物

**真正的"源"是图(`UNiagaraGraph`,editor-only)**,`UNiagaraScript` 只是图被编译出来的那个产物的容器。

### 4.2 Usage 是什么?

```cpp
UENUM()
enum class ENiagaraScriptUsage : uint8
{
    // 编辑器素材级
    Function,
    Module,
    DynamicInput,
    // 粒子级
    ParticleSpawnScript,
    ParticleSpawnScriptInterpolated,  // bInterpolatedSpawning 开启时用这个
    ParticleUpdateScript,
    ParticleEventScript,
    ParticleSimulationStageScript,    // Phase 10
    ParticleGPUComputeScript,         // GPU 模式的合并脚本
    // Emitter 级
    EmitterSpawnScript,               // 注意:被合并进 System
    EmitterUpdateScript,              // 注意:被合并进 System
    // System 级
    SystemSpawnScript,
    SystemUpdateScript,
};
```

这里真正值得消化的,是这个枚举**同时承载了三层语义**:

- **生命周期**:Spawn(一次性) vs Update(每帧)
- **作用域**:Particle / Emitter / System
- **编辑器角色**:Function / Module / DynamicInput 是"可插入其他脚本的素材",不是独立执行单元

光靠"类名"区分是不够的(它们都叫 `UNiagaraScript`),所以用 `Usage` 枚举做运行时分发。

### 4.3 重要事实:`EmitterSpawn/Update` 不可单独编译

从 `UNiagaraScript.h` 里翻到这一行:

```cpp
bool IsCompilable() const { return !IsEmitterSpawnScript() && !IsEmitterUpdateScript(); }
```

**`EmitterSpawn` 和 `EmitterUpdate` 两种 Usage 的脚本 `IsCompilable()` 返 false**。这印证了第 1.2 节提到的"Emitter 级脚本被合并进 System 脚本"的事实——它们不是独立的字节码单元,编译管线会把它们融进 `SystemSpawnScript` / `SystemUpdateScript` 里。

这是为什么 `UNiagaraEmitter` 的非 editor 字段里没有 `EmitterSpawn/Update` 脚本——**它们不在运行时存在**。

### 4.4 编译产物三件套

#### `FNiagaraVMExecutableDataId`:身份证

```cpp
USTRUCT()
struct FNiagaraVMExecutableDataId
{
    FGuid CompilerVersionID;
    ENiagaraScriptUsage ScriptUsageType;
    FGuid ScriptUsageTypeID;
    uint32 bUsesRapidIterationParams : 1;
    uint32 bInterpolatedSpawn : 1;
    uint32 bRequiresPersistentIDs : 1;
    FNiagaraCompileHash BaseScriptCompileHash;  // 图内容哈希
    TArray<FNiagaraCompileHash> ReferencedCompileHashes;  // 依赖图的哈希
    // ...
};
```

**这个结构体回答一个问题**:"给定当前状态,这份脚本的编译产物是否还有效?"

它的 `operator==` + `GetTypeHash` 被用作 [[Wiki/Concepts/UE4/UE4-ddc|DDC(Derived Data Cache)]] 的 key——UE 的 DDC 是**机器级的编译产物缓存**,可以在团队间共享。两个人编译同一个图,只要这个 Id 一样,第二个人从 DDC 拉缓存即可,不用真跑编译。

#### `FNiagaraVMExecutableData`:字节码容器

```cpp
USTRUCT()
struct FNiagaraVMExecutableData
{
    TArray<uint8> ByteCode;                    // VectorVM 字节码本体
    TArray<uint8> OptimizedByteCode;           // 针对当前平台的二次优化(非序列化)
    int32 NumTempRegisters;
    TArray<FNiagaraVariable> Attributes;       // 此脚本读写了哪些粒子属性
    TArray<FNiagaraScriptDataInterfaceCompileInfo> DataInterfaceInfo;
    TArray<FVMExternalFunctionBindingInfo> CalledVMExternalFunctions;
    // ... 还有很多
};
```

这里有两个点值得关注:

1. `OptimizedByteCode` 是**非序列化**的——加载后会异步跑 `AsyncOptimizeByteCode()` 针对本机平台做二次优化(不同 CPU 指令集 / 访存模式)。
2. `Attributes` 是 editor 编译出来的"**此脚本读写了哪些粒子属性**"——这是后面 `bTrimAttributes`(Emitter 字段)能正确剔除无用属性的信息源。

#### `FNiagaraShaderScript`:GPU 侧

`ScriptResource` 指向一个 `FNiagaraShaderScript`——**GPU 模式的 compute shader 资源**。这个类型定义在 `NiagaraShader` 模块,Phase 8 才详细讲。此处只需要知道:**同一个 `UNiagaraScript` 对象,CPU 侧有 `CachedScriptVM`(字节码),GPU 侧有 `ScriptResource`(shader),互不干扰**。哪一个被用取决于 Emitter 的 `SimTarget`。

### 4.5 `RapidIterationParameters`:用户可调参数

```cpp
UPROPERTY()
FNiagaraParameterStore RapidIterationParameters;
```

这是一个**初学者容易混淆**的东西。Niagara 里有**三种**可调参数:

| 参数存储 | 位置 | 命名空间 | 改动是否触发重编译 |
|---|---|---|---|
| `User.*` | `UNiagaraSystem::ExposedParameters` | 用户暴露(BP/Sequencer 可读写) | 否 |
| `RapidIteration.*` | `UNiagaraScript::RapidIterationParameters` | 模块输入的实时调节 | 否(除非 `bBakeOutRapidIteration`) |
| `Module.*` 等普通输入 | 图内 | 内部连线 | 是 |

**RapidIteration 是一个性能优化**:当你在编辑器里拖一个数字滑块(比如某模块的 `Particle Lifetime`),你不希望每次拖动都触发全图重编译——太慢。所以这些参数被提到 `RapidIterationParameters` 里,改它不重编译,字节码照跑,只是参数表里的值变了。

烘焙时可以选 `bBakeOutRapidIteration`(Emitter 级或 System 级开关),把这些参数"烘死"回字节码里,代价是发布包里改不了,但运行时少一次 indirection。

**这个细节会在 Phase 5 读 `NiagaraScriptExecutionContext` 时再彻底讲清**,此处知道它存在、知道它的性能意图即可。

### 4.6 Script 小结

- **Usage 决定角色**(Particle/Emitter/System × Spawn/Update,加几种特殊)
- **CPU / GPU 双形态**,同一类共存(`CachedScriptVM` + `ScriptResource`)
- **编译产物 = VMExecutableDataId(身份证 + DDC key)+ VMExecutableData(字节码 + 元数据)+ ShaderScript(GPU 侧)**
- **EmitterSpawn/Update 不独立编译**,被合并进 System 脚本
- **三种参数存储**:User / RapidIteration / 图内连线,各有不同重编译触发策略

走到这里,你其实已经可以理解一个 Niagara 特效的**运行时全景**的 80%——剩下的最后一个谜团是:`UNiagaraScript` 和 `UNiagaraEmitter` 都有个 `Source` / `GraphSource` 字段,指向一个 `UNiagaraScriptSourceBase`。这玩意是什么?

---

## 5. 图源与 editor/runtime 解耦

`NiagaraScriptSourceBase.h` 是 Phase 1 里**最短的文件**(124 行),但它解决的架构问题值得单开一节讲。

### 5.1 问题的由来

`UNiagaraScript::Source`(editor-only)指向一个"图源"。"图"是什么?——是 Niagara 编辑器里你看到的那张节点连线网络。具体实现是 `UNiagaraGraph`,继承自 UE 通用的 `UEdGraph`(蓝图图的基类)。

但**`UNiagaraGraph` 住在 `NiagaraEditor` 模块**,不在 `Niagara`(runtime)模块。为什么?因为编辑器图依赖大量编辑器基础设施(`UEdGraph`、`UEdGraphNode`、slate UI 等),这些东西在 shipping 包里**根本不存在**。

可是 `UNiagaraScript` 住在 `Niagara` 模块(runtime),它想在 editor 场景下持有一个图的指针……**怎么持?**

### 5.2 解决方案:抽象基类在 runtime,实现在 editor

`NiagaraScriptSourceBase.h` 里就这一个类:

```cpp
/** Runtime data for a Niagara system */  // ← 注释过时,此类几乎不承载 runtime data
UCLASS(MinimalAPI)
class UNiagaraScriptSourceBase : public UObject
{
    // 一堆 virtual 方法,默认空实现
    virtual FGuid GetChangeID() { return FGuid(); }
    virtual bool IsSynchronized(const FGuid& InChangeId) { return true; }
    virtual void ComputeVMCompilationId(...) {}
    virtual TSharedPtr<FNiagaraCompileRequestDataBase, ESPMode::ThreadSafe>
        PreCompile(...) { return nullptr; }
    // ...
};
```

**关键点**:

- `UNiagaraScriptSourceBase` 住在 `Niagara`(runtime)模块,作为**抽象基类**
- 所有方法是虚方法 + 默认空实现(返回 `nullptr` / `false` / 空 `FGuid`)
- `UNiagaraScript::Source` 的类型是这个基类指针
- **真正的实现类 `UNiagaraScriptSource`** 继承它,住在 `NiagaraEditor` 模块,内部持有 `UNiagaraGraph* NodeGraph`
- 在 shipping 包里,`NiagaraEditor` 模块不被链接,`UNiagaraScript::Source` 要么是 nullptr,要么是被剥离的数据

**这是"抽象基类在底层模块,具体实现在上层模块"的典型解耦模式**。运行时代码能通过基类指针持有图源、调用通用接口,但不依赖具体的编辑器实现。

### 5.3 顺带一提的两个辅助类型

同一文件里还定义了两个与编译管线相关的纯 C++ 类(非 UClass):

- `FNiagaraCompileRequestDataBase`:编译请求的抽象接口,具体实现也在 NiagaraEditor 模块
- `FNiagaraCompileOptions`:编译选项(TargetUsage / Additional HLSL defines 等)

它们是同一设计哲学的体现——**运行时模块声明接口,编辑器模块填实**。

### 5.4 小结:为什么这个文件重要

读这个文件本身学不到太多实质内容(反正都是空方法),但它**给整个 Niagara 的模块划分定了调**:

> Niagara 插件**严格区分** runtime 模块(上线要用)和 editor 模块(开发期才用)。`UNiagaraScriptSourceBase` 是这条边界在 C++ 层面的一个具体实现。

理解这一点后,你再看 `UNiagaraScript::Source` 字段时就不会困惑——**"为什么这个字段指向一个看起来啥也不干的类"**?因为它在 runtime 看到的确实就是个空壳,真东西在 editor 模块。

---

## 6. 全景回看:五个类如何组成一个特效

现在把 Phase 1 的所有零件拼起来。一个保存在磁盘上的 Niagara 特效资产,在内存里大概是这个样子:

```
┌──────────────────────────────────────────────────────────────────┐
│  UNiagaraSystem  (一个 .uasset)                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  SystemSpawnScript (UNiagaraScript*)                       │ │
│  │    └─ CachedScriptVM: bytecode, attributes, DI info...    │ │
│  │    └─ Source: UNiagaraScriptSourceBase* (editor only)     │ │
│  │  SystemUpdateScript (UNiagaraScript*)                      │ │
│  │    └─ ... 同上                                             │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │  ExposedParameters: FNiagaraUserRedirectionParameterStore │ │
│  │     └─ User.Intensity, User.Color, ...                    │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │  EmitterCompiledData[]                                     │ │
│  │  SystemCompiledData  ← 编译期预计算的参数→偏移映射         │ │
│  │  EmitterExecutionOrder[]                                   │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │  EmitterHandles[]: TArray<FNiagaraEmitterHandle>           │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ FNiagaraEmitterHandle #0                             │ │ │
│  │  │   Id: {GUID}                                         │ │ │
│  │  │   Name: "SparkEmitter"                               │ │ │
│  │  │   bIsEnabled: true                                   │ │ │
│  │  │   Instance → UNiagaraEmitter *                       │ │ │
│  │  │               ┌──────────────────────────────────┐   │ │ │
│  │  │               │  UNiagaraEmitter                 │   │ │ │
│  │  │               │  SimTarget: CPUSim               │   │ │ │
│  │  │               │  bLocalSpace / RandomSeed / ...  │   │ │ │
│  │  │               │  SpawnScriptProps.Script →  *----+---+-→ UNiagaraScript
│  │  │               │  UpdateScriptProps.Script → *----+---+-→ UNiagaraScript
│  │  │               │  GPUComputeScript → *            │   │ │ │
│  │  │               │  EventHandlerScriptProps[]       │   │ │ │
│  │  │               │  RendererProperties[]            │   │ │ │
│  │  │               │  SimulationStages[] (Phase 10)   │   │ │ │
│  │  │               │  GraphSource → UScriptSourceBase │   │ │ │
│  │  │               │  Parent / ParentAtLastMerge      │   │ │ │
│  │  │               └──────────────────────────────────┘   │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  │  ┌──────────────────────────────────────────────────────┐ │ │
│  │  │ FNiagaraEmitterHandle #1  ( ... 同构 ... )            │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

**读懂这张图,Phase 1 就算通关了。**

---

## 7. 五条关键洞察(带走)

1. **System 的本质是 `TArray<FNiagaraEmitterHandle>`** — 其他所有字段都是围绕这个数组构建的上下文
2. **Handle 是 System 对 Emitter 的引用包装**,不是运行时 Instance — Handle 里的 `Instance` 字段仍在 Asset 层,真正运行时 Instance 是 Phase 3 的 `FNiagaraEmitterInstance`
3. **Emitter 级脚本(EmitterSpawn/Update)不独立编译**,被合并进 System 脚本 — `IsCompilable()` 对它们返 false 是铁证
4. **`UNiagaraScript` 同时承载 CPU 和 GPU 两种执行形态**,通过 `SimTarget` 分发 — 一个类,两条执行路径
5. **`UNiagaraScriptSourceBase` 是 editor/runtime 模块解耦的接缝** — "抽象基类在底层、实现在上层"的教科书案例

---

## 8. Phase 1 留下的问题(等 Phase 3+ 回答)

读完 Phase 1,下面这些问题还没答案,明确记下,以后 phase 回填:

- **`EmitterExecutionOrder.kStartNewOverlapGroupBit`** 在哪里被消费?——Phase 3 看 `FNiagaraSystemSimulation` 的并行 tick 时回答
- **`RapidIterationParameters` vs `ExposedParameters` vs `User.*`** 三者的完整协作机制?——Phase 5 看 `NiagaraScriptExecutionParameterStore` 时回答
- **`FNiagaraVMExecutableData::DIParamInfo`** 作者自己吐槽"不该在这里"的技术债,真正的归属点在哪?——Phase 7/8
- **`UNiagaraEmitter::Parent / ParentAtLastMerge`** merge 算法能传播什么、不能传播什么?——需要读 `NiagaraEditor/Private/INiagaraMergeManager` 实现,属于 Phase 1 范围外的加餐

---

## 9. Phase 2 预告

下一站:**Component 层**(`NiagaraComponent.h` + `NiagaraActor.h` + `NiagaraFunctionLibrary.h`)。

Phase 1 回答的是"**资产在磁盘上长什么样**"。Phase 2 回答的是"**这个资产怎么被放进 World、怎么被 BP 调用**"——也就是,Asset 如何与场景实例化机制(`USceneComponent` / `AActor`)嫁接。

Phase 2 很短(只有 3 个文件),难度与 Phase 1 同级(⭐⭐),主要是**接口清晰 + 生命周期流程**。读完后,你就能回答:

> 当 BP 里一个 `SpawnSystemAtLocation` 节点被执行,从 `UNiagaraSystem` Asset 到一个真正在场景里跑粒子的对象,中间发生了什么?

Phase 3 之后才会进入"运行时粒子真的在 tick"这部分硬核内容。

---

## 深入阅读

如果你想细究某个具体类的**全部字段、全部方法、代码行号标注**,去对应的原子 Source 页(也是 Claudian 在 LLM 检索时最常用的形式):

### 原子源摘要(逐文件)
- [[Wiki/Sources/Stock/NiagaraSystem]] — `NiagaraSystem.h` 全字段清单 + 代码片段引用
- [[Wiki/Sources/Stock/NiagaraEmitter]] — `NiagaraEmitter.h` 同上
- [[Wiki/Sources/Stock/NiagaraEmitterHandle]] — `NiagaraEmitterHandle.h` 同上
- [[Wiki/Sources/Stock/NiagaraScript]] — `NiagaraScript.h` 同上(含编译产物详解)
- [[Wiki/Sources/Stock/NiagaraScriptSourceBase]] — `NiagaraScriptSourceBase.h` 同上

### 实体页(按"类"索引,跨 source 稳定入口)
- [[Wiki/Entities/Stock/UNiagaraSystem]]
- [[Wiki/Entities/Stock/UNiagaraEmitter]]
- [[Wiki/Entities/Stock/FNiagaraEmitterHandle]]
- [[Wiki/Entities/Stock/UNiagaraScript]]
- [[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]

### 上下文(Phase 0 概念,前置)
- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — UCLASS/USTRUCT/UPROPERTY 宏
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — Asset vs Instance 二元模型
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — 设计哲学对比
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — SimTarget 分叉的根源

### 导航
- 学习路径总图:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]

---

*本读本由 [[Claudian]] 基于 Phase 1 的 5 个原子 Source 页综合生成,2026-04-19。*
