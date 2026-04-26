---
type: synthesis
created: 2026-04-26
updated: 2026-04-26
tags: [aiagents, desktop-pet, team-distribution, config-layering, flipbook, vfx]
sources: 1
aliases: [桌宠团队版, 团队分发架构, Desktop Pet Team Distribution]
---

# 桌宠的"团队分发版"架构

> 本页是 [[Readers/AIAgents/桌宠 AI 入口的从零方案]] 的延伸专题——**VFX 团队场景**特有的三件事:Flipbook 特效 / 团队配置同步 / 设置页 GUI。原读本默认"个人 pet",本页处理"团队 mascot"第二人设。

## 0. 两个人设的本质差别

| | 个人版(原读本) | 团队版(本页) |
|---|---|---|
| 用户 | 一个人(自己) | N 个 VFX 同事 |
| 配置 | JSON 文件直接编辑 | GUI 必须 + 集中管理 |
| MCP server | 自己想加什么加什么 | admin 维护清单,users 跟着用 |
| 特效形象 | Live2D 一只就够 | 需要团队产能"长期供应" → Flipbook |
| 维护 | 自己改自己用 | 改一处影响 N 人,要审、要 lock、要 revert |

> [!abstract]
> 个人版是**单进程单用户工具**;团队版是**有 admin 和 user 角色的内部分发软件**。后者的工程量在配置层、不在 UI 层。

---

## 1. Flipbook 特效:VFX 团队的杀手级功能

### 为什么是它

VFX 团队的工作就是产 sprite sheet / sub-UV / Niagara render。让桌宠**消费团队自产特效**——

```
Live2D 角色  =  "身体"(rig 驱动,常驻,会眨眼会看)
Flipbook    =  "特效环绕"(事件触发,瞬态,花瓣/光柱/魔法阵)
```

这正是 Niagara 的"Material vs Particle"分工——**团队最熟悉的语义**。桌宠从"个人玩具"升级成"**团队产能展示舞台**":每个人都能扔自己的特效进去。

### 技术零成本

`pixi-live2d-display` 底层是 PixiJS。PixiJS 自带 `AnimatedSprite` —— sprite sheet flipbook **天生支持**:

```typescript
import { Spritesheet, AnimatedSprite, BaseTexture } from "pixi.js";
const sheet = new Spritesheet(BaseTexture.from("magic_burst.png"), atlasJson);
await sheet.parse();
const burst = new AnimatedSprite(sheet.animations.magic);
burst.animationSpeed = 0.5;
burst.loop = false;
burst.play();
stage.addChild(burst);  // 跟 Live2D 角色同舞台
```

**0 新依赖**。Live2D 在的地方 Flipbook 自动在。

### 触发系统

| 触发源 | 例子 | 引入时机 |
|---|---|---|
| **手动**(右键菜单 / 设置页按钮) | "撒花" / "施法" | P1.5(无需 LLM) |
| **定时 / 闲置** | 每整点 ambient sparkle / idle 30s 转头 | P1.5 |
| **用户事件** | 系统通知到达 → 头顶冒💡 | P2 后 |
| **LLM 触发** | Claude 调 tool 时 → 思考魔法阵;说"完成"时 → 烟花 | **P3+**(需 Hub 事件总线) |

### 资产格式 & 团队工作流

约定 sprite sheet + atlas JSON 格式(PixiJS 标准 / TexturePacker JSON),约定目录:

```
config/effects/
├── magic_burst/
│   ├── sheet.png
│   └── atlas.json
├── petal_swirl/
│   ├── sheet.png
│   └── atlas.json
└── manifest.json   # 哪些 pack 启用 / trigger 映射
```

**任何团队成员**都能产出新特效:
- TA / 工艺员:Houdini / AE / Niagara 渲染 → 烘到 sprite sheet → 扔目录
- UI 美术:绘 sprite sheet + 写 atlas
- 程序:写新触发规则到 manifest

> [!tip] 这是"个人项目 → 团队 IP"的关键一跃
> 没有 Flipbook 的桌宠是"你的"——团队成员看一眼觉得不错但不会有归属感。有 Flipbook 之后,**它能消费每个人的产出**——同事丢一个 sprite sheet 就能进 share build,这种贡献门槛低到所有人都能进来。心理学杠杆比技术杠杆大。

---

## 2. 团队配置同步:三层 + git repo source

### 配置层模型(借鉴 VS Code / Git / 大多成熟工具)

```
优先级:低 → 高(高的覆盖低的)
─────────────────────────────────────────────
Layer 1:  App defaults     (随安装包发,代码硬编码兜底)
Layer 2:  Team config      (admin 维护,git repo 同步)        ← 新增层
Layer 3:  User local       (个人偏好;能 override layer 2 除非 lock)
Layer 4:  Runtime overrides (设置页改一下立刻生效,持久化到 layer 3)
```

每个 key admin 可以 mark `"locked": true`(用户不可改)或默认(用户可改)。

### Team config source 四选项对比

| 选项 | 怎么用 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|
| **A. 共享网络盘**(`\\studio\tools\desktop-pet\config\`) | 桌宠启动时读 UNC 路径 | 简单粗暴 | 无版本 / 无审计 / 内网外失效 / 容易踩坑(多人写同一文件) | 公司有强内网,小团队凑合 |
| **B. Git repo**(私有 GitLab/GitHub) ⭐ | 桌宠定期 `git pull` 一个 config repo | **TA 友好(团队天然爱 git);PR 走代码审;有 diff/blame/revert;离线可用** | 桌宠端要内置 git 调用 | TA 团队的最佳选 |
| **C. HTTPS endpoint** | admin 挂 JSON,桌宠 fetch | 灵活 | 要维护 server / 鉴权 / 缓存 | 有专门 SRE/IT 时 |
| **D. 现成同步**(Notion / 飞书多维表 API) | 通过 API 拉 | admin 已熟 GUI | 重 / 限流 / 协议依赖第三方 | admin 不会用 git 时 |

**本议题选 B** —— 团队是 TA,git 是母语;PR 工作流让"加一个 agent"变成可审可 revert 的代码事件,出问题能追溯。

### 实现轮廓(B 方案)

桌宠仓库自带一个 **team config sync 模块**:

```typescript
// hub/config/team-sync.ts
import { simpleGit } from "simple-git";
const TEAM_REPO_LOCAL = path.join(userDataDir, "team-config");

export async function syncTeamConfig(repoUrl: string) {
  if (!fs.existsSync(TEAM_REPO_LOCAL)) {
    await simpleGit().clone(repoUrl, TEAM_REPO_LOCAL);
  } else {
    await simpleGit(TEAM_REPO_LOCAL).pull();
  }
  return loadConfigFrom(TEAM_REPO_LOCAL);  // 解析为 Layer 2
}
```

启动时 sync 一次 + 设置页"立刻同步"按钮 + 后台每小时静默 pull。失败时降级到上次缓存。

### Team config repo 长什么样

`git@studio:vfx-team/desktop-pet-config.git`:

```
desktop-pet-config/
├── agents.json          # MCP server 清单(团队默认)
├── effects/             # 团队特效资产(用 Git LFS)
│   ├── magic_burst/
│   └── petal_swirl/
├── settings-defaults.json  # 默认设置 + 哪些字段 lock
├── README.md            # admin 操作手册
└── CHANGELOG.md         # 每次 PR 标注影响范围
```

每加一个 agent / 特效 = 一个 PR。Lead approve 后 merge,半小时内全团队桌宠静默更新。

---

## 3. Admin / User 权限模型

### Lock 机制(per-key)

```jsonc
// settings-defaults.json (Layer 2)
{
  "agents": {
    "value": [
      { "name": "houdini-mcp", "command": "...", "locked": true },     // 用户不可禁用
      { "name": "shotgun-mcp", "command": "...", "locked": false }     // 用户可禁用
    ]
  },
  "character": {
    "live2dModel": { "value": "hiyori", "locked": false },             // 用户可换皮肤
    "scale": { "value": 0.4, "locked": false }
  },
  "effects": {
    "enabled": { "value": ["magic_burst", "petal_swirl"], "locked": true }  // 团队统一特效集
  },
  "hotkeys": {
    "summon": { "value": "Ctrl+Space", "locked": false }
  }
}
```

设置页 UI:
- **Lock 项**:显示"由团队管理 🔒" + 灰显不可改
- **未 lock 项**:用户改了存到 Layer 3(`user-settings.json`)
- **Reset to team default** 按钮:清除 Layer 3 该 key,回退到 Layer 2

### Admin 是谁

议题选 **几个 lead/TA**(不是单 admin,也不是全员):

- 1 人:简单但 bus factor = 1,放假就没人改
- 几个 lead:**推荐**;有 PR 互审;团队规模 5-30 人最优
- 全员可改:扁平但容易乱;只在<5 人非正式团队适用

实现上,**桌宠端不区分 admin**——admin 身份只体现在"谁有 git repo 写权限"。这是 git 自带的权限模型,我们什么都不用做。

---

## 4. 设置页(从 P4 提前到 P0.5)

团队成员不可能编 JSON。设置页**必须**前置。

### 结构

```
设置页(`/settings` 路由,Vue Router)
├── 角色      —— Live2D 模型 / 缩放 / 位置 / 透明度
├── 特效      —— Flipbook pack 启用 / 触发规则        [P1.5 后启用]
├── Agents    —— MCP server 增删 / 启用                [P3 后启用]
├── 快捷键    —— Ctrl+Space 等
├── 团队      —— team config repo URL / 上次同步时间 / 立刻同步 / "我是 admin 模式"开关
└── 关于      —— 版本 / 更新通道 / 检查更新
```

### Admin 模式(可选,加在"团队"tab)

打开后多出来:
- "导出当前设置为 team config"按钮——简化 admin 工作流(不用手写 JSON)
- "克隆 team-config 仓库"按钮——快速进入 admin 工作流

普通用户看不到这些按钮(基于"我是 admin 模式"开关切换;不是真正的权限控制,只是 UI affordance,真正的权限在 git 上)。

---

## 5. 调整后的阶段路线(全局视角)

```
原计划:  P0 壳 → P1 对话 → P2 MCP → P3 自定义 server → P4 记忆+设置 → P5+
团队版:  P0 壳 + P0.5 配置层 → P1 对话+基本设置页 → P1.5 Flipbook
        → P2 MCP → P3 自定义 server + LLM-trigger Flipbook
        → P4 记忆 → P4.5 Team config sync → P5 distribution
```

| 新增/调整阶段 | 内容 | 工时 |
|---|---|---|
| **P0.5**(原 P4 部分提前) | 配置三层架构 + 简单设置页(角色 + 快捷键 2 tab) | +1-2 天 |
| **P1.5**(新) | Flipbook 渲染层 + 手动触发 + 资产目录约定 | +2-3 天 |
| **P3** 扩展 | 让 LLM tool_call 触发 Flipbook(事件总线) | +0.5 天 |
| **P4.5**(新) | Team config git sync + admin 模式 + lock 机制 | +3-5 天 |
| **P5 distribution** | 安装包 + auto-update + 部署文档 | +2-3 天 |

整体工期从原"1-2 月 v0.1"延到 **2-3 月 v0.1**。**值得**——没有 P0.5/P4.5,后面所有 setting 改起来都疼,而且团队没法用。

---

## 6. 关键开放问题

- **Git LFS 还是直接 binary**:effects/ 下 sprite sheet 可能 MB 级,**Git LFS 是必须**——但配 LFS 增加 admin 门槛。备选:effects 单独走 HTTPS endpoint,只有 settings JSON 走 git。
- **离线场景**:同事在家用 VPN 连不上 repo 怎么办——上次 sync 的 cache 兜底是必要的。需明确"离线多久后强制提示"。
- **冲突处理**:user override 一个项后,admin 把它 lock 了,怎么办?推荐:UI 显示"团队已 lock,你的本地修改已隐藏";不破坏用户数据,只是不生效。
- **特效预览**:设置页"特效"tab 应能 preview——这要求特效资产能脱离桌宠主舞台单独播放。组件抽象上要预留。
- **多 vault / 多用户场景**:notes-server 的 VAULT_ROOT env 在团队版中是不是 user-local(每人指自己 vault)?**几乎肯定是**——团队成员各自 vault,不能 lock。

---

## 7. 工程影响:"个人版"还存在吗?

值得明确:

- **个人版**仍然成立,作为"开箱即跑"的 default:不配 team repo URL 时,Layer 2 为空,行为退化为原读本计划
- **配置 schema 兼容**:个人版的 user-settings.json 跟团队版的 team-defaults.json 是同一 schema,只是不带 `locked` 字段
- **同一份代码两种部署**:程序员个人玩 = 个人版;装公司 build = 团队版。**不要分两套代码库**

这是"配置层架构正确"的复利体现——同一栈,通过配置启停团队特性。

---

## 相关

- [[Wiki/Concepts/AIAgents/Desktop-pet-as-ai-hub]] — 议题概念(本页是其团队场景的延伸)
- [[Wiki/Syntheses/AIAgents/Desktop-pet-stack-comparison]] — 选型矩阵
- [[Wiki/Syntheses/AIAgents/Mcp-host-implementation]] — MCP host 实施手册
- [[Readers/AIAgents/桌宠 AI 入口的从零方案]] — 主题读本(P0-P5 详细路线)
- [[Wiki/Concepts/AIAgents/Generative-agents-architecture]] — agent 长期运行的架构原则,与 team config 同步的"持续可用"目标共鸣

## 深入阅读(未来若展开)

- VS Code Settings 分层模型(default / workspace / user)的实践教训
- Git-based config 在内部工具分发上的成熟先例(Atlas / Backstage / Crossplane)
- Sprite sheet 烘焙最佳实践(Houdini → SubUV → JSON atlas 流水线)
