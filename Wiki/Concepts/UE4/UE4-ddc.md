---
type: concept
created: 2026-04-20
updated: 2026-04-20
tags: [ue4, engine, cache, infrastructure]
sources: 1
aliases: [DDC, Derived Data Cache, 派生数据缓存]
---

# DDC (Derived Data Cache)

> UE4 的**机器级编译产物缓存**。把"从 .uasset 原始数据到可运行二进制"的转换结果存下来,下次用同一个 key 直接命中,绕过重算。团队内可共享同一份 DDC,一个人编译,全组复用。

## 一句话角色

UE 里很多资产在磁盘上存的是"源数据"(如 Niagara 的节点图、Shader 的 HLSL 源、贴图的压缩前像素、Mesh 的原始顶点流),**运行时需要的是"派生数据"**(字节码、编译后的 shader bytecode、压缩后的 DXT/BC 贴图、LOD mesh 等)。DDC 就是存这些"派生产物"的缓存层,按 key 查找命中则跳过昂贵的编译/压缩/处理步骤。

**为什么存在**:很多派生数据的生成非常慢(几秒~几分钟),但结果是**输入的纯函数**——只要输入不变,结果就不变。缓存它是显然的优化。Epic 把这个抽象做成了贯穿引擎的公共设施,`Engine/Source/Runtime/DerivedDataCache/` 里是实现。

## 核心事实 / 字段速查

| 维度 | 说明 |
|---|---|
| **Key** | 一个字符串,由"插件名 + 版本 + 输入内容哈希 + 相关依赖哈希"拼成。**key 相同就代表派生数据相同**,所以 key 设计是 DDC 正确性的关键 |
| **Value** | 任意 `TArray<uint8>` 二进制 blob,各系统自行序列化/反序列化 |
| **存储后端** | 多级:内存 cache → 本地文件 cache(`%LOCALAPPDATA%\UnrealEngine\Common\DerivedDataCache\`)→ Shared DDC(SMB / 云 / Zen)→ 未命中则走真实编译 |
| **团队共享** | Shared DDC 指向团队网络盘或云存储,一个人编译产物入库,其他人直接拉 |
| **失效** | 无主动失效机制——失效通过"key 变了"自然发生。所以**版本号是 key 里必填字段** |
| **可手动清** | `Engine\Binaries\Win64\UnrealVersionSelector.exe` 或删本地 DDC 目录 |

## ⚠️ 常见陷阱

- **key 设计漏了某个输入** → 输入变了,key 没变,命中错误缓存 → 表现为"改了代码但不生效",删 DDC 才好。定位极困难
- **编译器/插件升级后没 bump 版本号** → 旧 DDC 里的产物反序列化炸
- **Shared DDC 的网络慢** → 命中率虽高但每次拉包慢过本地重算,得评估;UE 4.27+ 有 Zen 加速
- **DDC 只对"纯函数式派生"有效** → 依赖 CPU/GPU 硬件环境或外部工具版本的输出,最好把那些也放进 key

## Niagara 怎么用 DDC

Niagara 是 DDC 的重度用户——节点图编译非常耗时:

- [[Wiki/Entities/Stock/UNiagaraScript|UNiagaraScript]] 的 `CachedScriptVMId`(`FNiagaraVMExecutableDataId`)**直接作为 DDC key**
- key 的构成:基础版本号 + `BaseScriptCompileHash`(图本体哈希) + `ReferencedCompileHashes`(所有被引用子图哈希) + editor-only 元信息
- 两个人在不同机器打开**同一个**未改动的 NiagaraSystem,`operator==` 判等 → DDC 命中 → 第二个人跳过编译。**Niagara 团队协作的流畅度很依赖这个缓存**
- 相关函数:`BuildNiagaraDDCKeyString / GetNiagaraDDCKeyString`(生成 key)、`BinaryToExecData / ExecToBinaryData`(二进制 ↔ `FNiagaraVMExecutableData` 结构体)
- 详见 [[Wiki/Sources/Stock/NiagaraScript]] 的 DDC 一节 + [[Readers/Niagara/Phase 1 - 从 System 到图源抽象基类]] § 5

## 其他典型用户

- **Shader**:HLSL → bytecode 编译,DDC key 含 shader 源哈希 + 目标平台 + 编译器版本
- **贴图**:源 PNG/EXR → 平台相关的压缩格式(DXT/BC/ASTC),key 含源哈希 + 压缩设置 + 平台
- **StaticMesh / SkeletalMesh**:源几何 → LOD 链、distance field、渐变 LOD 等
- **Cooked Content 的中间态**:部分 cook-on-the-fly 场景

## 相关

- [[Wiki/Concepts/UE4/UE4-uobject-系统|UObject 系统]] — UAsset 序列化层,DDC 在其上
- [[Wiki/Concepts/UE4/UE4-资产与实例|资产 vs 实例]] — DDC 只缓存"资产级"的派生数据,不缓存运行时实例状态
- [[Wiki/Entities/Stock/UNiagaraScript]] — Niagara 里 DDC 的主要消费者
- [[Wiki/Concepts/Niagara/Niagara-vs-cascade|Niagara vs Cascade]] — "显式编译字节码 + DDC 缓存"是 Niagara 相对 Cascade 的质变之一

## 深入阅读

- 源引用:[[Wiki/Sources/Stock/NiagaraScript]] — Niagara DDC key 构造的代码级细节
- 主题读本:[[Readers/Niagara/Phase 1 - 从 System 到图源抽象基类]] § 5(UNiagaraScript)

## 开放问题 / 待深入

- UE 5.x 引入的 Zen Store 如何改变 DDC 行为?(本 wiki 暂未涉及 UE 5)
- key 里 editor-only 字段如果 cook 后不存在,是否有影响? 按 `WITH_EDITORONLY_DATA` 条件编译控制,Phase 5/6 可追
- DDC 的网络层失败重试和超时策略 — 本页未展开,属于 `DerivedDataCache/Private` 实现细节

## 引用来源

- 本页从 [[Wiki/Sources/Stock/NiagaraScript]] 对 DDC 的使用方式反推而来。DDC 本身的源码(`Engine/Source/Runtime/DerivedDataCache/`)尚未作为独立源 ingest。如后续需要,可补 ingest 为 [[Wiki/Sources/Stock/]] 下的代码源。
