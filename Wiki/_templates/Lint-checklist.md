<!--
本文件是 [[CLAUDE]] §3.3 Lint 操作的配套 checklist。
用途:每次运行 lint 时 Read 此文件,按顺序执行,最后按标准格式出报告。

和 _templates/ 下页面模板(Entity-Concept.md / Source.md / Reader.md ...)的区别:
- 页面模板:创建**新页**时,复制骨架填内容
- 本文件:执行**lint 操作**时,按 checklist 扫描+报告

二者都属"LLM 在做特定操作时按需 Read 的 schema 参考",都放 _templates/。若今后操作级 playbook 积累到 ≥3 份,再考虑拆分为独立的 _playbooks/。
-->

# Lint Checklist — wiki 体检作业规程

> 触发:用户说"帮我 lint 一下"/"跑一次 lint"/"给 wiki 做个体检"等。
> 产出:一份结构化 lint 报告 + log.md 追加一条 `## [YYYY-MM-DD] lint | ...` 条目。

---

## 0. 前置:摸家底(并行执行)

```
Glob  Wiki/**/*.md
Glob  Raw/**/*.md
Glob  *.md                        # 根目录 index/README/CLAUDE/log
Grep  -n "\[\[([^\]]+)\]\]"  Wiki # 全量 wikilink(可能输出很大,用 head_limit=0 + persist)
Grep  -n "^(type|created|updated|tags|sources|aliases|repo|source_root|source_path|source_ref|source_commit|twin|source_type):" Wiki
```

**关键数字**:wiki 总页数(不含 `_templates/`)、Raw 文件数、wikilink 总数——报告里开头给一行。

> ⚠️ 输出通常很大(数十 KB),不要一次塞 LLM 上下文。用 `head_limit=0` 加 persist,**必要时派 Explore / general-purpose agent** 带着明确的文件清单 + 输出格式去精细分析。本 vault 规模小(50 页量级)主 agent 手搓可接受;大于 100 页强制 delegate。

---

## 1. 孤儿页检测

**定义**:Wiki/ 下某页(排除 `_templates/`、`Wiki/Overview.md`)**没有**任何其他 wiki 文件通过 `[[...]]` 链入。

**分级**:
- 🔴 **零入链**(连 index/log/Overview 也不链)—— 通常是 ingest 时忘登记
- 🟡 **仅索引入链**(只有 index/log/Overview 链)—— 内容页面之间孤立,LLM 检索时会漏
- 🟢 **仅 1 条非索引入链**—— 不一定是问题,但若是主题读本 / 枢纽页要警觉

**输出格式**:`path | 非索引入链数 | 谁链接了它(有则列出)`

---

## 2. Broken wikilinks

**检查**:每条 `[[X]]` 或 `[[X|alias]]` 的目标能否解析。目标解析规则:
- 精确 basename 匹配(含无 .md 后缀形式)
- 完整路径匹配(如 `[[Wiki/Concepts/Topic/abc]]`)
- 命中某页 frontmatter 的 `aliases` 字段

**排除**(不是真正 broken):
- `_templates/` 内的占位符(`[[xxx]]`、`[[…]]`、`[[CLAUDE]]`、`[[Wiki/Entities/Stock/UFoo]]` 等模板示例)
- 表格里被转义的 `\|` 别名语法(`[[target\|display]]` 是合法的)
- `[[Raw/...]]` 指向 Raw 目录下实际文件的——这类应逐一验证真的存在

**输出格式**:`source_file:line | [[broken-target]] | 建议(修/删/建页)`

---

## 3. Filesystem vs log 对账 ⭐

**常被忽视**:log.md 声称"新建 [[X]]"但 filesystem 里根本没有 X。成因:ingest 中途崩掉 / 用户手动删除但 log 没改 / 错字。

**方法**:
- 扫 log.md 里所有 `新建:[[...]]` / `归档为:[[...]]` / `新增:[[...]]` 陈述
- 对每个声称已建的页,Glob 或 Read 验证真实存在
- 反向:扫 filesystem 找 log 里从未记录 ingest 的页面(疑似绕过 bookkeeping)

**输出格式**:`log:行号 | 声称的页面 | filesystem 状态 | 建议(补建/改 log/删 log 行)`

---

## 4. 缺失概念页(高频术语裸奔)

**定义**:某术语/名称在 ≥3 个不同 wiki 页的**正文**(非 wikilink)中被提到,但没有独立 Concept / Entity 页。

**方法**:
- 对 Overview.md 列的"待建页"列表,逐个 Grep 出现频次
- 对本仓典型"可能漏掉"的候选(DDC / VectorVM / Embedding / FNiagaraSystemInstance 这类),主动 Grep
- 被 aliases 字段覆盖的不算漏(如 `VectorVM` 作为 `Niagara-cpu-vs-gpu模拟.md` 的 alias)

**输出格式**:`term | 出现页数 | 典型出现位置 | 建议页路径`

**判断是否要建**:≥3 页是门槛,但也看术语是否有"独立解释价值"。若只是被反复提到但都能靠上下文理解、不需独立解释,可不建(本 wiki 重原子化,但不强求每个词都有页)。

---

## 5. Frontmatter 一致性

**必查**(所有非 `_templates/` 页):

| 检查项 | 合规标准 |
|---|---|
| 必需字段 | `type` / `created` / `updated` / `tags` 齐备 |
| 时间逻辑 | `updated >= created`(小于即错) |
| `type` 合法值 | 必须是 `entity` / `concept` / `source` / `synthesis` / `overview` 之一 |
| Reader 专属 | `type: synthesis` + `tags` 含 `reader` |
| 代码页专属 | `Entities/Stock`、`Entities/Project`、`Sources/Stock`、`Sources/Project` 下必须有 `repo` / `source_root` / `source_path` / `source_ref` / `source_commit` |
| Source `source_type` | `article` / `paper` / `book` / `note` 之一 |
| `aliases` | **不是必需**,但 Entity/Concept 最好有(利于 wikilink 解析) |

**输出格式**:`path | 具体问题`

---

## 6. 缺失交叉引用

**"应链而未链"的典型模式**,逐一检查:

1. **读本 ↔ 原子页双向链闭合**:该议题读本是否下链全部被综合的原子页?这些原子页的"深入阅读"是否回链读本?
2. **同主题概念页内部交叉**:同一 topic 下的 Concept 页是否互链(至少"逻辑相邻"的应链);看 `相关` 节
3. **新建页后的反向 wiring**:新建页若在其他页里被 plain text 提过,要把 plain text 改为 wikilink
4. **index.md 覆盖**:每个 wiki 页是否都被 index 列出
5. **Overview.md 的待建列表**是否和现状同步(已建的要从"待建"移除)

**输出格式**:`gap | 位置 | 建议`

---

## 7. 潜在矛盾 / 陈旧表述

**检查**:同一概念在多页的措辞强弱、事实细节是否一致。重点看:
- 同主题两个 Concept 页对核心定义的表述
- Source 页与其派生 Concept/Entity 的事实要点一致性
- log.md 记录的"新增"描述和当前 wiki 现状

**判断分寸**:不深挖,只浮出"可疑对",让人类裁决。措辞强弱不同 ≠ 真矛盾,但值得标注。

**输出格式**:`页 A | 页 B | 分歧点`

---

## 8. 读本签名 / 模板痕迹

**检查**:
- 所有 Reader 页末尾是否有 `*本读本由 [[Claudian]] 基于 ... 综合生成,YYYY-MM-DD。*` 格式签名
- 所有页是否误留模板占位符(如 `YYYY-MM-DD`、`<议题>`、`[[xxx]]`、`[[...]]`)
- Reader 开头的"本页是 XX 的主题读本"导语是否存在

**输出格式**:`path | 问题`

---

## 9. 报告输出格式

报告统一 6-9 个 `###` 小节,每节按上面指定的格式逐条列举。干净项明确写"无问题"。

**模板**:

```markdown
# Lint 报告 — YYYY-MM-DD

> 规模:N 个 wiki 页 + M 个 raw 源 + K 条 wikilink

### 1. 孤儿页
...

### 2. Broken wikilinks
...

### 3. Filesystem vs log 对账
...

### 4. 缺失概念页
...

### 5. Frontmatter 一致性
...

### 6. 缺失交叉引用
...

### 7. 潜在矛盾
...

### 8. 读本 / 模板痕迹
...

## 建议处置顺序
🔴 高:...
🟡 中:...
🟢 低:...
```

---

## 10. 收尾 checklist

- [ ] 报告输出给用户(结构化,不省略"无问题")
- [ ] **人工验证**至少 2 个可疑点(尤其 §3 filesystem 对账、§4 缺失概念页)——agent 会有误报
- [ ] 追加 log.md 条目 `## [YYYY-MM-DD] lint | <摘要>`,列:触发 / 关键发现(含数字) / 建议下一步 / 不处理的(含理由)
- [ ] **不主动修复**——除非用户批准。lint 是诊断不是治疗;用户看过报告后决定哪些要动
- [ ] 若修复,修完再按分档(🔴🟡🟢)分别追加 log 条目,保历史可追

---

## 与其他 op 的关系

- **ingest** 不 lint:ingest 自带 bookkeeping(§3.1 第 5-8 步),不需要全量扫描
- **每周 / 每 N 次 ingest 后**建议跑一次完整 lint,成本够低收益够大(小 vault 20-30 分钟)
- **大 refactor 前**建议先 lint:拿到基线,refactor 完再 lint,diff 出回归问题
