---
type: concept
created: 2026-04-18
updated: 2026-04-18
tags: [UE4, uobject, reflection, 基础概念]
sources: 0
aliases: [UObject, UCLASS, UObject 系统, UE4对象系统]
---

# UE4 UObject 系统

> UE4 中所有"托管对象"的根基——提供垃圾回收、反射、序列化的统一基础设施。

---

## 概览

如果你用过 Java 或 Python，你知道所有类都有一个共同的根类（`Object`），语言运行时能自动管理对象的内存和元数据。C++ 本身没有这套东西，于是 Epic 在引擎里**手工实现了一套**：这就是 `UObject` 系统。

UE4 中，凡是继承自 `UObject` 的类，引擎就能：

| 能力 | 含义 |
|---|---|
| **垃圾回收（GC）** | 没有任何对象引用它时，自动释放内存，不需要手动 `delete` |
| **反射（Reflection）** | 运行时知道"这个类有哪些字段、哪些函数"，蓝图和编辑器靠这个工作 |
| **序列化（Serialization）** | 能保存到 `.uasset` 文件，也能从文件加载回来 |
| **属性面板显示** | 编辑器里的 Detail Panel 能展示并编辑字段 |
| **网络复制（Replication）** | 标记了 `Replicated` 的字段能自动同步到客户端 |

---

## 关键宏（C++ 语法糖）

你读 UE 代码时会反复看到这些宏，不理解它们等于看天书。

### `UCLASS()`

写在 C++ 类声明之前，告诉 UE 引擎"把这个类纳入 UObject 系统"。

```cpp
UCLASS()
class NIAGARA_API UNiagaraSystem : public UObject
{
    GENERATED_BODY()
    // ...
};
```

- `UCLASS()` 括号里可以加选项，如 `BlueprintType`（允许蓝图使用）、`Abstract`（不能直接实例化）等
- `GENERATED_BODY()` 是必须的占位宏，展开后插入引擎需要的胶水代码

**类比**：`UCLASS` 就像在 Python 里写 `@dataclass` 或注册到某个框架——它让框架"认识"这个类。

### `USTRUCT()`

同 `UCLASS`，但用于**结构体**（没有 GC，轻量级，可以栈分配）。

```cpp
USTRUCT(BlueprintType)
struct FNiagaraVariable
{
    GENERATED_BODY()
    // ...
};
```

UE 中用 `F` 前缀命名结构体（如 `FVector`、`FNiagaraVariable`），用 `U` 前缀命名 UObject 派生类（如 `UNiagaraSystem`）。

### `UPROPERTY()`

写在字段前，让引擎知道这个字段的存在（GC、序列化、编辑器显示）。

```cpp
UPROPERTY(EditAnywhere, BlueprintReadWrite)
TArray<FNiagaraEmitterHandle> Emitters;
```

- `EditAnywhere`：在编辑器的任何地方都能编辑
- `BlueprintReadWrite`：蓝图可读可写
- `Transient`：不序列化（运行时临时数据）
- `TArray<T>`：UE 的动态数组，类似 `std::vector`

### `UFUNCTION()`

写在函数前，让函数可以被蓝图调用或通过反射调用。

```cpp
UFUNCTION(BlueprintCallable, Category="Niagara")
void Activate(bool bReset = false);
```

---

## UObject 的命名前缀约定

| 前缀 | 含义 | 例子 |
|---|---|---|
| `U` | 继承自 UObject 的类 | `UNiagaraSystem`、`UTexture2D` |
| `A` | 继承自 AActor 的类 | `ANiagaraActor`、`ACharacter` |
| `F` | 普通 C++ 结构体或类（不受 GC 管理） | `FNiagaraVariable`、`FVector` |
| `I` | 接口类 | `INiagaraMergeManager` |
| `E` | 枚举 | `ENiagaraScriptUsage` |
| `T` | 模板类 | `TArray<T>`、`TSharedPtr<T>` |

---

## UObject 与普通 C++ 对象的关键区别

**普通 C++ 对象**：
```
new SomeClass()  →  你负责管理内存  →  delete 它
```

**UObject**：
```
NewObject<UNiagaraSystem>()  →  引擎 GC 管理  →  无引用时自动释放
```

这意味着：
- **不能用 `new`/`delete`**，必须用 `NewObject<T>()` 创建
- **不能用裸指针持有 UObject**（GC 不知道你引用了它，会提前回收）——必须用 `UPROPERTY()` 标记的指针或 `TObjectPtr<T>`
- **`F` 前缀的结构体**（如 `FNiagaraSystemInstance`）是普通 C++ 对象，可以用 `TSharedPtr<T>`，GC 不管它们

---

## 智能指针速查

Niagara 代码里常见：

| 类型 | 含义 | 用于 |
|---|---|---|
| `TObjectPtr<T>` | UObject 的安全引用 | UObject 字段，替代裸指针 |
| `TWeakObjectPtr<T>` | 弱引用 UObject（不阻止 GC） | 跨系统引用，不想阻止对象销毁时 |
| `TSharedPtr<T>` | 引用计数智能指针 | F 前缀的普通 C++ 对象 |
| `TSharedRef<T>` | 不可为 null 的 TSharedPtr | 同上，但保证非空 |
| `TUniquePtr<T>` | 独占所有权 | 生命周期清晰的 F 对象 |

---

## 和 Niagara 的关系

Niagara 中**资产类（Asset）全是 UObject**，因为它们需要被编辑器显示、保存到磁盘：

- `UNiagaraSystem` → 继承 `UObject`，序列化为 `.uasset`
- `UNiagaraEmitter` → 同上
- `UNiagaraScript` → 同上
- `UNiagaraRendererProperties` 的各子类 → 同上

而**运行时实例（Instance）通常是普通 F 结构体**，因为它们是临时的运行时对象：

- `FNiagaraSystemInstance` → 普通 C++ 类（`TSharedPtr` 管理）
- `FNiagaraEmitterInstance` → 同上
- `FNiagaraDataSet` → 同上

这就是"资产 vs 实例"分离的技术基础，详见 [[Wiki/Concepts/UE4/UE4-资产与实例]]。

---

## 相关
- [[Wiki/Concepts/UE4/UE4-资产与实例]] — 基于 UObject 的资产/实例二元模型
- [[Wiki/Syntheses/Niagara/Niagara-learning-path]] — 学习路径总览

## 开放问题 / 待深入
- `CDO`（Class Default Object）的概念：每个 UClass 有一个"默认对象"，`GetDefaults()` 返回它；Niagara 里 Emitter 的 `MergedEmitter` 机制与 CDO 相关，Phase 1 时再深入
