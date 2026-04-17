# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

## [2026-04-17] bootstrap | 仓库初始化
- 按 Karpathy LLM Wiki 方法论搭建三层架构
- 新建目录:`raw/{articles,papers,books,notes,assets}`、`wiki/{entities,concepts,sources,syntheses}`
- 新建:[[CLAUDE]](schema)、[[index]](目录)、[[log]](本文件)、[[wiki/overview]](占位)
- 源文档:`Karpathy Wiki 方法论.md`(暂放根目录)
- 下一步:等待第一个 raw source。

## [2026-04-17] ingest | Karpathy — LLM Wiki
- source: [[raw/notes/Karpathy Wiki 方法论]](从根目录迁入 raw/notes/)
- 新建:
  - [[wiki/sources/karpathy-llm-wiki]]
  - [[wiki/concepts/llm-wiki-方法论]]
  - [[wiki/concepts/rag]]
  - [[wiki/concepts/memex]]
  - [[wiki/entities/karpathy]]
- 更新:[[index]](登记 5 个新页)、[[wiki/overview]](首次有实质内容,确立"当前主题=方法论自举")、[[CLAUDE]](修正 wikilink 指向 raw/notes)
- 要点:自举式 ingest——用这套方法论本身 ingest 这套方法论。wiki 现在可以正式运转。
- 下一步:扔进来任意一个真实 source,验证流程。
