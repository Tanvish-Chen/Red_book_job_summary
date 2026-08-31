# 小红书 AI 求职雷达

追踪小红书上 AI 方向求职信息的静态数据看板：企业招聘帖、面试面经（含真题展开）、经验攻略。
纯静态页面零依赖——双击 `index.html` 即可本地打开，也可部署到 GitHub Pages。

**完整实施文档（架构 / 采集流程 / 复核规范 / 风控经验）见 [DEVELOPMENT.md](DEVELOPMENT.md)，那是唯一权威文档。**

![数据源](https://img.shields.io/badge/数据源-MediaCrawler-red) ![依赖](https://img.shields.io/badge/依赖-零-green)

## 本地使用

直接双击 `index.html`（数据内嵌在 `data.js`，无跨域问题）。
功能：分类标签、全文搜索、公司/地点/方向/时效筛选、按时间或热度排序、真题展开、原文链接直达。

## 数据更新与部署

采集 → 标注 → 构建 → 发布的完整步骤见 [DEVELOPMENT.md §4](DEVELOPMENT.md)。
推送到本仓库后，在 GitHub Settings → Pages 中启用即可自动更新。

## 免责声明

数据采集自小红书公开笔记，仅供个人求职学习，请勿商用或二次分发内容本身；
分类与字段由规则+人工/LLM 辅助提取，可能有偏差，投递前务必点击「原文链接」核对。
