---
name: youtube-skill
description: YouTube 操作总入口。协调 youtube-downloader、youtube-search、use-notebooklm 三个子 skill，根据用户意图路由到正确的执行路径。
tier: router
---

# YouTube Skill

YouTube 相关操作的统一编排入口。不直接执行搜索/下载/分析，而是将用户意图映射到正确的子 skill 并协调执行。

## ⛔ 硬性关卡：搜索前必须做意图校准

**本 skill 被触发时，如果用户请求涉及搜索视频，严禁直接路由到 youtube-search。** 必须先完成意图校准：

1. 用户说"找 X 视频" → **停**，不要接"好的我来搜"
2. 用 2-3 个具体视频例子让用户确认方向（参考 youtube-search 的「意图校准」章节）
3. 校准完成后，再路由到 youtube-search 执行

```
错误流程: "找 AI 生产力视频" → 直接搜 "AI productivity" ❌
正确流程: "找 AI 生产力视频" → "你要的是 Notion AI 实操、自动化工作流搭建，还是 AI 工具测评？" → 确认后搜 ✅
```

唯一例外：同一会话中已校准过的延续搜索。

## 子 Skill 矩阵

| 子 Skill | 仓库 | 职责 | 触发条件 |
|----------|------|------|---------|
| **youtube-downloader** | [yang0/youtube-downloader](https://github.com/yang0/youtube-downloader) | 下载视频（解决认证墙） | "下载这个视频"、"yt-dlp 失败了" |
| **youtube-search** | [yang0/youtube-search](https://github.com/yang0/youtube-search) | 搜索视频、提取评论、下载头像 | "找关于 X 的视频"、"提取评论" |
| **use-notebooklm** | [yang0/use-notebooklm](https://github.com/yang0/use-notebooklm) | NotebookLM 深度分析视频内容 | "深挖这个视频内容"、"生成播客" |

## 路由决策树

```
用户提到了 YouTube 链接或搜索意图？
├─ 要搜索视频？
│   ├─ ⛔ 先做意图校准（不可跳过）
│   └─ 校准完成后 → 路由到 youtube-search
│
├─ 要下载视频？
│   └─ 路由到 youtube-downloader
│       先用 yt-dlp 尝试，失败则走 CDP cookies + bgutil PoToken
│
├─ 要搜索视频 / 提取评论 / 下载头像？
│   └─ 路由到 youtube-search
│       搜索 → 筛选 → 提取评论 → 下载头像，按需串接
│
├─ 要深挖视频内容 / 提取教程 / 生成播客？
│   └─ 路由到 use-notebooklm
│       认证 → 创建 notebook → 导入 source → 渐进式提问 → 产出章节文件
│
├─ 要完整链路：搜索 → 分析 → 沉淀？
│   └─ 协调两个子 skill：
│       1. youtube-search 搜索视频
│       2. use-notebooklm 深度分析选中的视频
│       3. 产出结构化教程
│
└─ 意图不清？
    └─ 先澄清，再路由
```

## 典型链路

### 链路 A：搜索 → 分析 → 教程

```text
用户: "找 YouTube 上 AI 创业的高流量视频，挑几个最好的深挖成教程"

1. youtube-search
   bun scripts/search_videos.js --queries "ai startup" --output-dir ./data
   → 拿到视频列表，用户挑选

2. use-notebooklm
   python scripts/cdp_login.py --launch-chrome          # 认证
   notebooklm create "AI Startup Analysis"
   notebooklm source add "<picked-url>" -n "<id>"
   notebooklm source wait "<source-id>" -n "<id>"
   # 渐进式 5-8 轮提问
   → 产出 chapters/*.md
```

### 链路 B：下载 → 分析

```text
用户: "下载这个视频，然后做个语音摘要"

1. youtube-downloader
   提取 CDP cookies → yt-dlp 下载视频/音频

2. use-notebooklm
   导入已下载的音频 → generate audio overview
```

### 链路 C：纯搜索

```text
用户: "找关于人类学田野调查的 Vlog"

1. youtube-search
   bun scripts/search_videos.js --queries "anthropology fieldwork vlog"
   → 列出视频 URL + 频道 + 观看量
```

## 与子 Skill 的关系

- **不替代**任何子 skill
- **不重复实现**子 skill 的功能
- 只在需要跨 skill 协调时介入
- 子 skill 保持独立可运行，不依赖本 skill

## 前置环境

所有子 skill 的依赖。至少需要：
- Bun、Python 3、yt-dlp
- Chrome（已登录 YouTube/Google）
- `notebooklm-py` CLI
- `curl_cffi`、`websocket-client`
- bgutil PoToken server（下载时需要）

## 故障排除

| 症状 | 可能原因 | 检查哪个子 skill |
|------|---------|-----------------|
| 搜索无结果 | cookies 过期 | youtube-search → 刷新 CDP cookies |
| 下载被拒 | bot 检测 | youtube-downloader → 走 CDP 流程 |
| NotebookLM 认证失败 | SID cookie 缺失 | use-notebooklm → `cdp_login.py --launch-chrome` |
| 子 skill 命令不存在 | 未在正确目录执行 | 确认 `which notebooklm`、`bun --version` |
