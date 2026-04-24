<!--
本文件是 [[CLAUDE]] §4 / §9.5 的配套模板(代码 source)。
用途:ingest 代码源文件时,在 `Wiki/Sources/<repo>/<文件名>.md`(如 `Sources/Stock/`)使用。
代码源**不在 Raw/ 下**——源真相是 code root 里的真实文件。本页记录"这次 ingest 读了什么、看出了什么"。
-->

---
type: source
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [topic, source-code, ...]
sources: 1
repo: stock | project
source_root: UE-4.26-Stock
source_path: Engine/Source/Runtime/.../Foo.h
source_ref: "4.26"
source_commit: b6ab0dee9        # git short SHA / p4 CL / "p4: unknown"
---

# 文件名或模块名

- **Repo**:stock(或 project)
- **路径**:`Engine/Source/Runtime/.../Foo.h` [(GitHub)](可选链接)
- **快照**:commit `b6ab0dee9` / CL `12345`
- **同仓协作文件**:`Foo.cpp`、`FooInstance.h` …
- **Ingest 日期**:YYYY-MM-DD
- **学习阶段**:Phase X.Y(若属学习路径)

## 职责 / 这个文件干什么的
2-5 句。

## 关键类型 / 函数 / 宏
- `UFoo`:…
- `FFooProxy`:…

## 依赖与被依赖
- 上游(此文件 include / 使用):`Bar.h`、…
- 下游(谁用它):`Baz.cpp`、…

## 关键代码片段

> `Foo.h:L120-L135` @ `b6ab0dee9`
> ```cpp
> // 只贴真正值得记的片段,不复制整文件
> ```

## 涉及实体 / 概念
- [[Wiki/Entities/Stock/Niagara/UFoo]]、[[Wiki/Concepts/Topic/xxx]]

## 与另一仓差异(如已 ingest 孪生)
- 对比见 [[Wiki/Syntheses/Topic/foo-stock-vs-project]]

## 开放问题 / 待深入
- …
