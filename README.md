# YouTube Python 爬虫工具

使用 Python 将 YouTube 视频、转录、Shorts、评论和频道抓取为 JSON。无需 YouTube Data API 密钥，不受配额限制。基于 [Bright Data YouTube 爬虫工具 API](https://www.bright.cn/products/web-scraper/youtube?utm_source=github&utm_medium=referral&utm_content=youtube-scraper) 构建。新账户每月可获得 5,000 点免费额度。

[![实时检查](https://github.com/bright-cn/youtube-scraper/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/youtube-scraper/actions/workflows/live.yml) [![上次验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/youtube-scraper/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-python-抓取-youtube-视频) · [无需 API 密钥](#无需-youtube-data-api-即可抓取-youtube) · [CLI](#批量-youtube-搜索爬虫-cli) · [端点](#抓取-youtube-视频转录shorts频道和评论) · [转录](#抓取-youtube-视频转录) · [数据](#youtube-视频数据字段json) · [错误](#youtube-爬虫工具错误) · [MCP 和 Claude Code](#youtube-mcp-服务器和-claude-code-技能) · [免费额度](#这个-youtube-爬虫工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/youtube/introduction) · [支持](#支持)

使用 Python 将 YouTube 数据转换为 JSON：包括带观看次数、点赞数、转录和章节的视频及 Shorts、搜索结果、话题标签、频道、评论和播客剧集。涵盖全部三个 Bright Data YouTube 爬虫工具及其十种方法，每种方法对应一个函数。无需 YouTube Data API 密钥，不受配额限制，无需浏览器，也没有依赖。基于 [Bright Data YouTube 爬虫工具 API](https://www.bright.cn/products/web-scraper/youtube?utm_source=github&utm_medium=referral&utm_content=youtube-scraper) 构建。

此外还提供：用于关键词搜索的单命令 CLI，以及完全不需要 Python 的 [Bright Data CLI](#youtube-mcp-服务器和-claude-code-技能)。

## 如何使用 Python 抓取 YouTube 视频

需要 Python 3.10 或更高版本，无需安装任何内容：这里仅使用标准库。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得[5,000 点免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
import json
import os
import time
import urllib.parse
import urllib.request

API = "https://api.brightdata.com/datasets/v3"
HEADERS = {
    "Authorization": f"Bearer {os.environ['BRIGHTDATA_API_TOKEN']}",
    "Content-Type": "application/json",
}


def call(path, body=None):
    data = None if body is None else json.dumps(body).encode()
    request = urllib.request.Request(API + path, data=data, headers=HEADERS)
    with urllib.request.urlopen(request) as response:
        return json.load(response)


query = urllib.parse.urlencode({
    "dataset_id": "gd_lk56epmy2i5g7lzu0k",
    "type": "discover_new",
    "discover_by": "keyword",
    "limit_per_input": 3,
    "include_errors": "true",
})
job = call(f"/trigger?{query}", [{"keyword": "python tutorial", "num_of_posts": "3"}])
while call(f"/progress/{job['snapshot_id']}")["status"] not in ("ready", "failed"):
    time.sleep(10)
for video in call(f"/snapshot/{job['snapshot_id']}?format=json"):
    if "error" in video:
        print("error row:", video["error"])
        continue
    print(video["views"], video["likes"], video["title"][:50])
```

```
error row: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
8099775 172885 Python Full Course for Beginners
13396612 377300 Python Full Course for free 🐍
```

预计需要两到十一分钟：API 会运行搜索任务，循环则等待任务完成。按[每个视频](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中有三项内容并非可选。

`limit_per_input` 会限制运行规模和费用。`num_of_posts` 请求相同的数量；在此爬虫工具中，它是文本字段，因此会以 `"3"` 发送。

`include_errors=true` 会让 API 将无法采集的视频报告为包含 `error` 键的数据行。API 默认关闭此选项，因此失败的搜索看起来会与没有结果的搜索相同。

之所以包含 `error row` 分支，是因为在 2026 年 11 月 9 日，某次搜索中的部分视频返回了 "Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined"，而其余视频则完整返回。请参阅 [YouTube 爬虫工具错误](#youtube-爬虫工具错误)。

## 无需 YouTube Data API 即可抓取 YouTube

YouTube 爬虫工具 API 不需要 Google Cloud 项目、YouTube Data API 密钥或 OAuth，也没有每日配额。唯一凭据是 Bright Data API 密钥，通过 `BRIGHTDATA_API_TOKEN` 设置。此软件包及其命令也能找到由 `bdata login` 存储的密钥。它会采集未登录访问者可看到的内容，包括转录。

## 批量 YouTube 搜索爬虫 CLI

此仓库中的命令会同时针对多个关键词运行 Python 快速入门，并写入一个 JSON 文件。

```bash
pip install git+https://github.com/bright-cn/youtube-scraper
youtube-scraper "python tutorial" "home workout" --limit 2
```

```
Finding up to 2 videos per keyword, for: python tutorial, home workout
Usually two to ten minutes each. Billed per video, 5,000 free credits a month.
asking  'python tutorial'...
got     'python tutorial': 1 videos, 1 error rows: Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
asking  'home workout'...
got     'home workout': 2 videos

Saved 3 videos as JSON to youtube.json (51 fields per video)
```

在终端中，`asking` 行会被原地更新的旋转进度指示器和计时器替代，因此你可以看到任务正在运行以及已运行多长时间。

```
--limit N      videos per keyword, default 5, minimum 1
--out PATH     output file, default youtube.json
```

`python -m youtube_scraper` 同样可用。该命令会在 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你曾运行过一次 `npx -p @brightdata/cli bdata login`，则会从保存的登录信息中查找，供你以及该终端中的任何编码智能体使用。

你也可以导入它而不是直接运行，以便按关键词获取 `ok`、`note` 和 `error`，而不是原始数据行。`scrape` 不会因单个错误关键词抛出异常；读取 `videos` 前请检查 `ok`：

```python
from youtube_scraper import scrape

for outcome in scrape(["python tutorial", "home workout"], limit=1):
    if outcome.ok:
        video = outcome.videos[0]
        print(f"{outcome.keyword}: {video['title'][:45]} ({video['views']} views)")
    else:
        print(f"{outcome.keyword} failed: {outcome.error}")
```

```
python tutorial: Curso de PYTHON desde CERO (Completo) (6251934 views)
home workout: 25 MIN FULL BODY HIIT for Beginners - No Equi (34828112 views)
```

只有当某个关键词返回的每一行都是错误行时，该关键词才算失败。如果搜索同时返回部分视频和部分错误行，视频会保留，错误则会存入 `note`。

## 抓取 YouTube 视频、转录、Shorts、频道和评论

Bright Data 提供三个 YouTube 爬虫工具，分别用于视频、频道和评论，共有十种调用方式，每一种都记录在 [YouTube 爬虫工具 API 文档](https://docs.brightdata.com/api-reference/scrapers/social-media-apis/youtube-videos-collect-by-url)中。此软件包为每种方式提供一个函数。以下每个代码片段都是完整的：在上方的 `pip install` 后按原样粘贴即可。它们每周一都会在 Actions 中运行，规模较小的检查则每隔一天运行一次，顶部徽章会显示最新结果。

| 你已有 | 你想要 | 方法 |
| --- | --- | --- |
| 视频或 Shorts URL | 观看次数、点赞数、转录、章节 | `videos(urls)` |
| 类似 "python tutorial" 的搜索 | 匹配的视频 | `search("python tutorial", 5)` |
| 搜索和 YouTube 筛选条件 | 经过筛选的视频 | `search_filters("python tutorial", 5, upload_date="This month")` |
| 话题标签 | 相关视频和 Shorts | `hashtag("baking", 5)` |
| 视频 URL | 相关评论 | `comments(urls, 10)`，按每条评论计费 |
| 频道 URL | 订阅者数、观看次数、视频数量 | `channels(urls)` |
| 关键词 | 匹配的频道 | `channels_search("home cooking", 5)` |
| 播客播放列表 URL | 其中的剧集 | `podcast(url, 5)` |
| 探索页面 URL | 其中的视频 | `explore("https://www.youtube.com/gaming/games", 5)` |
| 频道的 `/videos` URL | 其最近的视频 | `channel_videos(url, 5)`，2026 年 11 月 9 日上游出现故障 |

每个函数都接受一个 URL 或 URL 列表。列表会作为一个任务处理，而不是每个 URL 一个任务。每个函数都会返回数据行列表，API 无法采集的数据行会包含 `error` 键，而不是记录。接受数量限制的函数会将 API 支持的任何其他输入作为关键词参数传递，并使用 API 中的原始名称：`search("python tutorial", 5, start_date="10-01-2026")`。

### 通过 URL 抓取 YouTube 视频

```python
from youtube_scraper import videos

video = videos("https://www.youtube.com/watch?v=jNQXAC9IVRw")[0]
print(video["title"], "|", video["youtuber"], "|", video["date_posted"][:10])
print(video["views"], "views,", video["likes"], "likes,", video["num_comments"], "comments")
print(video["video_length"], "seconds,", video["category"], "| tags:", ", ".join(video["tags"]))
```

```
Me at the zoo | @jawed | 2005-04-24
448085592 views, 20132396 likes, 10646129 comments
19 seconds, Film & Animation | tags: me at the zoo, jawed karim, first youtube video
```

通过 URL 采集视频耗时 12 到 178 秒。Shorts URL 的工作方式相同。

### 抓取 YouTube 视频转录

```python
from youtube_scraper import videos

url = "https://www.youtube.com/watch?v=jNQXAC9IVRw"
video = videos(url, transcription_language="English")[0]
for line in video["formatted_transcript"][:3]:
    text = " ".join(line["text"].split())
    print(f"{line['start_time'] / 1000:5.1f}s  {text}")
print(len(video["transcript"]), "characters in all")
```

```
  1.2s  All right, so here we are, in front of the elephants
  5.3s  the cool thing about these guys is that they have really...
  8.0s  really really long trunks
217 characters in all
```

每条视频记录都包含两种形式的转录：`transcript` 是纯文本，`formatted_transcript` 是带时间的文本行，`start_time` 和 `end_time` 以毫秒为单位。`transcript_language` 列出 YouTube 提供的语言，`chapters` 列出视频章节。文本会保留 YouTube 的换行符。

### 使用筛选条件抓取 YouTube 搜索结果

```python
from youtube_scraper import search_filters

found = search_filters(
    "python tutorial", 2, upload_date="This month", type="Video", duration="Under 4 minutes"
)
for video in found:
    if "error" in video:
        print("error row:", video["error"])
        continue
    print(video["date_posted"][:10], video["video_length"], "s |", video["title"][:50])
```

```
2026-10-12 79 s | Python for Beginners: Complete Tutorial & Practice
2026-10-23 82 s | Python Tutorial: Functions and Parameter Passing
```

筛选条件采用 YouTube 自己的标签，并根据固定列表进行验证。"This month"、"Video" 和 "Under 4 minutes" 会被接受；"4-20 minutes" 会返回 HTTP 400，`["duration", "This value is not allowed"]`。

### 通过话题标签抓取 YouTube Shorts

```python
from youtube_scraper import hashtag

for video in hashtag("baking", 2):
    if "error" in video:
        print("error row:", video["error"])
        continue
    print(video["post_type"], video["views"], video["url"])
```

```
post 5183839 https://www.youtube.com/watch?v=zdpxBdtBL4w
post 3349958 https://www.youtube.com/watch?v=3NvvOVrm1uY
```

传入不带 `#` 的标签。`start_date` 和 `end_date`、`MM-DD-YYYY` 可将范围缩小到指定时间窗口。

### 从视频中抓取 YouTube 评论

```python
from youtube_scraper import comments

for comment in comments("https://www.youtube.com/watch?v=jNQXAC9IVRw", 3):
    print(comment["likes"], "|", comment["username"], "|", comment["comment_text"][:50])
```

```
4800000 | @SanDiegoZoo | We're so honored that the first ever YouTube video
1 | @roshac995 | 10M comment
1 | @دمینېنغمېپښتو | Happy 21years youtube
```

数量限制由 `num_of_comments` 输入控制：请求 3 条，43 秒内返回 3 条。`sort_by` 和 `load_replies` 作为关键词参数传入。

### 抓取 YouTube 频道数据

```python
from youtube_scraper import channels

channel = channels("https://www.youtube.com/@NASA/about")[0]
print(channel["name"], channel["handle"])
print(channel["subscribers"], "subscribers,", channel["videos_count"], "videos,", channel["views"], "views")
```

```
NASA @NASA
15100000 subscribers, 6201 videos, 1134859779 views
```

速度最快的调用：12 秒。

### 通过关键词查找 YouTube 频道

```python
from youtube_scraper import channels_search

for channel in channels_search("home cooking", 2):
    print(channel["subscribers"], channel["name"], channel["url"])
```

```
3300000 HomeCookingShow https://www.youtube.com/@homecookingshow
1400000 HomeCooking Tamil https://www.youtube.com/@homecookingtamil
```

### 抓取 YouTube 播客剧集

```python
from youtube_scraper import podcast

playlist = "https://www.youtube.com/playlist?list=PLbnMTcZEga8QCKJvxsJVQV4uHD6bENdnP"
for episode in podcast(playlist, 2):
    print(episode["date_posted"][:10], episode["views"], "|", episode["title"][:60])
```

```
2026-11-09 7455 | Are you risking your career if you post about work on social
2026-11-02 16738 | She gave all her employees flexible work. Here’s how she kep
```

使用播客播放列表 URL 或频道的 `/podcasts` URL。Python SDK 没有用于此功能的方法。

### 抓取 YouTube 探索和游戏页面

```python
from youtube_scraper import explore

for video in explore("https://www.youtube.com/gaming/games", 2):
    print(video["live_status"], video["views"], "|", video["title"][:50])
```

```
is_live 3974 | ADDRESSING THE AGENDA - AMPHOREUS WATCHPARTY 1900 
is_live 4698 | 【#崩壊スターレイル 】アプデもうくる…！アッハのPV初見同時視聴！【 Vtuber/ 白亜リラ 】
```

游戏页面主要是直播，因此 `live_status` 为 `is_live`，`views` 统计截至当前的直播数据。

### 抓取 YouTube 频道的视频

```python
from youtube_scraper import channel_videos

for row in channel_videos("https://www.youtube.com/@NASA/videos", 2):
    if "error" in row:
        print("error:", row["error_code"], row["error"])
    else:
        print(row["date_posted"][:10], row["title"][:60])
```

```
error: parse_error Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
error: parse_error Parse error: ORIGINAL_LANGUAGE_TO_COUNTRY is not defined
```

该输出是真实结果。2026 年 11 月 9 日，测试的每个频道（@NASA 和 @MrBeast，共运行五次）都在约九分钟后仅返回此错误。调用和输入与实时架构一致，每周检查也会运行此代码块，因此恢复后徽章会显示结果。在此之前，可使用带频道名称的 `search`，或对已知 URL 使用 `videos`，以获取相同记录。

### 立即触发，稍后获取

对于规模超过少量搜索的任务，不要让进程阻塞一小时。触发任务，保存快照 ID，准备好后再获取。快照可下载 30 天。

```python
import time

from youtube_scraper import CHANNELS, fetch, status, trigger

snapshot = trigger(CHANNELS, [{"url": "https://www.youtube.com/@NASA/about"}])
print("snapshot:", snapshot)
while (state := status(snapshot)) not in ("ready", "failed"):
    time.sleep(10)
print("status:", state)
print("fetched:", fetch(snapshot)[0]["name"])
```

```
snapshot: sd_mw95rfbd114h5umqne
status: ready
fetched: NASA
```

`trigger` 接受任何数据集、`discover_by` 和 `limit_per_input`，因此可调用表中的每种方法。以下每项输入均已于 2026-11-09 通过实时 API 验证：

| 方法 | dataset_id | discover_by | 输入 |
| --- | --- | --- | --- |
| 通过 URL 获取视频 | `gd_lk56epmy2i5g7lzu0k` | 无 | `{"url"}`，可选 `{"country", "transcription_language"}` |
| 搜索 | `gd_lk56epmy2i5g7lzu0k` | `keyword` | `{"keyword", "num_of_posts"}`，`num_of_posts` 作为文本 |
| 带筛选条件的搜索 | `gd_lk56epmy2i5g7lzu0k` | `search_filters` | `{"keyword_search"}` 加筛选标签 |
| 话题标签 | `gd_lk56epmy2i5g7lzu0k` | `hashtag` | `{"hashtag", "num_of_posts"}` |
| 频道视频 | `gd_lk56epmy2i5g7lzu0k` | `url` | `{"url", "num_of_posts"}`，一个 `/videos` URL |
| 探索页面 | `gd_lk56epmy2i5g7lzu0k` | `explore` | `{"url"}`，可选 `{"all_tabs"}` |
| 播客 | `gd_lk56epmy2i5g7lzu0k` | `podcast_url` | `{"url"}` |
| 通过 URL 获取频道 | `gd_lk538t2k2p1k3oos71` | 无 | `{"url"}` |
| 通过关键词获取频道 | `gd_lk538t2k2p1k3oos71` | `keyword` | `{"keyword"}` |
| 评论 | `gd_lk9q0ew71spt1mxywf` | 无 | `{"url", "num_of_comments"}` |

`bdata pipelines info <dataset_id> <method>` 会打印 API 验证的输入字段。

## YouTube 视频数据字段（JSON）

大多数人需要的视频字段：

```
url  title  youtuber  views  likes  num_comments  date_posted  video_length  transcript  tags
```

代码未硬编码任何字段列表。API 返回的任何内容都会进入每条数据行以及命令生成的文件。

<!-- fields:start -->
每天根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。记录会包含适用于它的字段，以及 `timestamp` 和 `input`；示例文件中的第一个视频有 51 个键。

<details>
<summary>全部 49 个视频字段，<code>gd_lk56epmy2i5g7lzu0k</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | URL | 已采集视频的完整 YouTube 观看 URL |
| `title` | 文本 | YouTube 视频的标题 |
| `youtuber` | 文本 | 从频道浏览端点提取的频道用户名 |
| `youtuber_md5` | 文本 | youtuber 字段的 MD5 哈希值 |
| `video_url` | URL | 从 YouTube 自适应格式中提取或通过 signatureCipher 解码得到的视频文件直接流式传输 URL |
| `video_length` | 数字 | 视频时长，以秒为单位 |
| `likes` | 数字 | 视频获得的点赞数 |
| `views` | 数字 | 视频获得的观看次数 |
| `date_posted` | 日期 | 视频发布日期 |
| `description` | 文本 | 视频随附的文本描述 |
| `num_comments` | 数字 | YouTube 显示的视频评论数（YouTube 的计数包括顶级评论和回复） |
| `subscribers` | 数字 | YouTube 频道的订阅者数量 |
| `music` | 对象 | 视频中使用的音乐信息。仅当 YouTube Content ID 系统识别出视频中的音乐时才会出现 |
| `video_id` | 文本 | YouTube 视频的唯一标识符 |
| `channel_url` | URL | 根据频道规范浏览端点生成的、经过 URL 编码的 YouTube 频道 URL |
| `preview_image` | 图像 | 视频预览图的 URL |
| `shortcode` | 文本 | 视频的唯一短标识符，与 video_id 相同，用于所有 YouTube 视频类型，包括普通视频和 Shorts |
| `verified` | 布尔值 | 表示频道是否已通过验证 |
| `handle_name` | 文本 | 视频页面上显示的 YouTube 频道名称 |
| `avatar_img_channel` | URL | YouTube 频道头像图片的 URL |
| `is_sponsored` | 布尔值 | 表示视频是否包含 YouTube 声明的付费内容叠加层 |
| `related_videos` | 数组 | 当前视频旁边侧边栏中显示的视频 URL 列表 |
| `license` | URL | YouTube 视频元数据中声明的视频许可证 URL |
| `viewport_frames` | 文本 | 根据 YouTube 流媒体数据得出的主要视频流格式分辨率 |
| `current_optimal_res` | 文本 | 来自 YouTube 流媒体格式元数据的当前和最佳视频分辨率及帧率 |
| `codecs` | 文本 | 从 YouTube 自适应流格式中提取的视频和音频编解码器字符串，格式为视频编解码器和音频编解码器 |
| `color` | 文本 | 从自适应格式颜色信息中提取的视频流色彩原色 |
| `quality` | 文本 | YouTube 报告的第一个自适应流格式的质量标识符 |
| `quality_label` | 文本 | YouTube 报告的易读视频质量标签 |
| `post_type` | 文本 | 根据 URL 结构推断的 YouTube 内容类型。可能的值：标准视频为 post，YouTube Short 为 short，或 N/A |
| `youtuber_id` | 文本 | YouTube 内部频道 ID，与频道用户名不同 |
| `transcript` | 文本 | 根据 YouTube 字幕轨道整理的视频完整纯文本转录。优先使用英文字幕；如果没有英文，则回退到第一个可用语言 |
| `formatted_transcript` | 数组 | 来自 YouTube 字幕轨道的带时间戳转录片段，每个片段包含以毫秒为单位的开始时间、结束时间和持续时间，以及字幕文本 |
| `hashtags` | 数组 | 视频标题或描述中出现的话题标签列表，每项包含话题标签文本及其 YouTube 搜索链接 |
| `tags` | 数组 | 视频标签 |
| `next_recommended_videos` | 数组 | 视频最后几秒内作为结束画面卡片显示的视频，包括以毫秒为单位的时间和显示样式 |
| `recommended_videos` | 数组 | YouTube 在当前视频旁边的侧边栏中推荐的视频 |
| `transcript_language` | 数组 | YouTube 上该视频可用的字幕和转录语言列表，包括每个轨道是否为自动生成 |
| `chapters` | 数组 | 创作者定义的章节标记列表，每项包含标题、以毫秒为单位的开始时间和预览图 |
| `transcription_language` | 文本 | 调用方请求用于获取转录的语言代码，在采集时作为输入参数传入 |
| `is_age_restricted` | 布尔值 | 视频是否有年龄限制，根据非 OK 的可播放状态以及 YouTube 提供的年龄相关原因推断 |
| `channel_url_decoded` | 文本 | 频道 URL 经过百分号解码后的纯文本版本 |
| `category` | 文本 | YouTube 从预定义类别分类体系中分配给视频的内容类别 |
| `audio_tracks` | 数组 | 音轨 |
| `made_for_kids` | 布尔值 | 表示视频是否被指定为儿童内容，根据 YouTube 迷你播放器的播放模式推断 |
| `player_notification_text` | 文本 | 视频播放器内向用户显示的通知文本 |
| `ai_content_label` | 布尔值 | 表示视频是否被创作者标记或被 YouTube 自动检测为 AI 生成或经过大幅 AI 修改。 |
| `live_status` | 文本 | 视频的直播状态，等同于 yt-dlp 的 live_status。值为 "is_live"、"was_live" 或 "not_live" 之一，根据 YouTube 的信息字符串（例如 "Streamed live 29 minutes ago"）解析；普通视频默认为 "not_live" |
| `comments` | 数组 | 视频最相关的 10 条评论 |

</details>

<details>
<summary>全部 21 个频道字段，<code>gd_lk538t2k2p1k3oos71</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | URL | 直接链接到 YouTube 个人资料的 URL |
| `handle` | 文本 | 每个用户的唯一标识（例如 @name） |
| `handle_md5` | 文本 | 此字段存储 handle 字段的 MD5 哈希值。 |
| `banner_img` | URL | 横幅图片的 URL |
| `profile_image` | URL | 个人资料图片的 URL |
| `name` | 文本 | 个人资料名称 |
| `subscribers` | 数字 | 个人资料的订阅者数量 |
| `Description` | 文本 | 用户在 YouTube 上介绍自己或分享兴趣与爱好时编写的简短描述或陈述 |
| `videos_count` | 数字 | 个人资料拥有的视频数量 |
| `created_date` | 日期 | 个人资料创建日期 |
| `views` | 数字 | 个人资料获得的观看次数 |
| `Details` | 对象 | 频道详情 |
| `Links` | 数组 | 用户可能在 YouTube 个人资料中链接的外部网站 |
| `identifier` | 文本 | 标识符 |
| `discovery_input` | 对象 | 发现请求的输入值 |
| `id` | 文本 | 频道 ID |
| `has_podcast` | 布尔值 | 是否有播客的指示值 |
| `top_videos` | 数组 | 热门视频 |
| `featured_video` | URL | 精选视频 |
| `featured_channels` | 数组 | 精选频道 |
| `collaborations` | 数组 | 合作视频 |

</details>

<details>
<summary>全部 16 个评论字段，<code>gd_lk9q0ew71spt1mxywf</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `comment_id` | 文本 | 每条评论的唯一标识符 |
| `comment_text` | 文本 | 评论的文本内容 |
| `likes` | 数字 | 评论获得的点赞数 |
| `replies` | 数字 | 评论的回复数 |
| `username` | 文本 | 评论者的用户名 |
| `username_md5` | 文本 | 此字段存储 username 字段的 MD5 哈希值。 |
| `user_channel` | 文本 | 评论者 YouTube 频道的 URL |
| `date` | 文本 | 评论发布日期 |
| `url` | URL | 发布该评论的 YouTube 视频网址 |
| `video_id` | 文本 | YouTube 视频的唯一标识符 |
| `replies_value` | 数组 | 与此评论关联的回复对象数组 |
| `replies_without_names` | 数组 | 已移除个人身份信息（用户名）的回复对象数组，仅包含经过 MD5 哈希处理的用户标识符 |
| `user_id` | 文本 | 评论者 YouTube 频道的唯一标识符 |
| `mentioned_timestamps_minutes` | 数组 | 评论中提到的时间戳数组，采用分:秒格式（例如 '2:35'） |
| `mentioned_timestamps_seconds` | 数组 | 评论中提到的时间戳数组，转换为总秒数 |
| `date_iso` | 日期 | ISO 格式的日期 |

</details>
<!-- fields:end -->

<details>
<summary>来自 <code>youtube-scraper "python tutorial" --limit 1</code> 的真实输出文件开头</summary>

```json
{
  "generated_at": "2026-11-09T11:11:27.087936+00:00",
  "keywords": [
    {
      "keyword": "python tutorial",
      "videos": [
        {
          "url": "https://www.youtube.com/watch?v=ix9cRaBkVe0&pp=ygUPcHl0aG9uIHR1dG9yaWFs",
          "title": "Python Full Course for free 🐍",
          "youtuber": "@BroCodez",
          "youtuber_md5": "70d4b03c4bfb5ad224f20557ab307492",
          "video_url": "https://rr2---sn-gjo-w43s.googlevideo.com/videoplayback?expire=1794244154&ei=2qnxavSNINSRr9gP69bsuAg&ip=198.171.211.118&id=o-AEkMpU30iXwNtnvbnAON2eG7AttyInlAGwwG01pqa8vk&source=youtube&requiressl=yes&xpc=EgVo2aDSNQ%3D%3D&met=1794222554%2C&mh=Zm&mm=18%2C29&mn=sn-gjo-w43s%2Csn-qjp5q5-5c&ms=aub%2Crdu&mv=m&mvi=2&pl=20&rms=aub%2Caub&pcm2=no&initcwndbps=4588250&spc=mpRQ_yRcDVkz9LJzvmLderwqwDBiIUMpTYdAB7VfYGkrZ1qHmjDsEg04x0RuWVnOkP6pff4&svpuc=1&ns=J3JNrNRnfM1v_ggVbplpJE8Z&sabr=1&rqh=1&mt=1794222100&fvip=1&epbp=1&keepalive=yes&fexp=51565115%2C52250119%2C52323638&c=WEB&n=CEJtyzpbOfeJMl3fqr&sparams=expire%2Cei%2Cip%2Cid%2Csource%2Crequiressl%2Cxpc%2Cpcm2%2Cspc%2Csvpuc%2Cns%2Csabr%2Crqh&sig=AE0s2JYwRAIhAMhhV5p8KVeGw4he3AkzCNLLYPzPT4t486MJRurbxitQAh8iv4sOa7TLD6KtHYWPeneTG-0mmYQpfhE6P1dDN5LT&lsparams=met%2Cmh%2Cmm%2Cmn%2Cms%2Cmv%2Cmvi%2Cpl%2Crms%2Cinitcwndbps&lsig=APaTxxMwRQIgArVc0ZpH7fcKdlGNogRr8Klbk5W446L1nRcJnl4t5ccCIQD1KIwxicqzTiGivwMcCr0H0bipJUkyHA3pAPC6IRw2NA%3D%3D",
          "video_length": 43200,
          "likes": 377299,
          "views": 13396410,
          "date_posted": "2024-08-20T13:00:54.000Z",
          "description": "#python #tutorial #beginners \nPython tutorial for beginners' full course 2024\n\n*Learn Python in 1 HOUR* ⏱ : https://www.youtube.com/watch?v=8KCuHHeC_M0\n*My original Python 12 Hour course* 🐍 :  https://www.youtube.com/watch?v=XKHEtdqhLK8\n*Full Python playlist* 📃: https://www.youtube.com/watch?v=Sg4GMVMdOPo&list=PLZPZq0r_RZOOkUQbat8LyQii36cJf2SWT\n\n                           [⭐ = project]\n#1   (00:00:00) python tutorial for beginners 🐍\n#2   (00:05:49) variables ❎\n#3   (00:16:05) type casting 💱\n#4   (00:21:15) user input ⌨️\n#5   (00:32:42) ⭐ madlibs game 📖\n#6   (00:37:55) arithmetic & math 📐\n#7   (00:51:46) if statements 🤔\n#8   (01:00:06) ⭐ calculator program 🧮\n#9   (01:05:59) ⭐ weight conversion program 🏋️\n#10 (01:09:59) ⭐ temperature conversion program 🌡️\n#11 (01:13:58) logical operators 🌦️\n#12 (01:21:28) conditional expressions ❓\n#13 (01:27:03) string methods 〰️\n#14 (01:39:08) string indexing ✂️\n#15 (01:46:35) format specifiers 💬\n#16 (01:51:55) while loops ♾️\n#17 (01:58:53) ⭐ compound interest calculator 💵\n#18 (02:06:28) for loops 🔁\n#19 (02:11:33) ⭐ countdown timer program ⌛\n#20 (02:17:28) nested loops ➿\n#21 (02:23:03) lists, sets, and tuples 🍎\n#22 (02:38:08) ⭐ shopping cart program 🛒\n#23 (02:45:21) 2D collections ⬜\n#24 (02:53:59) ⭐ quiz game 💯\n#25 (03:03:27) dictionaries 📙\n#26 (03:11:33) ⭐ concession stand program 🍿\n#27 (03:19:42) random numbers 🎲\n#28 (03:24:16) ⭐ number guessing game 🔢\n#29 (03:32:37) ⭐ rock, paper, scissors game 🗿\n#30 (03:42:06) ⭐ dice roller program ⚂\n#31 (03:52:12) functions 📞\n#32 (04:02:50) default arguments 👍\n#33 (04:08:56) keyword arguments 🗝️\n#34 (04:15:40) *args & **kwargs 📦\n#35 (04:30:33) iterables 🔂\n#36 (04:37:04) membership operators 🔎\n#37 (04:45:56) list comprehensions 📃\n#38 (04:56:17) match-case statements 📆\n#39 (05:02:13) modules 📨\n#40 (05:08:51) scope resolution 🔬\n#41 (05:14:22) if name == 'main': 📥\n#42 (05:23:34) ⭐ banking program 💰\n#43 (05:38:34) ⭐ slot machine 🎰\n#44 (05:58:45) ⭐ encryption program 🔐\n#45 (06:07:26) ⭐ hangman game 🕺\n#46 (06:32:32) python object oriented programming 🚗\n#47 (06:44:50) class variables 🎓\n#48 (06:53:06) inheritance 👨‍👦‍👦\n#49 (07:00:02) multiple inheritance 🐟\n#50 (07:08:04) super() 🔴\n#51 (07:21:10) polymorphism 🎭\n#52 (07:29:15) duck typing 🦆\n#53 (07:33:34) static methods ⚡\n#54 (07:39:31) class methods 🏫\n#55 (07:46:16) magic methods 🌟\n#56 (07:59:51) @property ⚙️\n#57 (08:07:33) decorators 🎊\n#58 (08:14:57) exception handling 🚦\n#59 (08:20:46) file detection 🕵️‍♂️\n#60 (08:27:47) writing files ✍\n#61 (08:41:33) reading files 🔍\n#62 (08:48:29) dates & times 📅\n#63 (08:54:46) ⭐ alarm clock ⏰\n#64 (09:05:03) multithreading 🧵\n#65 (09:13:45) request API data ↩️\n#66 (09:22:19) PyQt5 GUI intro 🖥️\n#67 (09:31:27) PyQt5 labels 🏷️\n#68 (09:40:23) PyQt5 images 📷\n#69 (09:46:28) PyQt5 layout managers 🧲\n#70 (09:53:07) PyQt5 buttons 🛎️\n#71 (10:00:12) PyQt5 checkboxes ✅\n#72 (10:06:42) PyQt5 radio buttons 🔘\n#73 (10:15:55) PyQt5 line edits 💬\n#74 (10:21:55) PyQt5 CSS styles 🎨\n#75 (10:32:48) ⭐ digital clock program 🕒\n#76 (10:48:38) ⭐ stopwatch program ⏱\n#77 (11:06:05) ⭐ weather API app ☀️\n\nPython Interpreter: https://www.python.org/\nPyCharm IDE: https://www.jetbrains.com/pycharm/\n\n*Copyright Disclaimer*:\nThis video is the intellectual property of Bro Code. All rights reserved. No part of this video may be reproduced, distributed, or transmitted in any form or by any means, including but not limited to recording, uploading, or other electronic or mechanical methods, without my written permission, except in the case of brief quotations embodied in critical reviews and certain other noncommercial uses permitted by copyright law.",
          "subscribers": 3440000,
          "music": {
            "song": "Block Party",
  ...
```

包含一个视频及其所有字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## YouTube 爬虫工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发出任何请求前以状态码 2 退出。请设置令牌，或运行 `bdata login`。 |
| `failed  'keyword': Parse error: ...` | 以状态码 1 退出。该搜索中的每个视频都触发了爬虫工具中的解析错误。2026 年 11 月 9 日，该错误显示为 `ORIGINAL_LANGUAGE_TO_COUNTRY is not defined`，另一次为 `Cannot read properties of undefined (reading 'runs')`。请重新运行，或使用其他关键词。 |
| `got     'keyword': 1 videos, 1 error rows: Parse error: ...` | 以状态码 0 退出。同一错误影响了部分视频；其余视频已写入文件。 |
| `failed  'keyword': timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。请重新运行，或稍后使用 `fetch()` 获取快照。 |
| `failed  'keyword': HTTP 401: ...` | 令牌已设置，但不正确。 |

任何失败都会以状态码 1 退出，因此可安全地使用运行结果控制脚本是否继续执行。

通过函数调用时，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入，例如不在其列表中的筛选标签。消息会指出相应字段。 |
| 包含 `error` 键的数据行 | API 对单个视频或输入的响应：`parse_error`，或无法访问的页面。其他数据行不受影响。 |
| `TimeoutError` | 任务超过了 900 秒等待期限。将 `timeout=` 传递给 `run`，或使用 `trigger` 并稍后获取。 |

## YouTube MCP 服务器和 Claude Code 技能

无需 Python，也无需安装任何内容。粘贴以下两行；第一行会打开一次浏览器，也可以通过 SSH 或在 CI 中使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_lk538t2k2p1k3oos71 collect_by_url "https://www.youtube.com/@NASA/about" --pretty
```

1 条记录、20 个字段，2026 年 11 月 9 日耗时 35 秒。CLI 会省略值为 null 的字段。

CLI 会通过数据集 ID 运行爬虫工具，包括所有 YouTube 爬虫。`bdata pipelines search youtube` 会列出它们的 ID 和方法，`--limit-per-input N` 会限制发现数量，默认为 10。

`npx skills add brightdata/skills` 会向 Claude Code、Cursor 和 Codex 提供这些命令和文档，因此之后可以使用自然语言操作。完整指南：[为你的编码智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管助手、不方便使用终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行任何预构建爬虫工具，包括 YouTube 爬虫：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体也可以自行开设账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 支持的其他所有集成，从 LangChain 到 Zapier 和 n8n：[集成](https://docs.brightdata.com/integrations/introduction)。

<details>
<summary>CLI 的 YouTube 命令，均于 2026 年 11 月 9 日运行验证</summary>

在每条命令前添加 `npx -p @brightdata/cli`，或使用 `npm i -g @brightdata/cli` 安装一次。每条命令都会打印 JSON，并按记录计费。

三个爬虫工具，均使用 `bdata pipelines run <dataset_id> <method>` 运行：视频 `gd_lk56epmy2i5g7lzu0k`、频道 `gd_lk538t2k2p1k3oos71` 和评论 `gd_lk9q0ew71spt1mxywf`。

```bash
bdata pipelines run gd_lk9q0ew71spt1mxywf collect_by_url --url "https://www.youtube.com/watch?v=jNQXAC9IVRw" --num_of_comments 3 --format csv -o comments.csv
```

3 条评论、24 列，耗时 37 秒。

```bash
bdata pipelines run gd_lk56epmy2i5g7lzu0k discover_by_keyword --keyword "python tutorial" --num_of_posts 2 --limit-per-input 2 --timeout 900 --pretty
```

599 秒内返回 2 行：一条视频记录，以及一条包含 [YouTube 爬虫工具错误](#youtube-爬虫工具错误)中所述解析错误的数据行。发现任务可能超过 CLI 默认的 600 秒，因此请保留 `--timeout 900`，或添加 `--async`，稍后再使用 `bdata pipelines results <snapshot_id>` 获取。

`bdata pipelines info <dataset_id> <method>` 显示爬虫工具的输入字段。`bdata pipelines run --help` 显示 `--format json|csv|ndjson|jsonl`、`-o FILE`、`--timeout` 和 `--async`。

</details>

## 这个 YouTube 爬虫工具免费吗？

新的 Bright Data 账户每月可获得 [5,000 点免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。每个 YouTube 爬虫工具都按记录计费，即按视频、频道或评论计费。[定价](https://www.bright.cn/pricing/web-scraper)页面列出了费率。

## 支持

此仓库中的错误：[提交 issue](https://github.com/bright-cn/youtube-scraper/issues)，[CONTRIBUTING.md](CONTRIBUTING.md) 说明了需要提供哪些信息。关于 API、你的账户或额度的任何问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## YouTube Node.js 爬虫工具

同一款 YouTube 爬虫工具也提供 Node.js 版本：[brightdata/youtube-scraper-node](https://github.com/brightdata/youtube-scraper-node)。

## 更多 Bright Data 开源爬虫工具

采用相同方式构建并提供实时验证示例的其他 Python 爬虫工具：

- [TikTok Python 爬虫工具](https://github.com/brightdata/tiktok-scraper)
- [Instagram Python 爬虫工具](https://github.com/bright-cn/instagram-scraper-python)
- [ChatGPT Python 爬虫工具](https://github.com/brightdata/chatgpt-scraper)
