# 小红书 AI 求职雷达

追踪小红书上 AI 方向求职信息的静态数据看板：招聘信息、面试面经、经验攻略。
纯静态页面，无任何依赖——双击 `index.html` 即可在本地打开，也可以直接部署到 GitHub Pages。

**开发与运维文档见 [DEVELOPMENT.md](DEVELOPMENT.md)**（架构、采集操作流程、风控经验、Roadmap）。

![预览](https://img.shields.io/badge/数据源-MediaCrawler-red) ![依赖](https://img.shields.io/badge/依赖-零-green)

## 本地使用

直接双击 `index.html` 即可（数据内嵌在 `data.js`，无跨域问题）。
如需本地起服务：`python -m http.server -d . 8000`，然后访问 <http://localhost:8000>。

页面功能：分类标签（招聘/面经/攻略）、全文搜索、公司/方向筛选、按时间或热度排序、真题列表展开、原文链接直达。

## 数据更新流程

```
MediaCrawler/data/xhs/jsonl/*.jsonl   ← 采集原始数据（SAVE_DATA_OPTION=jsonl）
pipeline/extracted.json               ← LLM 结构化提取（分类/公司/岗位/地点/真题）
pipeline/build_site.py                ← 去重合并，生成本目录的 data.js
```

每一轮更新：

1. 在 `MediaCrawler` 目录运行采集：`uv run main.py --platform xhs --lt qrcode --type search`（关键词在 `config/base_config.py` 的 `KEYWORDS` 中修改）；
2. 用 LLM 整理新增笔记，把结构化字段补进 `pipeline/extracted.json`（key 为 note_id）；
3. 重新构建：`uv run python ../pipeline/build_site.py`（在 MediaCrawler 目录下执行）；
4. 提交并推送本仓库，GitHub Pages 自动更新。

未提取的笔记会在页面"待分类"标签下显示，不会丢失。

## 部署到 GitHub Pages

```bash
# 在本目录（site/）内
git init && git add . && git commit -m "init: 求职雷达看板"
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

然后 GitHub 仓库 → Settings → Pages → Branch 选 `main` / 目录 `(root)` → Save。
一两分钟后即可通过 `https://<你的用户名>.github.io/<仓库名>/` 访问。

## 免责声明

数据采集自小红书公开笔记，仅供个人求职学习使用，请勿用于商业用途或二次分发内容本身；
分类与字段由 LLM 辅助提取，可能存在偏差，投递前务必点击「原文链接」核对最新信息。
