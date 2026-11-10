# 供在此仓库中工作的编码智能体使用

更改任何内容前，请先阅读本文。以下每一行均已于 2026-11-09 使用实时 API、`brightdata-sdk` 2.5.2 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，以此处记录的实时 API 行为为准。

## 身份验证

- `token()` 会读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（Python SDK 读取的也是同一文件）。设置该变量或登录一次，是唯一需要人工完成的步骤；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅会读取 `BRIGHTDATA_API_TOKEN`，无法获取 CLI 登录信息。

## API 的行为方式

- 三个数据集：视频 `gd_lk56epmy2i5g7lzu0k`（通过 URL 采集，通过 `keyword`、`search_filters`、`hashtag`、`url`、`explore`、`podcast_url` 发现）、频道 `gd_lk538t2k2p1k3oos71`（通过 URL 采集，通过 `keyword` 发现）和评论 `gd_lk9q0ew71spt1mxywf`（通过 URL 采集）。
- Python SDK 封装了十种方法中的九种，不包括播客。此软件包使用标准库调用 `https://api.brightdata.com/datasets/v3`，且没有任何依赖，因此十种方法的工作方式完全相同。请保持这一点。
- 每次调用都是异步任务：`/trigger`，轮询 `/progress/<id>`，然后调用 `/snapshot/<id>?format=json`。运行时间从 12 秒（频道）到 250 秒（播客播放列表）不等。`TIMEOUT` 为 900。
- 如果状态轮询因网络错误失败，会持续重试直至截止时间；2026-10-24，使用同一循环的 Node 运行因一次连接中断而丢失了一项付费任务。HTTP 错误仍会抛出异常。
- 始终发送 `include_errors=true`。API 默认会丢弃错误行，这会让失败的输入看起来像空结果。
- 每次发现请求都会发送 `limit_per_input`，因此不会按所有匹配项计费。`num_of_posts` 和 `num_of_comments` 也得到了遵循：请求 2 条，返回 2 条；请求 3 条评论，返回 3 条。
- API 验证的输入字段名称：
  - 关键词搜索：`keyword`，`num_of_posts` 是文本字段（使用 "3"，而不是 3）。
  - 搜索筛选条件：`keyword_search`，以及根据固定列表验证的 YouTube 筛选标签。"This month"、"Video" 和 "Under 4 minutes" 可通过；`duration` "4-20 minutes" 会返回 400，提示 "This value is not allowed"。
  - 话题标签：`hashtag` 不含 #，`num_of_posts` 为数字。
  - 评论：`num_of_comments`，可选参数为 `sort_by`、`load_replies`。
  - `bdata pipelines info <dataset_id> <method>` 会打印实时输入架构。
- **频道视频（通过 `url` 发现）存在上游故障。** 2026-11-09，针对 @MrBeast 和 @NASA 的三次 `/videos` 运行均在约九分钟后返回六行中的六行 `parse_error`，错误为 "Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined"。README 展示了该输出。当它恢复返回正常数据行后，请重新运行 README 代码块并替换所示输出。
- 视频记录中的 `description` 可能属于另一个视频："Me at the zoo" 返回的描述内容是关于微塑料的。在此问题修复前，切勿在 README 输出中显示 `description`。
- 视频记录包含 `transcript`、`formatted_transcript`（以毫秒为单位的开始和结束时间）以及 `chapters`。转录文本包含换行符。
- 架构可能随时更改。切勿硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此仓库中工作

- `pytest` 可离线运行且不需要令牌。`ruff check .` 必须通过。
- CI 会在空白机器上执行 README 中的命令，并通过准确的 H2 文本查找对应章节。如果重命名标题，也要在 ci.yml 中同步修改。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则运行较小规模的实时检查。代码片段必须完整且可单独粘贴运行，展示的输出必须真实。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，每天由 `.github/scripts/field_tables.py` 根据 https://docs.brightdata.com/scrapers-full.json 重新生成。请勿手动编辑。
- README 顶部的“上次验证”徽章行由每日运行自动重写。请勿手动编辑。
- 标题针对搜索进行了优化，例如“YouTube Python 爬虫工具”“如何抓取 YouTube……”。添加章节时请保持这一模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发功能。
