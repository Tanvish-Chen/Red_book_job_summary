# 开发说明文档 · 小红书 AI 求职雷达

> 最后更新：2026-08-31 · 数据规模：220 条笔记（面经 129 / 攻略 66 / 招聘 24）
> 在线看板：`https://<用户名>.github.io/Red_book_job_summary/`（需先开启 Pages，见「部署」）

---

## 1. 项目是什么

一个**三段式数据管线**：用 MediaCrawler 采集小红书公开笔记 → LLM+规则做结构化提取 → 生成零依赖静态看板网页，部署到 GitHub Pages。目标是持续追踪 AI 方向求职信息（校招岗位、面经真题、经验攻略）。

```
┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  MediaCrawler    │   │  pipeline/       │   │  site/          │
│  (采集器,上游库) │──>│  提取与构建脚本   │──>│  看板静态页      │
│  data/xhs/jsonl  │   │  extracted.json  │   │  index.html     │
└─────────────────┘   └──────────────────┘   └────────┬────────┘
                                                       │ git push
                                                       v
                                             GitHub Pages 上线
```

## 2. 目录结构（工作区 `D:\小红书爬取`）

```
D:\小红书爬取\
├── MediaCrawler\              # 上游采集库（GitHub: NanmiCoder/MediaCrawler）
│   ├── config\base_config.py  #   ★ 采集配置（关键词/条数/频率）——最常改
│   ├── data\xhs\jsonl\        #   原始数据落盘处（按日期追加）
│   ├── browser_data\          #   浏览器登录态缓存（删除=下次要重新扫码）
│   └── media_platform\xhs\    #   ★ 已打本地补丁，升级上游时会丢（见 §5）
├── pipeline\                  # 自研数据管线
│   ├── extracted.json         #   ★ 结构化标注库（LLM 维护，note_id → 字段）
│   ├── build_site.py          #   合并去重 → 生成 site/data.js
│   └── auto_extract.py        #   规则预标注（分类/公司/城市/真题抽取）
└── site\                      # ★ 独立 git 仓库（推 GitHub 的就是它）
    ├── index.html             #   看板页（无外部依赖，双击可开）
    ├── data.js                #   构建产物（window.XHS_DATA = {...}）
    ├── README.md              #   使用简介
    └── DEVELOPMENT.md         #   本文档
```

## 3. 日常操作：跑一轮采集

### 3.1 配置（`MediaCrawler/config/base_config.py`）

| 参数 | 当前值 | 说明 |
|---|---|---|
| `KEYWORDS` | 逗号分隔 | 每轮要搜的词；一轮别超过 6-8 个 |
| `CRAWLER_MAX_NOTES_COUNT` | 30 | 每个关键词抓多少条 |
| `CRAWLER_MAX_SLEEP_SEC` | 6 | 请求间隔（秒）。**别低于 5**，见 §6 风控 |
| `ENABLE_GET_COMMENTS` | False | 评论抓取开关；开启后请求数翻倍 |
| `SAVE_DATA_OPTION` | jsonl | 保持 jsonl（管线依赖此格式） |
| `CRAWLER_TYPE` | search | 可改 creator 爬指定博主全部笔记 |

### 3.2 执行

```bash
cd /d/小红书爬取/MediaCrawler
uv run main.py --platform xhs --lt qrcode --type search
```

- 首次或登录态失效时会弹 Chrome 窗口等扫码（窗口保持约 1 小时，已改过源码）；
- 登录态健康时全自动，无需人工。正常一轮 6 词 × 30 条 ≈ 25-35 分钟。

### 3.3 采集后处理（一轮三步）

```bash
cd /d/小红书爬取/MediaCrawler

# ① 规则预标注（只处理新 note_id，不覆盖已有标注）
uv run python ../pipeline/auto_extract.py        # 加 --dry-run 只看不写

# ② （LLM 复核）把 pending 与明显误分类的条目改对，
#    或把 summary/questions 补充进 pipeline/extracted.json

# ③ 重建看板并上线
uv run python ../pipeline/build_site.py
cd ../site && git add data.js && git commit -m "数据更新" && git push
```

## 4. extracted.json 数据契约

key 为小红书 note_id，value 字段：

```jsonc
{
  "note_id_xxx": {
    "category": "job",          // job | interview | insight | noise
    "company": "字节跳动",       // 尽量标准名；外号映射见 auto_extract.py NICKNAMES
    "positions": ["AI产品经理"],
    "locations": ["北京"],
    "deadline": "8.13起投递",    // 有截止/时间线才填
    "direction": "AI产品",       // Agent/RAG/LLM/NLP/CV/AI产品/...
    "summary": "一句话摘要",      // 40-80字，提取核心信息
    "tags": ["2027届", "官方"],
    "questions": ["面试题1", ...], // 面经类：逐条抽出真题
    "auto": true                 // true=规则草稿，复核后可去掉
  }
}
```

规则：`auto_extract.py` 永不覆盖已有条目；没有条目的 note_id 在页面上显示为「待分类」。noise 类默认在「全部」标签下隐藏。

## 5. 本地补丁清单（升级 MediaCrawler 时需重打）

对上游 `media_platform/xhs/` 打过 3 处补丁，`git pull` 上游更新会冲突/丢失：

1. **core.py · get_comments**：包 try/except，单条评论失败跳过不炸整轮（DataFetchError/RetryError/KeyError）；
2. **core.py · get_note_detail_async_task**：HTML 回退路径包 RetryError；
3. **core.py · 三处 asyncio.gather**：`return_exceptions=True` + 结果循环跳过 BaseException 实例；
4. **login.py · check_login_state**：扫码等待窗口 600 次→3000 次（约 12 分钟→1 小时）。

## 6. 风控经验（重要，血泪换来的）

- **账号级标记**：风控跟账号走，不跟 IP。被标记表现：登录成功但详情 API 100% 返回 461 验证码。解法只有静默 24h+ 或换号；
- **触发条件**（我们踩过的）：短时间多轮搜索 + 登录态反复失效重登（一天内丢了 3 次登录态后触发）；
- **安全参数**：间隔 ≥6 秒、每轮 ≤200 条、轮次间隔 ≥45 分钟、一轮关键词 ≤8 个；
- **健康信号**：日志里 `CAPTCHA appeared` 计数。开局偶发 1-2 次可无视；连续出现应立即停（TaskStop），静默后再来；
- 评论抓取是详情请求的 2 倍压力，面经数据主要在正文里，非必要不开。

## 7. 部署（GitHub Pages）

site/ 本身是独立 git 仓库，remote 已指向 `Tanvish-Chen/Red_book_job_summary`。

首次开通（浏览器操作，一次性）：
1. 打开 `https://github.com/Tanvish-Chen/Red_book_job_summary` → **Settings** → **Pages**；
2. Source 选 **Deploy from a branch**，Branch 选 `main` / `(root)` → **Save**；
3. 等 1-2 分钟，访问 `https://tanvish-chen.github.io/Red_book_job_summary/`。

之后每次 `git push` 页面自动更新。本地预览直接双击 `site/index.html`（数据内嵌无跨域问题）。

## 8. 后续开发建议（Roadmap）

按性价比排序：

1. **Cookie 登录免扫码**：浏览器 F12 → Application → Cookies → 复制 `web_session` 填入 `config/base_config.py` 的 `COOKIES`，`LOGIN_TYPE` 改 `cookie`。配合现有 1 小时扫码窗口，无人值守成功率大幅提高；
2. **博主主页模式**：面经数据密度最高的博主（现有数据里nickname可统计）用 `CRAWLER_TYPE="creator"` 定向全量爬取，比关键词搜索精准；
3. **面经聚合页**：把 129 条面经的 questions 按 direction 聚合去重，生成「按方向复习清单」——build_site.py 里已有全部原料，加一个 group-by 视图即可；
4. **GitHub Actions 定时构建**：push 时自动跑 build_site.py 校验 data.js 与 extracted.json 一致性；
5. **接入 xiaohongshu-mcp**（github.com/xpzouying/xiaohongshu-mcp）：临时查询场景（"最近谁在招 X"）走 MCP 实时查，不用跑全量采集。

## 9. 常见问题

| 症状 | 处置 |
|---|---|
| 启动后一直等扫码 | 正常，1 小时窗口内任意时刻扫码即可；或按 §8.1 切 cookie 模式 |
| 日志大量 `CAPTCHA appeared` | 立即停止本轮，账号静默 24h，期间勿高频使用 |
| `uv run` 报 Python 版本错 | 依赖 `requires-python>=3.11`，uv 会自动下载，勿用系统 3.10 直跑 |
| 页面数据没更新 | 检查是否忘了跑 build_site.py，或 git push 网络失败重试 |
| xlsx 报 no module | 旧格式兼容警告可无视；jsonl 才是主数据源 |
