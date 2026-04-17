---
type: entity
category: factory
created: 2026-04-17
updated: 2026-04-17
tags: [project-game, kuro-effect-system, effect, current, factory]
sources: 1
repo: project-game
source_root: project-game
source_path: Plugins/Kuro/KuroGameplay/Source/KuroGameplay/Public/KuroEffectSystemNew/EffectSpec/EffectSpecFactory.h
source_ref: branch_3.4 (p4)
source_commit: "p4: CL 7086024"
aliases: [FEffectSpecFactory]
---

# FEffectSpecFactory(Spec 工厂)

> New 独有的**Spec 对象工厂**。14 行头文件,2 个静态方法,但它是整个 New 系统"数据驱动构造"的关键中枢——把 [[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]] 的 `SpecType` 一步 dispatch 成对应 `IEffectSpecBase` 派生类。

## 接口

```cpp
namespace KuroEffect
{
    class FEffectSpecFactory
    {
    public:
        static TSharedPtr<IEffectSpecBase> CreateEffectSpec(const FEffectSpecData& EffectSpecData);
        static EEffectSpecDataType GetEffectSpecType(const UEffectModelBase* EffectModelBase);
    };
}
```

就 2 个 `static` 方法:
1. **`CreateEffectSpec`** — 按数据创建运行时对象
2. **`GetEffectSpecType`** — 反向查询:给一个 DA,返回其 `EEffectSpecDataType`

**.h 里没实现**——全部在 .cpp(Batch 6 细读)。但从类型签名和上下文可以推断实现。

## `CreateEffectSpec` 的职责(推断)

```cpp
// 伪代码,.cpp 未读
TSharedPtr<IEffectSpecBase> FEffectSpecFactory::CreateEffectSpec(const FEffectSpecData& EffectSpecData)
{
    switch (EffectSpecData.SpecType)   // EEffectSpecDataType
    {
    case EEffectSpecDataType_Niagara:
        return MakeShared<FEffectModelNiagaraSpec>();
    case EEffectSpecDataType_Group:
        return MakeShared<FEffectModelGroupSpec>();
    case EEffectSpecDataType_MultiEffect:
        return MakeShared<FEffectModelMultiEffectSpec>();
    case EEffectSpecDataType_Decal:
        return MakeShared<FEffectModelDecalSpec>();
    case EEffectSpecDataType_PostProcess:
        return MakeShared<FEffectModelPostProcessSpec>();
    case EEffectSpecDataType_Audio:
        return MakeShared<FEffectModelAudioSpec>();
    // ... 17 种
    default:
        return nullptr;
    }
}
```

关键点:
- 返回 **`TSharedPtr<IEffectSpecBase>`** 而非裸指针——ownership 明确
- 构造时**不传任何参数**——Spec 的真正初始化推迟到 [[entities/project-game/kuro-effect-system-new/effect-spec-base|FEffectSpec<T>::Init(EffectData, InitModel)]] 调用
- **空构造的 Spec 不能立即用**——必须先 `SetHandle`、再 `InitLifeTime`、再 `Init`

## `GetEffectSpecType` 的职责

给一个 `UEffectModelBase*`,返回其对应的 `EEffectSpecDataType`:
```cpp
// 伪代码
EEffectSpecDataType FEffectSpecFactory::GetEffectSpecType(const UEffectModelBase* EffectModelBase)
{
    if (EffectModelBase->IsA<UEffectModelNiagara>()) {
        UEffectModelNiagara* N = Cast<UEffectModelNiagara>(EffectModelBase);
        if (N->bPointCloudNiagara) return EEffectSpecDataType_PointCloudNiagara;  // 细化
        return EEffectSpecDataType_Niagara;
    }
    if (EffectModelBase->IsA<UEffectModelGroup>()) return EEffectSpecDataType_Group;
    /* ... */
}
```

**为什么要这个方法?**

`FEffectSpecData` 在 db 配置时已经有 SpecType——但运行时如果**只拿到一个 UEffectModelBase\***(比如从 TS 直接传入,或者动态创建的 DA),需要反推类型。

注意 **`bPointCloudNiagara`** 的细分:从 Old 的 "Niagara + flag" 到 New 的 "独立 SpecType"——不同的 SpecType 会走 [[#CreateEffectSpec]] 的不同分支,产出不同 Spec 子类,运行时路径完全分离。

## 在 Spawn 流程里的位置

```
FEffectSystem::SpawnEffect(WorldContext, Transform, Path, Reason, ...)
         │
         ▼
查 EffectForSpecMap 得 FEffectSpecData(含 SpecType)
         │
         ▼
FEffectSpecFactory::CreateEffectSpec(SpecData)   ←──── 就在这里
         │
         ▼
得到 TSharedPtr<IEffectSpecBase> Spec
         │
         ▼
Handle->CreateEffectSpec(SpecData) 或类似:
      Spec->SetHandle(Handle)
      Spec->InitLifeTime(SharedSpec)
         │
         ▼
Handle 持有 Spec,业务回 SpawnEffect 得 EffectId
         │
         ▼
异步 LoadObject(Path) 拿到 UEffectModelBase*
         │
         ▼
Handle->Init(EffectData, InitModel)
      实际是:Spec->Init(EffectData, InitModel)
           - EffectModel = Cast<T>(EffectData)
           - OnInit(...)   ← 子类专属初始化(如 Niagara 创建 Component)
           - EffectInitModel->DoHandleInitCallback(Success/Fail)
```

**Factory 的职责清晰**:**只负责构造对应类型的空 Spec**。填充数据(`UEffectModelBase*`)、建立关系(`SetHandle`)、启动异步资源加载(LoadObject)都不是它的事——分层非常干净。

## 与 Old 的对比

**Old 没有集中的 Factory**,Spec 构造分散在:
- `FKuroEffectSystem::RegisterEffectHandle`(7 个重载)里 `new` 对应 Spec 类型
- `FKuroEffectSystemInterface::FindEffectSpecBase` 作为查询接口

这意味着 Old 加一种新 Spec 要改:
1. DA 类(新类型)
2. Spec 类(新 hpp)
3. `FKuroEffectSystem` 增加重载(或改已有)
4. TS 侧 `RegisterEffectHandle` 调用点
5. `FKuroEffectSystemInterface` 查询逻辑(如需)

New 加新 Spec 只改:
1. DA 类(若需)
2. Spec 类(新 hpp)
3. **Factory 的 `CreateEffectSpec` + `GetEffectSpecType`**(**集中在一处**)

—— 新增类型的成本明显降低。

## 与其他实体的关系

- **输入**:[[entities/project-game/kuro-effect-system-new/effect-spec-data|FEffectSpecData]]
- **输出**:[[entities/project-game/kuro-effect-system-new/effect-spec-base|TSharedPtr<IEffectSpecBase>]]
- **调用者**:[[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle::AfterConstruct]] / [[entities/project-game/kuro-effect-system-new/effect-handle|FEffectHandle::CreateEffectSpec]] / 别处(Batch 6 细查)
- **依赖 17 种 FEffectModelXxxSpec**:见 [[entities/project-game/kuro-effect-system-new/effect-spec-subclasses|New Spec 子类目录]]

## Twin(Old 版对应)

**Old 没有直接对应**。功能散落在 `FKuroEffectSystem::RegisterEffectHandle` 重载 + `FKuroEffectSystemInterface`。New 把它抽成显式 Factory **是架构的一大进步**——直接的体现:新增 Spec 类型从"改 5 处" → "改 2 处"。

## 引用来源

- 原始代码:`F:\...\KuroEffectSystemNew\EffectSpec\EffectSpecFactory.h`(14 行)
- `.cpp` 实现:Batch 6 读
- 被调用处:Handle 的 `AfterConstruct(FEffectSpecData)` 或 System 的 Spawn 路径

## 开放问题

- **实现位置**:`FEffectSpecFactory.cpp` 在哪?(大概率 `Private/EffectSystemNew/EffectSpec/EffectSpecFactory.cpp`)
- **异常处理**:不认识的 SpecType 怎么办?(返回 nullptr 还是 MakeShared<FEffectSpecBase>?)
- **`GetEffectSpecType` 的 PointCloudNiagara 分支** 是不是在这里判定?还是另外设有路径?
- **Spec 构造后的复用**:Factory 每次都 MakeShared 一个新对象,还是能从对象池拿?(`FEffectSystem` 有 LRU 池,但池化的是 Handle 不是 Spec——Spec 是 Handle 的一部分,随 Handle 复用)
