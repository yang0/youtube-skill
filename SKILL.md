---
name: youtube-skill
description: YouTube 操作总入口。以 intent-identification 为入口澄清意图，协调 youtube-downloader、youtube-search、use-notebooklm、youtube-to-pdf 四个子 skill 执行。
tier: router
---

# YouTube Skill

YouTube 相关操作的统一编排入口。**intent-identification 是入口**，不直接执行搜索/下载/分析。

```
用户请求 → intent-identification（澄清意图）→ intent_contract_v1 → 路由到子 skill
```

## 子 Skill 矩阵

| 子 Skill | 仓库 | 职责 | 调用时机 |
|----------|------|------|---------|
| **intent-identification** | [yang0/intent-identification](https://github.com/yang0/intent-identification) | 入口：澄清意图，产出 intent_contract_v1 | 每次请求最先调用 |
| **youtube-search** | [yang0/youtube-search](https://github.com/yang0/youtube-search) | 搜索视频、提取评论、下载头像 | 合同明确为搜索类任务 |
| **youtube-downloader** | [yang0/youtube-downloader](https://github.com/yang0/youtube-downloader) | 下载视频（解决认证墙） | 合同明确为下载任务 |
| **use-notebooklm** | [yang0/use-notebooklm](https://github.com/yang0/use-notebooklm) | NotebookLM 深度分析视频内容 | 合同明确为分析类任务 |
| **youtube-to-pdf** | [yang0/youtube-to-pdf](https://github.com/yang0/youtube-to-pdf) | 视频→PDF教程（NotebookLM提取+md-to-pdf渲染） | 合同明确为教程/文档产出 |

## 执行流程

### ⛔ 硬性关卡

**0. 认证检查（每次 YouTube 操作前）**
- 所有 yt-dlp 操作（搜索/下载/评论）都依赖有效 cookies
- 如果上次提取 cookies 超过 24 小时，或任何操作报认证错误 → 先执行 Quick Auth Recovery
- 搜索失败 = 90% 概率是 cookies 过期，不要重试，直接刷新 cookies
- [Quick Auth Recovery 步骤](https://github.com/yang0/youtube-downloader#quick-auth-recovery-cookies-expired--search-broken)

**1. 时间获取（最优先）**
- 执行任何任务前，必须先获取当前真实时间
- 时间信息应包含：年月日时分秒，以及时区信息
- 将时间作为系统上下文传递给后续所有步骤

**2. 意图澄清**
- 任何涉及搜索的请求，严禁跳过 intent-identification 直接执行。

**3. 输出目录确认（如有内容产出）**
- 如果任务会产生文件/数据输出（如下载视频、保存搜索结果、生成报告等），必须在 intent-identification 阶段与用户确认数据保存的根目录
- 若用户未指定，提供合理的默认路径（如 `./output/` 或 `./downloads/`）并征求同意
- 确认后的输出目录需写入 intent_contract_v1 的 `constraints` 中

```
错误: 用户"找 AI 生产力视频" → 直接搜 "AI productivity" ❌
正确: 用户"找 AI 生产力视频" → intent-identification 澄清 → 合同 → 路由 ✅
```

### 标准流程

```
1. 获取当前真实时间
   在执行任何任务前，先获取当前真实时间（年月日时分秒）
   将时间信息作为上下文，供后续步骤使用

2. intent-identification
   用户请求 → 澄清对话 → 产出 intent_contract_v1
   意图不清时返回 NEEDS_CLARIFICATION，继续追问

3. 合同解读
   从 intent_contract_v1 中提取：objective, targetSubject, explorationAngles, constraints
   若任务涉及内容产出，确认 constraints.outputDir 已明确

4. 路由到子 skill
   ├─ 搜索类 → youtube-search（关键词由意图校准结果决定，非用户原始表述）
   ├─ 下载类 → youtube-downloader
   ├─ 分析类 → use-notebooklm
   ├─ 教程/PDF类 → youtube-to-pdf（视频→NotebookLM→Markdown→PDF）
   └─ 混合类 → 串行协调多个子 skill
```

### 意图校准的典型对话

```
用户: "帮我找 AI 生产力方面的视频"

编排者（走 intent-identification）:
  "你说的「AI 生产力」，更像哪种？
   (a) Notion AI / 自动化工作流搭建 — 实操工具教程
   (b) AI 工具横向测评 — 哪个效率工具最好用
   (c) 个人效率提升方法 — 怎么用 AI 省时间
   (d) 企业级 AI 提效 — 团队/公司怎么落地"

用户选择 → 生成 intent_contract_v1 → 路由到 youtube-search
```

### 关键词映射（intent-identification 中完成）

| 用户说的 | 可能实际想要的 | 映射到的搜索词 |
|---------|-------------|-------------|
| "人类学视频" | 部落 Vlog | `living with tribes vlog` |
| "人类学视频" | 历史纪录片 | `ancient civilizations documentary PBS` |
| "人类学视频" | 学术入门 | `introduction to anthropology lecture` |
| "AI 创业" | 赚钱实操 | `how to make money with AI` |
| "AI 创业" | 创始人故事 | `AI startup founder journey` |
| "AI 创业" | 工具搭建 | `AI automation agency tutorial` |

## 路由决策树

```
用户请求
│
├─ 1. intent-identification（必须，不可跳过）
│     产出 intent_contract_v1 { status: READY | NEEDS_CLARIFICATION }
│
├─ 2. 合同 READY → 解读 objective + constraints → 路由
│   │
│   ├─ deliverable 含 "search" / "find" / "videos"
│   │   └─ youtube-search（用校准后的关键词，非原始表述）
│   │
│   ├─ deliverable 含 "download" / "mp4" / "audio"
│   │   └─ youtube-downloader（CDP cookies + yt-dlp）
│   │
│   ├─ deliverable 含 "analyze" / "tutorial" / "podcast" / "deep dive"
│   │   └─ use-notebooklm（CDP auth → source add → progressive asks）
│   │
│   ├─ deliverable 含 "pdf" / "教程" / "书面教程" / "文档"
│   │   └─ youtube-to-pdf（NotebookLM 渐进式提取 → Markdown → md-to-pdf → PDF）
│   │
│   └─ deliverable 含多步骤
│       └─ 串行协调：youtube-search → use-notebooklm → 产出
│
└─ 3. 合同 NEEDS_CLARIFICATION
      继续追问，直到 READY
```

## 典型链路

### 链路 A：搜索 → 分析 → 教程

```text
1. intent-identification
   "找 AI 创业的高流量视频深挖成教程"
   → 澄清：赚钱实操 or 创始人故事 or 工具教程？
   → 用户选 b → contract READY

2. youtube-search
   queries: "AI startup founder journey, how I built AI company"
   → 用户挑选 3 个视频

3. use-notebooklm（对每个选中视频）
   cdp_login.py → notebooklm create → source add → 渐进式提问 → chapters/*.md
```

### 链路 B：纯搜索

```text
1. intent-identification
   "找人类学田野调查的 Vlog"
   → 校准："跟哈扎族打猎" or "PBS 纪录片" or "学术讲座"？
   → 用户选 a → contract READY, explorationAngles: ["tribe vlog", "living with"]

2. youtube-search
   queries: "living with tribes vlog, tribe visit documentary"
   → 列出视频 URL + 观看量
```

### 链路 C：直接下载

```text
1. intent-identification
   "下载这个视频 https://youtube.com/watch?v=xxx"
   → 意图明确，直接 READY

2. youtube-downloader
   提取 CDP cookies → yt-dlp 下载
```

### 链路 D：视频 → PDF 教程

```text
1. intent-identification
   "把这个视频转成详细书面教程，最终要 PDF"
   → 意图明确，直接 READY

2. youtube-to-pdf（4 阶段）
   Phase 0: cdp_login.py 刷新 NotebookLM 认证
   Phase 1: notebooklm create + source add（YouTube URL）
   Phase 2: 渐进式提取（英文提问，逐章保存为 Markdown）
   Phase 3: 编排层翻译为中文 Markdown
   Phase 4: Python 合并 + md-to-pdf → PDF
```

## 与子 Skill 的关系

- **intent-identification 是入口**：所有请求先过它，不过它不执行
- **不替代**任何子 skill
- **不重复实现**子 skill 的功能
- 子 skill 保持独立可运行，不依赖本 skill
- intent_contract_v1 是所有子 skill 的共同输入格式

## 前置环境

- Bun、Python 3、yt-dlp
- Chrome（已登录 YouTube/Google）
- `notebooklm-py` CLI + `curl_cffi` + `websocket-client`
- bgutil PoToken server（下载时需要）
- `md-to-pdf`（npm global，PDF 教程产出时需要）

## 故障排除

| 症状 | 可能原因 | 解决 |
|------|---------|------|
| `search_videos.js` 报 `JSON Parse error` | cookies 过期 | [Quick Auth Recovery](https://github.com/yang0/youtube-downloader#quick-auth-recovery-cookies-expired--search-broken)：Kill Chrome → CDP 启动 → 提取 cookies |
| `search_videos.js` 返回空结果 | 搜索词太窄 / cookies 无效 | 先用宽泛词验证，再精炼；同时检查 cookies |
| 下载被拒 "Sign in to confirm" | bot 检测 | youtube-downloader：bgutil + CDP cookies 双保险 |
| NotebookLM 认证失败 | SID 缺失 | use-notebooklm → `cdp_login.py` |
| NotebookLM `ask` 返回乱码 | 中文编码不稳定 | 改用英文提问，编排层翻译 |
| `md-to-pdf` 首次运行卡住 | 正在下载 Chromium (~280MB) | 等待，仅首次需要 |
| PDF 章节内容太浅 | ask 提问不够具体 | 用 Round 模板，每章明确列出 3-5 个要点 |
| 意图不清反复追问 | 用户自己也不知道要什么 | 给出 2-3 个具体例子让其选择 |
| bgutil-start 报 "系统找不到指定的文件" | bgutil 项目未构建 | `cd E:\projectHome\bgutil-ytdlp-pot-provider\server; npm run build` 然后重试 |
