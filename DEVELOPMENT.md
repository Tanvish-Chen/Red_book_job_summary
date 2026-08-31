# 小红书 AI 求职雷达 · 项目实施文档（唯一权威版）

> 最后更新：2026-09-01 · 数据规模：322 条笔记（招聘 48 / 面经 157 / 攻略 105 / 无关 11 / 待分类 1）+ 181 条评论
> 在线看板：https://tanvish-chen.github.io/Red_book_job_summary/
> 项目 GitHub：https://github.com/Tanvish-Chen/Red_book_job_summary
> 本文档是项目**唯一**的完整实施文档。根目录曾有一份旧版 `DEVELOPMENT.md`，已删除，勿再创建副本。

---

## 1. 项目是什么

一条三段式数据管线，目标是**系统化追踪 AI 方向求职信息**（企业招聘帖、面经真题、求职攻略），服务于临近毕业的求职季：

```
MediaCrawler（采集） → pipeline（规则预标 + 人工复核 + 构建） → site（零依赖静态看板） → GitHub Pages
```

- **上游**：开源爬虫 MediaCrawler（本地魔改，见 §7），扫码登录后按关键词搜索小红书公开笔记，落盘 jsonl
- **中游**：`auto_extract.py` 规则引擎预标 → 人工/LLM 复核 `extracted.json` → `build_site.py` 去重合并生成 `data.js`
- **下游**：`site/index.html` 单文件看板（搜索/筛选/排序/时效标记/真题展开/原文链接），`git push` 即发布

## 2. 目录结构

```
D:\小红书爬取\
├── MediaCrawler\                    # 上游爬虫（uv 虚拟环境，Python ≥3.11）
│   ├── config\base_config.py        #   关键词/条数/间隔/登录方式等
│   ├── media_platform\xhs\core.py   #   容错补丁（§7）
│   ├── media_platform\xhs\login.py  #   扫码等待 3000 次（约 50 分钟）
│   └── data\xhs\jsonl\*.jsonl       #   原始数据（追加式、按日期分文件）
├── pipeline\
│   ├── auto_extract.py              #   规则引擎（只处理新 note_id，不覆盖已复核条目）
│   ├── build_site.py                #   去重合并 + 截止时间解析 → site/data.js
│   └── extracted.json               #   结构化标注库（核心资产）
└── site\                            # 独立 git 仓库 → GitHub Pages
    ├── index.html                   #   看板应用（无构建步骤，改完刷新即生效）
    ├── data.js                      #   构建产物（勿手改）
    ├── README.md                    #   仓库简介（指向本文档）
    └── DEVELOPMENT.md               #   本文档
```

## 3. 环境准备（一次性）

| 项 | 要求 | 备注 |
|---|---|---|
| Python | ≥3.11（MediaCrawler 的 uv 环境） | **所有脚本必须用 `uv run` 执行**；系统 Python 缺 openpyxl，会静默跳过 xlsx 旧数据（实测教训：220→217 条） |
| Chrome | 任意现代版 | CDP 模式（`ENABLE_CDP_MODE=True`）复用本机 Chrome，反检测更好 |
| 小红书账号 | 健康账号 1 个 | 被风控标记的账号会反复要求扫码，详见 §6 |

关键配置（`MediaCrawler/config/base_config.py`）：

| 配置 | 当前值 | 说明 |
|---|---|---|
| `KEYWORDS` | 招聘向 6 词 | 上一轮面试词已备份在注释里；每轮换主题 |
| `CRAWLER_MAX_NOTES_COUNT` | **40** | 每词抓 2 页（20 条/页）。注意取整：设 30 实际只抓 1 页 20 条 |
| `CRAWLER_MAX_SLEEP_SEC` | 6 | 每条详情间隔，风控安全线 |
| `MAX_CONCURRENCY_NUM` | 1 | 并发 1，勿调高 |
| `ENABLE_GET_COMMENTS` | False | 评论请求量减半的关键；需要时再开 |
| `ENABLE_GET_MEIDAS` | False | 图片未下载（配图内容丢失，见 §8） |
| `SAVE_DATA_OPTION` | jsonl | 追加式落盘 |
| `SAVE_LOGIN_STATE` | True | 登录态持久化到 `browser_data/` |

## 4. 日常更新流程（一轮约 1 小时）

**⚠️ 时段纪律：只在你醒着且能看手机的时段跑采集。** 2026-08-31 深夜挂机跑，会话失效弹二维码无人扫，整晚零产出。登录失效无法自愈（必须人工扫码），本项目**不做**推送叫醒方案（用户明确拒绝），因此夜间无人值守必然失败。一轮采集仅 15-30 分钟，白天跑零成本。

### 4.1 采集

```bash
cd MediaCrawler
uv run main.py --platform xhs --lt qrcode --type search
```

- 登录态有效：直接开跑（日志出现 `Current search keyword`）
- 弹出二维码：用小红书 App 扫码（窗口约 50 分钟）
- 实测：6 词 × 2 页约 14 分钟，一轮新增约 100 条（关键词间有重复，属正常）
- 数据落盘到 `data/xhs/jsonl/search_contents_<日期>.jsonl`

**每轮采集后检查日志**：确认 6 个关键词都跑完、无 `461`/`请通过验证`/ERROR；出现 461 = 账号进惩罚期（§6）。

### 4.2 规则预标

```bash
uv run python ../pipeline/auto_extract.py            # 增量，只为新笔记生成草稿（auto:true）
uv run python ../pipeline/auto_extract.py --dry-run  # 只预览不落盘
```

规则能力：标题强信号判类、公司词典+外号映射（鹅厂→腾讯）、城市、`X月X日` 截止时间、编号列表抽真题。
**实测准确率**：分类约 75%，公司字段会误匹配（如"卡士乳业"帖被标成小红书），截止时间会把"抽奖时间/发布时间"当成截止（如华为帖 9 月 4 日实为抽奖日）。所以草稿**必须复核**。

### 4.3 人工复核（质量的生命线）

直接编辑 `pipeline/extracted.json`（key=note_id）。复核要点（血泪清单）：

1. **分类边界**：
   - "凉经/面经/面试记录/投递时间线" → `interview`（规则常因标题含"秋招"误判为 job）
   - 企业官方招聘帖/内推帖 → `job`
   - 攻略/科普/薪资盘点/工具推荐/心态贴 → `insight`
   - 焦虑吐槽/提问/个人求职求助/学术招生 → `noise`
2. **截止时间陷阱**：抽奖时间 ≠ 截止、发布日 ≠ 截止、"8.13 起投递"是开始不是截止。格式统一写 `X月X日`（构建脚本按此解析出日期做过期判断）；长期有效/无截止留空
3. **公司字段**：内推帖的公司从正文判断，别信规则；`阿里巴巴`与`阿里`统一写`阿里`，`淘天`独立保留
4. **关键信息入摘要**：内推码、投递邮箱、面向届别（27 届/28 届）；"招聘对象为 2028 届"这类届别不符的要写警示
5. 复核完删掉该条目的 `"auto": true` 标记；完全无法判断的条目整条删除，看板会显示为"待分类"
6. **批量复核时不要用空字段覆盖规则已抽好的真题列表**（实测教训：一次补丁抹掉 120+ 题，需重跑规则恢复）

### 4.4 构建

```bash
uv run python ../pipeline/build_site.py
```

预期输出示例：`原始数据：322 条笔记 / 181 条评论 … 招聘 48 | 面经 157 | 攻略 105`。
检查：`待分类` 数量是否可接受；出现"有 N 条笔记待提取"提示说明有笔记没进 `extracted.json`。

### 4.5 本地验证

双击 `site/index.html`（或浏览器打开 `file:///D:/小红书爬取/site/index.html`）：
核对五个统计卡片数字、招聘页签摘要、时效标记（⏰在招/即将截止/已过期删除线）、地点筛选、真题展开。**构建后浏览器会缓存 data.js，务必强制刷新**。

### 4.6 发布

```bash
cd ../site && git add data.js index.html DEVELOPMENT.md && git commit -m "数据更新" && git push
```

一两分钟后 Pages 生效。

## 5. 看板功能速览

- 五统计卡片 + 分类标签页（招聘/面经/攻略/待分类，无关默认隐藏、可搜索出来）
- 全文搜索（标题/正文/摘要/公司/岗位/标签，可搜"内推码"）
- 筛选：公司 / **地点** / 方向 / **时效**（在招/即将截止≤14天/已过期/未注明）
- 排序：时间 / 热度
- 面经卡片可展开真题列表；每卡片带小红书原文链接（投递前务必点原文核对）

时效逻辑：`build_site.py` 把 `X月X日` 解析为日期（年份取发布年，跨年自动+1），前端按当天计算状态。静态页状态随时间自动变化，无需重建。

## 6. 风控与登录态（重要，省学费）

1. **账号级标记**：高频搜索+登录会把账号打进惩罚期（详情 API 全 461，搜索列表正常）。换账号立即恢复 → 标记在账号不在 IP
2. **惩罚期处理**：立即停跑（继续只会加深标记），静默数小时~1 天；期间轻度正常使用有助恢复
3. **健康节奏**（全程 0 验证码）：6 秒间隔、每词 2 页、6 个词/轮、轮间冷却 ≥45 分钟、关评论抓取
4. **登录态**：健康账号会话可跨启动保留；**会话可能在任意时刻被服务端失效，失效=必须人工扫码，无自动恢复手段**。这就是"只在白天跑"纪律的根因
5. 数据是增量落盘+按 note_id 去重的：中断**不丢已采数据**，补扫码重跑即可续上

## 7. 对 MediaCrawler 的本地魔改（升级时重放）

| 文件 | 改动 | 原因 |
|---|---|---|
| `media_platform/xhs/core.py` | `get_comments` try/except 跳过；`get_note_by_id_from_html` 回退路径包 RetryError；4 处 `asyncio.gather` 加 `return_exceptions=True` 并过滤异常实例 | 单条笔记风控失败会炸掉整个进程 |
| `media_platform/xhs/login.py` | 扫码等待 600→3000 次 | 原 10 分钟窗口太短 |
| `config/base_config.py` | `CRAWLER_MAX_NOTES_COUNT` 30→40 | 30 实际只抓 1 页 |

上游 `git pull` 会冲突，以本地补丁为准。已知补丁副作用：会话中途失效时会"静默空转"完剩余关键词（容错换稳定的代价），所以每轮结束要看日志核对产出量。

## 8. 已知问题与改进方向

| 问题 | 改进 |
|---|---|
| 图片内容丢失（很多岗位详情/真题在配图） | 开 `ENABLE_GET_MEIDAS=True` + 多模态模型提取 |
| 真题率约 27%（43/157） | 同上 + 复核时补录 |
| 博主定向采集未用 | `CRAWLER_TYPE="creator"` + 填 `config/xhs_config.py` 的 `XHS_CREATOR_ID_LIST`（目前是上游示例链接） |
| 181 条评论未展示 | 看板卡片展开评论区（评论常有内推码/补充题目） |
| 分类复核仍是人工 | 接 LLM API 批量重标 `auto:true` 条目 |
| 招聘帖截止时间覆盖率低（2/48） | 多数帖子本就不写截止（数据源特性），靠"未注明"筛选兜底 |
| 部署手动 | GitHub Actions 监听 jsonl 自动构建（服务器扫码问题无解，采集仍须本地） |

## 9. 快速参考

```bash
# 一轮完整更新（全部在 MediaCrawler 目录，全部用 uv）
uv run main.py --platform xhs --lt qrcode --type search   # ① 采集（白天跑！可能需扫码）
uv run python ../pipeline/auto_extract.py                 # ② 规则预标
#   ③ 人工复核 ../pipeline/extracted.json（§4.3 清单）
uv run python ../pipeline/build_site.py                   # ④ 构建
#   ⑤ 双击 ../site/index.html 验证（强制刷新）
cd ../site && git add . && git commit -m "数据更新" && git push   # ⑥ 发布
```

## 10. 免责声明

数据采集自小红书公开笔记，仅供个人求职学习；分类与字段由规则+人工/LLM 辅助生成，可能有偏差，投递前务必点击「原文链接」核实。请遵守 MediaCrawler 的非商业学习许可（LICENSE）。
