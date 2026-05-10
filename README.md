# YouTube Skill

YouTube 操作统一编排入口。协调三个独立子 skill：

- **[youtube-downloader](https://github.com/yang0/youtube-downloader)** — 视频下载（CDP auth + bgutil）
- **[youtube-search](https://github.com/yang0/youtube-search)** — 搜索 / 评论 / 头像
- **[use-notebooklm](https://github.com/yang0/use-notebooklm)** — NotebookLM 深度内容分析

本 skill 是纯文档型（router tier），不包含可执行脚本。所有执行逻辑由子 skill 各自承担。

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
