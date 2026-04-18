# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

## [2026-04-18] refactor | raw/wiki 顶层+笔记首字母大写化、UE 全大写
- 变更:顶层目录 `raw/` → `Raw/`、`wiki/` → `Wiki/`；所有 raw 子目录 `Articles/Papers/Books/Notes/Assets/`
- 变更:笔记文件名英文首字母大写：`overview → Overview`、`rag → Rag`、`memex → Memex`、`karpathy → Karpathy`、`llm-wiki-方法论 → Llm-wiki-方法论`、`niagara-* → Niagara-*`、`ue4-* → UE4-*` 等
- 变更:内容里所有 "UE"（含 `ue4`、`ue 官方的`、`ue4对象系统`）全大写为 `UE4`/`UE`
- 更新:所有 .md 文件的 wikilink 路径同步（.obsidian/app.json 的 `attachmentFolderPath` 改为 `Raw/Assets`）
- 更新:[[CLAUDE]](§2 目录结构 Raw/Wiki 大写、§3 路径、§4 模板、§5 log 示例、§6 链接约定、§9.5 ingest 路径)
- 踩坑:PowerShell 的 `-ne` 对字符串**默认大小写不敏感**，首次 batch 替换误判 "无变化"。改用 `.Equals()` 后正常

## [2026-04-18] refactor | wiki 目录大写化 + 主题分类
- 变更:所有 wiki 子目录改为大写开头（`concepts/` → `Concepts/` 等）
- 变更:各目录内按主题建子文件夹：`Concepts/{Methodology,UE4,Niagara}/`、`Entities/{Methodology,Stock,Project}/`、`Sources/{Methodology,Stock,Project}/`、`Syntheses/{Niagara}/`
- 迁移:10 个 wiki 文件移至新路径，全部内部 wikilink 同步更新
- 更新:[[CLAUDE]](§2 目录结构、§3 路径引用、§4 模板示例、§5 log 格式、§6 链接约定、§9.5 代码 ingest 路径)、[[index]]、[[Wiki/Overview]]（批量 wikilink 替换）

## [2026-04-17] bootstrap | 仓库初始化
- 按 Karpathy LLM Wiki 方法论搭建三层架构
- 新建目录:`Raw/{articles,papers,books,notes,assets}`、`Wiki/{entities,concepts,sources,syntheses}`
- 新建:[[CLAUDE]](schema)、[[index]](目录)、[[log]](本文件)、[[Wiki/Overview]](占位)
- 源文档:`Karpathy Wiki 方法论.md`(暂放根目录)
- 下一步:等待第一个 raw source。

## [2026-04-17] ingest | Karpathy — LLM Wiki
- source: [[Raw/Notes/Karpathy Wiki 方法论]](从根目录迁入 Raw/Notes/)
- 新建:
  - [[Wiki/Sources/Methodology/Karpathy-llm-wiki]]
  - [[Wiki/Concepts/Methodology/Llm-wiki-方法论]]
  - [[Wiki/Concepts/Methodology/Rag]]
  - [[Wiki/Concepts/Methodology/Memex]]
  - [[Wiki/Entities/Methodology/Karpathy]]
- 更新:[[index]](登记 5 个新页)、[[Wiki/Overview]](首次有实质内容,确立"当前主题=方法论自举")、[[CLAUDE]](修正 wikilink 指向 Raw/notes)
- 要点:自举式 ingest——用这套方法论本身 ingest 这套方法论。wiki 现在可以正式运转。
- 下一步:扔进来任意一个真实 source,验证流程。

## [2026-04-18] ingest | Niagara Phase 0 — 基础心智模型（4 概念页）
- 新建:
  - [[Wiki/Concepts/UE4/UE4-uobject-系统]]（UObject/UCLASS/UPROPERTY 宏、GC、反射、命名前缀、智能指针速查）
  - [[Wiki/Concepts/UE4/UE4-资产与实例]]（Asset vs Instance 二元模型、Component 桥梁、为什么分离）
  - [[Wiki/Concepts/Niagara/Niagara-vs-cascade]]（设计哲学对比、SIMD vs 逐粒子、脚本阶段说明）
  - [[Wiki/Concepts/Niagara/Niagara-cpu-vs-gpu模拟]]（VectorVM、Compute Shader、选择依据、源码分叉点）
- 更新:[[index]](新增 UE4 基础 + Niagara 基础两个 Concepts 分类)、[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 0 全部打勾)
- 要点:Phase 0 建立心智模型，无需读代码；重点是资产/实例二元模型 + CPU/GPU 分叉
- 下一步:Phase 1 — 读 NiagaraSystem.h / NiagaraEmitter.h / NiagaraScript.h

## [2026-04-18] synthesis | Niagara 源码学习路径制定
- 扫描 `stock` 仓 `Engine/Plugins/FX/Niagara/` 全部 7 个模块，749 个文件
- 新建:[[Wiki/Syntheses/Niagara/Niagara-learning-path]]
- 更新:[[index]](新增 Niagara Syntheses 分类)
- 要点:10 阶段学习路径，Phase 0(概念)→ Phase 9(世界管理)+ Phase 10(高级选修)，共约 69 个文件；每阶段含文件清单、学习要点、难度评级、进度 checkbox
- 下一步:从 Phase 0 概念页开始，或直接从 Phase 1 `NiagaraSystem.h` 开始逐文件 ingest

