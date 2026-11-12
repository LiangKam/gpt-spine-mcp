# Alternatives / 同类项目

> 调研时间：2026-11-12。星标数、最近推送日期和许可证都取自 GitHub API（仓库的
> `license` 字段与 LICENSE/README 原文）。这些数字变化很快，引用前请重新核对。

## TL;DR

- 目前**没有**一个比本项目明显更成熟、且同样做"AI 驱动 PSD → Spine 全自动绑定 + 动画"的开源项目。
- 最常被拿来对比的两个方案，都不是同一件事：
  - **官方 PhotoshopToSpine**（`EsotericSoftware/spine-scripts`）：PSD → Spine 输入的事实标准，
    但只导出图层图片和位置/绘制顺序，不做骨骼绑定，也不做动画。本项目直接使用它的导出结果作为输入。
  - **DragonBones**：一套完整的免费骨骼动画生态（编辑器 + PS 插件 + MIT 运行时），但属于另一个格式生态，
    绑定和动画仍需在编辑器里手工完成。
- 2026 年出现了一批 AI/MCP 驱动的 Spine 小项目（见[第 3 节](#3-ai--mcp-驱动的-spine-项目直接竞品)），
  其中有几个与本项目同源（同一条 fork 链）。它们大多只有几周到几个月历史、基本没有社区使用，
  在真实资产上都还没有验证；按"成熟度"算，没有一个明显领先。
- 本项目的差异化：
  1. **语义推断**：`spine_semantics.py` 根据图层名/路径做角色分类（role、side、state、material、父子关系），
     支持自定义命名档案。
  2. **智能绑定档案**：`spine_smart_rig.infer_profile` 自动判定 `biped` / `quadruped` / `winged` / `prop`，
     据此生成骨骼、网格、权重和 IK。
  3. **MCP / Agent 集成**：`server.py` 用 FastMCP 暴露 19 个工具，另有 `openai_agent.py`
     （OpenAI Agents SDK 客户端）和 `gpt-spine` CLI。模型负责把美术需求写成数字化的 motion spec，
     确定性的工具负责落盘和校验（`apply_motion_spec` / `validate_motion` / `audit_preview`）。
  4. **无需 Spine 授权也能跑通**：运行时 JSON、图集、预览和校验都是纯 Python；只有导出 `.spine` 工程才需要正版 Spine CLI。

对照基准是本项目 README 中的自我定位：*"These are deterministic production starters, not an
art-director replacement."* 当前版本 v0.7.1；v1.0.0 计划在完成真实资产兼容性测试后发布。

## 1. 对比总表

图例：✅ 自动完成 · 🟡 部分 / 半自动 · ✋ 需在编辑器中手工完成 · ❌ 不支持

| 方案 | 输入 | 骨骼 | 网格 / 权重 | IK | 动画生成 | AI / MCP | 许可证 | 维护活跃度（最近推送） | 需要商业软件 |
|---|---|---|---|---|---|---|---|---|---|
| **gpt-spine-mcp（本项目）** | 分层 PSD、PhotoshopToSpine 导出 | ✅ 按语义和档案生成 | ✅ 四角网格 + 单骨自动权重 | ✅ 头部/身体 IK | ✅ 多状态起步动画、FX 包、motion spec 编译器 | ✅ FastMCP + OpenAI Agents SDK | MIT | 活跃 | 仅导出 `.spine` 时需要 Spine CLI |
| [PhotoshopToSpine](https://github.com/EsotericSoftware/spine-scripts/tree/master/photoshop)（官方脚本） | Photoshop 文档 | ❌（只有 root；图层标签可做少量结构） | ❌ | ❌ | ❌ | ❌ | BSD 风格（`photoshop/LICENSE`，© Esoteric Software） | 活跃（2026-09） | 需要 Photoshop；导入需要 Spine 编辑器 |
| Spine 编辑器内置 Import PSD | PSD | ✋ | ✋ | ✋ | ✋ | ❌ | 商业 | 活跃 | 需要 Spine |
| [DragonBones](https://github.com/DragonBones)（Pro 编辑器 + 运行时） | PS 插件导出、编辑器 | ✋ | ✋ | ✋ | ✋（手工） | ❌ | 运行时 MIT；编辑器免费但不开源 | 运行时仍有零星更新（JS 2026-01、C# 2026-05）；DesignPanel 最近 2021、Tools 最近 2022 | 编辑器免费 |
| [COA Tools](https://github.com/ndee85/coa_tools) / [COA Tools 2](https://github.com/Aodaruma/coa_tools2)（Blender） | PS / GIMP / Krita 导出脚本 | ✋（Blender 内） | 🟡 COA Tools 2 可自动生成网格；权重手工 | ✋ Blender IK | ✋ Blender 时间轴 | ❌ | GPL-3.0 | 原版最近 2024-05；COA Tools 2 最近 2026-09（已在 Blender 5.1 测试） | 否；但输出不是 Spine 格式 |
| [Live2D Cubism](https://www.live2d.com/) | PSD（编辑器原生导入） | ✋ 变形器 / 参数 | ✋ | — | ✋ | ❌ | 编辑器商业；SDK 为专有许可，年营收 ≥1000 万日元的企业需要 Release License | 活跃 | 需要 Cubism Editor |
| [Inochi2D](https://github.com/Inochi2D/inochi-creator)（开源 Live2D 替代） | PSD 导入（`source/creator/io/psd.d`） | ✋ | ✋ 网格变形 | — | ✋ | ❌ | BSD-2-Clause | SDK 2026-11；Creator 2025-06 | 否；但输出不是 Spine 格式 |

## 2. 各方案说明

### 2.1 官方 PhotoshopToSpine（`EsotericSoftware/spine-scripts`）

- 位置：脚本在 [`EsotericSoftware/spine-scripts`](https://github.com/EsotericSoftware/spine-scripts) 的
  `photoshop/PhotoshopToSpine.jsx`，**不在** `spine-runtimes` 仓库里。约 500 星，2026-09 仍有推送。
- 作用：把每个图层导出为 PNG，再生成一份保留位置和绘制顺序的 Spine JSON，用于"Import Data"。
  可以用图层名标签（如 `[bone]`、`[slot]`、`[skin]`、`[mesh]`）提前声明结构，但这些都是手工标注。
- 官方 README 写明：Spine 编辑器现已支持**直接导入 PSD**，速度最多快约 5 倍，而且一般更推荐用它。
- 许可证：`photoshop/LICENSE` 是 BSD 风格（© 2012-2017 Esoteric Software）。注意它和
  `spine-runtimes` 的 *Spine Runtimes License* 不同，后者要求使用者持有 Spine 编辑器授权。
- 与本项目的关系：本项目把它的导出（layout JSON + `images/`）当作标准输入之一，在此基础上做绑定、动画、图集和校验。

### 2.2 DragonBones（Egret）

- 定位：一套免费的 2D 骨骼动画方案，包括 DragonBones Pro 编辑器、PS/Flash 导出插件，以及多平台运行时
  （JS/TS、C#、C++、Java、AS3）。
- 许可证：运行时仓库为 MIT；官网仓库为 CC BY-NC-SA 4.0；GitHub 组织里没有编辑器源码，即编辑器"免费但不开源"。
- 现状：JS/C#/C++ 运行时仍有零星提交；`DesignPanel`（最近 2021）和格式转换工具 `Tools`（最近 2022）基本停更。
  组织里新出现了一个 `Loongbones-doc`（Web 编辑器文档）仓库，前景还不清楚。
- 与 Spine 的互通：`dragonbones-tools`（`2db` / `db2`）可以在 DragonBones 与 Spine JSON 之间转换，
  但版本较旧，对 Spine 4.x 的兼容性需要自行验证。
- 结论：生态完整，但绑定和动画全部靠人工，没有 AI/自动化层，也不产出原生 Spine 4.2 工程。

### 2.3 COA Tools / COA Tools 2（Blender）

- 定位：Blender 里的 2D cutout 绑定 / 动画套件，工作方式和 Spine/Spriter 类似。附带 Photoshop、GIMP
  （COA Tools 2 另有 Krita）的图层导出脚本。
- 原版 `ndee85/coa_tools`（约 970 星，GPL-3.0）最近推送是 2024-05；社区续作 `Aodaruma/coa_tools2`
  （GPL-3.0）面向 Blender 3.4+，README 说明最新版已在 Blender 5.1 测试，并新增了自动网格生成。
- 结论：骨骼、权重、IK、动画都在 Blender 里手工完成，输出是 Blender 场景或 JSON（原版附带 Godot 导入器），
  不是 Spine 工程。如果团队已经在用 Blender，这是零成本选项，但不能替代 Spine 流水线。

### 2.4 Live2D 与开源替代 Inochi2D

- **Live2D Cubism**：Cubism Editor 原生支持分层 PSD 导入，属于商业软件。Cubism SDK 的 Framework
  源码是公开的，但许可证是专有的（Live2D Open Software License / Proprietary Software License），
  年营收 ≥1000 万日元的企业还要签 *Cubism SDK Release License*。所以它**不算完全开源**。
  它的模型是"变形器 + 参数"，和 Spine 的骨骼 + 网格不是同一种范式。
- **Inochi2D**（BSD-2-Clause，SDK 约 1.9k 星、Creator 约 1.3k 星）：开源的 Live2D 式绑定工具，
  Inochi Creator 支持 PSD 导入。绑定过程是手工的，输出是自家的 Inochi2D 格式。
- **pixi-live2d-display**（MIT）这类项目只负责播放 Live2D 模型，不做导入或绑定。
- 结论：适合 VTuber 或立绘类角色，不适合 Spine 运行时流水线。

### 2.5 GitHub 上的其他 PSD → Spine 自动化项目

搜索关键词：`psd to spine`、`psd2spine`、`spine auto rig`、`photoshop spine export`、`psd spine json`、`spine mcp`。

| 项目 | 做什么 | 许可证 | 状态 |
|---|---|---|---|
| [Little-Z7/psd2spine](https://github.com/Little-Z7/psd2spine) | PSD → PNG + 带骨架的 `skeleton.json`；`smart` 模式结合 MediaPipe 姿态识别、图层名和 bbox 自动绑出人形骨架，可选网格 + 权重 | 未声明 | 4 星，2026-06 |
| [Anivia1337/psd-to-spine](https://github.com/Anivia1337/psd-to-spine) | 浏览器内 PSD → PNG + Spine 3.8 JSON，所有图层挂在 root 下（相当于 Web 版 PhotoshopToSpine） | 未声明 | 2026-10 |
| [NomadOSStudios/nomados-rig-animator](https://github.com/NomadOSStudios/nomados-rig-animator) | 本地浏览器 cutout 动画工具：读取 PSD，手工画骨骼和打关键帧，导出 Unity prefab 和 Spine 4.0 JSON | MIT | 1 星，2026-10 |
| [guiguiyan930-source/character-spine-layering](https://github.com/guiguiyan930-source/character-spine-layering) | 从单张立绘拆出分层 PSD，再生成 Spine 4.2 骨骼初稿和换图附件 | 未声明 | 1 星，2026-10 |
| [Jeffreynuez/ComfyUI-AutoSplit-Orchestrator](https://github.com/Jeffreynuez/ComfyUI-AutoSplit-Orchestrator) | 用 SAM 3 + 深度图把平面立绘切成带名称、有绘制顺序的部件，并导出 Spine 工程（拆图上游，不做动画） | MIT | 1 星，2026-10 |
| [hadashiA/psd2spine](https://github.com/hadashiA/psd2spine)、[quatmo/spine_export](https://github.com/quatmo/spine_export) | 早期 PSD/AE → Spine 导出脚本 | MIT / Apache-2.0 | 停更（2016 / 2014） |

这些项目要么只做图层导出（相当于 PhotoshopToSpine 的替代品），要么只做拆图或骨架初稿，
都没有覆盖"绑定 + 动画 + 校验 + Agent 编排"整条链路。

## 3. AI / MCP 驱动的 Spine 项目（直接竞品）

这一类和本项目最接近，评估替代方案时应重点看这里。

| 项目 | 定位 | 输入 | 绑定 / 动画 | 许可证 | 成熟度信号 |
|---|---|---|---|---|---|
| [GenielabsOpenSource/spine-animation-ai](https://github.com/GenielabsOpenSource/spine-animation-ai) | Claude Skill：用 SIFT + RANSAC 根据参考图定位部件，生成骨骼 JSON 和 idle/walk/run 等动画，输出 Web Player 预览 | 部件图 + 参考图（不是 PSD 流水线） | 骨骼 + 动画；README 没写网格、权重和 IK | **PolyForm Noncommercial 1.0**（商用需另外授权，不是 OSI 开源） | 约 670 星，2026-08 |
| [ampersante/spine2d-animation-mcp](https://github.com/ampersante/spine2d-animation-mcp) | MCP：PSD → Spine 4.2 骨骼、图集；本项目最早的上游灵感来源 | PSD | 只有骨骼（Phase 0），**还没有动画** | MIT | 约 130 星，README 自述尚未在真实 PSD 上验证 |
| [egorfedorov/spine-mcp](https://github.com/egorfedorov/spine-mcp) | 本项目的直接上游（v0.1.0 基础） | PSD / PhotoshopToSpine | 绑定 + 手写动画 | MIT（README 声明） | 11 星，2026-08 后未更新 |
| [kumikilongyeyo/CLAUDE-SPINE](https://github.com/kumikilongyeyo/CLAUDE-SPINE) | 同源分支（基于 egorfedorov/spine-mcp 重构）：轮廓网格、热扩散权重、IK、物理、2.5D 转身、FX，并在 `spine-core` 运行时中回放验证 | PSD | 绑定 + 动画 + FX | MIT（LICENSE 原文） | 0 星，创建于 2026-10，2026-11 仍有推送 |
| [kumikilongyeyo/ART-PIPELINE-MCP](https://github.com/kumikilongyeyo/ART-PIPELINE-MCP) | 同源分支，模型无关的 PSD → Spine → After Effects FX 流水线；README 中专门写了 "Why not GPT-SPINE?" 一节，把自己定位为本项目的单一后端继任者 | PSD | 自述包含网格、热权重、IK、物理、约 28 个 FX 配方、49 个工具、1169 个测试 | MIT（LICENSE 原文） | 0 星，v0.2.0，最近推送 2026-10 |
| [RevStudio/Rev2D](https://github.com/RevStudio/Rev2D) | 面向 AI Agent 的 Live2D/Spine 式绑定引擎：单个 JSON 描述整套绑定，确定性无头渲染，18 个 MCP 工具 | 自有 `.r2d.json`、单张立绘、Spine JSON 导入 | 骨骼、权重网格、IK、物理、动画；可导出到 Spine 运行时 | MIT | 40 星，创建于 2026-09，自述 2600+ 测试 |
| [zhoushengmin/spine-mcp-server](https://github.com/zhoushengmin/spine-mcp-server) | 70 个 MCP 工具，用来操作**现有** Spine 3.8.75 工程 | `.spine` 工程 | 编辑或生成动画（依赖编辑器） | 未声明 | 27 星 |
| [1425sd/spine-MCP](https://github.com/1425sd/spine-MCP) | 只包装官方 Spine CLI（导入、导出、打包），不做绑定 | `.spine` / JSON | ❌ | Apache-2.0 | 25 星，仅 Windows |
| [SAM-AEL/spine-studio](https://github.com/SAM-AEL/spine-studio) | 浏览器端 Spine 3.8 查看器 + 程序化自动绑定 / 5 个动画 | 精灵部件 / sprite sheet | 自动骨骼 + 程序化动画 | MIT / Spine 条款 | 0 星 |

解读：

- **GenielabsOpenSource/spine-animation-ai** 社区关注度最高，但许可证禁止商用，输入也不是分层 PSD
  流水线，没有 MCP 服务器，也没有网格、权重和 IK。不能作为本项目的直接替代。
- **ampersante/spine2d-animation-mcp** 只做到骨骼，没有动画，功能上被本项目覆盖。
- **CLAUDE-SPINE / ART-PIPELINE-MCP** 和本项目同源，宣称的功能面更广（热权重、物理、AE 桥接），
  但都没有社区采用记录，也都还没有真实资产验证。如果需要物理、After Effects FX 或模型无关部署，
  建议拿同一批真实 PSD 对比测试后再决定，不要只看 README。
- **Rev2D** 是设计最完整的"Agent 原生"绑定引擎（MIT，测试多），但它的核心是自有格式，
  Spine 只是导入/导出目标，而且不从分层 PSD 出发。适合愿意换掉 Spine 编辑器工作流的团队。

## 4. 结论

1. **按"PSD → Spine 4.2 全自动绑定 + 动画 + Agent 编排"这个标准衡量，没有比本项目明显更成熟的开源选择。**
   - 官方 PhotoshopToSpine：最稳定、最标准，但只做图层导出，不做绑定和动画。
   - DragonBones：生态完整、免费，但属于另一个格式生态，靠手工制作，主编辑器不开源，周边工具基本停更。
   - COA Tools、Inochi2D、Live2D：都能用 PSD 做 2D 绑定，但都是手工流程，输出也不是 Spine。
2. AI/MCP 方向的同类项目都很新（2025-2026 年创建），绝大多数没有真实生产验证。
   在成熟度上，本项目与同源分支 CLAUDE-SPINE / ART-PIPELINE-MCP 大致处在同一阶段。
3. 本项目的差异化：
   - **语义推断**：`spine_semantics` 的 role、side、state、material 分类，可自定义命名档案，`inspect` 时报告语义猜测。
   - **智能绑定档案**：`biped` / `quadruped` / `winged` / `prop` 自动判定，并据此生成骨骼、网格、权重、IK 和主运动。
   - **MCP / Agent 集成**：FastMCP 工具集、OpenAI Agents SDK 客户端、motion spec 编译器加 `validate_motion` /
     `audit_preview` 校验，把模型创意和确定性落盘分开。
   - **无授权可运行**：纯 Python 产出运行时 JSON、图集和预览；Spine CLI 只在导出 `.spine` 时需要。
4. 局限（与 README 一致）：输出是确定性的生产起点，不能替代美术指导；关键资产的网格拓扑、权重、约束目标和裁剪形状仍需人工复核。v1.0.0 要等真实资产兼容性测试完成后才发布。
