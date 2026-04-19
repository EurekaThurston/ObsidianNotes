---
type: synthesis
created: 2026-04-19
updated: 2026-04-19
tags: [niagara, UE4, learning-path, phase-0, reader, foundation]
sources: 4
aliases: [Phase 0 读本, Niagara 心智模型读本, Niagara 前置概念]
---

# Phase 0 读本 — 上阵前的四层脑内地图

> 本页是 Niagara 学习路径 [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 0 的**主题读本**——详细、精确、满满当当,一次读完即掌握 Phase 1+ 所需的全部前置心智模型,不需要跳转。
>
> 定位与 [[Wiki/Readers/Niagara/Phase1-asset-layer-读本|Phase 1 读本]]对偶——Phase 0 建地图,Phase 1 在地图上走路。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。

---

## 0. Phase 0 要回答的问题

Phase 1 起会开始读真代码(5 个 header,3000+ 行)。直接跳过去会遇到一堆"看不懂的魔法":

- 为什么类名前面有 `U`、`F`、`A` 这种前缀?
- `UCLASS()`、`UPROPERTY()`、`GENERATED_BODY()` 是什么鬼?
- 为什么一个 `UNiagaraSystem` 的字段里没有"当前粒子位置",而是只有"参数定义"?
- 文档老说 "Asset" 和 "Instance",它们到底差别在哪?
- Niagara 到底相对 Cascade 好在哪?我们为什么要学这个新的?
- `SimTarget` 的 CPU / GPU 两个值一改,实际发生了什么?

**这些问题的答案都不在 Niagara 插件里**,而在 UE4 的更底层概念 + Niagara 的宏观设计哲学里。Phase 0 就是在读代码之前,把这四层地图先叠好:

```
┌──────────────────────────────────────────────────────┐
│  Layer 4:  Niagara CPU vs GPU 模拟                   │
│            SimTarget 字段分叉出两条执行路径            │
└──────────────────────────────────────────────────────┘
                        ▲
                        │ 建立在
┌──────────────────────────────────────────────────────┐
│  Layer 3:  Niagara vs Cascade(设计哲学)              │
│            数据显式化、脚本可编程、SIMD               │
└──────────────────────────────────────────────────────┘
                        ▲
                        │ 建立在
┌──────────────────────────────────────────────────────┐
│  Layer 2:  UE4 Asset vs Instance 二元模型             │
│            磁盘描述 vs 运行时活体,Component 是桥     │
└──────────────────────────────────────────────────────┘
                        ▲
                        │ 建立在
┌──────────────────────────────────────────────────────┐
│  Layer 1:  UE4 UObject 系统                          │
│            UCLASS / UPROPERTY / GC / 反射 / 序列化    │
└──────────────────────────────────────────────────────┘
```

读完本文你应当能:

> 用自己的话解释:"**一个 `UNiagaraSystem` 资产和一个正在运行的 `FNiagaraSystemInstance` 有什么区别?它们为什么要分开?**"

下面按这四层从下往上走。

---

## 1. Layer 1 — UE4 UObject 系统

UE4 不是普通 C++,**几乎所有"像对象"的东西都是 `UObject` 的子类**。理解了 UObject 系统,你就理解了 UE4 代码里所有奇怪的宏和命名约定的来源。

### 1.1 问题:C++ 本身缺什么

Java / Python 等语言自带这些能力:

- **GC**(垃圾回收):没人引用的对象自动释放
- **反射**:运行时能查"这个类有哪些字段和方法"
- **序列化**:对象可以保存到磁盘、从磁盘加载回来
- **统一根类**:所有类有个共同祖先(如 Java 的 `Object`)

**C++ 没有这些**。但一个 3D 引擎又特别需要:

- 编辑器要知道"你的 `UNiagaraSystem` 类有哪些字段",才能在 Detail Panel 里显示可编辑控件(需要反射)
- 资产要能存进 `.uasset`、从 `.uasset` 加载回来(需要序列化)
- 美术师放了个特效,游戏结束后没人引用了,要自动回收(需要 GC)
- 蓝图要能调用 C++ 函数、读 C++ 字段(需要反射)

### 1.2 解决:Epic 手工实现一套

Epic 用宏 + 代码生成 + UBT(Unreal Build Tool)做了这一整套基础设施。**所有继承 `UObject` 的类都自动拥有 GC / 反射 / 序列化 / 蓝图暴露 / 网络复制等能力**。

核心工具是**几个宏**——它们是 UE4 代码里看起来最"魔法"的东西:

#### `UCLASS()`

写在 C++ 类声明之前,告诉 UBT"把这个类纳入 UObject 系统":

```cpp
UCLASS(BlueprintType)
class NIAGARA_API UNiagaraSystem : public UFXSystemAsset
{
    GENERATED_UCLASS_BODY()
    // ...
};
```

- 括号里可以加选项:`BlueprintType`(BP 可用)、`Abstract`(不能实例化)、`MinimalAPI` 等
- `GENERATED_UCLASS_BODY()` / `GENERATED_BODY()` 是必须的占位宏,展开后由 UBT 插入大量胶水代码

**类比**:`UCLASS` 类似 Python 里的 `@dataclass` 或 Java 的注解——声明"我要被框架托管"。

#### `USTRUCT()`

同 `UCLASS`,但用于**结构体**(无 GC、轻量、可栈分配):

```cpp
USTRUCT()
struct NIAGARA_API FNiagaraEmitterHandle
{
    GENERATED_USTRUCT_BODY()
    // ...
};
```

#### `UPROPERTY()`

写在字段前,让引擎知道这个字段的存在(纳入 GC、序列化、编辑器显示):

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite)
TArray<FNiagaraEmitterHandle> EmitterHandles;
```

常见修饰符:`EditAnywhere`(编辑器可编辑)、`BlueprintReadWrite`、`Transient`(不序列化)、`VisibleAnywhere`(只读显示)、`AssetRegistrySearchable`(资产注册表可查)。

#### `UFUNCTION()`

写在函数前,让函数可被蓝图调用或反射调用:

```cpp
UFUNCTION(BlueprintCallable, Category="Niagara")
void Activate(bool bReset = false);
```

### 1.3 命名前缀——不是随便起的

这套宏配合一套命名约定,**你只要看一眼类名前缀就知道它的性质**:

| 前缀 | 含义 | 例子 |
|---|---|---|
| `U` | 继承自 `UObject`,被 GC 管理 | `UNiagaraSystem`、`UTexture2D` |
| `A` | 继承自 `AActor`(`A` 本身也是 U 的特殊子集,能放进场景) | `ANiagaraActor`、`ACharacter` |
| `F` | 普通 C++ 结构体/类,**不受 GC 管理** | `FNiagaraVariable`、`FVector`、`FNiagaraSystemInstance` |
| `I` | 接口类(纯虚) | `INiagaraMergeManager` |
| `E` | 枚举 | `ENiagaraScriptUsage`、`ENiagaraSimTarget` |
| `T` | 模板 | `TArray<T>`、`TSharedPtr<T>` |

**Niagara 里最关键的一组对比**——

- `UNiagaraSystem`(U 前缀)= 资产,被 GC 管理,存进 `.uasset`
- `FNiagaraSystemInstance`(F 前缀)= 运行时实例,**普通 C++ 对象**,由 `TSharedPtr` 管理

**这不是巧合**,是 UE4 设计思想的体现——见第 2 节 Asset/Instance 模型。

### 1.4 GC 带来的使用约束

一旦加入 UObject 系统,**你不能再用普通 C++ 的 `new`/`delete` 了**:

- 创建:用 `NewObject<T>()`
- 持有:必须通过 `UPROPERTY()` 标记的指针,或 `TObjectPtr<T>` / `TWeakObjectPtr<T>`
- **不能用裸指针**长期持有 UObject——GC 不知道你引用了它,会提前回收导致 crash

**F 前缀的对象**(普通 C++ 结构体)不受这些约束,用 `TSharedPtr<T>` / `TUniquePtr<T>` 管理即可。

### 1.5 智能指针速查

Niagara 代码里会反复遇到:

| 类型 | 管理什么 | 含义 |
|---|---|---|
| `TObjectPtr<T>` | UObject | 安全替代裸指针,配合 GC |
| `TWeakObjectPtr<T>` | UObject | 弱引用,不阻止 GC |
| `TSharedPtr<T>` | 普通 F 对象 | 引用计数(可 null) |
| `TSharedRef<T>` | 普通 F 对象 | 引用计数(不可 null) |
| `TUniquePtr<T>` | 普通 F 对象 | 独占所有权 |

### 1.6 本节小结

- UE4 的一切"被引擎托管的对象"都是 `UObject` 子类,通过 UCLASS/UPROPERTY 宏纳入系统
- 宏展开后由 UBT 生成反射/序列化/GC 胶水代码——你不需要理解实现细节,但得认识这些宏
- **U 前缀 vs F 前缀**的区别是 Niagara 里最高频的区分,贯穿 Phase 1 起所有代码

**有了这层基础,你看到 `UCLASS(BlueprintType) class UNiagaraSystem : public UFXSystemAsset { GENERATED_UCLASS_BODY() ... }` 就不再头疼**——你知道这只是在说"我是个纳入 UObject 系统的类,蓝图可见,继承 UE 特效资产基类"。

---

## 2. Layer 2 — Asset vs Instance 二元模型

这是 UE4 里最核心的设计模式,也是**读懂 Niagara 源码的第一把钥匙**。

### 2.1 一个日常类比

想象一份建筑蓝图 vs 一栋按图建造的楼:

- **蓝图**:画在纸上,描述"楼长什么样"。可以复印、存档、改了再建
- **楼**:实际存在的建筑,占地、住人、开灯——有自己的**状态**

UE4 里完全同样的模式:

| 概念 | 类比 | 技术形式 | 存在周期 |
|---|---|---|---|
| **Asset(资产)** | 蓝图/设计图 | UObject 子类,`.uasset` 文件 | 持久,存磁盘 |
| **Instance(实例)** | 实际建造的楼 | 普通 C++ 对象(F 类) | 临时,运行时 |

### 2.2 资产(Asset)

就是你在 Content Browser 里看到的那些文件。每个资产对应一个**序列化到磁盘**的 UObject 对象。

**资产的特征**:

- **只描述,不执行**:资产只保存"应该是什么样",不保存运行时状态
- **可共享**:同一个资产可以同时有 100 个实例,共享同一份资产数据
- **可序列化**:UObject 系统提供的能力(§1)

**Niagara 里的主要资产**(全部 U 前缀):

| 资产类 | 对应文件 | 说明 |
|---|---|---|
| `UNiagaraSystem` | `.uasset` | 整个特效系统的定义 |
| `UNiagaraEmitter` | `.uasset` | 一个 Emitter 的定义 |
| `UNiagaraScript` | 嵌在 Emitter 内 | 编译后的脚本(字节码) |
| `UNiagaraRendererProperties` 子类 | 嵌在 Emitter 内 | 渲染器配置 |

### 2.3 实例(Instance)

实例是资产在**运行时**的"化身"。在场景里放一个 `NiagaraComponent`、按 Play,引擎就会根据 `UNiagaraSystem` 资产**创建**一个 `FNiagaraSystemInstance`,这就是实例。

**实例的特征**:

- **持有运行时状态**:粒子当前位置、速度、生命值、当前时间
- **不持久**:特效播放完或游戏退出即销毁
- **不可共享**:场景里 100 个特效 = 100 个独立实例(状态各自独立)

**Niagara 里的主要实例**(全部 F 前缀,**非 UObject**):

| 实例类 | 对应资产 | 说明 |
|---|---|---|
| `FNiagaraSystemInstance` | `UNiagaraSystem` | 运行中的特效系统 |
| `FNiagaraEmitterInstance` | `UNiagaraEmitter` | 运行中的单个 Emitter |
| `FNiagaraDataSet` | — | 运行时粒子属性的实际内存 |

### 2.4 关系图

```
Content Browser(磁盘)              运行时(内存)
┌──────────────────────┐             ┌──────────────────────────┐
│  UNiagaraSystem      │──创建──────▶│  FNiagaraSystemInstance  │
│  (NS_Fire.uasset)    │             │  - 状态机                 │
│  - EmitterHandles[]  │   引用(只读) │  - 持有 EmitterInstances  │
│  - ParameterDefs     │◀────────────│  - 当前时间/帧计数        │
└──────────────────────┘             └──────────────────────────┘
         │                                      │
         │ 包含                                 │ 包含
         ▼                                      ▼
┌──────────────────────┐             ┌──────────────────────────┐
│  UNiagaraEmitter     │──创建──────▶│  FNiagaraEmitterInstance  │
│  - Scripts           │             │  - 粒子 DataSet           │
│  - RendererProps     │   引用(只读) │  - 当前粒子数量           │
└──────────────────────┘◀────────────│  - 运行时 DI 绑定         │
                                     └──────────────────────────┘
```

**关键点**:实例**引用**资产(只读取),**不修改**资产。**资产是只读模板**。

### 2.5 Component:连接资产与实例的桥梁

资产不会自己跑起来。游戏世界里,`UNiagaraComponent` 把资产和实例连起来:

```
场景里的 Actor
  └── UNiagaraComponent(组件)
         ├── 持有 UNiagaraSystem* 指针(资产引用)
         └── 持有 FNiagaraSystemInstance(实例,运行时创建)
```

BP 调用 `Spawn System At Location` 的背后:

1. 创建一个临时 Actor
2. 挂上 `UNiagaraComponent`
3. Component 根据你传入的 `UNiagaraSystem` 资产创建 `FNiagaraSystemInstance`
4. 开始 Tick

**这是 Phase 2 要读的内容**——从 Asset 到场景实例化的整条路径。

### 2.6 为什么要分离?

这个设计回答一个核心问题:**同一个特效在场景里同时出现 500 次,内存怎么办?**

- **不分离的方案**:每个特效完整复制一份 → 500 份脚本字节码 → 内存爆炸
- **分离的方案**:500 个实例**共享**同一份资产 → 资产只有 1 份 → 500 个实例各自只保存运行时状态(粒子位置/速度等)

额外好处:

- 资产可以在编辑器里修改后**热重载**,运行中的实例自动同步
- 资产可以被多个 Component 引用,实现"同一特效在不同场景复用"

### 2.7 快速记忆口诀

> **资产 = 食谱**(怎么做这道菜的描述)
> **实例 = 盘子里的菜**(正在吃的那份)
>
> 一份食谱可以同时做 100 盘菜;每盘菜的状态(吃了多少)独立。

### 2.8 ⚠️ 一个典型陷阱:Niagara 的 "Handle.Instance"

**这是 Phase 1 必定会踩的坑,先钉死**:

`FNiagaraEmitterHandle` 里有个字段叫 `Instance`,类型是 `UNiagaraEmitter*`。**这个 `Instance` 不是本节说的那个"运行时 Instance"**——它**仍在 Asset 层**,只是 "这个 Handle 所引用的 Emitter 资产副本"。

Niagara 里有**两层** "Emitter 的 Instance"概念:

- **Handle.Instance** = Emitter 资产副本(Asset 层)
- **FNiagaraEmitterInstance** = 运行时粒子模拟(Instance 层,Phase 3)

Phase 1 读 `NiagaraEmitterHandle.h` 时会专门讲这个陷阱。**现在先埋个雷**:看到 Niagara 代码里说 "Instance" 时,先问自己一句**"哪层 Instance?"**。

### 2.9 本节小结

- Asset = 磁盘上的只读模板,U 前缀,UObject,可共享
- Instance = 运行时的状态活体,F 前缀,普通 C++ 对象,每个独立
- Component 是连接桥梁,在场景里实际跑起来
- **"磁盘/只读 vs 内存/有状态"的分离是理解所有 Niagara 代码的基本坐标系**

---

## 3. Layer 3 — Niagara vs Cascade:为什么换?

到这里你已经理解了 UE4 的通用机制。现在要进 Niagara 插件之前,还需要知道一件事:**Niagara 为什么存在?它相对上一代粒子系统 Cascade 做了什么根本性的改变?**

这不只是"知道一点历史",而是**Niagara 源码里大量"看起来很复杂"的设计,其复杂性来源都是 Cascade 的某个限制**。理解这些动机,读代码时才不会困惑"这到底解决了什么问题"。

### 3.1 Cascade 的模型:黑盒模块

Cascade(UE3 延续,UE4 继续兼容)的特效由**固定模块**组合:Initial Velocity、Lifetime、Size by Life、Color over Life……每个模块是一个黑盒,**你只能填参数,不能修改它"怎么工作"**。

```
Cascade Emitter:
  ┌──────────────┐
  │ Spawn Rate   │  ← 固定逻辑,只改数值
  │ Initial Vel  │  ← 固定逻辑,只改数值
  │ Color/Life   │  ← 固定逻辑,只改数值
  └──────────────┘
```

**优点**:艺术家上手快。
**缺点**:要新效果就得程序员实现新模块;不同项目反复实现类似功能;粒子数据对用户不透明。

### 3.2 Niagara 的模型:完全可编程数据流

Niagara 的核心转变:**粒子行为由可视化脚本图定义**,图里任意连接,每个节点对应一个数学/逻辑操作。

```
Niagara Emitter Script(图示):
  [Particle.Position]  →  [Add]  →  [Particle.Position]
                               ↑
                    [Particle.Velocity * DeltaTime]
```

这意味着:

- **任意粒子属性可读写**:Update 脚本里读/写任意粒子属性
- **自定义属性**:可以给粒子加任意新字段(如 `Particle.MyCustomFloat`)
- **DataInterface**(数据接口):通过 DI 读外部数据(相机、骨骼、音频频谱……)

### 3.3 关键差异对照表

| 维度 | Cascade | Niagara |
|---|---|---|
| **设计哲学** | 黑盒模块组合 | 完全可编程数据流 |
| **行为定义** | 固定模块(内部不可改) | 可视化脚本图(完全自定义) |
| **数据可见性** | 粒子数据对用户不透明 | 所有属性完全可见可读写 |
| **CPU/GPU** | 主要 CPU(GPU 有限支持) | CPU + GPU 一等公民 |
| **性能模型** | 每粒子逐个处理 | **SIMD 批量处理**(VectorVM) |
| **源码位置** | `Engine/Source/Runtime/Engine/Classes/Particles/` | `Engine/Plugins/FX/Niagara/` |
| **当前状态** | 维护模式 | 主力开发 |

### 3.4 脚本阶段(Niagara 的固定执行流)

虽然脚本内容完全自定义,Niagara Emitter 的**执行阶段**是固定的——每帧按顺序跑几个脚本:

| 阶段 | 触发时机 | 作用 |
|---|---|---|
| `EmitterSpawnScript` | Emitter 首次激活 | 初始化 Emitter 级变量 |
| `EmitterUpdateScript` | 每帧 | 更新 Emitter 级变量(如发射速率) |
| `ParticleSpawnScript` | 新粒子诞生 | 初始化粒子属性 |
| `ParticleUpdateScript` | 每帧,所有活跃粒子 | 更新粒子属性 |
| `EventHandlerScript` | 收到事件 | 响应碰撞/死亡等 |
| `SimulationStageScript` | GPU 专用 | 多 pass 计算(流体等) |

**这些脚本类型对应 `UNiagaraScript::Usage` 枚举(`ENiagaraScriptUsage`)——Phase 1 读 `NiagaraScript.h` 时直接就会看到**。

### 3.5 SIMD 批量执行:VectorVM 的本质

Cascade 的逐粒子模型:

```
for particle in particles:
    process(particle)   ← 逐个处理,CPU cache 不友好
```

Niagara 的 VectorVM 模型:

```
process(particles[0..3])  ← 一次处理 4 个粒子(SIMD 4-wide)
process(particles[4..7])
...
```

VectorVM 是专为 SSE/AVX 指令集优化的"微型 CPU",粒子多时性能差异巨大。**具体实现在 `Engine/Source/Runtime/VectorVM/`(引擎内置,不在 Niagara 插件里)**,Phase 5 会再关联。

### 3.6 最深层的哲学转变:显式 vs 隐式

这是理解 Niagara 源码复杂度的**根本原因**——

**Cascade 的隐式**:
- 粒子有哪些属性?固定,写死在代码里
- 粒子数据怎么存?引擎内部黑盒,不对外暴露
- 模块怎么执行?按固定顺序

**Niagara 的显式**:
- 粒子有哪些属性?**完全动态**——脚本图用到哪些,编译时确定(`FNiagaraTypeDefinition` + `FNiagaraVariable`)
- 粒子数据怎么存?**显式的 DataSet**(`FNiagaraDataSet`),SoA 布局,每属性一个连续数组(Phase 4)
- 模块怎么执行?**编译成字节码**,由 VectorVM / GPU Compute Shader 执行(Phase 5/8)

**代价**:
- ✅ 极高的灵活性和可扩展性
- ✅ 完全的 GPU 原生支持
- ❌ **源码显著更复杂**(编译、绑定、反射系统、DI 机制……都是你要学的东西)

**带来的好处**:**Niagara 是一个可编程的粒子编译器 + 执行引擎**,Cascade 只是一个带参数的固定逻辑库。

### 3.7 ⚠️ 兼容性提示

UE4 同时允许 Cascade 和 Niagara 资产存在。Cascade 的资产是 `UParticleSystem`,Niagara 的是 `UNiagaraSystem`——**扩展名都是 `.uasset`,但类型完全不同,不可互换**。

读源码时偶尔会看到 `// Legacy` 或 Cascade 风格注释——那是历史遗留。Niagara 在 4.20 引入时没完全重写,`UFXSystemAsset`(两者共同基类)就是典型印记。

### 3.8 本节小结

- Cascade = 黑盒模块库;Niagara = 可编程数据流编译器
- Niagara 的复杂性**不是过度工程**,而是"数据/执行全部显式化"带来的必然代价
- **Niagara 的脚本阶段(Spawn/Update/Event/SimStage)是固定的**,但每阶段内部的脚本内容完全可编程——这个"固定骨架 + 可编程填充"是理解整个运行时流程的关键
- VectorVM 是 CPU 侧性能的关键,Phase 5 会深入

---

## 4. Layer 4 — Niagara CPU vs GPU 模拟

Niagara 相对 Cascade 的另一个质变,是它让 **GPU 粒子模拟成为一等公民**。每个 Emitter 可以独立选择在 CPU 还是 GPU 上跑——这个选择深刻影响它的能力边界和源码实现路径。

### 4.1 为什么同时支持两条路径?

粒子模拟的本质是**每帧对大量粒子执行相同计算**。"大量相同计算"天然适合并行,而 CPU 和 GPU 的并行能力天差地别:

| | CPU | GPU |
|---|---|---|
| 核心数 | 8 ~ 32 | 数千个计算单元 |
| 适合任务 | 复杂逻辑、分支多、需要游戏状态 | 简单重复、大规模并行 |
| Niagara 执行器 | **VectorVM**(SIMD 软件 VM) | **Compute Shader**(HLSL) |
| 典型粒子上限 | 数万 | 数十万 ~ 百万 |
| 调试难度 | 容易 | 困难 |

**结论**:不是非此即彼。两条路径各擅胜场,Niagara 保留两者并让用户每 Emitter 选择。

### 4.2 CPU 模拟:VectorVM 路径

CPU 模拟**不是**逐粒子跑普通 C++,而是跑在叫 **VectorVM** 的自定义虚拟机上:

- 脚本图被编译成**字节码**
- 字节码指令集专为 SIMD 设计,每条指令同时操作 4 或 8 个粒子
- 在 CPU 上利用 SSE/AVX 指令集并行执行

**类比**:VectorVM 是个专为粒子计算设计的"微型 CPU",每次处理一小批粒子循环到底。

**执行流**:

```
UNiagaraScript (字节码)
      │
      ▼
FNiagaraScriptExecutionContext  ← 绑定:字节码 + DataSet + DataInterface
      │
      ▼
VectorVM::Exec()  ← 实际执行,每次 4/8 粒子
      │
      ▼
FNiagaraDataSet  ← 粒子属性数组(Position, Velocity, ...) 被更新
```

**CPU 模拟的优势**:

- ✅ 可访问游戏状态(Actor 位置、物理、游戏逻辑变量)
- ✅ 精确碰撞(`NiagaraDataInterfaceCollisionQuery` 做完整 line trace)
- ✅ 事件系统(粒子间发送/接收事件)
- ✅ 调试友好(Niagara Debugger 可逐粒子检查)
- ✅ 所有 DataInterface 全支持

**CPU 模拟的局限**:

- ❌ 粒子数量上限低(超过几万帧率明显下降)
- ❌ 主线程/任务线程有 CPU 预算开销
- ❌ 无法利用 GPU 的天然并行

### 4.3 GPU 模拟:Compute Shader 路径

GPU 模拟把脚本**编译成 HLSL Compute Shader**,GPU 上执行。每个 GPU 线程处理一个粒子,成千上万线程并行。

**执行流**:

```
UNiagaraScript (GPU 字节码 / HLSL)
      │
      ▼
NiagaraShader (FNiagaraShader)  ← 编译为 GPU Compute Shader
      │
      ▼
FNiagaraEmitterInstanceBatcher  ← 每帧把 Dispatch 命令提交给渲染线程
      │
      ▼
GPU 执行  ← 数万线程并行处理所有粒子
      │
      ▼
GPU Buffer(粒子数据留在显存)  ─────────────▶ 渲染(Vertex Factory 直接读)
```

**关键优化**:GPU 模拟的粒子数据**留在 GPU 显存**,不拷回 CPU。渲染阶段 Vertex Factory 直接从 GPU Buffer 读——这是 GPU 粒子高效的核心。

**GPU 模拟的优势**:

- ✅ 海量粒子(百万级)
- ✅ 零 CPU 模拟开销(CPU 只提交命令)
- ✅ 渲染零拷贝(CPU↔GPU 不搬数据)

**GPU 模拟的限制**:

- ❌ 访问游戏状态受限(必须通过 DI 提前上传)
- ❌ 不支持所有 DataInterface(只有实现 `GetParameterDefinitionHLSL/FunctionHLSL` 的 DI 才能在 GPU 用)
- ❌ **Fixed Bounds 要求**:CPU 不知道 GPU 粒子在哪,所以 GPU Emitter **必须手动设置 Bounds**,否则遮挡剔除出错
- ❌ 调试极难
- ❌ 粒子数量预分配(`MaxParticleCount` 固定,不能运行时扩容)

### 4.4 如何选择

Niagara Editor 里每个 Emitter 右上角有 **"Sim Target"** 下拉:

- `CPU Sim`
- `GPU Compute Sim`

**选 CPU 当**:粒子 < ~5000、需精确碰撞、需事件、需复杂游戏状态
**选 GPU 当**:海量粒子(烟雾、液体、群体)、视觉为主、可接受 Fixed Bounds

### 4.5 源码里如何体现这个分叉

**Emitter 的 SimTarget 字段**(Phase 1 会读):

```cpp
// NiagaraEmitter.h
UPROPERTY(EditAnywhere, Category = "Emitter")
ENiagaraSimTarget SimTarget;  // CPUSim 或 GPUComputeSim
```

**运行时分叉点**:`FNiagaraEmitterInstance::Tick` 内按 `SimTarget` 走不同路径。

**双形态承载**:**同一个 `UNiagaraScript` 对象同时承载 CPU 和 GPU 两种执行形态**——CPU 侧有 `CachedScriptVM`(字节码),GPU 侧有 `ScriptResource`(shader 资源),Emitter 的 `SimTarget` 决定用哪一个。Phase 1 读 `NiagaraScript.h` 时会再次遇到。

### 4.6 混合使用

一个 `UNiagaraSystem` 可以**同时包含 CPU 和 GPU Emitter**。例如:

- GPU Emitter:数万火花(大量轻量)
- CPU Emitter:5 个大火球(少量但需碰撞)

两种 Emitter 通过 `FNiagaraSystemInstance` 统一管理,各走各的执行路径,最终一起渲染。

### 4.7 本节小结

- `SimTarget` 是 Emitter 的一个字段,决定走 VectorVM 还是 Compute Shader
- 两条路径能力不对等:CPU 灵活但粒子少;GPU 海量但访问受限
- **同一个 `UNiagaraScript` 类同时承载两种编译产物**——从 Phase 1 开始就能看到这个双形态
- Phase 5 深入 CPU 路径(VectorVM),Phase 8 深入 GPU 路径(Compute Shader + Batcher + Shader Map)

---

## 5. 四层地图回看

把 Phase 0 的四层贯通起来——读完下面这张图,你就可以放心进 Phase 1 了:

```
┌───────────────────────────────────────────────────────────────────┐
│  你双击 Content Browser 里 NS_Fire.uasset                         │
│    │                                                              │
│    ▼                                                              │
│  [Layer 1: UObject 系统] 告诉引擎"把它反序列化为 UNiagaraSystem"   │
│    │  这是个 U 前缀的类,被 GC 管理,有反射元数据                   │
│    │  UCLASS/UPROPERTY 的魔法就是在这发生                         │
│    ▼                                                              │
│  [Layer 2: Asset vs Instance] 这只是个"资产",只描述不执行          │
│    │  它里面没有"当前粒子位置",只有"特效的定义"                    │
│    │  在场景里放 NiagaraComponent → 引擎创建 FNiagaraSystemInstance│
│    │  实例持有运行时状态,引用资产为只读模板                        │
│    ▼                                                              │
│  [Layer 3: Niagara 哲学] 这个资产里装的是什么?                    │
│    │  不是固定模块,是若干 Emitter + 脚本图                         │
│    │  脚本图编译成字节码(VM)或 HLSL(GPU)                          │
│    │  属性(Particle.Position 等)完全动态,编译时确定                │
│    │  固定骨架(SystemSpawn/Update, ParticleSpawn/Update, ...)     │
│    │  + 可编程填充                                                │
│    ▼                                                              │
│  [Layer 4: CPU/GPU] 每个 Emitter 的 SimTarget 字段决定            │
│    │  CPUSim → VectorVM 跑字节码 → 数据在 CPU RAM                 │
│    │  GPUComputeSim → Compute Shader 跑 HLSL → 数据在 GPU VRAM    │
│    │  同一个 UNiagaraScript 同时承载两种编译产物                   │
│    ▼                                                              │
│  渲染阶段:VertexFactory 从 DataSet 读粒子数据,提交给 GPU 绘制     │
└───────────────────────────────────────────────────────────────────┘
```

---

## 6. 五条关键洞察(带走)

1. **U/F/A 前缀不是装饰**——它们告诉你"这个类受不受 GC 管理"。`U` = UObject = GC;`F` = 普通 C++ = 手动。所有 Niagara 代码的"资产 vs 实例"都沿着这个前缀分界
2. **Asset 是只读模板,Instance 持有运行时状态**——Niagara 里几乎每个概念都有 Asset(U 前缀)和 Instance(F 前缀)两个版本,是 Phase 1-3 的基本坐标系
3. **Niagara 的复杂性根源是"一切显式化"**——粒子属性、执行流、数据布局全是运行时/编译时可见的数据,这是它比 Cascade 复杂的核心原因,也是它能做到 GPU 原生支持和 SIMD 性能的基础
4. **脚本是"固定骨架 + 可编程填充"**——SystemSpawn/Update、ParticleSpawn/Update、EventHandler、SimulationStage 这些阶段是固定的,但每阶段里装什么完全由用户脚本图决定
5. **CPU/GPU 两条路径由 Emitter 的 SimTarget 字段分叉**,同一个 `UNiagaraScript` 类同时承载两种编译产物(`CachedScriptVM` 字节码 + `ScriptResource` GPU shader)——Phase 5 和 Phase 8 各自深入一条

---

## 7. Phase 0 留下的问题(等后续 Phase 回答)

- **`CDO`(Class Default Object)**:UClass 上有个"默认实例",`GetDefaults()` 返回它。有时你取到的不是运行中的对象,是 CDO——Niagara 的 Emitter merge 机制跟 CDO 相关。Phase 1 / 编辑器加餐时再展开
- **编译流水线的细节**:资产保存的不是原始图,而是编译后数据(`FNiagaraSystemCompiledData`)。编译/DDC 缓存的完整路径 Phase 1(`FNiagaraVMExecutableData`)会建立,Phase 5 才真正追到 VectorVM
- **事件系统如何跨 Emitter 传递**:Niagara 支持粒子发送/接收事件,但这套机制的实现(`FNiagaraEventReceiverProperties` / `FNiagaraEventGeneratorProperties`)只有到 Phase 3/4 读运行时 + 数据模型时才真正闭环
- **DataInterface 的 CPU/GPU 双实现**:`GetFunctions()` 对应 CPU 函数绑定,`GetFunctionHLSL()` 对应 GPU 代码生成——两边如何保持一致?Phase 7 专门讲

---

## 8. Phase 1 预告

下一站:**资产层三件套 Asset**(`NiagaraSystem.h` + `NiagaraEmitter.h` + `NiagaraEmitterHandle.h` + `NiagaraScript.h` + `NiagaraScriptSourceBase.h`,共 5 文件)。

Phase 0 建地图,Phase 1 在地图上走路。Phase 1 回答:

> 当一个 Niagara 特效处于"静止"状态(没在场景里播放),它在内存里的数据结构到底长什么样?

你会看到本文所有的抽象概念变成真实的 `UCLASS` / `USTRUCT` / `UPROPERTY`:

- `UNiagaraSystem`(U 前缀 → §1)= Asset(§2)= "容器 + 固定骨架"(§3)
- `UNiagaraEmitter` 里的 `ENiagaraSimTarget SimTarget`(§4)
- `UNiagaraScript` 同时持有 CPU 字节码(`CachedScriptVM`)和 GPU shader(`ScriptResource`)(§4)
- ……

读完 Phase 1 的 [[Wiki/Readers/Niagara/Phase1-asset-layer-读本|读本]],你就能回答 Phase 0 开头那个问题了:**"UNiagaraSystem 资产和 FNiagaraSystemInstance 实例差别在哪?"**

---

## 深入阅读

### Phase 0 的四个原始 Concept 页

- [[Wiki/Concepts/UE4/UE4-uobject-系统]] — Layer 1 全文(UCLASS/UPROPERTY 宏、前缀、GC、智能指针速查)
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — Layer 2 全文(食谱/菜类比、Component 桥、为什么分离)
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade]] — Layer 3 全文(哲学对比、脚本阶段、SIMD 模型、显式/隐式)
- [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] — Layer 4 全文(VectorVM、Compute Shader、DataInterface 两侧、混合使用)

### 下一 Phase

- [[Wiki/Readers/Niagara/Phase1-asset-layer-读本]] — Phase 1 读本
- 原子 Source/Entity 页入口见 [[Wiki/Syntheses/Niagara/Niagara-learning-path]]

### 总图

- [[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 10 阶段学习路径导航

---

*本读本由 [[Claudian]] 基于 Phase 0 的 4 个概念页综合生成,2026-04-19。*
