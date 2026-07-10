**[English](./README.md)** | [简体中文](./README.zh-CN.md)

---

# Automated Daily Report Update (PIP)

> 🔒 **Security notice (read first)**: This repo is public. **No API keys / tokens / passwords / app_secrets are committed.** All credentials live in:
> - `.env` (gitignored) — `LLM_API_KEY`, optional `FEISHU_BITABLE_APP_TOKEN`, `FEISHU_BITABLE_TABLE_ID`, `FEISHU_DM_CHAT_ID`
> - `~/.daily-digest/config.yaml` (your local machine, never shared) — `app_id`, `app_secret` for Feishu
>
> Copy `config/.env.example` → `config/.env` and fill in your own values before running. Never commit your `.env`.

## Automated Daily Report Update (PIP)

> Automated daily tech-digest update — two autonomous agents running on cron jobs. One collects daily raw data sources, uses an LLM to analyze, filter content, and produce structured daily reports. The other automatically synchronizes updated reports and delivers them to a Feishu group chat on a fixed schedule.

## What this is

A fully automated AI tech-digest system:

- 9 first-party sources (Qbitai / 36Kr / Juejin / Simon, etc.) scraped across three time windows: 9:50 / 10:10 / 10:40 every morning
- LLM summarization bucketed into 3 themes (AI / Agent / Industry), preserving **4 required fields** per item: star count / comment count / publish date / source
- Rendered as single-file HTML reports
- Feishu DM push + Bitable multi-dim-table sync, with built-in failure fallbacks

Zero manual intervention. Cron jobs produce the report; the operator only reads the result.

## Reports

Newest first. Filename convention: `DailyReportV1.0-YYYY-M-D.html` (no leading zero on month or day).

| Date | Report |
|------|--------|
| 2026-07-08 | [DailyReportV1.0-2026-7-8.html](科技日报V1.0-2026-7-8.html) |
| 2026-07-07 | [DailyReportV1.0-2026-7-7.html](科技日报V1.0-2026-7-7.html) |
| 2026-07-06 | [DailyReportV1.0-2026-7-6.html](科技日报V1.0-2026-7-6.html) |
| 2026-07-05 | [DailyReportV1.0-2026-7-5.html](科技日报V1.0-2026-7-5.html) |
| 2026-07-04 | [DailyReportV1.0-2026-7-4.html](科技日报V1.0-2026-7-4.html) |
| 2026-07-03 | [DailyReportV1.0-2026-7-3.html](科技日报V1.0-2026-7-3.html) |
| 2026-07-02 | [DailyReportV1.0-2026-7-2.html](科技日报V1.0-2026-7-2.html) |
| 2026-07-01 | [DailyReportV1.0-2026-7-1.html](科技日报V1.0-2026-7-1.html) |
| 2026-06-30 | [DailyReportV1.0-2026-6-30.html](科技日报V1.0-2026-6-30.html) |
| 2026-06-29 | [DailyReportV1.0-2026-6-29.html](科技日报V1.0-2026-6-29.html) |
| 2026-06-26 | [DailyReportV1.0-2026-6-26.html](科技日报V1.0-2026-6-26.html) |
| 2026-06-25 | [DailyReportV1.0-2026-6-25.html](科技日报V1.0-2026-6-25.html) |
| 2026-06-24 | [DailyReportV1.0-2026-6-24.html](科技日报V1.0-2026-6-24.html) |
| 2026-06-23 | [DailyReportV1.0-2026-6-23.html](科技日报V1.0-2026-6-23.html) |
| 2026-06-22 | [DailyReportV1.0-2026-6-22.html](科技日报V1.0-2026-6-22.html) |

> 6-27 / 6-28 weekends are intentionally skipped (configurable via `run_window`)

## Automation architecture

Two independent agents, scheduled via Windows `schtasks`:

```
[09:50 first run] → [10:10 retry] → [10:40 final]
   ↓
Agent 1: data collection
   - 9 sources via RSS / scraping
   - cross-day dedupe (URL + title + GitHub repo, 3-layer 14-day sliding window)
   - LLM summary + 3-theme bucketing
   - single-file HTML render
   ↓
Agent 2: Feishu delivery
   - Feishu group DM push (canonical name "Daily Report YYYY-MM-DD")
   - Bitable multi-dim-table sync
   - retry + double-verify on failure
   ↓
Report on disk + Feishu message delivered = zero human in the loop
```

## Key design decisions

| Design point | How it works | Why |
|--------------|--------------|-----|
| Cross-day dedupe | URL + L2 title + GitHub repo, 3 layers with a 14-day sliding window | Measured 5/50 items deduped (6-23 → 6-24) |
| LLM fallback chain | 19 → 1 path: timeout 60s + smart_truncate + markitdown cache flush + state.pushed reset | One stuck source should not fail the whole day |
| Anti-crawl bypass | `curl` over WinHTTP proxy → Python `urllib` with proxy disabled + UA + 2 retries | All of qbitai / 36kr / juejin / simon pass |
| Required fields | Star count / comment count / publish date / source | 4-dim signal density beats "AI summary" |
| Failure retry | 3 time windows (9:50 / 10:10 / 10:40) with nested double-verify | Single-point failure never loses the report |

## Bundled package

- [`daily-digest-pipeline-skill-v1.5.zip`](daily-digest-pipeline-skill-v1.5.zip) — fully self-contained reproduction package (9-source config + LLM summarization prompt + HTML render template + schtasks bat + Feishu API wrapper + cross-day dedupe state). Any machine can take it, change 4 fields, and inherit the entire digest pipeline

## Production track record

- v1.4 onwards: 4 consecutive days (2026-06-30 → 2026-07-03) with `rc=0` (no failures)
- 9 sources average success rate 19/30 → 30/30 (after anti-crawl bypass upgrade)
- Single report generation under 90s (including LLM calls)

## How to use

**As an operator**: clone this repo → unzip `daily-digest-pipeline-skill-v1.5.zip` → edit 4 fields in `config/sources.yml` (source URLs / Feishu webhook / Bitable app_token / delivery time) → run `install_deps.bat` + `register_cron.bat` → first report arrives at 9:50 the next morning

**As an auditor**: open any `DailyReportV1.0-*.html` directly, each is self-contained; compare dates to see trends

**As a reproducer**: read the bundled `daily-digest-pipeline-skill-v1.5.zip` (5 docs under `docs/` + full source under `scripts/`)

## License

- HTML report content: **CC BY 4.0** (attribution-share, free to repost)
- Code inside the bundled package: **MIT**
- Source content copyright: original authors (each report's footer cites them)

## Origin

- System design: project maintainer
- Bundled package version: v1.5 (from 2026-07)
- Evolution: 4 weeks of iteration v1.0 → v1.5 across 5 stages (dedupe / LLM fallback / anti-crawl / Feishu sync / production hardening)
