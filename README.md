# Daily Report Coordinator

External cron (cron-job.org) triggers daily stock reports after each market close. A-share and US equity each have their own cron in their own timezone.

## Reports

| Report | Repo | Frequency |
|--------|------|-----------|
| A-Share Daily | `A-Share-report` | Every A-share trading day |
| US Equity Daily | `US-Equity-report` | Every US trading day |
| Full-Market Monthly | `monthly-full-market-report` | Last US trading day (piggybacks on US equity trigger) |

## Trigger Schedule

| Report | Cron | Timezone | When |
|--------|------|----------|------|
| A-Share | `30 7 * * 1-5` | **UTC** | 07:30 UTC = 15:30 Beijing (close + 30min) |
| US Equity + Monthly | `30 16 * * 1-5` | **America/New_York** | 16:30 ET (close + 30min, DST-aware) |

## Deduplication (2 layers)

| Layer | Location | How |
|-------|----------|-----|
| Coordinator cache | `scheduler.yml` | GitHub Actions cache, keyed by event type + date |
| Python SQLite | Each `main.py` | 23h window — skips if `email_sent=1` found |

## Manual Trigger (won't affect auto cron)

Go to [Actions](https://github.com/zhoujinjia-gif/daily-report-trigger/actions) → Run workflow:

| Param | Default | Notes |
|-------|---------|-------|
| `force` | `false` | Set `true` to skip all dedup + trading day checks |
| `reports` | `a_share,us_equity` | Comma-separated; add `monthly` to test monthly |

Manual triggers use a separate cache key (`coordinator-manual-<DATE>`) and don't write cache — they never interfere with cron auto-triggers.

## Setup

### GitHub Secret

| Secret | Purpose |
|--------|---------|
| `REPO_PAT` | Classic PAT with `repo` scope for cross-repo dispatch |

### cron-job.org (2 jobs)

Both POST to `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches`:

| | Job A (A-Share) | Job B (US Equity) |
|---|---|---|
| Method | POST | POST |
| Headers | `Authorization: token ghp_xxx` | same |
| | `Content-Type: application/json` | same |
| Body | `{"event_type":"trigger_a_share"}` | `{"event_type":"trigger_us_equity"}` |
| Cron | `30 7 * * 1-5` | `30 16 * * 1-5` |
| Timezone | **UTC** | **America/New_York** |

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No report | cron-job.org history → daily-report-trigger Actions → downstream report Actions |
| Coordinator 204 but skipped | Dedup cache hit (already triggered today), expected behavior |
| Duplicate emails | Report repo Actions log, search `DEDUP` |
