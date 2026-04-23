---
type: synthesis
created: 2026-04-24
updated: 2026-04-24
tags: [ai, agent, texture, vfx, artist-tooling, landing-design, tool-use, reader]
sources: 1
aliases: [AI 贴图工具读本, Texture Tool Reader, 特效贴图 AI 工具读本]
---

# 让 AI 接特效贴图的长尾需求 — 架构与 GitHub 生态

> 本页是"AI 特效贴图工具"议题的**主题读本**——详细、精确、满满当当,一次读完即完整掌握这套工具**为什么要这么设计 / GitHub 生态能接到哪一步 / 怎么做 POC / 有哪些坑**,不需要跳转。
>
> 如需字段级查询或溯源,见末尾的 [[#深入阅读]]。

---

## 0. 这个议题要回答的问题

你是 VFX / TA 程序,美术每隔几天就抛一个新的贴图处理需求过来:缩放、去模糊、去 seam、四方连续化、通道打包、生成 variation、atlas 拼接、flipbook 切分、色彩空间转换……每次都让你加一个功能按钮。你不想这么干——**你不擅长 DCC 工具开发,而且美术的需求本质上是长尾的**,这不是工程能堆完的活。

> [!question] 读完后你应当能回答
> 1. 为什么"AI + 工具使用"是这类长尾需求的自然解法,而不是硬编码 / 不是 RAG
> 2. 为什么工具架构要分三层(原子工具 / 模型工具 / 代码沙箱),而不是全做成原子工具或全让 AI 写代码
> 3. VFX 贴图和"通用图像编辑"的本质差异在哪里——为什么直接把 AI 图像编辑工具拿来用会出问题
> 4. 2026-04 GitHub 生态已经能接到哪一步,你要自己写的究竟是什么
> 5. 为什么 ComfyUI + workflow-skill 很可能是当下最省力的基座
> 6. POC 应该从哪个单点切入,如何评估成败
> 7. 这个工具和 [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆|代码问答机器人]] 同构在哪里——这种同构暗示了什么"项目级 AI 应用"通用套路

叙事主线:

```
问题:美术需求千奇百怪
  ↓
AI agent 为什么是自然解(vs 硬编码 / vs RAG)
  ↓
三层架构 L1 / L2 / L3 的设计逻辑
  ↓
VFX 专属 guardrail(数据语义 / 位深 / 色彩空间 / tile)
  ↓
GitHub 现成生态按"解决什么问题"分类
  ↓
ComfyUI + workflow-skill 为什么是当下的 sweet spot
  ↓
POC 最小路径 + 风险
  ↓
与代码问答机器人同构
```

---

## 1. 为什么 AI agent 是长尾需求的自然解

美术需求长尾的本质:**有限高频 + 无限低频**。

- 高频:缩放、超分、去模糊、四方连续、通道打包——这五个可能覆盖 60% 的日常请求
- 低频:各种组合 / 变种 / 一次性需求——剩下 40% 每个都只出现过几次,但加起来体量大

**硬编码路线**:把每个高频需求做成工具按钮——前 5 个工程师愿意写,第 20 个就开始怨声载道,第 50 个 UI 都塞不下。这是**经典"工具组织"悖论**:需求 CAP 定理里,"覆盖度 / 可发现性 / 维护成本"三者不可兼得。

**纯 RAG 路线**(检索历史处理方案):文档域 RAG 不错,但贴图处理是**动作**不是**知识**,检索"上次怎么做"不能直接复现,美术还得自己翻译。不对口。

**AI agent + tool use 路线**:LLM 当**路由器 + 翻译器**:
- 翻译:美术自然语言("把这张云雾变清晰再做成四方连续") → 机器可执行的工具调用序列
- 路由:从一组工具里按需求类型、贴图类型、约束条件选最合适的
- 兜底:工具库没有直接对应的能力时,现场写 Python 代码实现

这三件事**LLM 比人工 UI 路由天然有优势**——它能理解模糊描述、能问澄清问题、能看图判断类型(多模态)、能串多步。

> [!abstract] 一句话
> AI agent 把"我要在 UI 上加一个新按钮"的工程成本,转化成"我给 LLM 补一句 prompt 或加一个轻量工具"的配置成本。成本曲线从阶梯变成平缓,长尾就能接住了。

---

## 2. 三层架构的设计逻辑

看起来最简单的设计是两种极端:

- **极端 A**:全做成原子工具(L1)。LLM 只挑,不写代码。——稳,但覆盖度烂,回到硬编码的老路
- **极端 B**:LLM 全写 Python。——灵活,但**慢 / 贵 / 不稳定**(每次都要生成→跑→改),高频需求被反复重新发明,还容易有幻觉 bug

正解是**按需求频率分层**:

```
L1 原子工具  ← 高频、稳定、确定性
    resize / crop / rotate / channel_pack / color_space_convert
    flipbook_split / atlas_pack / gamma / alpha_premult_toggle
  ↑ 下沉
  | 某个 L3 代码被重复生成>N 次 → 提炼为 L1 工具

L2 模型工具  ← 中频、需要 AI 模型
    Real-ESRGAN / SwinIR (超分 / 去模糊)
    Stable Diffusion img2img / inpaint (语义级编辑)
    tileable 生成(SD + asymmetric tiling / Tiled Diffusion)

L3 代码沙箱  ← 长尾、一次性、组合型
    LLM 现场写 Python(PIL / OpenCV / numpy / scipy)
    沙箱执行 → 失败看 error → 改代码 → 再跑
```

**为什么这么分?**

1. **L1 的价值是"不浪费 token"**:`resize(img, 1024, 1024)` 这种操作让 LLM 每次现场写代码,既慢又错误率高,没意义
2. **L2 的价值是"封装模型调用复杂度"**:Real-ESRGAN 的参数、模型权重路径、前后处理(sRGB↔Linear)这些细节如果让 LLM 每次都查文档,它记不住,靠谱工具必须封好
3. **L3 的价值是"兜底长尾"**:没有 L3,你又回到硬编码老路;有 L3,无限需求都能接住
4. **高频下沉机制**:如果某种 L3 写的代码被 LLM 重复生成超过 N 次,运维人(TA 或你)把它提炼为 L2 / L1。这是**工具库随使用自然生长**的机制——类似 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot|代码问答机器人]] 的 wiki 沉淀,都是"让 AI 用过的轨迹自动变成下一代的基础设施"

> [!tip] 选型口诀
> 稳定的、确定性的、被高频调用的 → L1
> 需要大模型 / GPU / 依赖重的 → L2
> 剩下的一切 → L3 兜底

---

## 3. LLM 做路由,不做像素

这是整套架构的**第一性原则**,违反它会出大事。

**为什么**:

- LLM 自己"画"像素(比如让 LLM 直接输出 base64 图)——幻觉爆炸,精度烂,成本高,**没有任何优势**
- 确定性工具做像素——`resize(1024, 1024)` 永远给你 1024×1024,不会有 1020×1024 的幽灵

所以分工是:

| 环节 | 谁做 | 为什么 |
|---|---|---|
| 理解需求 | LLM | 自然语言 → 结构化意图,这是 LLM 的本职 |
| 澄清问题 | LLM | "你这张是 data texture 还是颜色图?" 需要对话 |
| 选工具 | LLM | 基于需求 + 图类型 + 约束条件选 |
| 编排顺序 | LLM | 先去模糊再超分还是反过来?LLM 基于领域知识决定 |
| **执行像素操作** | **工具** | **确定性、可复现、无幻觉** |
| 验证结果 | LLM(多模态看图) + 工具(像素 diff) | 双重校验 |

这个分工跟 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] 里 agentic grep 的分工同构:**LLM 决定调什么,确定性基础设施实际执行**。这不是巧合,是"项目级 AI 应用"的通用套路(见 §8)。

---

## 4. VFX 贴图不是照片 — 这是 guardrail 的本质

如果你直接把 [imgflw](https://github.com/ototadana/imgflw) 这种"通用 LLM 图像编辑工具"拿来给美术用,很快会出事故。**原因是 VFX 贴图和"通用照片"在几个维度上本质不同**,而通用工具的默认设置是为照片优化的。

### 4.1 通道语义

通用图像编辑假设 RGBA 是颜色 + 透明度。**VFX 贴图里 RGBA 往往不是颜色**:

- **Distortion map**:RG 通道 = xy 方向偏移量,BA 往往空着或打包别的
- **Flow map**:RG 通道 = 流动方向
- **Packed mask**:R / G / B / A 各是一张独立的灰度 mask(metallic / roughness / AO / emissive)
- **Normal map**:RGB = xyz 方向,数值被重新映射到 [0,1]

> [!warning] 最致命的错误
> 对 distortion map 跑 Real-ESRGAN 超分。AI 超分模型在照片上训练,会"美化"边缘、"去噪"、"增加锐度"——但这些操作**对 xy 偏移数据是随机破坏**。结果是超分后的贴图在引擎里特效完全错乱,但视觉上"看起来好像更清晰了",美术不一定立刻发现。

**工具侧对策**:每张图带 `data_texture: bool` 标注,true 时**禁用所有 AI 超分 / 去噪 / 色彩调整**,只允许 bicubic / nearest 这种数学确定性 resize。LLM 的 system prompt 里写明这条。

### 4.2 位深与格式

通用工具的 pipeline 默认 8-bit PNG/JPG。游戏 VFX 常用:

- 16-bit PNG(灰度 mask 高精度)
- .exr / .hdr(HDR 烟雾、体积光、IBL)
- BC 系列压缩格式(实际用的时候)

> [!warning] 隐蔽的坑
> PIL / OpenCV 默认读 .exr 时容易做 tone mapping 把 HDR 压成 [0,1],原始高光信息就丢了。工具层必须用 `OpenEXR` / `imageio` 显式保持原位深。

### 4.3 Alpha 预乘 vs 直乘

同一张 RGBA 图,预乘(alpha-premultiplied)和直乘(straight alpha)的 RGB 数值**不同**。
- 做 resize 时,如果位深 / 预乘设置不一致,边缘会出脏点
- 预乘图做 color 操作前要先除 alpha,改完再乘回去

**工具侧对策**:每个可能影响颜色的工具签名显式接受 `premult: bool`,默认不假设。

### 4.4 色彩空间 — **最隐蔽**的坑

- 游戏颜色贴图存 sRGB,法线 / mask / HDR 存 Linear
- 大部分 AI 超分 / SD 模型训练在 sRGB 图上,默认输入 sRGB
- 如果你把 Linear 空间的 mask 图直接喂 Real-ESRGAN,模型会把它当成极暗的 sRGB 图处理——gamma 全错,结果完全变形

**工具侧对策**:所有进 L2 模型的图必须先显式转 sRGB,出来再转回原空间。工具签名里 `input_colorspace` / `output_colorspace` 显式参数,不默认。

> [!warning] 为什么最隐蔽
> 色彩空间错误**不会报错、不会明显变形**,只会让输出"亮度看起来怪怪的",美术可能会归因于"模型效果一般",把错误数据就这么用了。半年后上线之后才发现某些资产整体偏暗,查不到根源。

### 4.5 Tile 可见度

四方连续做完必须**2×2 平铺一次看接缝**,单张看不出来。这是所有 tileable 工具都必须强制的步骤,不能让 LLM 说"处理好了"就算完。

> [!abstract] VFX guardrail 的本质
> 通用 AI 图像编辑工具假设"像素是给人看的"。VFX 贴图很多情况下**像素是给 shader 算的**。这两种用途对"什么是正确的处理"有本质不同的定义,工具层必须感知到,不能依赖 LLM 记住。

---

## 5. GitHub 现成生态按"解决什么问题"分类

好消息是 2026-04 这个时间点,**组件非常齐全,你基本不用从零写**。但生态分散,按"解决什么问题"看清楚才好挑。

### 5.1 架构范本(看架构怎么切)

| 项目 | 核心思想 | 参考价值 |
|---|---|---|
| [GenArtist](https://github.com/zhenyuw16/GenArtist) (NeurIPS 2024 Spotlight) | MLLM 做 agent,把复杂任务**分解成子任务树** + 分步执行 + 自检自纠 | ⭐⭐⭐ 三段式架构范本,直接对应你的规划→执行→验证 |
| [AgentLego (InternLM)](https://github.com/InternLM/agentlego) | LLM 视觉工具 API 库,含生成 / 编辑 / 感知多类工具,接 LangChain / Lagent | L2 基础设施,看它怎么封工具接口 |
| [imgflw](https://github.com/ototadana/imgflw) | LLM 驱动图像编辑 demo,极简 | 入门 UI ↔ LLM ↔ 工具串联参考 |

**读法**:GenArtist 的自验证循环是这套方案里最"像人"的部分——**生成一步,看结果,不对就回退改参数再来**。你的工具要不要做这个能力,决定了用户体验差距。

### 5.2 基座选型:ComfyUI 生态(重点)

ComfyUI 是当下 VFX / 概念美术圈的事实标准节点式图像处理平台,覆盖 Stable Diffusion、超分、色彩、tile、3D 等几乎所有你会用到的能力。现在在它上面加 LLM 有多个成熟项目:

| 项目 | 特点 | 推荐度 |
|---|---|---|
| [twwch/comfyui-workflow-skill](https://github.com/twwch/comfyui-workflow-skill) | **自然语言 → ComfyUI workflow JSON**;34 内置模板 + 360+ 节点定义;**作为 Claude Code / Cursor 的 skill 工作** | ⭐⭐⭐ 跟本仓 Claude 生态最合拍 |
| [DanielPFlorian/ComfyUI-WorkflowGenerator](https://github.com/DanielPFlorian/ComfyUI-WorkflowGenerator) | 专门 LLM 微调,三阶段 pipeline 生成 workflow | 架构参考 |
| [heshengtao/comfyui_LLM_party](https://github.com/heshengtao/comfyui_LLM_party) | ComfyUI 内 LLM agent 框架,含 MCP / 多 LLM 适配 | 生态整合型 |

**为什么 ComfyUI + workflow-skill 是 2026 当下的 sweet spot**:

1. **节点已覆盖 90%+ 的 VFX 操作**:resize / 超分 / SD inpaint / tile / 通道 / 色彩——你不需要写这些
2. **workflow-skill 做了"自然语言 → workflow JSON"这步**:就是你要的"翻译"能力,而且是 Claude skill 格式,跟本仓方法论([[Wiki/Concepts/AIApps/Agent-skills]])无缝衔接
3. **美术本来就可能在用 ComfyUI**:工作流复用,不需要重新培训
4. **JSON workflow 可保存、可版本化、可共享**:解决"美术之间传经验"的问题

**你要自己写的只是**:VFX 专属 guardrail(§4)注入 LLM system prompt + 若干 VFX 专用节点(数据贴图检测器 / tile 预览器 / 色彩空间校验器)。

### 5.3 四方连续专项(高频单点,已非常成熟)

四方连续是美术最高频的单点需求之一,**至少有 5 种成熟方案**,按图类型挑:

| 项目 | 方法 | 适用 |
|---|---|---|
| [sagieppel/convert-image-into-seamless-tileable-texture](https://github.com/sagieppel/convert-image-into-seamless-tileable-texture) | **经典 average-blending + SD inpaint 双方法** | ⭐ baseline,代码可抄,一个工具内双算法分型 |
| [camenduru/seamless](https://github.com/camenduru/seamless) | SD 生成式 tileable | 风格化 / 重新生成 |
| [brick2face/seamless-tile-inpainting](https://github.com/brick2face/seamless-tile-inpainting) | A1111 插件,SD inpaint 做 seamless | 成熟 |
| [Tiled Diffusion (CVPR 2025)](https://github.com/madaror/tiled-diffusion) | 扩散模型原生支持 tiling | 前沿,效果最好 |
| [dream-textures (Blender)](https://github.com/carson-katri/dream-textures) | Blender 内 SD + seamless | 如果工作流在 Blender |

**选型逻辑**(LLM 应当基于图类型挑):

- 规则纹理(砖、木纹):频域对称化(经典算法,sagieppel 的 average-blending 分支)
- 有机纹理(烟雾、云、地表):offset + SD inpaint 接缝区(brick2face / sagieppel 的 SD 分支)
- 完全重新生成:Tiled Diffusion 或 camenduru/seamless
- 数据贴图:**只能经典算法**,不能用 SD inpaint(会破坏数据语义)

### 5.4 Adobe / PS 生态(备选,不推荐独占)

| 项目 | 特点 |
|---|---|
| [mikechambers/adb-mcp](https://github.com/mikechambers/adb-mcp) | Adobe 官方背景 PoC,MCP 让 AI 控制 PS / Premiere,已验证 Claude Desktop + OpenAI Agent SDK |
| [Photoshop MCP server](https://playbooks.com/mcp/photoshop) | 通用 PS MCP |

**为什么不推荐作主后端**:PS 不擅长 VFX 数据贴图(.exr / 通道语义 / tile 验证等),虽然美术熟,但覆盖度不够。可以作为**补充后端**,处理 PS 强项(图层管理、笔刷、蒙版)。

### 5.5 可泛读书签

- [Yuan-ManX/ai-game-devtools](https://github.com/Yuan-ManX/ai-game-devtools) — AI 游戏开发工具汇总(含 Texture / Shader / 3D 专题),生态变化快,定期翻

> [!abstract] 生态结论
> 2026-04 你要建这个工具,技术栈几乎已成:**ComfyUI(节点基础)+ workflow-skill(自然语言翻译)+ VFX guardrail(你写)+ 定制节点(数据贴图检测 / tile 预览)**。自研部分可能不到整个系统 20%。

---

## 6. POC 最小路径

借鉴 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] 的 POC 方法——**单点突破,小范围试点,四指标评估**。

### 6.1 为什么选"四方连续 + 超分"作 POC 起点

候选需求排序:

| 需求 | 需求热度 | 现成组件 | 视觉反馈直观 | 选型 |
|---|---|---|---|---|
| 四方连续 | 高 | ⭐⭐⭐(5+ 方案) | ⭐⭐⭐ | ✅ |
| 超分 / 去模糊 | 高 | ⭐⭐⭐(Real-ESRGAN) | ⭐⭐⭐ | ✅ |
| 通道打包 | 中 | ⭐(需要自己写) | ⭐ 不直观 | ❌ |
| flipbook 切分 | 中 | ⭐⭐ | ⭐⭐ | ❌ 次选 |

选"四方连续 + 超分"的理由:**现成组件最全 + 视觉反馈最直观 + 真实需求热度最高**。美术一眼看到 before/after,判断"这 AI 靠不靠谱"的成本最低,POC 验证信号最强。

### 6.2 POC 范围

- **时长**:2 周
- **试点人数**:3-5 个美术 / TA
- **功能**:只做"四方连续 + 超分 + 二者组合"这一小类
- **基座**:ComfyUI + `twwch/comfyui-workflow-skill` + 自定义的 VFX guardrail prompt + 2×2 tile 预览节点
- **交付形式**:美术描述需求 → 机器生成 workflow → 自动跑 → 出 before/after + 2×2 预览 → 美术决定采纳 / 回退

### 6.3 四指标评估

借鉴 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]]:

1. **采纳率**:AI 出的结果美术直接用的比例。<30% 说明翻译 / 模型没跑通,>70% 说明真正落地了
2. **幻觉 / 失败率**:"处理成功了"但实际结果错(色彩空间错、seam 没修好、细节丢失)的比例。这是**最致命**指标,>10% 就要停下 debug
3. **节省时间**:相比原工作流(PS / Substance / 手工)节省多少。<30% 说明价值不够,>50% 说明可以推广
4. **沉淀产出**:哪些需求高频出现,值得从 L3 下沉为 L2 / L1 工具;哪些 prompt 技巧值得固化

### 6.4 失败了怎么办

- **采纳率低**:通常是翻译步骤不行——美术说的词(如"做成无缝")LLM 不一定对应到正确操作。解决:**VFX 术语对照表**(跟 [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] 的"术语桥梁"同构)
- **幻觉率高**:通常是 guardrail 不够。补 prompt + 加工具层硬校验(data_texture 自动检测 + 强制拒绝非法组合)
- **时间没省**:通常是交互路径长。把高频操作下沉为 slash command / 模板 workflow

> [!tip] POC 纪律
> **幻觉率 > 采纳率 > 时间 > 沉淀**,这个优先级不能乱。美术对错误输出没有免疫力(他们假设工具给的就是对的),一次严重幻觉会污染半年信任——跟 [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] 里的观察一致。

---

## 7. 风险与陷阱

| 风险 | 场景 | 对策 |
|---|---|---|
| **破坏原资产** | AI 直接覆写输入图 | 强制输出到副本,原图只读挂载 |
| **幻觉式"改好了"** | LLM 报告成功但像素未动 / 错动 | 每步**强制像素级 diff 校验**,diff 为 0 或意外大 → 报警 |
| **色彩空间静默错误** | 输出看起来只是"颜色怪了一点",无报错 | 工具签名显式 colorspace 参数,**不默认** |
| **模型对 data texture 致命** | 对 normal / distortion / flow 跑 Real-ESRGAN | data_texture 标注 + 硬禁用 AI 模型 |
| **L3 代码债** | 长尾代码累积成屎山 | 定期 lint,高频下沉 L1;保留生成日志追溯 |
| **外部模型权重漂移** | 新版 Real-ESRGAN 效果变 | 锁定 model hash,不自动更新 |
| **批量处理失控** | 100 张图一次处理,一步错步步错 | 先单张 dry run,确认后批处理,失败可续跑 |

---

## 8. 全景回看 — 与代码问答机器人的同构

你会发现这套设计和 [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] 结构上高度同构:

| 维度 | Code QA Bot | Texture Tool |
|---|---|---|
| 服务对象 | 美术 / TA | 美术 / TA |
| AI 角色 | 只读 + 解释 | 读写 + 生产 |
| 基础设施 | Agentic grep(Grep/Glob/Read 三件套) | ComfyUI 节点 + Real-ESRGAN + SD |
| LLM 分工 | 路由 + 解释,不真"懂"代码 | 路由 + 翻译,不碰像素 |
| 长期记忆 | Wiki 术语对照表 | 高频工具下沉 + prompt 库 |
| 核心风险 | 幻觉 → 错误解释 | 幻觉 → 破坏原资产 |
| 沉淀回路 | 高频问题 → wiki 页 | 高频代码 → L2/L1 工具 |
| 翻译难点 | 自定义 symbol ↔ 美术词汇 | 美术需求 ↔ 机器操作 |

这种同构不是巧合。**项目级 AI 应用落地的通用套路**是:

1. **LLM 做路由 + 翻译,确定性基础设施做执行** — 避免 LLM 直接生产内容的幻觉风险
2. **渐进式工具层 / 记忆层 / 沉淀机制** — 让 AI 使用的轨迹自然沉淀为下一代基础设施(see [[Wiki/Concepts/Methodology/Llm-wiki-方法论]])
3. **guardrail 写在 system prompt + 工具层双重防御** — prompt 是提醒,工具是硬约束,两者都不能省
4. **POC 从单点突破** — 不要试图一步到位覆盖所有需求
5. **四指标评估,幻觉率优先** — 幻觉放第一位

> [!abstract] 一句话
> 建 AI 工具不是"让 AI 做事",是"让 AI 按已有基础设施把事做对"。基础设施 + guardrail 决定能走多远,LLM 只是让基础设施长出触手。

## 8 条关键洞察

1. 美术需求长尾,硬编码注定 scale 不过,AI agent + 工具使用是**自然解**
2. 三层架构 L1/L2/L3 按频率分,**高频下沉、长尾兜底**,不是工程洁癖,是必要设计
3. **LLM 做路由,工具做像素** — 违反这条必出幻觉事故
4. **VFX 贴图不是照片**:通道语义 / 位深 / alpha / 色彩空间四重差异,必须在工具层和 prompt 层双重防御
5. 色彩空间错误**最隐蔽**——不报错,只"看起来怪",半年后才爆雷
6. 2026-04 GitHub 生态**接近齐活**:ComfyUI + workflow-skill + 现成超分 / tile 方案 — 你要写的不到 20%
7. POC 选"四方连续 + 超分"——现成组件最多 + 视觉反馈最直观 + 热度最高
8. 这套和代码问答机器人**同构**,背后是"项目级 AI 应用"的通用套路:路由 / 沉淀 / 双重 guardrail / 单点 POC / 幻觉优先

## 自检问题(读完回答)

下面这些题需要把 §1-§8 串起来才能答得有根据,每题都不是"回原文检索字段"的浅题。

1. **三层架构反推**:假设你决定"反正长尾需求也就那几十种,我把它们全工程化进 L1,放弃 L3"。请写出具体会撞到的 3 个问题(提示:从需求发现、维护成本、能力上限三个角度想)
2. **data texture 事故链**:如果你把 distortion map 喂给 Real-ESRGAN 超分。请讲清楚**从美术点按钮到引擎里特效出 bug 的完整事件链**——从模型训练假设到像素值含义到最终渲染结果,串起来告诉我为什么会错、错在哪、什么时候被发现
3. **基座选型取舍**:项目组 <20 人美术,你在"ComfyUI + workflow-skill"和"自建 Gradio + Claude tool use"之间选。给 3 条判断标准,每条标准指向不同选择。
4. **Tile 预览必须性**:为什么做完四方连续必须 2×2 平铺看,单张看不行?给一个具体失败模式——"单张看完全没问题,但拼起来立刻暴露"的那种
5. **项目级 AI 应用通用套路**:把 §8 的对照表抽象成一个"下次你遇到另一个美术向 AI 工具需求(比如 UI 动画生成)时的脑内 checklist"。最少 5 条
6. **POC 选型反推**:为什么选"四方连续 + 超分"而不是"通道打包"?如果老板硬要你做通道打包的 POC,你会怎么说服他改选?用四指标的语言答
7. **色彩空间隐蔽性**:给一个**具体场景**——某张贴图 colorspace 错误被处理后,美术怎么看 / 引擎怎么用 / 上线后怎么爆雷 / 半年后怎么被归因——把半年周期讲清楚
8. **5 句话解释**:用 5 句话向一个只懂 PS、不懂 LLM 的资深 VFX TA 解释"为什么 AI 做路由、工具做像素"这套分工。不能用"agent / tool use / guardrail"这类术语

## 这个议题留下的问题

- **data-aware 超分**:VFX 数据贴图(flow / distortion / packed mask)有没有 data-aware 超分的学术方案?当前答案是"只用 bicubic",可能有更聪明的
- **与 Niagara / Material 工作流衔接**:能不能做到"在 Niagara 编辑器里选中一张贴图,一键喂给工具 + 回写"?需要 UE 插件层的集成
- **批处理的规划器**:100+ 贴图批处理时,AI 规划一次、批量跑、人抽检——这个"规划一次跑多次"的循环怎么设计?ComfyUI 原生不擅长
- **ComfyUI vs 自建 UI 的长期取舍**:ComfyUI 省力但可控性有限;自建投入大但能深度绑定项目(资产命名规范 / 导出约定 / P4 集成)。1 年、3 年、5 年时间尺度的成本曲线怎么画?
- **冷启动 + 信任塑造**:POC 阶段如何在一次严重幻觉就会摧毁信任的前提下,让美术愿意试?可能需要"低风险/只读预览模式"作第一阶段

## 下一步预告

- **短期**(本周):用 `twwch/comfyui-workflow-skill` 跑一个 benchmark 任务——"512 云雾 → 去模糊 + 4× 超分 + 四方连续 → 2048" —— 测生态实际覆盖度与幻觉率
- **中期**(1 月):若 benchmark 通过,列 5-10 个高频 VFX 需求,按 L1/L2/L3 归类,写 VFX guardrail prompt 草稿,开始 POC
- **长期**(3-6 月):POC 结果驱动:基座稳定 → 扩展需求覆盖;基座不够 → 评估自建 Gradio 路线;都不行 → 重审议题定义

## 深入阅读

### 本议题的原子 / 综合页

- 综合页(设计源):[[Wiki/Syntheses/AIApps/Ai-texture-tool-design]] — 设计方案三层架构 + GitHub 调研 + 风险 + POC

### 前置议题

- 概念:[[Wiki/Concepts/AIApps/Ai-agent]] — AI agent 是什么;思考→行动→观察循环
- 概念:[[Wiki/Concepts/AIApps/Agent-skills]] — Agent Skills 渐进式披露(workflow-skill 就是这类)
- 概念:[[Wiki/Concepts/AIApps/Harness-engineering]] — 为什么 prompt 和工具层双重防御
- 概念:[[Wiki/Concepts/AIApps/Hallucination]] — 为什么幻觉率优先级最高

### 跨主题联系(同构读本)

- [[Readers/AIApps/给美术做代码问答机器人 - 从 grep 到 wiki 复合记忆]] — 同构的"美术向 AI 应用落地"读本;路由 / 翻译 / 沉淀 / 双重 guardrail / 四指标评估的另一个实例
- [[Wiki/Syntheses/AIApps/Artist-code-qa-bot]] — 代码问答机器人的综合页,POC 方法论的源头

### 方法论背景

- [[Readers/Methodology/从 Memex 到 LLM Wiki]] — 本仓库方法论读本;这里的"高频工具下沉"与 wiki "使用轨迹沉淀为基础设施"是同一思想
- [[Readers/AIApps/AI 应用生态全景 2026]] — agent + tool use 在 2026 年 AI 生态的位置

---

*本读本由 [[Wiki/Entities/Claudian]] 基于 [[Wiki/Syntheses/AIApps/Ai-texture-tool-design]] 综合页 + 2026-04-24 与用户的设计讨论 + 当日 GitHub 生态 web 搜索生成,2026-04-24。*
