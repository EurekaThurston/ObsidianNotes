---
type: synthesis
created: 2026-04-22
updated: 2026-04-22
tags: [infofeeds, automation, feishu, github-actions, design]
sources: 0
aliases: [Feed 推送方案, InfoFeeds 设计文档]
---

# 周期性 Feed 推送系统方案设计

> 本文档是 2026-04-22 设计讨论的沉淀,用于 vault 迁移 / 会话中断后无缝衔接。新会话里 Claudian 读完本文即可接续实施。

## 0 · 一句话目标

**让符合我个人主题画像的新内容,每天定时通过飞书机器人推送给我;权重由 vault 内容驱动,但主动引入邻接探索与远场源,防止信息茧房;视觉内容(VFX)独立通路。云端运行,不依赖任何一台电脑。**

---

## 1 · 核心架构(最终版)

```
┌─ F:\Cortex\ObsidianNotes(public git repo)──┐
│  Raw/ Wiki/ Readers/ CLAUDE.md             │  ← 知识库本体,公开
│  （Feed/ 不放这里——隐私数据迁出）             │
└─────────────────────────────────────────────┘

┌─ F:\Cortex\InfoFeeds(private git repo)─────┐
│  _system/                                  │
│    topic-profile.json      ← Claudian 算   │
│    topic-overrides.yaml    ← 人手动调      │
│    feeds.yaml              ← RSS 源清单    │
│    creators.yaml           ← VFX 白名单    │
│    curated-sources.yaml    ← 远场源        │
│    seen.jsonl              ← 去重历史      │
│    feedback.jsonl          ← 👍👎 反馈     │
│  digests/YYYY-MM-DD.md     ← 每日推送归档  │
│  inbox/                    ← 📥归档的文章   │
│  scripts/digest.py         ← 主 pipeline   │
│  scripts/rebuild_profile.py                │
│  .github/workflows/digest.yml              │
└─────────────────────────────────────────────┘

       ↑ Claudian 本地(Max 订阅)
       · 初次 bootstrap:扫 vault 出 topic-profile
       · 周度重算权重
       · 处理 📥归档 → ingest 到 Raw/Articles/
       · 重度脑力活(review、lint、调源)

       ↓ GitHub Actions(每天 8:00 CST,纯 Python,不调 Claude)
       1. git pull InfoFeeds
       2. 读 yaml/json 配置
       3. feedparser 抓 RSS(互联网上)
       4. 对 seen.jsonl 去重
       5. 关键词匹配 topic-profile
       6. 按 ε-greedy 预算分配(60 核心 / 25 邻接 / 15 远场)
       7. 组装 Feishu interactive card
       8. POST 飞书 webhook
       9. git commit seen.jsonl + digests/ 回 repo
```

---

## 2 · 五个关键决策(含讨论脉络)

### 2.1 路径隔离 · 独立 `Feed/` 目录 → 最终改为独立仓

**问题**:feed 数据(seen/feedback 等)污染 vault。

**演进**:
1. 起初方案:vault 内建 `Feed/` 顶层目录
2. 考虑到 vault 是 public repo,`seen.jsonl` / `feedback.jsonl` / `topic-profile` 持续暴露阅读指纹——**敏感的是被动行为数据,不是笔记内容**
3. 最终:**Feed 迁出 vault 到独立 private repo `F:\Cortex\InfoFeeds`**

**为什么 vault 不用装 Feed**:
- 隐私数据与公开笔记物理隔离
- Action 运行时只需要 `InfoFeeds/`,不需要 `Wiki/` / `Readers/`——因为 `topic-profile.json` 是 Claudian 本地预计算好的序列化产物
- 未来换云端平台(Cloudflare Workers 等)只动 private repo
- vault 结构保持三层(Raw/Wiki/Readers)不被打破

### 2.2 权重计算 · 纳入 Readers + 手动覆盖层

**问题**:只按 Wiki 原子页数算权重片面——Niagara 天生概念密集,但不代表比 AI 重要。

**方案**:双层结构

**底层 · 自动计算**:

```
score(topic) =
    w1 × log(1 + atomic_pages)              # Wiki 原子页(log 压缩)
  + w2 × readers_count × 3                  # 读本强加权(w2 比 w1 高 3-5 倍)
  + w3 × readers_wordcount / 10000          # 读本字数(深度指标)
  + w4 × recency_boost(updated_in_30d)      # 近期活跃度
```

**核心洞察**:**读本 = 深度投入的硬信号**。愿意为一个主题写完一份读本,比原子页数量更能反映真实关心程度。

**上层 · 手动覆盖** (`topic-overrides.yaml`):

```yaml
# 直接覆盖最终权重
Niagara: 0.25           # 自动 0.35,压一压
AI-agent: 0.30          # 自动 0.05,想多看,抬起来
LLM-Wiki 方法论: auto   # 维持自动值

# 临时 boost,到期自动失效(学新东西)
boosts:
  - topic: Rust
    weight: 0.15
    expires: 2026-07-01
```

**分工明确**:Claudian 算,人只做 review + 感觉微调,无需手算。

### 2.3 防茧房 · 三层防御

**问题**:纯 vault 相似度匹配,6 个月后会变回音壁。

**方案**:

| 层 | 比例 | 机制 |
|---|---|---|
| **核心** | 60% | topic-profile 高权重主题 |
| **邻接** | 25% | embedding 距离近但权重低的主题(你可能感兴趣但没深耕) |
| **远场** | 15% | curated 高信号源,不过主题过滤器 |

**邻接示例**:
- 重仓 Niagara → 邻接推 Houdini VEX / Compute shader / SDF
- 重仓 LLM Wiki → 邻接推 information retrieval / Zettelkasten

**远场源白名单**(不靠主题,无差别曝光):
- HN 每周 Top 5、The Browser、Stratechery
- 你佩服但领域不同的人的博客

**反馈回路防塌缩**:
- 👍 远场内容 → 自动加 boost 到 overrides,新兴兴趣浮现
- 连续 N 天某核心主题 👎 → 自动降权
- **每月一次茧房体检**:Claudian 对比 6 个月前的分布,告诉你"变窄 / 变宽了 n%"

### 2.4 视觉内容 · VFX 独立通路

**问题**:UE VFX 工作重视觉,纯文字 pipeline 对视觉效果是瞎子。

**MVP 阶段方案(无 LLM 多模态)**:

1. **换源结构**——VFX 主题专用源:

| 类型 | 源 | 抓什么 |
|---|---|---|
| 社区 | realtimevfx.com | 帖子 + 首图/gif |
| Twitter | Nitter RSS 镜像(实例不稳,需轮换) | 推文 + 附带媒体 |
| YouTube | 频道 RSS(`/feeds/videos.xml?channel_id=`) | 标题 + 缩略图 + 描述 |
| ArtStation | 无 RSS,v2 再做 | —— |
| 国内 | B 站 VFX up 主、米哈游/鹰角技术分享 | 封面 |

2. **VFX creator 白名单**(`creators.yaml`,20–50 人):
   - Simon Trumpler、Fabio Sciedlarczyk、Asher、Sirhaian、Gabriel Aguiar...
   - **同行信号 > 算法猜测**——白名单内容不走主题匹配,直接入候选池

3. **飞书卡片带图**:interactive card 支持 image 元素,缩略图直接嵌进卡片,不点进去也看得到效果大概

4. **信号替代**(无 LLM 打分时的粗筛):
   - Twitter 点赞/转发 > 阈值
   - YouTube 播放/点赞比
   - Real Time VFX 回复数

**v2 升级(需要 LLM 视觉)**:候选缩略图 + 标题 + 描述一起丢给 Claude/Gemini 多模态打分"是否类 vault 里 X 主题的效果"。MVP 不做,观察一段时间再评估。

### 2.5 云端运行 · GitHub Actions,但**不用** Claude

**关键澄清**:
- 用户是 **Claude Max 订阅,非 API 版**
- Max 订阅**可以**通过 `CLAUDE_CODE_OAUTH_TOKEN` 在 Action 里跑 Claude Code,但会吃 Max 5 小时滚动额度、有被限流风险
- **更好的选择**:MVP 阶段 Action 里**根本不调 Claude**——纯 Python + RSS + 规则筛选,0 成本 0 风险

**最终路径 C · 混合架构**:

| 层 | 跑在哪里 | 用 Claude 吗 | 频率 |
|---|---|---|---|
| **日常 digest** | GitHub Actions(云端) | ❌ 纯 Python 规则 | 每天 1 次 |
| **权重重算** | 本地 Claudian | ✅ Max 订阅随便用 | 每周 1 次(人触发) |
| **📥归档 ingest** | 本地 Claudian | ✅ | 人手动触发 |
| **茧房体检** | 本地 Claudian | ✅ | 每月 1 次 |

**唯一需要的 Action secret**:`FEISHU_WEBHOOK_KEY`(+ 开启签名校验的 `FEISHU_WEBHOOK_SECRET`)。**不需要** Anthropic 的任何凭据。

**GitHub Actions 免费额度**:私有 repo 每月 2000 分钟,digest 一天几十秒,**月度成本 ≈ $0**。

---

## 3 · MVP 范围界定(明确"不做什么"比"做什么"更重要)

### ✅ MVP 做

- RSS 源抓取(Epic Dev Community、realtimevfx.com、YouTube 频道、arXiv、HN、独立博客)
- 关键词粗筛 + 权重排序
- ε-greedy 预算分配(60/25/15)
- Feishu interactive card(含缩略图、3 按钮)
- 去重(seen.jsonl)+ 归档(digests/)
- 本地 Claudian 的 bootstrap + 周度重算 + 归档 ingest

### ❌ MVP 不做(v2+)

- LLM 细筛打分(观察粗筛质量再说)
- LLM 多模态视觉打分(先靠 creator 白名单 + 社交度量)
- Nitter Twitter 源(实例不稳定,等 MVP 跑通)
- ArtStation(无 RSS,要自解析 HTML)
- 实时推送(日级 digest 已经够)
- 嵌入模型自动算邻接(先手工在 `curated-sources.yaml` 里标)

---

## 4 · 当前进度 & 待办(给新会话 Claudian 的接力棒)

### 已完成

- [x] 架构设计(本文档)
- [x] `F:\Cortex\` 母文件夹已建,`InfoFeeds` 已 move 进来
- [ ] `F:\ObsidianNotes` → `F:\Cortex\ObsidianNotes`(**用户操作中**)

### 下一步(用户先完成迁移)

1. **用户**:关闭 Obsidian,把 `F:\ObsidianNotes` 剪切到 `F:\Cortex\` 下
2. **用户**:重开 Obsidian,从 `F:\Cortex\ObsidianNotes` 载入 vault
3. **用户/Claudian**:把 `~/.claude/projects/F--ObsidianNotes` 改名为 `F--Cortex-ObsidianNotes` 以保留会话历史
4. **用户**:GitHub 上的 vault repo 本地路径变了,`git config` 不受影响,但你本地 git 工具如果记着旧路径可能要重新 add

### 新会话 bootstrap 阶段 1(Claudian)

5. **Claudian**:Glob `Wiki/**/*.md` 和 `Readers/**/*.md`,按 §2.2 公式算初版 `topic-profile.json`,落到 `F:\Cortex\InfoFeeds\_system\topic-profile.json`
6. **Claudian**:生成人类可读的 `topic-profile-explained.md`,每主题列原始统计(页数、读本数、字数、关键词样本)
7. **Claudian**:起草 `feeds.yaml`——根据 profile 里 top 主题,每主题推荐 3–5 个 RSS 源
8. **Claudian**:起草 `creators.yaml`——VFX 艺术家白名单初稿(20 人左右)
9. **Claudian**:起草 `curated-sources.yaml`——远场高信号源初稿

### 阶段 2(用户 review)

10. **用户**:看 `topic-profile-explained.md`,在 `topic-overrides.yaml` 里调感觉不对的权重
11. **用户**:在 `feeds.yaml` 里增删源
12. **用户**:creator 白名单补你关注的艺术家

### 阶段 3(Claudian 实现 pipeline)

13. **Claudian**:写 `scripts/digest.py`——feedparser 抓 RSS、去重、匹配、排序、预算分配、组装 card、POST webhook
14. **Claudian**:写 `scripts/rebuild_profile.py`——扫 vault 重算权重
15. **Claudian**:写 `.github/workflows/digest.yml`——cron 定时触发(推荐 UTC 0 点 = CST 8 点,用 `cron: '0 0 * * *'`)

### 阶段 4(上线)

16. **用户**:在飞书创建自定义机器人,拿 webhook URL 和签名 secret
17. **用户**:GitHub InfoFeeds repo → Settings → Secrets 加 `FEISHU_WEBHOOK_KEY` 和 `FEISHU_WEBHOOK_SECRET`
18. **Claudian**:先在本地 dry-run 一次(不发飞书,只打印 card JSON),确认格式对
19. **用户**:手动 trigger 一次 workflow(`Actions` → `Run workflow`),看真实推送效果
20. **用户**:挂机跑一周,收集 👍👎 数据
21. **Claudian**:根据第一周反馈微调权重公式 / 源选择

---

## 5 · ⚠️ 陷阱与注意事项

- **Windows 路径锁**:移动 `F:\ObsidianNotes` 前**必须完全退出 Obsidian**(包括托盘图标)。Claudian 跑在 Obsidian 插件里,会话会随移动中断
- **Claude Code 会话存储**:会话 jsonl 存在 `C:\Users\weitiange\.claude\projects\F--ObsidianNotes\`,**不在 vault 里**。但 key 是用路径转义的,vault 移动后必须同步改名此目录,否则历史"消失"
- **Nitter 实例**:近年不稳定,MVP 先跳过 Twitter 源;若将来做,需要维护可用实例列表
- **seen.jsonl 无限增长**:设清理策略,只保留最近 180 天
- **Action commit 历史膨胀**:每日 commit 会让 private repo 历史很长;可每月 squash 一次
- **飞书 webhook 签名**:建议开启签名校验,防止 URL 泄漏被乱发
- **GitHub Actions 时区**:cron 表达式默认 UTC,写 `'0 0 * * *'` 其实是 CST 早上 8 点。别被坑
- **YouTube RSS 限制**:只返回最新 15 条,高频发布的频道可能错过;需要配合 ETag 或数据库记录
- **VFX creator 账号漂移**:人改 ID / 销号常见,每季度需要 lint 一次 `creators.yaml`

---

## 6 · 关键洞察(本次讨论沉淀的核心认识)

1. **权重是"服务 LLM 的量化",但依据必须包含"人类投入的证据"**。单纯页数会被主题天然密集度误导,读本数量/字数才是深度指标
2. **隐私数据和知识笔记是两类东西**,物理隔离(不同 repo)优于字段隔离
3. **防茧房不是算法问题,是预算分配问题**。ε-greedy 比任何"智能推荐"都可靠
4. **视觉内容的一等公民信号是"谁发的",不是"内容是什么"**。creator 白名单 > 自动理解
5. **Max 订阅在云端的最优用法是不用**——把 Max 留给本地深度工作,云端用纯规则,两边各司其职
6. **"MVP 不做什么"比"MVP 做什么"价值更高**。能用关键词解决的不上 LLM,能用白名单解决的不上相似度,能用日 digest 解决的不做实时

---

## 7 · 自检问题(读完回答,验证真懂)

1. 为什么 `seen.jsonl` 不能放 public vault repo?把它放 public 会暴露哪类信息,这类信息和 `Wiki/Concepts/*` 的笔记本质区别是什么?
2. Niagara 原子页 40 个、读本 2 份;AI-agent 原子页 4 个、读本 3 份。套 §2.2 公式(假设 w1=1, w2=3),哪个主题权重更高?这个结果反映了什么设计意图?
3. 用户连续一周给远场源某条内容点 👍,系统应该怎么响应?如果系统**不**响应,会导致什么长期问题?
4. MVP 选择"Action 里不调 Claude"而不是"用 Max OAuth token"。这个决定的代价转移到了哪里?(提示:并没有免费午餐)
5. VFX creator 白名单的作用是"绕过主题匹配"。反过来想:如果把白名单做得太大(比如 500 人),会撞到什么问题?
6. 用 5 句话向不熟悉本方案的人解释:为什么权重要**从 vault 算**、又要**手动可覆盖**、还要**引入和 vault 无关的远场源**——这三件事看似矛盾的逻辑是什么?

---

## 8 · 留下的问题(待后续决策 / 讨论)

- **embedding 算邻接主题**:MVP 里用手工标,长期是否引入嵌入模型?用哪个?放本地还是云端?
- **飞书卡片消息体积**:8 条 × 每条带缩略图,单条消息会不会超飞书限制?需要实测
- **digest 归档 md 的格式**:是否要和 vault 的 wikilink 风格统一,方便未来搜?
- **多设备推送**:目前只一个飞书群。要不要支持推到 Telegram / 邮件多通道?(v3+)
- **"这条我看过"的反向信号**:如果用户在 vault 里已经 ingest 了某篇文章,同主题新内容是否降权?(可能过度工程)

---

## 9 · 深入阅读

- [[CLAUDE.md]] §3.4 读本产出规则(本文档的元规则)
- [[CLAUDE.md]] §9 code-roots(本方案之后可能借鉴其"逻辑定义同步 + 路径本地化"思路,把飞书 webhook 放 `.local` 文件)
- [[Readers/Methodology/从 Memex 到 LLM Wiki.md]] —— 方法论背景
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论.md]] —— 为什么 vault 能作为主题画像来源

---

## 10 · 签名

讨论发起:2026-04-22
设计方:Eureka(人类)× Claudian(LLM)
当前状态:**设计完成,等待 vault 迁移后 bootstrap**
下一次接续:新会话中 Claudian 读本文档 → 执行 §4 阶段 1 第 5 步
