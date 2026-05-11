# YouTube Skill

YouTube 操作统一编排入口。协调三个独立子 skill：

- **[youtube-downloader](https://github.com/yang0/youtube-downloader)** — 视频下载（CDP auth + bgutil）
- **[youtube-search](https://github.com/yang0/youtube-search)** — 搜索 / 评论 / 头像
- **[use-notebooklm](https://github.com/yang0/use-notebooklm)** — NotebookLM 深度内容分析

本 skill 是纯文档型（router tier），不包含可执行脚本。所有执行逻辑由子 skill 各自承担。

## 执行原则

1. **先获取时间** — 在执行任何任务前，先获取当前真实时间（年月日时分秒）作为上下文
2. **再澄清意图** — 通过 intent-identification 产出结构化合同
3. **确认输出目录** — 如有内容产出，在 intent-identification 阶段确认数据保存的根目录
4. **后路由执行** — 根据合同路由到对应子 skill

## 安装

```bash
git clone https://github.com/yang0/youtube-skill.git
```

三个子 skill 需单独 clone 到 `E:\projectHome\` 下：

```bash
git clone https://github.com/yang0/youtube-downloader.git ..\youtube-downloader
git clone https://github.com/yang0/youtube-search.git ..\youtube-search
git clone https://github.com/yang0/use-notebooklm.git ..\use-notebooklm
```
