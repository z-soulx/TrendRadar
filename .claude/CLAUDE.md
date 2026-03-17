# CLAUDE.md — TrendRadar AI Context

## Project Overview

TrendRadar is a Python-based hot news aggregator and analysis tool (v6.5.0). It crawls trending topics from multiple platforms (via NewsNow API), fetches RSS feeds, applies keyword or AI-based filtering, optionally runs AI analysis/translation, generates HTML reports, and delivers notifications through 9+ channels (Feishu, DingTalk, WeChat, Telegram, Email, ntfy, Bark, Slack, Webhook).

## Tech Stack

- **Language**: Python ≥ 3.10
- **Build**: Hatch (pyproject.toml)
- **Key Dependencies**: requests, PyYAML, feedparser, litellm (100+ AI providers), fastmcp 2.x, boto3 (S3), json-repair, tenacity, websockets
- **Storage**: SQLite (local) + S3-compatible (R2, OSS, COS) for remote
- **MCP Server**: FastMCP 2.0, supports stdio and HTTP transport

## Repository Structure

```
TrendRadar/
├── trendradar/                # Main Python package
│   ├── __init__.py            # __version__ = "6.5.0"
│   ├── __main__.py            # CLI entry: NewsAnalyzer.run()
│   ├── context.py             # AppContext — central config/service hub
│   ├── ai/                    # AI analysis, translation, filter (via LiteLLM)
│   ├── core/                  # Config loader, scheduler, frequency parser, analyzer
│   ├── crawler/               # Hotlist fetcher (NewsNow) + RSS fetcher
│   ├── storage/               # StorageManager, SQLite backend, remote S3 sync
│   ├── notification/          # NotificationDispatcher, channel senders, formatters
│   ├── report/                # HTML/text report generation
│   └── utils/                 # Time, URL helpers
├── mcp_server/                # MCP server for AI tool integration
│   ├── server.py              # FastMCP app with resources + tools
│   ├── tools/                 # data_query, analytics, search, config, notification, etc.
│   ├── services/              # cache, data, parser services
│   └── utils/                 # date parser, validators, errors
├── config/                    # Configuration files
│   ├── config.yaml            # Main config (platforms, notification, storage, AI, RSS)
│   ├── timeline.yaml          # Schedule templates (always_on, morning_evening, etc.)
│   ├── frequency_words.txt    # Keyword groups, filters, regex patterns
│   ├── ai_interests.txt       # AI interest descriptions for filtering
│   ├── ai_analysis_prompt.txt # AI analysis prompt template
│   ├── ai_translation_prompt.txt
│   ├── ai_filter/             # AI filter prompt files
│   └── custom/                # User custom configs (ai/, keyword/)
├── docker/                    # Docker setup (Dockerfile, compose, manage.py)
├── docs/                      # Web-based config editor (index.html)
├── .github/workflows/         # GitHub Actions (crawler, clean, docker)
├── output/                    # Generated data (SQLite, HTML, TXT)
├── pyproject.toml             # Build config, entry points
├── requirements.txt
└── version                    # Version file (6.5.0)
```

## Architecture & Data Flow

```
Config Loading → Crawl (Hotlist + RSS) → Store (SQLite/S3)
    → Filter (Keyword | AI) → Analyze (AI optional)
    → Report (HTML) → Notify (9+ channels)
```

### Core Components

| Component | Location | Responsibility |
|---|---|---|
| `AppContext` | `trendradar/context.py` | Central hub: config, storage, scheduler, report, notification |
| `NewsAnalyzer` | `trendradar/__main__.py` | Main orchestrator: crawl → filter → analyze → report → notify |
| `DataFetcher` | `trendradar/crawler/fetcher.py` | Fetches hotlist data from NewsNow API |
| `RSSFetcher` | `trendradar/crawler/rss/fetcher.py` | Fetches and parses RSS feeds |
| `StorageManager` | `trendradar/storage/manager.py` | SQLite + S3 storage backend |
| `Scheduler` | `trendradar/core/scheduler.py` | Time-based scheduling via timeline.yaml |
| `AIAnalyzer` | `trendradar/ai/analyzer.py` | AI-powered content analysis (via LiteLLM) |
| `AIFilter` | `trendradar/ai/filter.py` | AI interest-based filtering with tag system |
| `AITranslator` | `trendradar/ai/` | AI-powered translation |
| `NotificationDispatcher` | `trendradar/notification/dispatcher.py` | Multi-channel notification delivery |
| MCP Server | `mcp_server/server.py` | FastMCP 2.0 tools for AI clients |

### Report Modes

| Mode | Behavior |
|---|---|
| `daily` | Full day summary, includes all matched items + new items section |
| `current` | Current hotlist snapshot, shows currently trending items |
| `incremental` | Only new items since last run, no repeats |

### Filter Strategies

| Strategy | Config | Behavior |
|---|---|---|
| `keyword` | `filter.method: keyword` | Match against `frequency_words.txt` keyword groups |
| `ai` | `filter.method: ai` | AI classifies news against interest tags from `ai_interests.txt` |

## Entry Points

| Entry | Command | Description |
|---|---|---|
| CLI | `python -m trendradar` | Full pipeline: crawl → filter → report → notify |
| CLI | `python -m trendradar --doctor` | Environment health check |
| CLI | `python -m trendradar --show-schedule` | Show current schedule status |
| CLI | `python -m trendradar --test-notification` | Test notification channels |
| MCP Server | `trendradar-mcp` / `mcp_server.server:run_server` | Start MCP tool server |

## Build & Run

```bash
# Install dependencies
pip install -r requirements.txt
# or
pip install -e .

# Run
python -m trendradar

# Docker
cd docker && docker compose up -d
```

## Configuration

- **Main config**: `config/config.yaml` — platforms, notification webhooks, storage, AI, RSS
- **Schedule**: `config/timeline.yaml` — time-based execution plans
- **Keywords**: `config/frequency_words.txt` — keyword groups with AND/OR/NOT/regex syntax
- **AI interests**: `config/ai_interests.txt` — natural language interest descriptions
- **Environment variables**: Override config values (FEISHU_WEBHOOK_URL, AI_API_KEY, S3_*, etc.)

## CI/CD

- **GitHub Actions**: `.github/workflows/crawler.yml` runs hourly (minute 33)
- **Docker**: `docker/docker-compose.yml` with cron every 30 min

## Development Conventions

- **Language**: Source code comments and logs are in Chinese (zh-CN); code identifiers are in English
- **Config merging**: Environment variables override YAML config values
- **Storage abstraction**: `StorageManager` handles both local SQLite and remote S3 transparently
- **Singleton pattern**: `AppContext` lazily initializes storage and scheduler as singletons
- **Error handling**: Graceful degradation — AI filter falls back to keyword matching on failure
- **No tests directory**: Project does not currently include unit tests
- **Version tracking**: `version`, `version_mcp`, `version_configs` files + `__version__` in `__init__.py`
- **Git**: Single `master` branch, 3 commits

## Key Patterns

1. **Mode Strategy Pattern**: `NewsAnalyzer.MODE_STRATEGIES` defines behavior per report mode
2. **Pipeline Pattern**: `_run_analysis_pipeline()` chains data processing → stats → AI → HTML
3. **Batch Processing**: AI filter processes news in configurable batch sizes with intervals
4. **Deduplication**: AI filter tracks analyzed news IDs to avoid re-processing
5. **Schedule-driven Execution**: `Scheduler.resolve()` determines what to do (collect/push/analyze) based on current time and timeline.yaml
