# 小红书 AI 求职雷达 · 开发文档

> 本文档面向后续开发者（或未来的你），说明系统的架构、数据流、每一步怎么跑、踩过的坑和改进方向。
> 部署上线说明见 [`site/README.md`](site/README.md)。

## 1. 系统是什么

自动采集小红书上 **AI 方向求职信息**（企业招聘帖、面经真题、求职攻略），经 LLM 辅助结构化后，
生成一个**纯静态数据看板网页**——本地双击可开，也可部署到 GitHub Pages。

当前数据规模：220 条笔记（面经 129 / 攻略 66 / 招聘 24 / 无关 1），181 条评论。

## 2. 目录结构与数据流

```
D:\小红书爬取\
├── MediaCrawler\                  # 上游：NanmiCoder/MediaCrawler 爬虫（已本地魔改，见 §5）
│   ├── config\base_config.py      #   采集配置：关键词/条数/间隔/保存格式
│   ├── media_platform\xhs\core.py #   已打容错补丁（评论/详情失败不崩进程）
│   ├── media_platform\xhs\login.py#   扫码等待窗口 600→3000 次
│   └── data\xhs\jsonl\*.jsonl     #   原始数据落盘（SAVE_DATA_OPTION=jsonl）
│
├── pipeline\                      # 中游：数据处理（自研，核心资产）
│   ├── build_site.py              #   扫描原始数据 → 按 note_id 去重 → 生成 site/data.js
│   ├── auto_extract.py            #   规则引擎：新笔记自动预分类/抽公司/城市/截止时间/真题
│   └── extracted.json             #   结构化标注库（key=note_id，auto:true 为规则草稿）
│
└── site\                          # 下游：静态看板（独立 git 仓库 → GitHub）
    ├── index.html                 #   单文件应用：标签页/搜索/筛选/排序/真题展开
    ├── data.js                    #   构建产物：window.XHS_DATA = {...}（数据内嵌，无跨域）
    └── README.md                  #   部署说明（GitHub Pages）
```

**数据流**：`MediaCrawler 采集 → jsonl 落盘 → auto_extract 规则预标 → 人工/LLM 复核 extracted.json → build_site 生成 data.js → git push → Pages 生效`

## 3. 每一步怎么跑

所有命令在 `D:\小红书爬取\MediaCrawler` 目录下执行（依赖其 uv 虚拟环境，Python ≥3.11）。

### 3.1 采集一轮

1. 改 `config/base_config.py`：`KEYWORDS`（英文逗号分隔）；
2. 运行 `uv run main.py --platform xhs --lt qrcode --type search`；
3. 首次/登录态失效时 Chrome 会弹二维码（窗口约 1 小时），手机扫码；
4. 数据落盘到 `data/xhs/jsonl/search_contents_<日期>.jsonl`（追加式，同笔记自动去重存储）。

当前安全参数：`CRAWLER_MAX_NOTES_COUNT=30`（每词）、`CRAWLER_MAX_SLEEP_SEC=6`、
`ENABLE_GET_COMMENTS=False`（需要评论时再开）、`MAX_CONCURRENCY_NUM=1`。

### 3.2 提取标注

```bash
uv run python ../pipeline/auto_extract.py            # 增量：只为新 note_id 生成草稿（auto:true）
```

- 规则引擎：标题强信号优先判类（面经/招聘/攻略/无关），公司词典 + 外号映射（鹅厂→腾讯），
  城市列表、`X月X日` 截止时间、`1. 2. 3.` 编号真题抽取；
- **无把握的判 pending 不写入**，由人工/LLM 补；
- 复核方式：直接编辑 `pipeline/extracted.json`，把明显错的改掉（吐槽/讨论类误判为 job 最常见）。

### 3.3 构建与发布

```bash
uv run python ../pipeline/build_site.py   # 生成 site/data.js
cd ../site && git add data.js && git commit -m "数据更新" && git push
```

## 4. 看板功能（site/index.html）

- 五个统计卡片（总数/招聘/面经/攻略/待整理）
- 标签页过滤（noise 默认隐藏）、全文搜索（标题/正文/摘要/公司/岗位/标签）、公司/方向下拉、时间/热度排序
- 卡片字段：公司、岗位、📍地点、⏰截止时间、方向、标签；interview 类可展开真题列表
- 每张卡片带小红书原文链接（投递前务必点原文核对）

改页面只需编辑 `index.html` 后直接刷新浏览器（无构建步骤）。

## 5. 对 MediaCrawler 的本地魔改（升级时注意）

| 文件 | 改动 | 原因 |
|---|---|---|
| `media_platform/xhs/core.py` | `get_comments` 加 try/except（DataFetchError/RetryError/KeyError 跳过）；`get_note_by_id_from_html` 回退路径包 RetryError；4 处 `asyncio.gather` 加 `return_exceptions=True`，结果循环过滤 BaseException 实例 | 单条笔记风控失败会炸掉整个进程（两次崩溃根因） |
| `media_platform/xhs/login.py` | `check_login_state` 重试 600→3000 次 | 扫码窗口实际 10 分钟太短，拉长到约 1 小时 |

上游更新时 `git pull` 会冲突，**以本地补丁为准**重放（补丁很小，见上表）。

## 6. 风控经验（重要，省学费）

1. **账号级标记**：短时间高频搜索+登录会使账号进入惩罚期（详情 API 全部 461 验证码，搜索列表正常）。
   表现：登录态频繁失效、扫码后立即吃验证码。**换账号立即恢复** → 标记是账号级的，不是 IP 级；
2. **惩罚期处理**：立即停止（继续跑只会加深标记），静默数小时~1 天再试；被标记的账号期间正常
   轻度使用有助于恢复行为画像，但别高频操作；
3. **健康节奏**（本次全程 0 验证码的配置）：6 秒间隔、每词 ≤30 条、每轮 6 个关键词、
   轮间冷却 ≥45 分钟、关闭评论抓取（请求数减半）；
4. **登录态持久化**：正常账号 `SAVE_LOGIN_STATE=True` 下会话可跨启动保留（新账号验证过），
   被标记账号会被服务端强制下线（每次都要扫码）。

## 7. 已知问题与改进方向

- **分类准确率约 75-85%**（规则引擎），"吐槽/讨论 vs 招聘"边界仍会误判，靠人工复核兜底。
  改进：接入 LLM API 批量分类（extracted.json 的 auto:true 条目直接喂模型重标）；
- **图片内容丢失**：很多岗位详情/真题在笔记配图里，正文只有摘要。改进：开 `ENABLE_GET_MEIDAS=True`
  下载图片 + OCR/多模态模型提取；或用 `--type detail` 按 note_id 补抓；
- **博主主页模式未用**：`CRAWLER_TYPE="creator"` 可定向抓高质量面经博主的全部笔记（更准的真题来源），
  需在配置里填 `XHS_CREATOR_ID_LIST`；
- **评论数据只在首轮开了**：181 条评论已入库但看板未展示。改进：卡片展开评论区（评论里常有内推码/补充题目）；
- **xlsx 旧文件兼容**：build_site 仍读第一天导出的 xlsx（jsonl 已覆盖相同 20 条，实际冗余），
  未来可删 `data/xhs/*.xlsx`，代码里有 try/except 兜底；
- **部署自动化**：当前更新需手动 build+push。改进：GitHub Actions 监听 jsonl 变更自动构建，
  或定时 Action 拉起采集（需要解决服务器扫码问题——可换 cookie 登录方案）。

## 8. 快速参考

```bash
# 一轮完整更新（在 MediaCrawler 目录）
uv run main.py --platform xhs --lt qrcode --type search          # ① 采集（可能需扫码）
uv run python ../pipeline/auto_extract.py                        # ② 规则预标
#   ③ 人工复核 pipeline/extracted.json（可选）
uv run python ../pipeline/build_site.py                          # ④ 构建
cd ../site && git add . && git commit -m "数据更新" && git push    # ⑤ 发布

# 看板本地预览：双击 site/index.html
# 线上地址：https://tanvish-chen.github.io/Red_book_job_summary/
```

## 9. 免责声明

数据采集自小红书公开笔记，仅供个人求职学习；分类与字段由规则+LLM 辅助生成，可能有偏差，
投递前务必点击「原文链接」核实。请遵守 MediaCrawler 的非商业学习许可（LICENSE）。
