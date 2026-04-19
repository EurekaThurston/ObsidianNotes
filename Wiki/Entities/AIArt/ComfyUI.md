---
type: entity
created: 2026-04-19
updated: 2026-04-19
tags: [tool, inference, pipeline, comfyui]
sources: 1
aliases: [ComfyUI, Comfy]
---

# ComfyUI

> 节点式扩散模型推理 UI，**工程化最强**的 Stable Diffusion 前端。对生产管线（含飞书机器人对接）是**首选部署工具**。

---

## 概览

- **类型**：Diffusion 模型推理 / 管线编排工具
- **交互范式**：节点图（类似 Substance Designer / Blender Compositor）
- **仓库**：`github.com/comfyanonymous/ComfyUI`
- **特点**：每个节点是一个操作（load model、encode prompt、sample、save），连起来形成 workflow

---

## 关键事实 / 属性

| 属性 | 值 |
|---|---|
| 推荐度（生产管线） | ★★★★★ |
| 推荐度（探索/原型） | ★★★（新手节点图有门槛） |
| API 模式 | ✅ 完整支持（提交 workflow JSON） |
| 多 LoRA 挂载 | ✅ 任意数量，各自独立 strength |
| 插件生态 | 非常活跃 |
| 开源许可 | GPLv3 |

---

## 为什么对生产管线是首选

1. **Workflow 即代码**：所有管线配置是 JSON，可版本化、可审查、可复用
2. **API mode 强**：HTTP 提交 workflow → 返回图片，天然适合后端集成
3. **多 LoRA 挂载**：实现 [[Wiki/Concepts/AIArt/Multi-lora-composition]] 的必备平台
4. **扩展性**：ControlNet / IP-Adapter / 各种自定义节点应有尽有
5. **无商业许可限制**：自己部署，数据不出公司

---

## 典型 Workflow 节点栈（风格 LoRA 生产）

```
CheckpointLoader (加载基座，如 Illustrious XL)
  ↓
LoraLoader (挂 wwstyle_v1)
  ↓
LoraLoader (挂角色 LoRA，可选)
  ↓
CLIPTextEncode (正向 prompt)
  ↓
CLIPTextEncode (负向 prompt)
  ↓
KSampler (生成)
  ↓
VAEDecode (解码)
  ↓
SaveImage / PreviewImage
```

---

## 飞书机器人后端集成示例

```python
def handle_tag_selection(tag_combo):
    prompt = assemble_prompt(tag_combo)
    prompt = f"wwstyle_v1, {prompt}"  # 加触发词

    workflow = load_workflow_template("ww_style_v1.json")
    workflow["6"]["inputs"]["text"] = prompt
    workflow["10"]["inputs"]["lora_name"] = "wwstyle_v1.safetensors"
    workflow["10"]["inputs"]["strength_model"] = 0.8

    prompt_id = comfyui_client.queue_prompt(workflow)
    images = comfyui_client.wait_for_images(prompt_id)

    send_feishu_card(images)
```

架构：**飞书前端（tag 选择器）→ 后端路由 → ComfyUI API → 返回图**。

---

## 和同类工具对比

| 工具 | 定位 | 推荐度 |
|---|---|---|
| **ComfyUI** | 生产管线、工程化 | ★★★★★ |
| Forge | A1111 的性能优化分支 | ★★★★ |
| Automatic1111 webui | 老牌，插件多但迭代慢 | ★★★ |
| InvokeAI | 美术友好 UI，商用许可清晰 | ★★★ |

**对鸣潮管线**：ComfyUI 是唯一合理选择，因为要做飞书后端集成，workflow-as-code 必须。

---

## 推理 VRAM 要求

- SDXL + LoRA：8GB 可跑，12GB 舒适
- Flux + LoRA：24GB 起步

---

## 相关

- [[Wiki/Concepts/AIArt/Multi-lora-composition]] — ComfyUI 的真正威力
- [[Wiki/Concepts/AIArt/Lora]] — 挂载对象
- [[Wiki/Entities/AIArt/Kohya-ss]] — 上游训练工具
- [[Wiki/Entities/AIArt/Illustrious-XL]] / [[Wiki/Entities/AIArt/NoobAI-XL]] / [[Wiki/Entities/AIArt/Flux]] — 支持的基座

## 引用来源

- [[Wiki/Sources/AIArt/Lora-deep-dive]]（raw: [[Raw/Notes/Lora_Deep_Dive]]）

## 开放问题 / 待深入

- **Workflow 版本管理**：多人协作时，workflow JSON 如何 diff / review？
- **热重载 LoRA**：换 LoRA 文件不重启服务的最佳实践
- **性能优化**：SDXL + 多 LoRA 在 24GB 卡上单图延迟的优化空间
- **ControlNet + LoRA** 组合的标准 workflow 模板
