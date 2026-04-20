<!--
本文件是 [[CLAUDE]] §4 的配套模板。
用途:新建 Entity 或 Concept 页时,复制以下结构并填内容。
- Entity 路径:`Wiki/Entities/<topic>/<名称>.md`
- Concept 路径:`Wiki/Concepts/<topic>/<名称>.md`

紧凑约定:
- 单一 Source 派生的 Code Entity 页应控制在 30-50 行。详细字段清单放 Source 页,叙事/心智模型放读本,Entity 只作稳定跨 source 入口。
- 被多 Source 引用后可扩展为汇总枢纽,但仍保持精炼,细节下沉。
-->

---
type: entity | concept
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [tag1, tag2]
sources: 1        # 被多少个 raw source 支持;新 source 提及时 +1
aliases: [别名1, 别名2]

# ——以下字段仅代码相关页面(Entities/Stock|Project)使用,普通 entity/concept 删掉——
# repo: stock | project
# source_root: UE-4.26-Stock
# source_path: Engine/Source/...
# source_ref: "4.26"
# source_commit: b6ab0dee9
# twin: [[Entities/Project/XXX]]
---

# 名称

> 一句话定义。

## 一句话角色 / 概览
段落总述(1-2 段,≤5 句)。

## 核心事实 / 字段速查
- 表格或列表,尽量紧凑
- 重要陷阱用 ⚠️ 标

## 相关
- [[实体/概念 A]] — 关系
- [[实体/概念 B]] — 关系

## 深入阅读
- 源摘要(含全字段 + 代码引用):[[Wiki/Sources/Topic/xxx]]
- 主题读本(若有):[[Readers/Topic/<议题>-读本]] § N

## 开放问题 / 矛盾
- …
