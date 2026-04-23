---
type: synthesis
created: 2026-04-24
updated: 2026-04-24
tags: [ai, agent, code-retrieval, agentic-grep, artist-tooling, ue4, landing-design, reader]
sources: 4
aliases: [美术代码问答机器人读本, Artist Code QA Bot Reader]
---

# 给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆

> 本页是**"给美术做代码问答机器人"**这个 AI 落地议题的**主题读本**——详细、精确、满满当当,一次读完即完整掌握从底层检索机制(agentic grep)到项目级部署方案(persona / 沉淀回路 / 术语桥梁)的全部关键决策,不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]] 索引。
>
> **贯穿案例**:UE 4.26 Material Instance 的 `Dithered LOD Transition` 开关。本读本会用这一个真实案例,从"美术看到的勾选框" → "shader 宏" → "per-pixel discard" → "性能代价" → "判断建议"的全链路,同时作为**机器人应该产出的答案范本**。

---

## 0. 这个议题要回答的问题

> [!question]
> 1. 为什么值得给美术做一个代码问答机器人?不做行不行?
> 2. LLM 到底是怎么"检索代码"的?和 RAG 有什么区别?
> 3. 如果机器人不认识项目里自定义的 symbol,它能怎么办?
> 4. 具体怎么部署?架构、persona、沉淀、风险都要覆盖
> 5. 怎么做 POC 才不翻车?

**叙事主线**(自下而上):

```
问题痛点(美术经验主义) 
    → 检索机制(agentic grep,三件套)
        → 真实案例(Dithered LOD Transition,从 UI 到性能)
            → 诚实的弱点(未知 symbol 零命中)
                → 破解策略(锚点反推 + 约定穷举 + 种子爬行 + 追问)
                    → 长期出路(wiki 作为术语桥梁 + 长期记忆)
                        → 项目级落地(架构 / persona / 沉淀 / 风险)
                            → POC 最小路径
```

**读完后应能回答**:
- 为什么 agentic grep 比 RAG 更适合代码域?
- 机器人怎么回答"这个 XX 开关是干嘛的"这种问题?每一步做了什么?
- 项目里自定义的 symbol 机器人怎么处理?
- 部署时最容易翻车的点是什么?
- POC 的评估指标怎么选?

---

## 1. 为什么值得做:美术的经验主义瓶颈

项目里长期存在一个结构性矛盾:

- 引擎(原生 + 项目魔改)的**权威答案藏在源码**:材质参数、宏开关、静态开关、Niagara 模块参数、CVar ...
- 美术/TA/策划日常使用这些功能,但**不读 C++/shader**——他们的知识来自老同事口传、文档半全、以及"试试看"
- 想问程序员?打断别人工作流,且不 scale——100 个美术每人每周问 5 个,一个程序员周一就报废了

> [!abstract]
> 这不是"美术懒得学代码",这是**效率分工的必然**。美术该把时间花在美学、视觉、工作流上,不是花在读 shader 宏展开。问题在于:权威答案只在源码里,他们又不会读。

这就是 AI 能填的缝——一个机器人,美术用自然语言问,机器人**基于源码**用美术能听懂的话回答,附上证据。

关键是"基于源码"这四个字。不是基于训练数据的泛泛回答,不是基于一篇可能过时的 wiki,而是**现查、现引用、现翻译**——项目代码每天都在改,只有临场查才是权威的。

---

## 2. 检索底层机制:Agentic Grep

那怎么"临场查"?这里就触到了这次议题的核心概念——**agentic grep**。

### 2.1 三件套工具

Claudian 以及 Claude Code 等现代 AI 编码 agent,都是用这三样东西搜代码:

| 工具 | 本质 | 典型用法 |
|---|---|---|
| **Grep** | 底层是 ripgrep,字面/正则匹配文件内容 | 找 `MATERIAL_TWOSIDED` symbol、找 `#ifdef XXX` 所有出现 |
| **Glob** | 按文件名 pattern 找文件 | `**/MaterialExpression*.cpp`、`Engine/Shaders/**/*.ush` |
| **Read** | 按路径读文件(可指定行号) | 读 grep 命中行的上下文几十行 |

**关键点**:**没有 embedding、没有向量库、没有预建索引**。每次提问都是临场 grep → 看命中 → Read 上下文 → 再决定下一步 grep 什么。LLM 自己操作这三个工具,像人类 senior 工程师一样"导航"代码——这就是 "agentic retrieval"。

### 2.2 vs RAG 的根本差异

> [!question]
> 为什么不用 RAG?RAG 不是更"智能"吗?

| 维度 | RAG | Agentic Grep |
|---|---|---|
| 匹配 | 向量空间**语义近似** | **字面/正则** |
| 索引 | 预建,代码变要 reindex | 无,每次现查 |
| 维护 | 持续(chunk 策略、embedding 选型、重建) | 几乎为 0 |
| 精确度(代码) | 可能漏 symbol、幻觉片段 | `path:line` 硬证据 |
| 精确度(自然语言文档) | 强(能懂"猫"≈"喵星人") | 弱(字面不对零命中) |
| 未知 symbol | 语义近似兜底 | ⚠️ 零命中,需破解策略 |

> [!abstract]
> **代码是符号化系统。`DitheredLODTransition` 就是 `DitheredLODTransition`,不会被嵌入空间搞成"近似"**。字面匹配比语义近似**更准、更稳、更可追溯**。这和 [[Wiki/Concepts/Methodology/Llm-wiki-方法论|Karpathy 的 Wiki 方法论]]是同一哲学——符号化系统靠字面/链接胜过靠向量。

这不是说 RAG 没用。对自然语言长文档(比如一堆 PDF 论文),RAG 仍有优势。**在代码域,agentic grep 胜出**。

---

## 3. 贯穿案例:Dithered LOD Transition 的端到端追踪

光说机制太抽象。用一个真实例子看机器人应该怎么工作:**"材质实例里的 Dithered LOD Transition 开关是干嘛的?"**

这是一个典型的美术问题。下面按**机器人应该执行的检索链**一步步走,同时这也是**机器人最终应该产出的答案范本**。

### 3.1 第一步:判断 symbol + 路由仓

机器人脑内第一跳:"Dithered LOD Transition" → 候选 symbol:

- C++ 风格:`DitheredLODTransition` / `bDitheredLODTransition`
- Shader 宏风格:`DITHERED_LOD_TRANSITION` / `USE_DITHERED_LOD_TRANSITION`
- CVar 风格:`r.Mobile.AllowDitheredLODTransition`

⚠️ **注意**:这里 Claudian 能一口气想出这么多候选,是因为训练数据见过 UE。**换成项目自定义 symbol,这招失效**——这是 agentic grep 真实弱点,§5 会专门讲破解。

按 CLAUDE.md §9 路由:引擎原生功能,优先 `stock` 仓。

### 3.2 第二步:跨文件 grep + Read 上下文

机器人并行 grep 两种风格,跨 C++ 和 shader 两类文件。命中的关键文件:

1. `MaterialInstanceBasePropertyOverrides.h` — 美术在材质实例 UI 上勾的那个框的定义
2. `Material.h` — 父材质上的原始开关定义
3. `MaterialTemplate.ush` — shader 宏合成 + `ClipLODTransition` 实现
4. `LocalVertexFactory.ush` — 植被/HISM per-instance 处理
5. `DepthRendering.cpp` — stencil 优化路径
6. `RendererSettings.h` — mobile 平台总开关 + 性能警告

### 3.3 第三步:按叙事结构组织答案

这里是**机器人应该产出的答案**。美术视角,不是代码罗列,而是"从我能看到的 → 底层发生了什么 → 性能代价 → 我该怎么判断":

---

#### A. 一句话说清楚

**让 Mesh 在切 LOD 时,用"像素点状渐隐/渐显"(dither)过渡,而不是整个模型突然一跳**。配合 TAA 几乎看不出 LOD 切换的瞬间。

#### B. 美术能看到的地方

材质实例右侧面板 `General` / `Material` 折叠区有个勾选项,旁边带一个 Override 勾:

| 列 | 控件 | 作用 |
|---|---|---|
| ☑ Override | `bOverride_DitheredLODTransition` | 开了才覆盖父材质,**不开不生效** |
| ☑ Dithered LOD Transition | `DitheredLODTransition` | 真正的开关 |

源码证据(`MaterialInstanceBasePropertyOverrides.h` L28-46):
```cpp
/** Enables override of the dithered LOD transition property. */
uint8 bOverride_DitheredLODTransition : 1;
...
UPROPERTY(meta = (editcondition = "bOverride_DitheredLODTransition"))
uint8 DitheredLODTransition : 1;
```

> [!warning]
> **第一个常见踩坑**:美术勾了下面那个,没勾 Override,以为改了其实没生效。`editcondition` meta 就是这个 UI 约束。

#### C. 底层它到底做了什么

这是个 **static switch**——编译期决定。勾上后编译出的 shader 会多走一段"丢像素"的分支。

**(1) 开关驱动 shader 宏**

引擎把勾选状态变成 `USE_DITHERED_LOD_TRANSITION_FROM_MATERIAL`,shader 里合成为 `USE_DITHERED_LOD_TRANSITION`:

`MaterialTemplate.ush` L90-102:
```hlsl
#ifndef USE_DITHERED_LOD_TRANSITION
    #if USE_INSTANCING
        #define USE_DITHERED_LOD_TRANSITION USE_DITHERED_LOD_TRANSITION_FOR_INSTANCED
    #else
        #define USE_DITHERED_LOD_TRANSITION USE_DITHERED_LOD_TRANSITION_FROM_MATERIAL
    #endif
#endif
```

**(2) 像素 shader 里的 `ClipLODTransition`**

宏打开时,材质编译器在 PS 入口塞一个 clip() 调用:按每像素噪声值 vs LOD 过渡系数,**该丢就 discard**。CPU 端每帧把"当前 LOD 过渡进度"写进 `PrimitiveDither.LODFactor`(0→1),shader 就把越来越多像素 discard——看起来就是"渐隐":

`MaterialTemplate.ush` L2425-2432:
```hlsl
#elif USE_DITHERED_LOD_TRANSITION && !USE_STENCIL_LOD_DITHER
void ClipLODTransition(FMaterialPixelParameters Parameters)
{
    if (PrimitiveDither.LODFactor != 0.0)
    {
        ClipLODTransition(Parameters, PrimitiveDither.LODFactor);
    }
}
```

**(3) 植被/HISM 的 per-instance 特殊处理**

草、树、批量植被(HISM)每个 instance 的 dither cutoff 塞在 per-instance 数据里,让每棵树按自己到相机的距离独立渐隐。这就是属性 tooltip 里特意点到 **"when used with the foliage system"** 的原因:

`LocalVertexFactory.ush` L808-815:
```hlsl
#if USE_DITHERED_LOD_TRANSITION
    float RandomLOD = InstancingViewZCompareZero.w * Intermediates.PerInstanceParams.x;
    ...
    Intermediates.PerInstanceParams.w = /* per-instance dither cutoff */ ;
#endif
```

**(4) 两种实现路径(渲染器侧优化)**

引擎内部有两套路径,美术不用关心,但会间接影响性能:

| 路径 | 条件 | 特性 |
|---|---|---|
| **Per-pixel discard** | 默认 | 简单粗暴,但会破 Early-Z |
| **Stencil dither** | `r.StencilLODDither` 开 | 先在 PrePass 往 stencil 里写掩码,正式 BasePass 只拼 stencil test,**不破 Early-Z** |

#### D. 性能代价(⚠️ 美术要关心的)

`RendererSettings.h` L192-195 的 tooltip 原话(`> [!quote]` 引):

> [!quote]
> "Whether to support 'Dithered LOD Transition' material option on mobile platforms.  
> **Enabling this may degrade performance as rendering will not benefit from Early-Z optimization.**"

翻译:

1. **`discard` 打破 Early-Z 剔除**。Early-Z = "像素 shader 还没跑,硬件先比深度,没通过就不跑 shader"。shader 里有 `discard` 后,硬件不敢提前丢深度,必须**把整段 PS 跑完再决定**。PS 越重,代价越大
2. **Mobile 上代价更显著**,有专门的 `r.Mobile.AllowDitheredLODTransition` 开关(默认关)。Mobile 项目勾了材质上的 dither,**还得 Project Settings 里总开关也开**,否则勾了也不生效
3. **过渡期间绘制两份**:LOD0 和 LOD1 同时渲染、一个渐隐一个渐显,**短时间 overdraw 翻倍**。这是视觉平滑的必然代价

#### E. 什么时候开、什么时候不开

| 场景 | 建议 |
|---|---|
| 主角/重要角色、PS 很贵的材质 | ❌ 不开。切换瞬间一跳可接受,性能更重要 |
| 远景植被、草、HISM 批量物件 | ✅ 开。per-instance dither 效果极好,镜头平移看不出跳变 |
| Mobile 项目 | 谨慎开。先确认 `r.Mobile.AllowDitheredLODTransition=1`,再按需开 |
| 没开 TAA 的项目 | ❌ 基本别开。dither 噪声会非常明显,TAA 是用来糊掉它的 |

---

### 3.4 这个答案作为"产出范本"的关键特征

回头审视上面这个答案,它做到了几件对美术友好的事:

1. **先用美术视角起手**("一句话说清楚 / 美术能看到什么"),后才沉入底层
2. **代码块紧凑**,只保留关键几行,不 dump 整个函数
3. **每处引用都有 `path:line`**,程序员复核能快速定位
4. **性能代价单独成节**,并翻译了"Early-Z"这种术语
5. **给出使用建议表**,直接答"我该不该勾"这个核心问题
6. **用 Obsidian callout**(`> [!warning]` / `> [!quote]`)做视觉分层

机器人 prompt 里最好强制这套骨架,让输出稳定、不同 persona 只是细节粒度差异。

---

## 4. 诚实的弱点:未知 symbol 怎么办

如果你仔细看 §3.1,Claudian 一口气报出候选 symbol 那一步有"作弊"成分——它训练数据里见过 UE,脑子里已有这些拼写。

**换成项目自定义 symbol 呢?** 比如某个 plugin 里自定义的材质宏 `AKI_CUSTOM_FX_MODE`,LLM 训练时从没见过。这时 agentic grep **会卡住**——不知道该 grep 什么。

这是必须直面的真实弱点。破解策略按推荐度:

### 4.1 锚点反推(最好用)

美术不懂代码,但一定看过**某个字符串**:UI 勾选项文本、工具栏按钮、报错消息、日志 tag、控制台命令。这些字符串在代码里**一定存在**。

| 美术说的 | 机器人该 grep 什么 |
|---|---|
| "材质里那个 XX 开关" | `LOCTEXT.*"XX"` / `DisplayName="XX"` / `ToolTip="XX"` |
| "Log 里说 XX 失败了" | `"XX"`(字面,找 `UE_LOG` / `ensureMsgf`) |
| "控制台打 r.XX 就能开" | `"r.XX"`(`IConsoleVariable` 注册点就在附近) |
| "按 F5 会怎样" | `EKeys::F5` 或输入绑定名 |

UE 约定很稳定:**所有玩家/美术可见文本都走 `LOCTEXT` / `NSLOCTEXT` / `FText` / `DisplayName` meta**。grep 这些几乎 100% 命中。

> [!tip]
> 机器人 persona prompt 里应该**教它**第一跳先问美术:"你是在哪个面板看到的?那个开关旁边写的文字是什么?" 这个追问能把命中率从 30% 拉到 90%。

### 4.2 命名约定穷举

UE 和项目代码有稳定 prefix/pattern。同一概念同时试多个变体:

```bash
# 比如美术说"那个 XX 渲染开关"
grep -i "xx"                          # 不区分大小写,广撒网
grep "bXx\|bXX"                       # UE bool 约定
grep "XX_\|_XX\|USE_XX\|ENABLE_XX"    # shader 宏约定
grep "r\.XX\|r\.Render\.XX"           # CVar 约定
grep "FXx\|UXx\|AXx\|EXx"             # UE 类型前缀
```

### 4.3 种子爬行

命中**一个**——哪怕模糊相关——就能 Read ±50 行,相邻 `#include`、function、注释都是新 grep 种子。几跳之内通常能摸到目标。

LLM 看起来"知道要搜什么",很多时候前几步是偷偷试错——试失败的不显示,试成功的链接下去看起来很笃定。

### 4.4 停下追问

上面都不行时,**不猜是最重要的策略**。典型问法:

- "这个开关在哪个面板看到?截图能发吗?"
- "你能在 .uasset 里看到它附带的文字吗?哪怕错别字也行"
- "这是 Engine 改的还是 Plugin 加的?大概哪个模块?"

> [!warning]
> 机器人最怕的不是"我不知道",而是"乱说但听起来很专业"。System prompt 里**首要一条**是"未 grep 到证据绝不作答,说'没找到'比瞎编好"。美术对代码无免疫力,一次幻觉能污染半年信任。

---

## 5. 长期出路:Wiki 作为术语桥梁 + 长期记忆

§4 是临阵补救。长期看,有更好的解法——**本 vault 本身就是答案**。

[[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]]的 compounding 效应在项目代码场景兑现如下:

### 5.1 Wiki 作为词汇桥

Wiki 页的 `aliases` 字段、tag、叙事段落,本质就是"美术的自然语言 ↔ 代码 symbol"的翻译表。机器人第一跳**先查 wiki**,命中一条 alias 或 tag,就拿到了候选 symbol。

### 5.2 术语对照表(关键产物)

建议产出 `Wiki/Concepts/Glossary/术语对照表.md`:

| 美术 / 策划常说 | 代码 symbol | 所在模块 | 备注 |
|---|---|---|---|
| 半透明 | `bIsTranslucent` / `BLEND_Translucent` | MaterialShared | Blend mode 枚举 |
| 渐隐切 LOD | `DitheredLODTransition` / `USE_DITHERED_LOD_TRANSITION` | Material + Shaders | 有 mobile 单独开关 |
| 阴影不对 | `CastShadow` / `bCastShadowAsTwoSided` / `CastShadowAsMasked` | PrimitiveComponent + Material | 分 component / material 两层 |
| 项目 XX 切换 | 你们自定义的 `EAkiXXMode` | `AkiGameplay/Plugins/XX` | 跨插件 |

机器人第一跳**先查这张表**翻译词汇,再去 grep。

### 5.3 两条腿并跑

| 路径 | 覆盖 | 优势 | 劣势 |
|---|---|---|---|
| **临场 grep** | 所有未 ingest 的长尾 | 覆盖全,零维护 | 首次查慢,质量看 LLM |
| **wiki 命中** | 高频 + 已 ingest | 答案精确,有人类复核,附读本 | 需要 wiki 先建设 |

初期全靠 grep;wiki 积累到一定量后,**绝大多数高频问题直接命中**,grep 退居长尾。

### 5.4 优先 ingest 清单(自定义代码)

对项目自定义代码(机器人天然不认识),优先 ingest:

1. 自定义 `UPROPERTY` 集中的头(UI 上所有看得到的开关源头)
2. 自定义 shader 的 `.usf` / `.ush`(宏名就是未来的 grep 关键词)
3. Plugin 的 `README` / `.uplugin` 描述
4. 项目自定义 CVar 的注册点(grep `IConsoleManager::Get\(\).RegisterConsoleVariable`)

> [!abstract]
> **半年后,术语对照表 + wiki 就是项目 AI 的核心资产**。每次 ingest 补一行对照表,每次 Q&A 沉淀一页 Syntheses——复利巨大。这是 Karpathy 讲的"LLM Wiki 强于 RAG"在项目场景的直接兑现。

---

## 6. 项目级落地:架构 / Persona / 部署风险

### 6.1 架构选型

| 组件 | 选型建议 |
|---|---|
| **接入层** | 飞书 / 钉钉机器人 → Claude Code headless 模式 / Anthropic Agent SDK |
| **代码访问** | 跑在能 mount UE 工程的机器上(git + p4 client),复用 `code-roots.local.md` 思路 |
| **Persona** | 做 2-3 个子命令 / [[Wiki/Concepts/AIApps/Agent-skills\|Skill]]:`/ask-artist`、`/ask-tech-artist`、`/ask-programmer`——同一检索底座,不同解释粒度 |
| **可追溯** | 强制每条答案附 `path:line` + 原始代码块 |
| **禁幻觉** | System prompt:未 grep 到证据不许作答 |
| **沉淀回路** | 高价值 Q&A 自动入 `Raw/Notes/qa-log/` → 定期人工 ingest |
| **机密与审计** | 内网部署;不得发到公开群;保留访问日志 |

### 6.2 Persona 粒度示例

| Persona | 解释粒度 | 术语替换 | 代码展示 |
|---|---|---|---|
| `/ask-artist` | 类比日常经验、比喻优先 | "Early-Z"→"硬件提前偷懒丢像素",术语首次出现加括号 | 代码块少、只关键注释,重点给"勾这个 = 看到什么 + 性能代价" |
| `/ask-tech-artist` | 概念级 + 数据流 | 保留 `discard` / `Early-Z` / `per-instance`,加 1 句定义 | 关键代码块 inline,标 `path:line`,不省略 shader 宏展开 |
| `/ask-programmer` | 工程细节 + 性能模型 | 全部保留原词 | 全代码块 + 调用链 + 矛盾点分析 |

§3.3 的 Dithered LOD 答案是 `/ask-tech-artist` 粒度的——给纯美术还要再"糊"一层。

### 6.3 部署风险

> [!warning]
> **幻觉是头号敌人**。美术对代码无免疫力,一旦被误导很久纠不回来。System prompt 必须明确:
> - 未 grep 到证据 → 说"我没找到相关代码",绝不作答
> - 有证据但不确定 → 列出证据 + 标注"以下推断仅供参考,建议找程序员复核"

> [!warning]
> **机密边界**。项目代码通常算机密,机器人回复**不能**截图发到公开群。部署要内网、审计日志要留。

> [!warning]
> **p4 权限**。问到 `project-*` 仓代码时,机器人得有 p4 读权限——IT 过一下 account。

> [!warning]
> **术语漂移**。美术和程序员对同一功能叫法可能不同,机器人无法自发知道"美术说的 XX = 代码里的 YY"——需要术语对照表人工维护。

---

## 7. POC 最小路径:不要一上来就全铺开

> [!tip]
> **不要一上来就全项目铺开**。典型翻车模式:上来就想覆盖所有子系统 → 首次 Demo 有几次幻觉 → 美术信任崩塌 → 项目废弃。

最小可验证版本:

1. **选一个最常被问的子系统**(比如只覆盖材质 static switch)
2. **手动整理 1 份术语对照表**(10-20 行即可)
3. **只开 `/ask-material-artist` 这一个命令**
4. **找 3-5 个美术试用 2 周**
5. **跑通再扩**:加 Niagara、加 Lighting、加项目自定义模块...

### 评估指标

| 指标 | 目标 | 怎么测 |
|---|---|---|
| 采纳率 | > 60% | 美术是否按机器人答案操作 |
| 幻觉率 | < 2% | 程序员抽检答案对错 |
| 时间节省 | 显著 | 美术报告:少问程序员多少次 |
| 沉淀产出 | 每周 5-10 条 | 可 ingest 的 Q&A 数量 |

---

## 全景回看

把整条链再捋一遍:

```
【痛点】美术经验主义,权威答案在源码,问程序员不 scale
    ↓
【方案】机器人基于源码临场回答,用美术能听懂的话 + 证据
    ↓
【检索机制】Agentic Grep(Grep/Glob/Read 三件套),不走 RAG
    ↓
【范本】Dithered LOD Transition 案例:UI → shader 宏 → discard → 性能代价 → 建议
    ↓
【弱点】未知 symbol 零命中 —— UE 原生靠训练数据先验,自定义靠四策略:
    ├─ 锚点反推(LOCTEXT / DisplayName / ToolTip 可见字符串)
    ├─ 命名约定穷举(UE 前缀、shader 宏大写)
    ├─ 种子爬行(命中一个向外扩)
    └─ 停下追问(宁可多问一轮)
        ↓
【长期解】Wiki 作为术语桥梁 + 长期记忆:
    ├─ Wiki 页的 alias/tag 本身就是词汇桥
    ├─ 术语对照表(关键产物)
    ├─ 两条腿并跑:临场 grep(长尾) + wiki 命中(高频)
    └─ 优先 ingest 自定义代码入口
        ↓
【部署】架构 / 三 persona / 禁幻觉 / 机密 / p4 / 术语漂移
    ↓
【POC】选一子系统 + 3-5 美术 + 2 周 → 采纳率 / 幻觉率 / 节省时间 / 沉淀产出 → 跑通再扩
```

---

## 10 条关键洞察

1. **代码是符号化系统**,agentic grep 的字面匹配比 RAG 的语义近似更准、更稳、更可追溯
2. **三件套 Grep+Glob+Read 是"自己写 rg"**,每次现查,零维护,代价是依赖关键词命中
3. **LLM 训练数据先验是"作弊"的一部分**——UE 原生懂,项目自定义不懂,老实承认比装懂重要
4. **锚点反推是未知 symbol 的最好用救命稻草**:UE 约定所有 UI 文本走 `LOCTEXT` / `DisplayName`
5. **Wiki 是 agentic grep 的长期外脑**——wiki 越厚,grep 越少,命中越准
6. **术语对照表是项目 AI 的核心长期资产**,复利巨大,每次 ingest 补一行
7. **禁幻觉是部署首要纪律**:美术对代码无免疫力,一次瞎说污染半年信任
8. **Persona 分层**(artist / tech-artist / programmer)同检索底座,不同解释粒度
9. **答案骨架要稳定**:一句话 → 美术视角 → 底层机制 → 性能代价 → 使用建议
10. **POC 不要铺太大**:一个子系统 + 3-5 人 + 2 周评估 → 跑通再扩

---

## 自检问题(读完回答)

下面这些题需要把本读本 §2-7 串起来才能答得有根据,不是"回原文检索某段"就能应付的浅题。

1. **为什么不用 RAG 做代码检索?** 如果坚持用 RAG,具体会在什么场景出错?举 Dithered LOD Transition 这个案例里,RAG 相比 agentic grep 会漏掉哪些东西?

2. **训练数据先验的作弊成分**:Claudian 能对 UE 原生 symbol 一口气报出三种拼写,是因为训练见过。那对一个叫 `AkiCustomFXMode` 的项目自定义宏,机器人第一跳应该做什么?具体步骤,不许答"就 grep 呗"。

3. **锚点反推 vs 命名约定穷举**:这两招什么时候用哪招?如果美术既说不清在哪个面板看到、也描述不出可见文字,只说"就是那个渲染开关",你会选哪条路?

4. **Wiki 的 compounding 效应**:假设 wiki 积累了 200 页概念 + 完整术语对照表,机器人的命中率为什么会比纯 grep 高很多?不许答"因为数据多",要讲具体机制——wiki 里的**什么**帮助了**grep 的哪一步**?

5. **如果没有禁幻觉的 System prompt**,机器人的失败模式具体是什么?举一个 Dithered LOD Transition 可能被瞎答的版本,说明美术看完会做什么错误决策。

6. **为什么 POC 要从材质 static switch 开始**,而不是 Niagara?(提示:思考哪种子系统术语相对稳定、grep 命中容易、答案可被程序员快速复核。)

7. **Persona 分层的代价转移**:三个 persona 同一检索底座、只差解释粒度——这套设计的**维护成本**主要转移到哪里了?如果后期要加 `/ask-level-designer` 第四个 persona,具体要改什么?

8. **用 5 句话向不熟悉本议题的项目经理解释**:为什么值得投入 2 个人月做这个机器人?能讲清楚就是真懂。

---

## 议题留下的问题

- **POC 评估基线怎么定**:采纳率、幻觉率、节省时间——没有对照组怎么量化?→ 需要运行 1 周后回顾
- **冷启动问题**:自定义代码未 ingest 前,机器人会反复"没找到",可能挫伤使用信心。是否需要冷启动时批量 ingest 前 20 个最常被问的自定义模块?
- **蓝图/uasset 不可 grep**:纯蓝图逻辑和材质图无法走代码 grep 路径——需要"蓝图→文本 dump"工具链配合?这是本议题的显著盲区
- **更新漂移**:项目代码持续变,wiki 里的答案会过时;lint 周期和触发机制需要定(每周跑?每次大版本后跑?)
- **agent 数量 vs 质量非线性**:本议题暂按单 agent 设计。若未来需要并行跨三仓搜索,是否该升级到 [[Wiki/Concepts/AIApps/Multi-agent|multi-agent]] 架构?拐点在哪里?

---

## 下一步预告

如果本方案被采纳,建议的后续动作:

1. **立项 POC**,绑定一个具体子系统(推荐材质 static switch 起手)
2. **初版术语对照表**:人工整理 20-30 行,覆盖项目最常问的概念
3. **机器人 System prompt 骨架**:含答案结构模板 + 禁幻觉纪律 + persona 切换
4. **部署环境**:内网一台能 mount UE 工程的机器 + p4 client + 飞书/钉钉接入
5. **2 周美术试用 + 评估**
6. **沉淀回路接通**:Q&A 入 `Raw/Notes/qa-log/` → 定期 ingest 成 wiki 页

这些展开够再写一份实操 playbook,本读本只到方案层。

---

## 深入阅读

### 本议题的原子页
- 概念页:[[Wiki/Concepts/AIApps/Agentic-grep]] — agentic grep 检索机制 + 破解策略
- 综合页:[[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] — 项目级落地设计
- 源摘要:[[Wiki/Sources/AIApps/Code-retrieval-conversation]]
- Raw 对话:[[Raw/Notes/代码检索与美术问答机器人对话]]

### 前置议题
- [[Wiki/Concepts/Methodology/Rag|RAG]] — 对照面
- [[Wiki/Concepts/Methodology/Llm-wiki-方法论|LLM Wiki 方法论]] — 长期记忆框架
- [[Wiki/Concepts/AIApps/Hallucination|幻觉]] — 部署首要防御
- [[Wiki/Concepts/AIApps/Agent-skills|Agent Skills]] — Persona 实现路径
- [[Wiki/Concepts/AIApps/Multi-agent|Multi-agent / Subagent 架构]] — 未来架构升级路径
- [[Wiki/Syntheses/AIApps/Prompt-context-harness-evolution|Prompt → Context → Harness 三段论]] — 方法论母体

### 跨主题联系
- [[Readers/Methodology/从 Memex 到 LLM Wiki|方法论读本]] — Wiki 方法论完整脉络
- [[Readers/AIApps/AI 应用生态全景 2026|AI 应用生态读本]] — AI 技术栈背景
- [[Readers/AIApps/为什么上下文有限反而必须切多 Agent|Multi-agent 读本]] — 未来升级参考

### 代码索引(Dithered LOD Transition 贯穿案例引用的 stock 仓文件)
- `Engine/Source/Runtime/Engine/Classes/Materials/MaterialInstanceBasePropertyOverrides.h`
- `Engine/Source/Runtime/Engine/Classes/Materials/Material.h`
- `Engine/Shaders/Private/MaterialTemplate.ush`
- `Engine/Shaders/Private/LocalVertexFactory.ush`
- `Engine/Source/Runtime/Renderer/Private/DepthRendering.cpp`
- `Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h`

---

*本读本由 [[Wiki/Entities/Claudian|Claudian]] 基于本议题 4 个原子页综合生成,2026-04-24。*
