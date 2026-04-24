---
type: synthesis
created: 2026-04-24
updated: 2026-04-24
tags: [ai, agent, texture, vfx, artist-tooling, landing-design, tool-use]
sources: 1
aliases: [AI 贴图工具设计, Texture Tool Design, 特效贴图 AI 工具]
---

# AI 特效贴图处理工具 — 项目级落地设计

> 一份面向 VFX / TA 场景的 AI 工具设计:美术自然语言描述需求(缩放、去模糊、四方连续、通道打包、生成 variation 等),AI 临场选工具 / 写代码完成处理。
>
> 本方案是 [[Wiki/Concepts/AIFoundations/Ai-agent|AI Agent]] + Tool Use + Code Interpreter 在 VFX 贴图生产场景的具体应用,与 [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot]] 并列,都属于"美术向 AI 落地"模式族。

## 问题定义

**现状**:

- 美术对贴图的处理需求千奇百怪:缩放、去模糊、四方连续、通道打包、去 seam、生成 variation、色彩空间转换、flipbook 切分、atlas 拼接 ...
- 需求长尾:每次"出一个新需求",让 TA / 程序写一个新功能不 scale
- 现有工具(Photoshop / Substance / 内部工具)或门槛高、或覆盖不全、或缺少"语义级"能力(比如"把这张 512 云雾变清晰再 4 倍到 2048 且四方连续")

**目标**:一个 AI 驱动的贴图处理工具,美术自然语言描述需求,AI:

1. 理解需求 + 澄清歧义
2. 选合适的已有工具 / 模型 / 算法(基础操作)
3. 现场写 Python 代码兜底(长尾需求)
4. 出中间预览,允许"回到上一步改参数"
5. 处理完还能顺便标注风险(色彩空间错了、通道语义错了、接缝残留等)

## 核心机制

### 三层架构(A + B 混合)

```
用户自然语言 + 输入贴图
   ↓
[LLM 规划] 理解需求 → 澄清问题(必要时)→ 选工具链
   ↓
工具层(三级):
  L1 原子工具(高频、稳定)
    - resize / crop / rotate / channel_pack
    - color space convert / gamma
    - flipbook split/pack / atlas pack
  L2 模型工具(AI 能力)
    - Real-ESRGAN / SwinIR (超分 / 去模糊)
    - Stable Diffusion img2img / inpaint (语义级编辑)
    - tileable 生成(SD + asymmetric tiling)
  L3 代码沙箱(长尾兜底)
    - LLM 现场写 Python (PIL / OpenCV / numpy / scipy)
    - 沙箱执行,失败→看 error→修代码→再跑
   ↓
每步产中间图 + diff 预览 + 历史
   ↓
美术可回溯、改参数、分支
```

**关键设计原则**:

1. **LLM 负责路由,不负责像素**:LLM 决定"用 Real-ESRGAN 还是 SwinIR"、"先 resize 还是先 tileable",实际像素操作交给确定性工具
2. **高频入 L1/L2,长尾入 L3**:不要试图把"所有美术需求"做成原子工具,那是程序员视角的工程灾难
3. **贴图数据语义感知**:distortion map / flow map / channel-packed mask / HDR / sRGB vs Linear —— LLM 必须在 system prompt 里被教会"不是所有贴图都是照片"
4. **强制预览 + tile 验证**:做完四方连续必须 2×2 平铺显示;改完必须 before/after diff

### VFX 特有注意点(写进 system prompt)

| 陷阱 | 表现 | 工具侧对策 |
|---|---|---|
| 通道语义 | RGBA 不一定是颜色(xy 偏移 / flow / mask) | 每张图标注 `data_texture: true/false` 开关 |
| 位深 | 8-bit pipeline 碾平 16-bit / .exr / .hdr | 保持原位深直到显式要求 |
| Alpha 预乘 | 预乘 vs 直乘处理前要明确 | 工具显式接受 `premult: bool` 参数 |
| 色彩空间 | 游戏贴图多 Linear,AI 超分模型默认 sRGB | Linear → sRGB → 模型 → sRGB → Linear |
| Tile 可见度 | 单张看不出接缝 | 强制 2×2 预览 |

## 现有 GitHub 项目调研(2026-04)

搜索结论:**"LLM agent + 图像编辑" 生态已经相当成熟**,组件齐全可拼;**但"VFX 专用贴图工具"没有完全对口的现成品**,我们要做的是把已有组件按 VFX 场景拼装 + 加上 VFX 特有的 guardrail。

### A. LLM 图像编辑 agent(架构可直接借鉴)

| 项目 | 特点 | 参考价值 |
|---|---|---|
| [GenArtist](https://github.com/zhenyuw16/GenArtist) (NeurIPS 2024 Spotlight) | MLLM 做 agent,把复杂任务**分解成子任务树** + 自验证 + 自纠错 | ⭐ **架构范本**——分解→执行→自检的三段式直接对应我们的规划器 |
| [AgentLego](https://github.com/InternLM/agentlego) | LLM 视觉工具 API 库,含生成/编辑/感知多类工具,接 LangChain / Lagent | ⭐ 可直接当 L2 模型工具层的基础设施 |
| [imgflw](https://github.com/ototadana/imgflw) | LLM 驱动的图像编辑 demo 应用 | 入门参考,看它怎么串 UI ↔ LLM ↔ 工具 |

### B. Adobe / Photoshop MCP(如果想接 PS 生态)

| 项目 | 特点 |
|---|---|
| [adb-mcp](https://github.com/mikechambers/adb-mcp) | Adobe 官方背景的 PoC,通过 MCP 让 AI 控制 Photoshop / Premiere;已验证 Claude Desktop + OpenAI Agent SDK |
| [Photoshop MCP server](https://playbooks.com/mcp/photoshop) | 让 LLM 通过 MCP 调 PS 能力 |

**评估**:如果美术工作流深度依赖 PS,可以考虑以 PS 为执行后端。**但 VFX 场景很多需求 PS 不擅长**(通道语义、.exr、tileable 验证等),不建议完全绑定。

### C. ComfyUI + LLM(很可能是最省力的基座)

ComfyUI 已经是 VFX / 概念美术圈的事实标准,节点覆盖极广。把 LLM 架在 ComfyUI 上有多个成熟项目:

| 项目 | 特点 | 推荐度 |
|---|---|---|
| [twwch/comfyui-workflow-skill](https://github.com/twwch/comfyui-workflow-skill) | **自然语言 → ComfyUI workflow JSON**;34 内置模板 + 360+ 节点定义;**作为 Claude Code / Cursor 的 skill 工作** | ⭐⭐⭐ 跟我们方法论(Claude 生态 + agentic)最合拍 |
| [DanielPFlorian/ComfyUI-WorkflowGenerator](https://github.com/DanielPFlorian/ComfyUI-WorkflowGenerator) | 专门 LLM 微调版,三阶段 pipeline 生成 workflow 图 | 架构参考 |
| [heshengtao/comfyui_LLM_party](https://github.com/heshengtao/comfyui_LLM_party) | ComfyUI 内的 LLM agent 框架,含 MCP / 多 LLM 适配 | 生态整合型 |

**评估**:**ComfyUI 很可能是正解**——我们只需要在它上面加一层"VFX 贴图专属 skill",把通道语义 / 色彩空间 / tile 验证等 guardrail 注入 LLM 的 system prompt 即可。大部分节点(超分 / SD / 色彩处理 / tile)都已存在。

### D. 四方连续专项

| 项目 | 方法 | 适用 |
|---|---|---|
| [sagieppel/convert-image-into-seamless-tileable-texture](https://github.com/sagieppel/convert-image-into-seamless-tileable-texture) | 提供**经典 average-blending + SD inpaint** 两种方法 | 参考实现直接能抄 |
| [camenduru/seamless](https://github.com/camenduru/seamless) | SD seamless texture 生成器 | 生成式四方连续 |
| [brick2face/seamless-tile-inpainting](https://github.com/brick2face/seamless-tile-inpainting) | A1111 插件,SD inpaint 做 seamless | 成熟 |
| [Tiled Diffusion (CVPR 2025)](https://github.com/madaror/tiled-diffusion) | 扩散模型原生支持 tiling | 前沿,效果最好 |
| [carson-katri/dream-textures](https://github.com/carson-katri/dream-textures) | Blender 内 SD + seamless 选项 | 如果工作流在 Blender 里 |

**评估**:四方连续这一单点需求已有至少 5 种不同成熟方案,按图类型(规则 / 有机 / 生成式)分别挑合适方法。可以直接把 `sagieppel/convert-image-into-seamless-tileable-texture` 当 baseline 包成工具。

### E. 其他参考

- [Yuan-ManX/ai-game-devtools](https://github.com/Yuan-ManX/ai-game-devtools) — AI 游戏开发工具汇总(含 Texture / Shader / 3D 专题),**值得 bookmark 定期翻**

## 技术选型推荐

**最小可行版(周末级)**:
- 基座:**ComfyUI + twwch/comfyui-workflow-skill**(Claude 生态原生)
- UI:ComfyUI 自带 + 自然语言输入框
- L1 原子工具:ComfyUI 已有节点
- L2 模型工具:Real-ESRGAN / SD inpaint 节点已在
- L3 代码沙箱:ComfyUI 的 Python 节点 + 自定义节点
- Guardrail:VFX 专属 system prompt(见 §核心机制)

**生产级(一个月级)**:
- 基座自己搭:Gradio / Electron + Claude API (tool use)
- 深度绑定项目(能读项目资产命名规范、导出约定、TA 自定义 shader)
- 接入内部素材库 / 版本系统(P4)

## POC 最小路径

借鉴 [[Wiki/Syntheses/AIAgents/Artist-code-qa-bot#POC 最小路径|Artist-code-qa-bot 的 POC 方法]]:

1. **单点突破**:选一个高频痛点,优先级建议 **"四方连续 + 超分"组合**(需求高、现成组件全、效果直观)
2. **小范围试点**:3-5 个美术,两周
3. **四指标评估**:
   - 采纳率(AI 给的结果美术直接用的比例)
   - 失败率(需要回退到手工的比例)
   - 节省时间(相比原工作流)
   - 沉淀产出(哪些需求高频,值得做成 L1 原子工具)
4. **迭代**:高频需求下沉到 L1;长尾保持 L3 代码兜底

## 与 Artist-code-qa-bot 的对照

| 维度 | Code QA Bot | Texture Tool |
|---|---|---|
| 服务对象 | 美术 / TA | 美术 / TA |
| AI 角色 | 只读 + 解释 | 读写 + 生产 |
| 基础设施 | Agentic grep(无索引) | ComfyUI / Tool use + 代码沙箱 |
| 风险 | 幻觉 → 错误解释 | 幻觉 → 破坏原资产 |
| 沉淀 | Wiki 术语对照表 | 高频工具下沉 L1 + prompt 库 |
| 难点 | 自定义 symbol 术语桥梁 | VFX 数据语义 guardrail |

两者同构:**LLM 做路由,专业基础设施做执行,人类+wiki 做长期记忆**。

## 风险 & 坑

1. **破坏原资产**:AI 处理必须产副本,原图只读。版本化保存中间结果
2. **幻觉式"改好了"**:LLM 可能声称处理成功但像素没动。每步**强制像素级 diff 校验**
3. **色彩空间静默错误**:最隐蔽也最常见。所有工具签名显式接受 color space,不默认
4. **模型选型幻觉**:LLM 可能说"用 Real-ESRGAN 最好",但对法线贴图 / flow map 会毁掉数据。**data texture 禁用所有 AI 超分模型**,只许用 bicubic / nearest
5. **工程债**:长尾 L3 代码累积多了会变成屎山。定期 lint + 高频下沉 L1

## 留下的问题

- VFX 数据贴图(flow map / distortion / packed mask)AI 超分是否有任何可用方案?当前答案是"禁用",但可能有 data-aware 超分的研究
- 和项目的 material / Niagara 工作流怎么衔接?能不能做到"选中某 Niagara 发射器用的贴图,一键优化"
- 批处理 vs 交互:每次生产需要 100+ 贴图批处理时,如何让 AI 规划一次、批量跑、人类抽检?
- ComfyUI 作基座 vs 自建 UI:长期看哪个更适合项目组内部工具(维护成本 / 可控性)

## 下一步

1. **验证 ComfyUI + workflow-skill 的真实能力**:用"512 云雾 → 去模糊 + 4× 超分 + 四方连续 → 2048"作 benchmark 测一遍
2. **如果基座可用**:列 5-10 个高频 VFX 需求,按 L1/L2/L3 归类,写 VFX guardrail prompt
3. **如果基座不够**:评估自建最小 Gradio + Claude tool use 的投入产出
4. **定期回看** §现有项目调研(见 [[log]] 2026-04-24 条目),生态变化快

---

_来源:2026-04-24 与用户的设计讨论(§问题定义、§核心机制源于对话),§现有项目调研部分基于当日 web 搜索(见文中 GitHub 链接)_
