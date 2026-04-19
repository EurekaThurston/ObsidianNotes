# Log — Wiki 时间线

> 追加式日志。格式:`## [YYYY-MM-DD] <op> | <title>`
> 快速查看最近记录:`grep "^## \[" log.md | tail -10`

---

## [2026-04-20] refactor | vault 使用 / 视觉 5 项改进(post-lint 打磨)

- 触发:用户问"这个 vault 使用和视觉上还能怎么改",我给了三档建议,用户挑了真值得改的 4 项 + 开放问题汇总机制,一口气都做掉
- 新建:
  - `.gitattributes`(覆盖原仅 1 行版本)— 强制 `eol=lf` + 给常见文本/二进制类型显式标注,消除 Windows 下 git 刷屏的 CRLF 警告
  - [[.obsidian/snippets/readers.css]] — reading mode 专用 CSS:820px 行宽 + 1.75 行距 + H1/H2 底线分隔 + H3 accent 色 + callout 圆角 + 表格斑马条。启用方式:Obsidian 设置 → Appearance → CSS snippets → 打开"readers"
- 更新:
  - [[CLAUDE]] §3.1 — ingest 流程补第 9 步"自验(强制)":对本轮 log 里所有"新建 [[...]]"路径跑一次 Glob,差异立即停下告诉用户。解决这次 lint 发现的 cheatsheet 幽灵文件类型问题(即便本次是用户主动删除,自验一样会触发复核)。属 always-apply,进 CLAUDE.md 内嵌而非外移
  - [[Wiki/_templates/Lint-checklist]] 新增 §11"开放问题汇总"— 周期性扩展动作(不进标准 0-10 强制项),用 grep 抓全仓 `## 开放问题` 节聚合为临时快照页,让用户一次过一遍决定"已解 / 保留 / 升级为下次 ingest 目标"。建议频率:每月 1 次或每 5 次 lint 一次。元规则记录:没做成独立 op,因为与 lint 逻辑连续;满足"能复用就不新开操作"
  - [[Wiki/_templates/Reader]] 模板 — 引入 Obsidian Callouts 约定作为读本标准写法:9 类 callout(warning/question/abstract/quote/tip/note/example/info/折叠语法)替代裸 emoji,从此生成读本时直接按此约定产出
  - [[Wiki/Readers/Niagara/Phase1-asset-layer-读本]] — 做 4 处示范性 callout 转换:第 0 节 Phase 要回答的问题 → `[!question]`、§2.5 命名陷阱(原 ⚠️)→ `[!warning]`、§2.6 Handle 本质归纳 → `[!abstract]`、§3 UNiagaraEmitter 源码自述 → `[!quote]`。其余 4 个读本暂不批量改,下次 lint 时评估是否需要统一回溯
- 元规则应用记录(本轮共 3 次分层判断):
  - `.gitattributes` / CSS snippet → 文件系统级基建,不进 CLAUDE.md,原地落
  - ingest 自验 → 每次 ingest 都要做 = always-apply → 进 CLAUDE.md §3.1 作为步骤 9
  - 开放问题汇总 → 周期性使用 = 条件性 → 进 Lint-checklist.md 作为扩展节而非强制节;且选择"复用 lint" 而非"开新 op"
- 不做的:
  - 换 Obsidian 主题 / 装美化插件(边际收益低)
  - Dataview 自动生成 index(人工+LLM 维护的带描述索引质量优于自动列表)
  - 装 Templater(CLAUDE.md + _templates/ 已替代此角色)
  - Readers/ 批量改 callout(先看 Phase1 4 处示范的效果;若用户满意再批量)
- 下一步:
  - 用户 commit push 后,在 Obsidian 里打开 Phase1 读本,启用 readers.css snippet,对比视觉改善
  - 若示范满意,可批量把其余 4 个读本的 ⚠️ 和长引用块按新 callout 约定改写

---

## [2026-04-20] refactor | 新增 Lint checklist — 元规则第二次应用

- 触发:首次 lint 实践后,总结经验固化为可复用 checklist;用户提示"补吧,不过你要考虑好该放在哪"——明确点名让我做元规则分层判断
- 元规则分层判断(应用 [[CLAUDE]] §7 原则 6):
  - 每轮对话都要守? ❌ 只在 lint 时才用 → 外移,不进 CLAUDE.md
  - 候选位置:
    - (A) `Wiki/_templates/Lint-checklist.md` — 复用现有文件夹,但 `_templates/` 语义需稍扩
    - (B) 新建 `Wiki/_playbooks/Lint.md` — 语义最准,但为 1 个文件开文件夹过早抽象
    - (C) CLAUDE.md §3.3 inline — 违反分层,污染 always-load context
  - **选 A**:`_templates/` 本质是"LLM 在做特定操作时按需 Read 的 schema 参考",页面模板是它的第一子类,操作 checklist 是第二子类;开新文件夹是过早抽象,待累积 ≥3 份操作 playbook 再考虑拆分
- 新建:[[Wiki/_templates/Lint-checklist]] — 10 节完整作业规程,从前置 inventory → 8 类检查(孤儿 / broken / log 对账 / 缺失概念 / frontmatter / 交叉引用 / 矛盾 / 读本签名) → 报告格式 → 收尾 checklist;含 agent delegation 判断、人工验证硬要求、"诊断不是治疗"原则
- 更新:
  - [[CLAUDE]] §3.3 — 从"列 6 条检查项"改为"Read [[Wiki/_templates/Lint-checklist]] 按其执行",核心输出仍留主文件,细节外移
  - [[CLAUDE]] §4 — `_templates/` 表格从"页面结构模板"改为"schema 参考",增加 Lint-checklist 一行,显式承认了扩展语义
  - [[README]] — 三层架构图 / Wiki 树 / Schema 分层设计段落三处同步措辞("页面模板" → "schema 参考:页面模板 + 操作 checklist")
- 副产品/洞察:
  - 元规则本次应用**不只是"去哪"**,还催生了"要不要开新文件夹"的决策——分层判断扩展到结构层面,不只是内容层面
  - 首次 lint 的实操经验暴露了几条 checklist 里最重要的条款:§3 filesystem vs log 对账(本次发现 cheatsheet 幽灵文件就靠它)、§0 规模门槛决定是否 delegate agent、收尾"人工验证可疑点"(agent 会误报,本次就手动验证了 cheatsheet 丢失和 Claudian broken)
  - 未来若出现 Ingest playbook / Synthesis playbook,累积到 ≥3 份时触发 `_templates/` 分拆为 `_templates/` + `_playbooks/`
- 下一步:以当前 schema 再跑一次 lint(试验 checklist 自身),或直接进入 Niagara Phase 2 ingest

---

## [2026-04-20] synthesis | lint 🟡🟢 项清理 — 3 个新概念页 + 小问题修补

- 触发:🔴 项完成后用户"黄的也修一下",一口气把 🟡 和 🟢 全部扫清
- 新建 3 个高频缺失概念页(对应 lint 报告 🟡 中的三个)。每页标准 header + 概览 + 字段速查 + ⚠️ 陷阱 + 相关 + 深入阅读 + 开放问题;长度 70-95 行:
  - [[Wiki/Concepts/UE4/UE4-ddc]] — Derived Data Cache;UE4 机器/团队级编译产物缓存;Niagara 用 `FNiagaraVMExecutableDataId` 直接作 key,也记录 shader/texture/mesh 等其他典型用户
  - [[Wiki/Concepts/Methodology/Vibe-coding]] — Karpathy 2025-02 命名;定位为 Vibe/Spec/Harness 三阶段演进的起点;含对非开发角色的适用视角(美术/策划/管理者)
  - [[Wiki/Concepts/AIApps/Embedding]] — 把语义变成几何距离的数学底座;RAG 的检索层 + 多模态对齐 + 聚类/推荐/异常检测等延伸应用
- 交叉引用 wiring(让新页不成孤儿):
  - [[Wiki/Entities/Methodology/Karpathy]] 的"Vibe Coding 命名"条目改为 wikilink
  - [[Wiki/Concepts/Methodology/Rag]] 正文的"Embedding（向量嵌入）"改为 wikilink + 相关节增一条
  - [[Wiki/Entities/Stock/UNiagaraScript]] 相关节增一条 DDC 回链
  - [[Wiki/Sources/Stock/NiagaraScript]] 涉及实体节增一条 DDC 回链
  - [[Wiki/Readers/Niagara/Phase1-asset-layer-读本]] 第 472 行首次提到 DDC 改为 wikilink
  - [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]] 相关节增一条 Vibe Coding 链
- 🟢 小问题修补:
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]] line 16 `[[Wiki/Sources/Stock/]]` 目录式死链改为普通 code 样式
  - [[Wiki/Concepts/AIArt/Multi-lora-composition]] 相关节补一条 Caption-strategy 链
  - [[Wiki/Concepts/AIApps/Harness-engineering]] 相关节补 Llm / Mcp / Reasoning-model / Vibe-coding 四条,解决交叉密度偏低
- 更新:
  - [[index]] — Concepts 的方法论/UE4/AI 应用生态三个分区各新增一条
  - [[Wiki/Overview]] — "待建页"列表去掉 Embedding / Vibe Coding,加一行"已补建(2026-04-20)"标注;"下一步建议"的 AI 应用一行同步更新
- 回查 lint 报告剩余项的状态:
  - ✓ `FNiagaraSystemInstance`(7 处) — 留到 Phase 3 自然 ingest,不急
  - ✓ [[Wiki/Sources/AIApps/AI-primer-v2]] 缺 `aliases` — 尚未补,可下一轮 lint 一并处理(属 cosmetic)
  - ✓ `Niagara-vs-cascade` vs `Niagara-cpu-vs-gpu模拟` 对 Cascade GPU 能力措辞强弱不一致 — 未改(不算矛盾,只是措辞),留待合适时机统一口径
- 本轮合计影响文件:16 个(3 新建 + 13 更新)

---

## [2026-04-20] refactor | lint 🔴 项清理 — Claudian 页 + 读本回链 × 13 + cheatsheet 去引

- 触发:2026-04-20 lint 后用户决策:(1) Claudian 选方案 a 建实体页;(2) cheatsheet 已人工删除,清理所有残留引用
- 新建: [[Wiki/Entities/Claudian]] — 本仓 LLM 作者身份实体页,固化签名约定和运行模型,同时消解读本模板 Reader.md 的 `[[Claudian]]` 占位问题(原 6 处读本签名 + 1 处 learning-path 的 broken link 自动解析)
- 更新(读本回链,13 页,在"引用来源"节新增"主题读本(推荐通读)"行):
  - AIApps 8 页: [[Wiki/Concepts/AIApps/Llm]]、[[Wiki/Concepts/AIApps/Hallucination]]、[[Wiki/Concepts/AIApps/Context-window]]、[[Wiki/Concepts/AIApps/Ai-agent]]、[[Wiki/Concepts/AIApps/Mcp]]、[[Wiki/Concepts/AIApps/Harness-engineering]]、[[Wiki/Concepts/AIApps/Agent-skills]]、[[Wiki/Concepts/AIApps/Reasoning-model]] → 全部回链 [[Wiki/Readers/AIApps/AI-primer-v2-读本]]
  - AIArt 5 页: [[Wiki/Concepts/AIArt/Lora]]、[[Wiki/Concepts/AIArt/Base-model-selection]]、[[Wiki/Concepts/AIArt/Caption-strategy]]、[[Wiki/Concepts/AIArt/Trigger-word]]、[[Wiki/Concepts/AIArt/Multi-lora-composition]] → 全部回链 [[Wiki/Readers/AIArt/Lora-深度指南-读本]]
  - 执行:用 sed 批量替换"## 引用来源"节下的 source 行,7+4 个页面(Llm/Lora 用 Edit 先行完成)
- 删除引用(cheatsheet 用户已手动删除源文件,清理残留):log.md:230 原条目"[[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat-cheatsheet]] — 一页纸精简版..." 已去除。其余 line 224 的"一页纸简化版"只是"下一步猜测",非实际引用,保留
- 更新:[[index]] — Entities 方法论分区新增 Claudian 条目
- lint 报告三个 🔴 项(读本孤立 / cheatsheet 丢失 / Claudian broken link)全部解决;🟡🟢 未动,留待下一批次
- 副产品:读本模板 [[Wiki/_templates/Reader.md]] 末尾签名 `*本读本由 [[Claudian]] 生成*` 从此不再是 broken link
- 下一步:🟡 建 DDC / Vibe-coding / Embedding 三个高频缺失概念页;🟢 修若干小问题(Niagara-learning-path:16 目录 link、Multi-lora 缺 Caption-strategy 链、Harness 交叉密度等)

---

## [2026-04-20] lint | 首次体检 — 读本回链缺口 + 4 个缺失概念页 + 1 个丢失文件

- 触发:用户首次使用 lint 功能
- 范围:Wiki/ 下 47 个非模板页 + 根目录 index/README/log/Overview,排除 `_templates/` 占位符
- 方法:并行 grep 全量 wikilink + frontmatter,派 Explore agent 交叉分析,手动验证 2 个可疑点

### 关键发现

1. **系统性:读本被孤立** — 13 个 AIApps/AIArt 概念页没有一个回链对应主题读本
   - AIApps 8 页(Llm/Hallucination/Context-window/Reasoning-model/Ai-agent/Mcp/Harness-engineering/Agent-skills)未回链 [[Wiki/Readers/AIApps/AI-primer-v2-读本]]
   - AIArt 5 页(Lora/Base-model-selection/Caption-strategy/Trigger-word/Multi-lora-composition)未回链 [[Wiki/Readers/AIArt/Lora-深度指南-读本]]
   - 结果:[[Wiki/Readers/AIApps/AI-primer-v2-读本]] 全无非索引入链(接近孤儿)
   - **根因**:本仓的读本范式是在概念页之后才定型的(§3.4),历史概念页尚未按新模板回链
   - **建议**:批量补"深入阅读"节回链,可作为一次小型 refactor

2. **丢失文件:`How-to-prompt-ai-chat-cheatsheet`**
   - log.md:173 声称 2026-04-19 已创建此页("一页纸精简版"),但文件系统里不存在
   - 未在后续 commit 中看到删除记录
   - **建议**:确认是否需要补建,或从 log 移除该行(加"未落地"标注)

3. **损坏链接:`[[Claudian]]`** — 6 处读本签名 + 1 处 Niagara-learning-path 指向不存在的页面
   - 根因:读本模板 Reader.md 末尾签名约定 `*本读本由 [[Claudian]] 生成…*`,但 `Wiki/Entities/Claudian.md` 未建
   - **建议**两选一:(a) 建 `Wiki/Entities/Claudian.md` 作为本仓 LLM 作者实体;(b) 改签名为纯文本"由 Claudian 生成",避免占位式 broken link

4. **缺失概念页(高频术语但无独立页)**:
   - `FNiagaraSystemInstance`(7 处) — Phase 3 自然会建,暂不急
   - `DDC / Derived Data Cache`(6 处) — 建议建 [[Wiki/Concepts/UE4/UE4-ddc]],Niagara 编译身份证直接作 DDC key
   - `Vibe Coding`(6 处) — Overview 已列待建;建议建 [[Wiki/Concepts/Methodology/Vibe-coding]]
   - `Embedding`(5 处) — Overview 已列待建;建议建 [[Wiki/Concepts/AIApps/Embedding]]

5. **其他小问题**
   - `Niagara-learning-path.md:16` 指向目录 `[[Wiki/Sources/Stock/]]`(非页面),应删或改具体文件
   - `Multi-lora-composition` 不链 `Caption-strategy`(两者逻辑紧密),建议补
   - `Harness-engineering` 与其他 AIApps 概念页交叉密度偏低,建议补 `Llm / Reasoning-model / Mcp` 三条
   - `Wiki/Concepts/Niagara/Niagara-vs-cascade` 与 `Niagara-cpu-vs-gpu模拟` 对 Cascade GPU 能力描述措辞强弱不一致(轻微,不算矛盾)
   - `Sources/AIApps/AI-primer-v2.md` 缺 `aliases` 字段(其他 source 页均有)

### 不用处理的

- index.md 已覆盖全部 47 页 ✓
- 所有 Stock 代码页 frontmatter 完整(repo/source_root/source_path/source_ref/source_commit 齐备) ✓
- 所有读本 frontmatter 合规(type: synthesis + tags 含 reader) ✓
- Niagara 5 个实体页 ↔ Phase1 读本双向链完整 ✓
- 无"updated < created"时间异常 ✓

### 下一步建议(按优先级)

1. 🔴 **高**:查清 `How-to-prompt-ai-chat-cheatsheet` 丢失原因(补建 or log 标注)
2. 🟡 **中**:决定 `[[Claudian]]` 处理方式 → 批量修 7 处 broken link
3. 🟡 **中**:批量补 AIApps/AIArt 概念页的读本回链(13 页 × "深入阅读"节)
4. 🟢 **低**:建 DDC / Vibe-coding / Embedding 三个高频缺失概念页
5. 🟢 **低**:修 `Niagara-learning-path.md:16` 的目录 link

等待用户指示是否要在本轮就动手修,还是只记录等批次处理。

---

## [2026-04-20] refactor | 新增元规则:新流程/规则的归处分层判断
- 触发:用户指令"以后有涉及到新的流程的时候你都考虑一下是需要写到 CLAUDE.md 里还是单独开一个文档,包括这条也是"
- 元规则本身的归处分析(自洽应用):
  - 性质:每次考虑扩 schema 时的元判断
  - 频率:always-apply
  - 结论:进 CLAUDE.md §7,作为原则 6,和既有"不把 CLAUDE.md 当作不可变"(原则 5)配对
- CLAUDE.md §7 新增原则 6(全文):
  > 新流程/规则落地前先做分层判断:问一句"这是每轮对话都要遵守的,还是只在做 X 时才需要的?"
  > - 每轮都要守 → 进 CLAUDE.md 某一节,尽量简短
  > - 只在做 X 时要 → 外移到独立文件(Wiki/_templates/、议题的 wiki 页、或专门 playbook)
  > - CLAUDE.md 只放 always-apply 的核心,防 context rot
- 更新 [[README]]:
  - 三层架构图的 Schema 层注明"CLAUDE.md + Wiki/_templates/",Wiki/ 树加 `_templates/` 一栏
  - 新增一段 "Schema 分层设计" 解释"always-apply vs conditional"原则,指向 CLAUDE.md §7 原则 6
- 这条 refactor 是本轮 CLAUDE.md 瘦身的自然延续——瘦身是一次性动作,**分层判断原则**把这次的治理经验固化为未来的规则
- 下一步:任何后续 schema 扩展都应先问这一句;如果判断模糊就先在 log 里记录,待 2-3 次同类情形后回看规律再做选择

## [2026-04-20] refactor | CLAUDE.md 分层瘦身 — 模板外移,防 context rot
- 触发:用户关心的不是 token 成本,而是 CLAUDE.md 膨胀导致 context rot(长对话中 schema 内容占用注意力,挤占硬约束的遵守可靠性)
- 核心洞察:CLAUDE.md 里**每轮对话都要遵守的核心规则** vs **条件性使用的详细模板** 应当分层——前者 always-loaded,后者 on-demand Read
- 方案:
  - 新建 `Wiki/_templates/` 目录(4 份模板):
    - `Entity-Concept.md`(~50 行)— Entity/Concept 页结构
    - `Source.md`(~36 行)— 非代码 raw source 摘要
    - `Code-Source.md`(~54 行)— 代码源摘要,含 frontmatter + 代码片段引用
    - `Reader.md`(~82 行)— 主题读本完整骨架 + header 里嵌入 §3.4 硬性要求备查
  - CLAUDE.md 瘦身:468 行 / 21 KB → **274 行 / 11 KB**(-41%)
    - 删除纯考古:§3.4 历史债清单(8 行,事实永久在 log.md)
    - 删除冗余:§2 各子目录逐条自注释("Methodology/ - 方法论相关概念"等 ~15 行)、§5 log 三示例压到一示例、§3.4 "读本 vs 原子页"表与 §2 分工说明的重复表述
    - 外移:§4.2-4.5 四份完整 markdown 模板(原 ~130 行)压为 §4 一张"页面类型 → 模板文件"指针表
    - 压缩:§9.4 VCS 约定文字精简,§2 目录结构去冗余注释
- 预期收益:
  - **每轮对话 CLAUDE.md 占用从 ~7K tokens 降到 ~3.7K tokens**(-47%)
  - 更关键:剩下的 274 行几乎全是 always-apply 规则,噪声比显著下降
  - 长对话中对 §7 Don'ts、§3.4 读本规则等硬性约束的遵守可靠性应当上升
  - 代价:创建新页时多一次 `Read Wiki/_templates/<type>.md`(每次 50-80 行),但这是 task-local 聚焦,与"每轮全量加载"完全不同的 attention 经济学
- 设计原则(写进 CLAUDE.md 头部 preamble):"本文件只含每轮对话都要遵守的核心规则。大块页面模板另存 `Wiki/_templates/`,仅在真的要创建对应页面时 Read"
- 下一步:观察后续几次 ingest/读本创建的实际体验;如果模板外移后某个模板高频被 Read,考虑把核心一句话约束提回 CLAUDE.md

## [2026-04-20] refactor | 读本独立顶层目录 Wiki/Readers/
- 触发:用户指出读本散在 `Syntheses/Methodology/`、`Syntheses/AIArt/`、`Syntheses/AIApps/`、`Syntheses/Niagara/` 四个子目录里,和普通专题(三段论演进、How-to-prompt、学习路径总图)混在一起"有点乱,到处都是"
- 方案:新增 `Wiki/Readers/` 顶层目录,5 份读本全部迁过去;`Syntheses/` 回归原定位——非读本类的专题综合
- 迁移(全部用 `git mv` 保留历史):
  - `Wiki/Syntheses/Methodology/Llm-wiki-方法论-读本.md` → `Wiki/Readers/Methodology/Llm-wiki-方法论-读本.md`
  - `Wiki/Syntheses/AIArt/Lora-深度指南-读本.md` → `Wiki/Readers/AIArt/Lora-深度指南-读本.md`(`Wiki/Syntheses/AIArt/` 本来只有读本,搬完即删空目录)
  - `Wiki/Syntheses/AIApps/AI-primer-v2-读本.md` → `Wiki/Readers/AIApps/AI-primer-v2-读本.md`
  - `Wiki/Syntheses/Niagara/Phase0-心智模型-读本.md` → `Wiki/Readers/Niagara/Phase0-心智模型-读本.md`
  - `Wiki/Syntheses/Niagara/Phase1-asset-layer-读本.md` → `Wiki/Readers/Niagara/Phase1-asset-layer-读本.md`
- CLAUDE.md 更新:
  - §2 目录说明:新增 `Readers/` 顶层目录的完整描述 + 明确 "`Syntheses/` vs `Readers/` 的分工"小节
  - §3.4:归档路径从 `Wiki/Syntheses/<topic>/` 改为 `Wiki/Readers/<topic>/`,新增"文件命名与路径"具体示例
  - §4.5:模板标题注明路径 `Wiki/Readers/<topic>/`
  - 历史债清单(§3.4 末尾)3 条 ✅ 条目的链接同步更新到新路径
- 连带同步改链接(用 sed 跨 12 个文件批量替换):
  - 5 份读本之间的互相引用
  - 5 个 Phase 1 Entity 页的"深入阅读"链接
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]] Phase 0/1 顶部读本引流
  - [[index]]、[[Wiki/Overview]]、以及历史 log 条目里的 wikilink
- [[index]] 分区重构:`Readers(主题读本)` 提升为**顶层分区**,与 `Entities / Concepts / Sources / Syntheses` 平级(不再作为 Syntheses 的子分区);Syntheses 分区回到"非读本类专题综合"的精简列表
- 本条 log 描述时的路径均为**迁移后新路径**(旧路径仅在本条内提到一次)
- 下一步:读本独立目录生效,路径稳定,之后 Phase 2 读本直接入驻 `Wiki/Readers/Niagara/`

## [2026-04-20] synthesis | 历史债清算 — 三份主题读本一次性补齐
- 触发:4-19 CLAUDE.md §3.4 引入"主题读本"规则时,历史上三个主题(方法论/AIArt/AIApps)因 ingest 时规则尚未存在而未产出读本,已在当时钉为"历史债清单"。今日用户命令清算
- 新建 3 份读本,每份都按 CLAUDE.md §4.5 通用骨架(问题驱动叙事 + 代码/原文 inline + 陷阱 ⚠️ 高亮 + 深入阅读指回原子 + 开放问题):
  - [[Wiki/Readers/Methodology/Llm-wiki-方法论-读本]] — 方法论议题读本,~500 行,叙事主线"时间线(Memex 1945 → RAG → LLM Wiki 2026)+ 哲学线(关联文档 → 查询时拼碎片 → 持续编译)"交织,五节 Memex/RAG/方法论骨架/Karpathy/本仓库具体化,末尾举 2026-04-17 第一次 ingest 作为完整例子
  - [[Wiki/Readers/AIArt/Lora-深度指南-读本]] — AI 美术议题读本,~900 行,叙事主线"战略(为什么离开 MJ,三个结构性短板)→ 技术(LoRA 原理 → 基座选型 → Caption 反常识 → Trigger Word → Multi-LoRA)→ 工具(kohya_ss + ComfyUI)→ 工程(6 个月路线图 + 合规 + 硬件 + 评估准则)"
  - [[Wiki/Readers/AIApps/AI-primer-v2-读本]] — AI 应用生态议题读本,~1000 行,叙事主线"LLM 本质 → 三个怪癖(幻觉/Context Rot/不稳定)→ 新面孔(推理/多模态/Agent)→ 连接外部(MCP/RAG/记忆)→ 三段论演进 → Harness 四柱 → Agent Skills → 2026 技术栈三层 → Karpathy 三节点 → Vibe/Spec/Harness Coding"
- 更新:
  - [[CLAUDE]] §3.4 历史债清单全部标 ✅,明确"今后新主题 ingest 同步产出读本,不再累积"
  - [[index]] Syntheses 分区新增方法论/AIArt 两个分类,三份读本各自登记
  - [[Wiki/Overview]] 三个主题段落顶部各加"📖 主题读本(推荐初读)"引流
- 总产出:3 份读本,约 **2400 行**叙事内容
- 收获/洞察:
  - 三份读本互相交织——方法论读本引用 AIApps 的 Karpathy,AIArt 读本引用方法论读本作为工作范式,AIApps 读本引用 AIArt 作为 AI 应用的具体落地。读本之间形成**内聚但可独立消化**的知识网络
  - 写读本比写原子页**压力大很多**——必须做完整叙事,不能跳章节,不能省略 trap。但读完单篇就"拿走全部知识"的体验远超读多篇原子页
  - CLAUDE.md §4.5 的模板和"问题驱动 / 代码 inline / 陷阱 ⚠️"硬性要求实际写作中非常好用,保证了三份读本的一致风格
- 下一步:读本规则全面生效,Phase 2(Niagara Component 层)起按新规则直接产出读本。AIArt / AIApps 新增原子页时,对应读本相关章节同步更新

## [2026-04-19] refactor | 导读 → 主题读本,规则泛化到所有议题
- 触发:用户指出"导读"命名有歧义(听着像摘要),而实际要的是**详细、精确、满满当当、一次读完即掌握全部知识、不用跳来跳去**的长文
- 同时指出:这种产物不应只存在于结构化学习路径(如 Niagara 各 Phase),**应覆盖所有议题**——任何有深度的 topic(扫盲源、方法论专题、跨源综合)都该有配套的人类阅读入口
- 改名:`Phase0-心智模型-导读.md` → `Phase0-心智模型-读本.md`;`Phase1-asset-layer-导读.md` → `Phase1-asset-layer-读本.md`(用 `git mv` 保留历史)
- CLAUDE.md 升级:
  - §3.4 重写 "Phase 导读" → "**主题读本(Topic 级线性读物)**",**触发条件从"学习路径阶段收尾"泛化到四种**(结构化路径 Phase / 新主题首次入驻 ≥3 原子页 / 多源交叉综合 / 用户显式要求)
  - 明确界定:"读本不是摘要",四条硬性特征(详细/精确/满满当当/一次读完)
  - 新增"原子页 vs 读本"分工对照表,明确两者互补不替代
  - 新增**历史债清单**:AIArt、AIApps、Methodology 三个主题当时 ingest 时没建读本,用户需要时按清单回补
  - §4.5 模板从 "Phase 导读" 重命名为 "主题读本",改写为通用骨架(适用所有议题,非仅学习路径)
  - §4.2 "Code Entity 紧凑约定" 中的 "Phase 导读" 改称 "主题读本"
- 连带同步改链接:
  - 5 个 Entity 页(Phase 1):`导读 § N` → `主题读本 § N`
  - [[Wiki/Syntheses/Niagara/Niagara-learning-path]]:Phase 0/1 顶部"线性读物"提示统一改为"主题读本"
  - [[index]]、[[Wiki/Overview]]:所有 Phase 0/1 链接同步
- 历史 log 条目(4-19 Phase 0 导读补齐、Phase 1 导读 + 方法论升级)**保留原文**,如实记录当时称"导读"的事实
- 下一步:Phase 2 按新规则产出 "Phase 2 读本";用户需要时可回补 AIArt / AIApps 主题读本

## [2026-04-19] synthesis | Niagara Phase 0 导读(补齐)
- 新建:[[Wiki/Readers/Niagara/Phase0-心智模型-读本|Phase0-心智模型-导读]] — Phase 0 的线性读物,把四个概念页(UObject / Asset-Instance / Niagara-vs-Cascade / CPU-vs-GPU)编成自下而上一条叙事链  *(注:4-19 晚些时候重命名为"读本",详见本条上方的 refactor 条目)*
- 叙事结构:四层脑内地图(Layer 1 UObject → Layer 2 Asset/Instance → Layer 3 Niagara 哲学 → Layer 4 CPU/GPU 分叉),每层末尾小结 + 最后一节"四层地图回看"贯通
- 埋雷:第 2.8 节专门提前钉死"`FNiagaraEmitterHandle::Instance` 不是运行时 Instance"这个 Phase 1 必踩的命名陷阱,让读者到 Phase 1 时有预期
- 更新:[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 0 节顶加导读链接)、[[index]]、[[Wiki/Overview]]
- 动机:补齐方法论升级后 Phase 0 的导读缺位;今后任何 Phase 完成都要有配套线性读物

## [2026-04-19] synthesis | Niagara Phase 1 导读 + 方法论升级
- 触发:用户指出原子化 Source/Entity 页不符合人类线性阅读习惯(频繁跳转、碎片化)
- 核心产出:[[Wiki/Readers/Niagara/Phase1-asset-layer-读本|Phase1-asset-layer-导读]] — Phase 1 的**教科书章节**,500+ 行线性叙事,从 Content Browser 切入讲到图源抽象基类,关键代码片段 inline,不强制跳转  *(注:4-19 晚些时候重命名为"读本")*
- 方法论升级(写入 [[CLAUDE]]):
  - 新增 §3.4 "Phase 导读":结构化学习路径每阶段收尾强制产出线性读物,定位"教科书章节"与原子页 spec 角色互补;准确/不遗漏 > 简短,不为压缩而压缩
  - 新增 §4.5 "Phase 导读页结构"模板
  - 修订 §4.2 "Code Entity 页的紧凑约定":单 source 派生的 entity 页控制在 30-50 行,字段细节下沉到 Source,叙事下沉到导读,Entity 只作稳定跨 source 入口
- Entity 页瘦身(按新约定):[[Wiki/Entities/Stock/UNiagaraSystem]]、[[Wiki/Entities/Stock/UNiagaraEmitter]]、[[Wiki/Entities/Stock/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/UNiagaraScript]]、[[Wiki/Entities/Stock/UNiagaraScriptSourceBase]] 全部重写,每页 35-55 行,保留"一句话角色 + 核心字段速查 + 常用方法 + 深入阅读指回 Source/导读 + 开放问题"
- 更新:[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 1 节顶部加导读链接)、[[index]](Syntheses/Niagara 分区登记导读)、[[Wiki/Overview]](Niagara Phase 1 行加导读引流)
- 收获:
  - 原子页(LLM 检索友好)+ 导读(人类阅读友好)的双层产出,今后作为结构化学习路径的标准动作
  - Entity 的合理边界:单 source 派生时瘦身;多 source 交叉时再扩为汇总枢纽
- 下一步:Phase 2(Component 层,3 文件)按新模板走,ingest 完收尾产一份简版导读(预计 250 行左右)

## [2026-04-19] ingest | Niagara Phase 1 — Asset 层三件套(5 文件)
- 代码源(stock,`b6ab0dee9`,UE 4.26)5 个文件,全部在 `Engine/Plugins/FX/Niagara/Source/Niagara/Classes/`:
  - `NiagaraSystem.h`、`NiagaraEmitter.h`、`NiagaraEmitterHandle.h`、`NiagaraScript.h`、`NiagaraScriptSourceBase.h`
- 新建(10 页):
  - Sources (5): [[Wiki/Sources/Stock/NiagaraSystem]]、[[Wiki/Sources/Stock/NiagaraEmitter]]、[[Wiki/Sources/Stock/NiagaraEmitterHandle]]、[[Wiki/Sources/Stock/NiagaraScript]]、[[Wiki/Sources/Stock/NiagaraScriptSourceBase]]
  - Entities (5): [[Wiki/Entities/Stock/UNiagaraSystem]]、[[Wiki/Entities/Stock/UNiagaraEmitter]]、[[Wiki/Entities/Stock/FNiagaraEmitterHandle]]、[[Wiki/Entities/Stock/UNiagaraScript]]、[[Wiki/Entities/Stock/UNiagaraScriptSourceBase]]
- 更新:[[index]](新增 "Niagara 代码实体" 分区 + "代码源摘要 — stock" 分区)、[[Wiki/Syntheses/Niagara/Niagara-learning-path]](Phase 1 全部打勾 + 标注完成日期)
- 要点:
  - **Asset 三件套的关系网清晰了**:`UNiagaraSystem`(容器,持有 `TArray<FNiagaraEmitterHandle>`)→ `FNiagaraEmitterHandle`(USTRUCT 包装,含唯一 Id + Name + enabled + `UNiagaraEmitter*`)→ `UNiagaraEmitter`(脚本 + 渲染器 + 继承 merge)→ `UNiagaraScript`(编译后产物,字节码 + GPU shader 双形态)
  - **关键命名陷阱**:`FNiagaraEmitterHandle::Instance` 是 Emitter 资产副本(仍在 Asset 层),**不是**运行时 Instance(后者叫 `FNiagaraEmitterInstance`,Phase 3 再看)
  - **EmitterSpawn/EmitterUpdate 脚本不可单独编译**(`UNiagaraScript::IsCompilable` 对这两者返 false),它们被合并进 System 脚本
  - **editor/runtime 模块分离**:`UNiagaraScriptSourceBase` 是抽象基类在 `Niagara`,实现 `UNiagaraScriptSource + UNiagaraGraph` 在 `NiagaraEditor`,这是"runtime 能持图源指针但不 link editor 图实现"的解耦点
  - 编译产物三件套:`FNiagaraVMExecutableDataId`(身份证/DDC key)+ `FNiagaraVMExecutableData`(字节码 + 属性 + DI 绑定)+ `FNiagaraShaderScript`(GPU 侧,Phase 8)
- 下一步:Phase 2 — `NiagaraComponent.h / NiagaraActor.h / NiagaraFunctionLibrary.h`(Component 层,从场景入口视角把 Asset 连到 World)
- 待 Phase 3+ 回看的 open questions(分散记录在各页"开放问题"节):
  - `EmitterExecutionOrder.kStartNewOverlapGroupBit` 的 parallel tick 消费点
  - `RapidIterationParameters` vs `ExposedParameters` vs `User.*` 命名空间的协作关系
  - `FNiagaraVMExecutableData::DIParamInfo` 的技术债(GPU 信息不该在 VM 数据里)
  - `UNiagaraEmitter::Parent / ParentAtLastMerge` 的 merge 语义(能传播什么)

## [2026-04-19] ingest | AI 应用技术发展脉络与核心概念扫盲手册 v2 — 新主题 AIApps 入驻
- source: [[Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册 v2]]（Eureka × Claude，面向零基础读者的 AI 应用生态综合扫盲）
- 上游素材：B 站视频 BV1zSDMBUE5o + 配套飞书文档
- 处理流程：v1 由 Eureka 基于视频/飞书先让 Claude 起草 → Eureka 邀请本 Claudian 做事实审阅 → 审出 v1 若干瑕疵（Vibe Coding 归属、Fowler 三分类/四支柱自相矛盾、MCP 捐赠细节、agentskills.io 域名疑点等）+ 建议补缺（幻觉/推理模型/多模态/Embedding/Subagent/HITL 等）→ Claudian 重写 v2 → v2 ingest 入库 → v1 删除
- 新建主题目录：`Wiki/{Concepts,Entities,Sources,Syntheses}/AIApps/`
- 新建：
  - Source (1): [[Wiki/Sources/AIApps/AI-primer-v2]]
  - Concepts (8): [[Wiki/Concepts/AIApps/Llm]]、[[Wiki/Concepts/AIApps/Hallucination]]、[[Wiki/Concepts/AIApps/Context-window]]、[[Wiki/Concepts/AIApps/Reasoning-model]]、[[Wiki/Concepts/AIApps/Ai-agent]]、[[Wiki/Concepts/AIApps/Mcp]]、[[Wiki/Concepts/AIApps/Harness-engineering]]、[[Wiki/Concepts/AIApps/Agent-skills]]
  - Synthesis (1): [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution]]
  - Entity (1): [[Wiki/Entities/AIApps/OpenClaw]]
- 更新：
  - [[Wiki/Entities/Methodology/Karpathy]]（追加 Context Engineering 倡导者 + Vibe Coding 命名者两个身份；sources 1→2）
  - [[Wiki/Concepts/Methodology/Rag]]（补 Embedding 向量嵌入机制解释 + 与 AIApps/Hallucination、Mcp 的交叉引用；sources 1→2）
  - [[index]]（新增 AIApps 在 Entities/Concepts/Syntheses/Sources 四个分区的条目）
  - [[Wiki/Overview]]（主题从 3 扩至 4，新增 AIApps 小节、重写主题知识图、扩充开放问题）
- 删除：Raw/Articles/AI 应用技术发展脉络与核心概念扫盲手册.md（v1，被 v2 取代）
- 要点：
  - wiki 首次覆盖"AI 应用生态"主题，与既有 Methodology / Niagara / AIArt 并列
  - 主线叙事：Prompt → Context → Harness 三段论；核心洞察"模型正在商品化，竞争在 Harness 和 Skills 两层"
  - 对非技术读者的第一纪律已固化：**AI 给的具体事实必核验（幻觉的结构性导致）**
  - 修正 v1 的几个事实错误，同时补全了幻觉、推理模型、多模态、Embedding、Subagent、HITL、Context Rot 等 v1 缺失的关键概念
- 下一步：用户可能后续会：(a) 让我按 v2 补建 Transformer / Embedding / Vibe Coding 等提及但未建页面；(b) 让我为 wiki 跑一次 lint 检查 AIApps 内部交叉引用是否闭环；(c) 拿这套文档给团队推广时要求生成一页纸简化版

## [2026-04-19] query | 如何向 AI 提问（chat 场景）— 团队推广用
- 触发:团队内推广 AI 使用,观察到"结果质量 ↔ 使用意愿"的正反馈循环,但大多数人不知如何有效提问
- 归档为:
  - [[Wiki/Syntheses/Methodology/How-to-prompt-ai-chat]] — 详细版（心智模型 + 5 要素 + 8 技巧 + 反模式 + 好/差对照 + 团队推广经验）
- 更新:[[index]](Syntheses 新增"方法论"分区)
- 核心洞察:糟糕的 prompt 来自错误的心智模型（把 AI 当搜索引擎/全知神谕）——校正心智模型后,技巧都是推论

## [2026-04-19] ingest | LoRA 深度指南 — AI 美术生成管线首次入驻
- source: [[Raw/Notes/Lora_Deep_Dive]]（鸣潮美术向 TA 的 LoRA 落地方案）
- 新建主题目录:`Wiki/{Concepts,Entities,Sources}/AIArt/`
- 新建:
  - Source: [[Wiki/Sources/AIArt/Lora-deep-dive]]
  - Concepts (5): [[Wiki/Concepts/AIArt/Lora]]、[[Wiki/Concepts/AIArt/Base-model-selection]]、[[Wiki/Concepts/AIArt/Caption-strategy]]、[[Wiki/Concepts/AIArt/Trigger-word]]、[[Wiki/Concepts/AIArt/Multi-lora-composition]]
  - Entities (5): [[Wiki/Entities/AIArt/Illustrious-XL]]、[[Wiki/Entities/AIArt/NoobAI-XL]]、[[Wiki/Entities/AIArt/Flux]]、[[Wiki/Entities/AIArt/Kohya-ss]]、[[Wiki/Entities/AIArt/ComfyUI]]
- 更新:[[index]](新增 AI 美术 Concepts + Entities + Sources 三个分区)、[[Wiki/Overview]](从 1 主题扩至 3 主题综合,重写知识图)
- 要点:wiki 首次覆盖 AI 美术生成领域,和既有方法论/Niagara 主题完全独立；核心洞察是 "LoRA 让游戏团队把美术 DNA 固化成可版本化的文件"、Caption 反常识策略、基座选型的许可陷阱（Flux Dev 非商用）
- 下一步:用户可能要验证本机 kohya_ss 环境,或询问具体章节的实践细节

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

