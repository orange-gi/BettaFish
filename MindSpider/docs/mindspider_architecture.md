# MindSpider 架构与行业扩展说明

## 1. 项目背景与模块总览
MindSpider 是一个围绕“热点发现 → 关键词抽取 → 多平台情感爬取”闭环设计的舆情采集系统，主仓库由以下模块组成：

- `BroadTopicExtraction/`：面向新闻热榜的采集与 AI 话题解析。核心子模块包括 `get_today_news.py`（多源热榜 API 聚合）、`topic_extractor.py`（基于 DeepSeek/OpenAI 的关键词摘要生成）以及 `database_manager.py`（写入 `daily_news`/`daily_topics`）。
- `DeepSentimentCrawling/`：对关键词进行跨平台搜索爬取，封装了 `keyword_manager.py`（关键词读取/回溯/兜底）、`platform_crawler.py`（对 MediaCrawler 的二次封装）以及 MediaCrawler 原生多平台实现（微博/知乎/抖音/快手/B站/贴吧/小红书等）。
- `schema/`：数据库初始化脚本，包含 MindSpider 自定义表与对 MediaCrawler 内容表的字段扩展。
- `main.py`：统一 CLI 入口，负责串联 `--broad-topic` 与 `--deep-sentiment`，并提供 `--complete` 一键执行。
- `config.py`：`pydantic-settings` 驱动的配置中心，支持 MySQL/PostgreSQL、DeepSeek/OpenAI 兼容 API 等参数读取。

辅助资产包括 `requirements.txt`（依赖锁定）、`img/`（示意图）、`docs/`（本说明）等。

## 2. 关键依赖
- **数据库层**：`pymysql`/`aiomysql`/`sqlalchemy` 负责 MySQL 连接，`asyncpg`/`psycopg[binary]` 对 PostgreSQL，`aiosqlite` 可用于本地轻量测试。
- **网络与调度**：`httpx`、`requests`、`aiofiles` 等满足多源热榜抓取与异步 I/O。
- **AI/数据处理**：`openai` SDK 配合 `DeepSeek` 或其他兼容模型；`pandas`、`numpy`、`regex`、`tqdm` 辅助结构化处理与分析。
- **MediaCrawler 相关**：`playwright` 负责浏览器自动化，`Pillow`/`opencv` 用于验证码/图片处理，`redis` 做缓存，`fastapi`/`uvicorn` 支持服务化接口，另有 `jieba`、`wordcloud`、`matplotlib` 做文本可视化。
- **通用工具**：`beautifulsoup4`、`lxml`、`loguru` 等。

## 3. 数据库与核心数据结构
Schema 由 `schema/mindspider_tables.sql` 定义，主要包含：

| 表 | 作用 | 关键字段 |
| --- | --- | --- |
| `daily_news` | 存储多源热榜新闻 | `news_id`（含 source+date）、`source_platform`、`rank_position`、`crawl_date` |
| `daily_topics` | AI 抽取的每日话题 | `topic_id`（带日期）、`keywords` JSON、`topic_description`、`processing_status` |
| `topic_news_relation` | 话题 ↔ 新闻的关联 | `relation_score`、`extract_date` |
| `crawling_tasks` | 多平台任务排程与统计 | `task_status`、`search_keywords` JSON、`total_crawled`/`success_count` |

此外对 MediaCrawler 原生表（`xhs_note`、`douyin_aweme`、`bilibili_video`、`weibo_note` 等）增加了 `topic_id`、`crawling_task_id` 字段，实现从话题到具体内容的全链路追踪，并提供 `v_topic_crawling_stats`、`v_daily_summary` 视图用于运营统计。

## 4. 数据流
### 4.1 BroadTopicExtraction
1. `NewsCollector` 调用 `newsnow.busiyi.world` 多个热榜端点（微博/知乎/B站/今日头条/抖音/GitHub/酷安/财联社/雪球等），统一落库到 `daily_news`。
2. `TopicExtractor` 将多源新闻摘要传入 DeepSeek/OpenAI，输出 JSON 结构的 `keywords + summary`，写入 `daily_topics`。
3. 关键词会被缓存到 `data/daily_keywords.txt`，为下游爬虫或调试提供直接引用。

### 4.2 DeepSentimentCrawling
1. `KeywordManager` 读取当日 `daily_topics`；若缺失则回溯最近 7 天或使用默认关键词表。
2. `PlatformCrawler` 将 MindSpider 数据库配置写入 MediaCrawler，并在 `config/base_config.py` 里动态注入批量关键词 / 保存方式 / 抓取配额。
3. `run_multi_platform_crawl_by_keywords` 为每个平台一次性执行 Playwright 自动化，并统计成功/失败任务、内容/评论数量。所有内容表记录都会带 `topic_id`、`crawling_task_id`，方便聚合。

### 4.3 CLI 编排
`python main.py --broad-topic` 触发阶段 1；`--deep-sentiment --platforms wb zhihu --max-keywords 30 --max-notes 40` 触发阶段 2；`--complete --test` 在测试配额下串行跑完整流程。`--date`、`--platforms`、`--keywords-count`、`--max-keywords`、`--max-notes` 提供按需调参能力。

## 5. 新能源车 / 充电桩 / AI 话题支持
1. **源头覆盖**：由于热榜源包含 `wallstreetcn`、`cls-hot`、`xueqiu` 等偏财经/产业的平台，新能源车、充电桩、AI 相关资讯一旦登上热榜即可被 `BroadTopicExtraction` 捕获。可通过 `python BroadTopicExtraction/main.py --sources wallstreetcn cls-hot xueqiu` 提升这类内容权重。
2. **LLM 抽取自适应**：`TopicExtractor` 的提示词要求返回“适合社交媒体检索的关键词”，因此当新闻标题中出现“新能源车”“充电桩”“人工智能”等词时会自然落入关键词列表，并被下游多平台搜索复用。
3. **兜底策略**：若当日关键词不足，`KeywordManager` 会回溯最近 7 天或回落到默认词表；可以在 `_get_default_keywords` 中加入自定义行业词，或直接在 `daily_topics` 表中写入特定关键词，实现强制覆盖。
4. **多平台落地**：一旦关键词包含新能源/AI 主题，`run_multi_platform_crawl_by_keywords` 会在小红书、抖音、快手、B站、微博、贴吧、知乎全面检索帖子/评论，实现对行业舆情的全渠道采集。
5. **实践建议**：先运行 `python main.py --broad-topic --sources wallstreetcn cls-hot xueqiu --keywords-count 80` 确认 `daily_topics` 含目标关键词，再执行 `python main.py --deep-sentiment --platforms wb zhihu bili --max-keywords 20 --max-notes 30 --test` 校验抓取链路是否完整。必要时可在 `TopicExtractor._build_analysis_prompt` 强调新能源/AI 优先级，以提高识别率。

---
如需继续扩展行业覆盖，可在 `get_today_news.py` 中添加新的行业资讯源，或在 `MediaCrawler/media_platform/` 下实现对应平台的登录与抓取逻辑，从而保持“发现—分析—爬取—落库”流程的一致性。
