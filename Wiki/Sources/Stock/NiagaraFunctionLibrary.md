---
type: source
created: 2026-04-20
updated: 2026-04-20
tags: [niagara, UE4, source-code, function-library, blueprint]
sources: 1
aliases: [NiagaraFunctionLibrary.h, UNiagaraFunctionLibrary 源]
repo: stock
source_root: UE-4.26-Stock
source_path: Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraFunctionLibrary.h
source_ref: "4.26"
source_commit: b6ab0dee9
---

# NiagaraFunctionLibrary.h

- **Repo**: stock
- **路径**: `Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraFunctionLibrary.h`
- **快照**: commit `b6ab0dee9`(分支 `Eureka` / 基于 4.26)
- **文件规模**: 93 行
- **同仓协作文件**: `NiagaraFunctionLibrary.cpp`、`NiagaraComponent.h`、`NiagaraComponentPool.h`、`NiagaraParameterCollection.h`、`VectorVM.h`
- **Ingest 日期**: 2026-04-20
- **学习阶段**: Phase 2 — 场景入口 · Component 层 2.3

## 职责 / 这个文件干什么的

定义 `UNiagaraFunctionLibrary`——**Niagara 给 Blueprint 和游戏代码的静态工具集**,继承 `UBlueprintFunctionLibrary`。功能按用途分四组:

1. **动态生成特效**:`SpawnSystemAtLocation` / `SpawnSystemAttached`——BP 里调用最频繁的两个 Niagara 函数
2. **User 参数的"重型"覆盖**:专门覆盖 StaticMesh / SkeletalMesh / Texture / VolumeTexture 等 DI 承载的对象参数(轻量参数走 `UNiagaraComponent::SetVariableXxx`)
3. **Niagara Parameter Collection 访问**:类比材质的 ParameterCollection,跨特效共享参数
4. **VectorVM Fast Path 接线点**:CPU 脚本执行的性能优化路径注册——完全 C++ internal,本 Phase 只登记存在

## 关键类型 / 函数 / 宏

### 主类

- `UNiagaraFunctionLibrary : public UBlueprintFunctionLibrary`(L24):
  ```cpp
  UCLASS()
  class NIAGARA_API UNiagaraFunctionLibrary : public UBlueprintFunctionLibrary
  ```
  - 整个类都是 `static` 函数,符合 BP FunctionLibrary 规范

### 组 1:Spawn API

两个对等的入口:

- `SpawnSystemAtLocation(WorldContext, SystemTemplate, Location, Rotation, Scale, bAutoDestroy, bAutoActivate, PoolingMethod, bPreCullCheck) → UNiagaraComponent*`(L34):
  - 在世界坐标生成独立特效(爆炸、命中、环境播放触发)
  - 内部流程:创建/从 Pool 取一个 `UNiagaraComponent`,`SetAsset(SystemTemplate)`,设置 transform,`Activate`
  - `bPreCullCheck=true` 默认开,会先问 `FNiagaraScalabilityManager` 是否要 cull
  - `UFUNCTION` 元标:`WorldContext="WorldContextObject"`(BP 隐式参数)、`UnsafeDuringActorConstruction="true"`

- `SpawnSystemAttached(SystemTemplate, AttachToComponent, AttachPointName, Location, Rotation, LocationType, bAutoDestroy, bAutoActivate, PoolingMethod, bPreCullCheck) → UNiagaraComponent*`(L37):
  - 生成时自动挂到指定 Component 的 socket 上,跟随父变换(武器刀光、Buff 特效)
  - 内部启用 Component 的 **auto-attachment 子系统**(`bAutoManageAttachment=true` + `AutoAttachParent / AutoAttachSocketName / AutoAttachLocationRule/RotationRule/ScaleRule`)
  - `EAttachLocation::Type LocationType` 控制 `Location/Rotation` 的解释(World/Relative/SnapToTarget 等)

- `SpawnSystemAttached(..., FVector Scale, ...)`(L39):**C++ only 重载**(无 `UFUNCTION`),比 BP 版多一个 `Scale` 参数
  - 参数顺序与 BP 版不同:`Scale`/`bAutoActivate` 位置调换(历史遗留,BP 版补加 Scale 到内部另一重载)

### 组 2:User 参数覆盖(非标量)

标量参数(float/int/bool/Vec/Color)走 `UNiagaraComponent::SetVariableXxx`——18 个 setter。**对象型参数**(Mesh/Texture)走 FunctionLibrary——因为它们是 DataInterface 的内部对象,需要特殊处理:

| 函数 | 行 | 用途 |
|---|---|---|
| `OverrideSystemUserVariableStaticMeshComponent(Comp, Name, StaticMeshComponent)` | L43 | 让 Niagara Static Mesh DI 采样这个 MeshComponent |
| `OverrideSystemUserVariableStaticMesh(Comp, Name, StaticMesh)` | L46 | 同上,但直接给 UStaticMesh*(没有 MeshComponent 时) |
| `GetSkeletalMeshDataInterface(Comp, Name)` | L49 | **C++ only** 读 DI 指针,不是 UFUNCTION |
| `OverrideSystemUserVariableSkeletalMeshComponent(Comp, Name, SkelMeshComponent)` | L53 | 让 Skeletal Mesh DI 采样这个骨骼网格组件 |
| `SetSkeletalMeshDataInterfaceSamplingRegions(Comp, Name, TArray<FName> Regions)` | L57 | 修改采样区域(只能采指定 bone 区域的顶点) |
| `SetTextureObject(Comp, Name, Texture)` | L61 | 覆盖 Texture DI |
| `SetVolumeTextureObject(Comp, Name, VolumeTexture)` | L65 | 覆盖 Volume Texture DI |
| `GetDataInterface(UClass*, Comp, Name)` | L68 | **C++ only** 按类取 DI |
| `GetDataInterface<TDIType>(Comp, Name)` | L71-L75 | **模板包装** 上面那个(编译期类型安全) |

> [!note] 注释里的一段"Leap device"遗迹
> L20-L22 的类注释说 "positions & orientations ... assuming the Leap device is located at the origin"——显然是从 Leap Motion 体感控制器的某个早期例子复制过来的陈旧注释,和当前代码无关,可以忽略。

### 组 3:Parameter Collection

- `GetNiagaraParameterCollection(UObject* WorldContext, UNiagaraParameterCollection* Collection) → UNiagaraParameterCollectionInstance*`(L82):
  - 类比 `UMaterialParameterCollection`——"全局可变参数表",跨特效/材质共享
  - 返回一个 world-bound 的实例,可以 set/get 参数

### 组 4:VectorVM Fast Path(Phase 5 预留)

这部分是**纯 C++ internal**,服务 CPU VM 脚本执行的性能优化。本 Phase 只登记接口存在:

- `static const TArray<FNiagaraFunctionSignature>& GetVectorVMFastPathOps(bool bIgnoreConsoleVariable = false)`(L84):列出所有注册了 "fast path" 的 VM 操作
- `static bool DefineFunctionHLSL(const FNiagaraFunctionSignature&, FString& HlslOutput)`(L85):把 VM 函数定义翻译成 HLSL(供 GPU 版用)
- `static bool GetVectorVMFastPathExternalFunction(const FVMExternalFunctionBindingInfo&, FVMExternalFunction& OutFunc)`(L87):把 VM 函数绑定到 C++ 实现

私有存储:
- `static TArray<FNiagaraFunctionSignature> VectorVMOps`(L91)
- `static TArray<FString> VectorVMOpsHLSL`(L92)
- `static void InitVectorVMFastPathOps()`(L90):初始化静态表

深入留给 [[Wiki/Sources/Stock/NiagaraScriptExecutionContext]](Phase 5)时展开。

## 依赖与被依赖

**上游(此文件 include / 使用):**
- `CoreMinimal.h`、`ObjectMacros.h`
- `Engine/EngineTypes.h`(`EAttachLocation`)
- `Kismet/BlueprintFunctionLibrary.h`(父类)
- `NiagaraComponentPool.h`(`ENCPoolMethod`)
- `NiagaraCommon.h`(共享类型/枚举)
- `VectorVM.h`(Fast Path 类型)

**下游(谁用它):**
- BP 图——绝大多数"生成特效"的 BP 节点都在这里
- C++ 游戏代码——同上(`SpawnSystemAtLocation` 是最常用入口)
- 内部代码——VectorVM Fast Path 被 VM 编译器调用
- **不**被其他 Niagara 运行时类调用——FunctionLibrary 只是对外 facade

## 关键代码片段

> `NiagaraFunctionLibrary.h:L29-L34` @ `b6ab0dee9` — 最常用的那个函数
> ```cpp
> /**
> * Spawns a Niagara System at the specified world location/rotation
> * @return            The spawned UNiagaraComponent
> */
> UFUNCTION(BlueprintCallable, Category = Niagara, meta = (Keywords = "niagara System",
>     WorldContext = "WorldContextObject", UnsafeDuringActorConstruction = "true"))
> static UNiagaraComponent* SpawnSystemAtLocation(const UObject* WorldContextObject,
>     class UNiagaraSystem* SystemTemplate, FVector Location,
>     FRotator Rotation = FRotator::ZeroRotator, FVector Scale = FVector(1.f),
>     bool bAutoDestroy = true, bool bAutoActivate = true,
>     ENCPoolMethod PoolingMethod = ENCPoolMethod::None, bool bPreCullCheck = true);
> ```
> 默认参数含义:自销毁、自激活、不用池、预检剔除。**"不用池"是默认值 = 性能隐患**(每次 spawn 都是全新 Component)——见下陷阱区。

> `NiagaraFunctionLibrary.h:L71-L75` @ `b6ab0dee9` — 模板包装的类型安全 DI 取值
> ```cpp
> template<class TDIType>
> static TDIType* GetDataInterface(UNiagaraComponent* NiagaraSystem, FName OverrideName)
> {
>     return (TDIType*)GetDataInterface(TDIType::StaticClass(), NiagaraSystem, OverrideName);
> }
> ```
> 一个很工程的小设计:上一层 `GetDataInterface(UClass*, ...)` 返回基类指针,这里用模板补一层 cast,编译期查类型。

## 涉及实体 / 概念

- [[Wiki/Entities/Stock/UNiagaraFunctionLibrary]] — 本文件定义的主类
- [[Wiki/Entities/Stock/UNiagaraComponent]] — Spawn 的返回类型与参数载体
- [[Wiki/Entities/Stock/UNiagaraSystem]] — Spawn 的 Template 参数
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — Spawn = Asset → Component → Instance 的标准链路

## 与既有 wiki 的关系

- 是 [[Wiki/Sources/Stock/NiagaraComponent]] 的 BP 外立面——FunctionLibrary 本身不持有状态,所有状态都在返回的 Component 对象上
- VectorVM Fast Path 的存在印证了 [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]] 中 "CPU 模拟不等于慢"——针对常用算子有专门快速路径

## 开放问题 / 待深入

> [!warning] `PoolingMethod::None` 是默认值
> `SpawnSystemAtLocation` 默认不使用池。每次调用都会 new 一个 `UNiagaraComponent` + 一个 `FNiagaraSystemInstance`,播完销毁。频繁生成的小特效(命中、子弹拖尾)必须显式传入 `ENCPoolMethod::AutoRelease` 等池化策略,否则 GC 压力显著。

> [!warning] `UnsafeDuringActorConstruction`
> Spawn 函数不能在 Actor 构造(Constructor / OnConstruction)阶段调用——此时 World 未完全初始化。UE 会在运行时直接 ensure。

- `SpawnSystemAttached` 的 C++ 重载(带 Scale,L39)为什么参数顺序和 BP 版(L37)不一致?历史债务还是有意设计?→ 翻 git blame 可知,但对本 Phase 非关键
- `bPreCullCheck=true` 默认开会不会造成"生成即消失"的错觉?scalability 的 PreCull 决策表 → [[Wiki/Sources/Stock/NiagaraScalabilityManager]](Phase 9)
- `OverrideSystemUserVariableStaticMesh` 直接给 `UStaticMesh*`(而非 Component)时,DI 的采样如何取 transform?→ Phase 7 `NiagaraDataInterfaceStaticMesh`
- VectorVM Fast Path 注册了哪些算子?全部?还是 hot path 子集?→ Phase 5
- 为什么 `GetSkeletalMeshDataInterface` / `GetDataInterface(UClass*, ...)` 不是 `UFUNCTION`?——返回 `UNiagaraDataInterface*` 暴露给 BP 意义不大,BP 无法直接操作 DI
