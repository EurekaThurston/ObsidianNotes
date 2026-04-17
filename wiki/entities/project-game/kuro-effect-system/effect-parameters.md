---
type: entity
category: parameter-system
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, legacy, parameters]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystem/EffectParameters
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 6985991"
aliases: [KuroEffectParameters, FKuroEffectParametersBase, FKuroEffectMaterialParameters, FKuroEffectNiagaraParameters]
---

# EffectParameters 家族(运行时参数层)

> 老版的**运行时参数收集 + 下发**抽象;把"特效运行中业务侧要下发到 NiagaraComponent/MaterialInstanceDynamic 的参数"统一成一组类。目录下 4 个文件。

## 整体结构

```
                  KuroEffectParameters (struct)           ← 纯数据,6 个 TMap
                            ▲
                            │(持有)
                    FKuroEffectParametersBase (class)     ← virtual Collect/Apply 抽象
                            ▲
                            │
             FKuroEffectMaterialParameters                ← 给 MaterialInstanceDynamic 用
                            
             FKuroEffectNiagaraParameters (standalone!)   ← 不走 Base,自有 TArray-based 实现
```

**注意** `FKuroEffectNiagaraParameters` **不继承 `FKuroEffectParametersBase`**——它自成一派。下面解释为什么。

## KuroEffectParameters(裸数据结构)

```cpp
struct KUROGAMEPLAY_API KuroEffectParameters
{
    TMap<FName, FKuroCurveFloat>        FloatCurveMap;
    TMap<FName, FKuroCurveLinearColor>  LinearColorCurveMap;
    TMap<FName, FKuroCurveVector>       VectorCurveMap;

    TMap<FName, float>         FloatConstMap;
    TMap<FName, FLinearColor>  LinearColorConstMap;
    TMap<FName, FVector>       VectorConstMap;

    void ApplyNiagara(UNiagaraComponent* NiagaraComp, float Time, bool bForceApply);
    void ApplyMaterial(UMaterialInstanceDynamic* MID, float Time, bool bForceApply);
    void Apply(UObject* Target, float Time, bool bForceApply);  // 分派
};
```

**设计点**:
1. **Curve 和 Const 分开**:同一个参数要么"随时间变化"(Curve),要么"固定值"(Const)。这两类永远不会同一个 key 同时生效——Apply 时先查 Const,再查 Curve,**或者反之**(具体优先级在 .cpp,Batch 6 确认)。
2. **三类值型**:Float / LinearColor / Vector——对应着色器/Niagara 里能配的参数类型。没有 `bool`,因为 bool 不存在"过渡曲线"的概念。
3. **Apply 分派**:`Apply(UObject*)` 根据 `Target` 的类型(`UNiagaraComponent*` / `UMaterialInstanceDynamic*`)**走不同下发路径**——Niagara 用 `SetVariableXxx`,Material 用 `SetScalar/VectorParameterValue`。

**注意**这里是 `struct`(不是 `UObject`),而且类名没有 `F` 前缀——这就是个纯 POD 级数据容器,不参与 UE 反射。

## FKuroEffectParametersBase(收集抽象)

```cpp
class FKuroEffectParametersBase
{
protected:
    KuroEffectParameters EffectParameter;    // 持有上面那个
    bool HasCurveParameters = false;         // 缓存:有没有 Curve(影响是否要 Tick)
public:
    virtual ~FKuroEffectParametersBase() {}
    
    // —— Collect API:业务侧往参数包里添加 ——
    virtual void CollectFloatCurve(FName Name, const FKuroCurveFloat& Value);
    virtual void CollectFloatConst(FName Name, float Value);
    virtual void CollectLinearColorCurve(FName Name, const FKuroCurveLinearColor& Value);
    virtual void CollectLinearColorConst(FName Name, const FLinearColor& Value);
    virtual void CollectVectorCurve(FName Name, const FKuroCurveVector& Value);
    virtual void CollectVectorConst(FName Name, const FVector& Value);
    
    // —— Remove API:清理 ——
    virtual void RemoveFloatCurveOrConst(FName Key);
    virtual void RemoveLinearColorCurveOrConst(FName Key);
    virtual void RemoveVectorCurveOrConst(FName Key);
    
    // —— Apply ——
    void Apply(UObject* Target, float Time, bool ForceApply);
};
```

**Collect vs Apply** 的分离很关键:
- **Collect** 阶段:业务代码调 `CollectFloatCurve(...)` 把参数写进 map。**不立刻生效**。
- **Apply** 阶段:每帧 Tick 时 `Apply` 把当前 Time 对应的值写到目标组件。

这样业务可以"提前几帧下发一个曲线参数",系统自己在正确的时间点把值推下去。

所有 `Collect*` / `Remove*` 是 virtual——子类可以拦截做额外处理(比如 Material 子类标 dirty)。

## FKuroEffectMaterialParameters(材质专用)

```cpp
class FKuroEffectMaterialParameters : public FKuroEffectParametersBase
{
    bool IsMaterialParameterDirty = false;   // ← 本类唯一新增字段
public:
    FKuroEffectMaterialParameters();
    FKuroEffectMaterialParameters(const TMap<FName, FKuroCurveFloat>& Floats,
                                  const TMap<FName, FKuroCurveLinearColor>& Colors);
    
    // 重写所有 Collect*:触发 IsMaterialParameterDirty = true
    virtual void CollectFloatCurve(FName Name, const FKuroCurveFloat& Value) override;
    virtual void CollectLinearColorCurve(...) override;
    virtual void CollectVectorCurve(...) override;
    virtual void CollectFloatConst(...) override;
    virtual void CollectVectorConst(...) override;
    virtual void CollectLinearColorConst(...) override;
    virtual void RemoveFloatCurveOrConst(...) override;
    virtual void RemoveVectorCurveOrConst(...) override;
    virtual void RemoveLinearColorCurveOrConst(...) override;

    void Tick(UObject* Target, float Time);    // 带 Dirty 检查的 Tick
    bool NeedTick() const;                     // 根据 HasCurveParameters + Dirty 决定
};
```

**关键优化:Dirty flag**:
- Material 参数下发(MID.SetScalarParameterValue / SetVectorParameterValue)是**每帧成本**——对每个材质实例每个参数做一次字符串查找+写入。
- 但**只有曲线参数需要每帧更新**。常量参数 Collect 之后只要下发一次。
- `Dirty flag` 和 `NeedTick()` 让 `FEffectSpec::Tick` 能精确知道"这个特效这一帧需不需要 re-Apply material"。

---

## FKuroEffectNiagaraParameters(为什么独立成一派)

文件顶部有段**宝贵注释**:

> ```
> //作为UObject在Ts函数传参时的效率是比Struct高的
> //不过在Ts中new一个新的UObject走反射消耗在四五十微妙，如果new得多可以考虑作为一个纯C++类，然后不走反射，走模板绑定
> ```

翻译这段思考:
1. 业务侧用 TS(Puerts)下发 Niagara 参数时,中间会经过 C++ 侧
2. `UObject` 在跨语言边界传参比 `struct` 高效(因为反射 infra 已缓存)
3. **但** TS 侧每次 `new` 一个 UObject 来当容器要走反射构造,耗时 40-50 微秒
4. **如果频繁调用(每帧多次),反射开销成瓶颈 → 改成纯 C++ class + 模板绑定**

这个文件就是选项(4)的产物——**它不是 UObject,也不继承 Parameters Base**,因为它要做得尽量轻。

### 结构

```cpp
// 4 种参数项
struct FParameterFloat       { FName Name; float Value; }
struct FParameterLinearColor { FName Name; FLinearColor Value; }
struct FParameterVector      { FName Name; FVector Value; }
struct FParameterArrayVector { FName Name; TArray<FVector> Value; }

class FKuroEffectNiagaraParameters
{
public:
    TArray<FParameterFloat>       UserParameterFloat;
    TArray<FParameterLinearColor> UserParameterColor;
    TArray<FParameterVector>      UserParameterVector;
    TArray<FParameterArrayVector> UserParameterArrayVector;
    TArray<FParameterFloat>       MaterialParameterFloat;
    TArray<FParameterLinearColor> MaterialParameterColor;
    
    FKuroEffectNiagaraParameters();
    FKuroEffectNiagaraParameters(const FKuroEffectNiagaraParameters&);   // 深拷贝
    FKuroEffectNiagaraParameters(const FKuroEffectNiagaraParametersStruct&);  // 从 Struct(反射版本)构造
};
```

### 关键差异(vs Base 家族)
- **TArray 而不是 TMap**:数组 cache-friendly,小规模参数集合(典型 <10)遍历比 hash 查找更快。代价是重复 Name 不会合并——但注释暗示业务方负责不重复。
- **User vs Material**:分两组——`UserParameterXxx` 下发到 Niagara User 参数,`MaterialParameterXxx` 下发到 Niagara 的材质参数。对应 Niagara System 内部的参数 scope。
- **没有 Curve!** 全是常值——曲线型参数走 `FKuroEffectParametersBase` 那条路,或者直接写在 DataAsset 的 `FloatParameters` Map 里(见 [[entities/project-game/kuro-effect-system/effect-model-base|UEffectModelNiagara]])。
- **`FKuroEffectNiagaraParametersStruct`** 是**反射版本**(USTRUCT,可跨 TS),ctor 允许从它构造到运行时版本——**跨边界一次 conversion 成本代替每帧反射**。

这是非常典型的"脚本 vs 运行时双版本"优化:
- 脚本/配置用 USTRUCT(可反射,易调试,可序列化)
- 运行时 hot path 用纯 class(零开销)
- 在边界一次性转换

## 使用场景对照

| 什么时候用哪个 |
|---|
| 从 TS 下发即时 Niagara 参数(高频) | `FKuroEffectNiagaraParameters` |
| DataAsset 里静态配置 Niagara 参数 | `UEffectModelNiagara::FloatParameters` 等 Map(Old),从 DA 读 |
| 业务代码下发动态材质参数(支持 Curve) | `FKuroEffectMaterialParameters`(通过 `Handle::CollectFloatCurve` 入口) |
| 业务下发 Niagara 曲线(特效进程中某参数从 0 渐变到 1) | `FKuroEffectParametersBase` ApplyNiagara(推测,.cpp 再确认) |

## 与其他实体的关系

- **用户**:[[entities/project-game/kuro-effect-system/effect-handle|FEffectHandle]] 的 `CollectFloatCurve` / `CollectVectorCurve` / `CollectLinearColorCurve` / `CollectEffectFloatConst` / ... / `RemoveEffectXxxCurveOrConst` —— 这些方法把参数 collect 到 Spec 内部持有的 `FKuroEffectMaterialParameters` 或 `FKuroEffectNiagaraParameters`
- **Spec(Batch 3)** 每种类型的 Spec 持有对应的 Parameters 实例;Tick 时调 Apply
- **目标组件**:`UNiagaraComponent` 和 `UMaterialInstanceDynamic` 是 Apply 的终点

## Twin(New 版对应)

**New 的参数层分裂到多个地方**:
- 运行时 Niagara 参数仍然叫 `FKuroEffectNiagaraParameters`(**同名同类**,看起来没重写)
- Handle 持有 `TUniquePtr<FKuroEffectNiagaraParameters> NiagaraParameters`(参见 [[entities/project-game/kuro-effect-system-new/effect-handle|New Handle]])
- Material 曲线通过 `FEffectHandle::CollectMaterialFloatCurve(...)` 下发(New Handle 的 Material 方法改了前缀)
- **Base 家族(`FKuroEffectParametersBase` + Material 子类)在 New 里是否保留?** 本批未读到——可能在 `EffectSpec/` 或 Private(Batch 3/6)
- 脚本侧新增 `FKuroParameterFloat` 等反射 USTRUCT(在 `KuroEffectSystemNew/Reflection/KuroEffectDefine.h`,Batch 5 细读)

## 引用来源

- 原始代码:
  - `F:\...\KuroEffectSystem\EffectParameters\KuroEffectParameters.h`
  - `F:\...\KuroEffectSystem\EffectParameters\KuroEffectParametersBase.h`
  - `F:\...\KuroEffectSystem\EffectParameters\KuroEffectMaterialParameters.h`
  - `F:\...\KuroEffectSystem\EffectParameters\KuroEffectNiagaraParameters.h`
- [[sources/project-game/kuro-effect-system/overview|Old 系统 overview]]

## 开放问题

- `KuroEffectParameters::Apply` 里 Const 和 Curve 共存时优先级?(.cpp 答)
- `FKuroEffectMaterialParameters::NeedTick()` 具体条件(`HasCurveParameters && (!Dirty || Dirty?)`)
- **Const 参数在 New 里消失了?** Batch 1 note:New 的 Handle 只有 `CollectMaterialXxxCurve`,没见 `CollectMaterialXxxConst`——是被合并成 `CollectMaterialXxxCurve` + `bUseCurve=false` 了?
- `FKuroEffectNiagaraParameters` 在 New 里**完全同名**——是直接复用?还是有同名不同定义?(Batch 5 确认)
