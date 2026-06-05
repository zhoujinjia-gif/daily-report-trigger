# Daily Report Coordinator

Unified scheduler that dispatches stock reports after each market close via cron-job.org external cron.

## Reports Managed

| Report | Repo | Frequency |
|--------|------|-----------|
| A-Share Daily | `A-Share-report` | Every A-share trading day |
| US Equity Daily | `US-Equity-report` | Every US trading day |
| Full-Market Monthly | `monthly-full-market-report` | Last US trading day of month |

## Cron Schedule

Two independent cron jobs on cron-job.org:

| Report | Cron Expression | Timezone | Event Type | Context |
|--------|----------------|----------|------------|---------|
| A-Share | `30 7 * * 1-5` | **UTC** | `trigger_a_share` | 07:30 UTC = 15:30 Beijing (A-share close 15:00 + 30min) |
| US Equity | `30 16 * * 1-5` | **America/New_York** | `trigger_us_equity` | 16:30 ET = ~30 min after US close, DST auto-adjusted |

Each market gets its own cron in its own timezone — no single-cron compromises.

> GitHub built-in `schedule` triggers are NOT used (verified unreliable timing).

## Architecture

```
cron-job.org
    ├─ Job A: 30 7 * * 1-5 UTC → POST trigger_a_share
    └─ Job B: 30 16 * * 1-5 America/New_York → POST trigger_us_equity
                │
                ▼
          GitHub API: repository_dispatch
                │
                ▼
          scheduler.yml (6 jobs)
                │
                ├─ dedup-check        (cache-based, per-event-type isolation)
                ├─ check-market       (US trading day + month-end detection)
                ├─ dispatch-a-share ──► A-Share-report
                ├─ dispatch-us-equity ► US-Equity-report
                ├─ dispatch-monthly ──► monthly-full-market-report  (month-end only)
                └─ mark-dispatched    (write cache, auto-trigger only)
```

## Deduplication (3 layers)

| Layer | Location | Mechanism |
|-------|----------|-----------|
| 1 | Coordinator cache | GitHub Actions cache per event type, same-day guard |
| 2 | Report repo workflow | 12h GitHub Actions run history check |
| 3 | Python SQLite | 23h window check on `email_sent` records |

## Manual Trigger

Go to Actions → Daily Report Coordinator → Run workflow:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `force` | `false` | Skip all dedup and trading day checks |
| `reports` | `a_share,us_equity` | Comma-separated: `a_share`, `us_equity`, `monthly` |

Manual triggers do **not** write dedup cache, so they don't interfere with auto triggers.

**Examples:**
- Weekend test: `force=true`, reports=default
- Test monthly: `force=true`, reports=`a_share,us_equity,monthly`
- A-share only: `force=true`, reports=`a_share`

## Setup

### Required Secret

| Secret | Purpose |
|--------|---------|
| `REPO_PAT` | GitHub Personal Access Token for cross-repo dispatch |

### Creating REPO_PAT

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. Select `repo` scope (full repo access)
4. Copy token → Add to this repo's Settings → Secrets → Actions as `REPO_PAT`

> **Important:** Classic PAT with `repo` scope is required. Fine-grained PATs are incompatible with the repository dispatch API.

### cron-job.org Configuration

Two cron jobs required:

#### Job A: A-Share Trigger

| Field | Value |
|-------|-------|
| URL | `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches` |
| Method | POST |
| Headers | `Authorization: token ghp_xxxx` (your Classic PAT) |
| Body | `{"event_type":"trigger_a_share"}` |
| Cron | `30 7 * * 1-5` (UTC timezone) |

#### Job B: US Equity Trigger

| Field | Value |
|-------|-------|
| URL | `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches` |
| Method | POST |
| Headers | `Authorization: token ghp_xxxx` (your Classic PAT) |
| Body | `{"event_type":"trigger_us_equity"}` |
| Cron | `30 16 * * 1-5` (America/New_York timezone) |

> ⚠️ Delete ALL old cron jobs (especially those sending `auto_trigger`) to prevent duplicate dispatches.

### Target Repo Secrets

Each report repo needs its own secrets (SMTP credentials, API keys) — see their respective READMEs.

## File Structure

```
daily-report-trigger/
├── .github/workflows/scheduler.yml   # Single workflow file
├── README.md
├── README-CN.md
└── history/                          # Design docs & specs
    ├── daily-report-trigger-architecture-v2.md
    ├── daily-report-trigger-architecture-v3.md
    ├── daily-report-trigger-v2-prompt.md
    ├── daily-report-trigger-v3-prompt.md
    └── coordinator-v3.yml            # v3 workflow backup
```

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| No reports at all | cron-job.org not firing | Check cron-job.org execution history for both jobs |
| A-share report arrives before market opens | Wrong timezone on Job A | Ensure Job A uses `30 7 * * 1-5` **UTC** timezone |
| All jobs skipped | Already dispatched today | Use `force=true` |
| 403 on dispatch | PAT is Fine-grained, not Classic | Recreate as Classic PAT with `repo` scope |
| `is_trading_day=false` | Weekend or holiday | Use `force=true` for testing |
| Monthly not triggered | Not month-end | Check check-market logs for `is_month_end` |
| Dispatch succeeded but no email | Report repo Python dedup blocked it | Check report repo Actions logs for `DEDUP` |
| Multiple reports in one day | Stale cron jobs on cron-job.org | Delete all old cron jobs, keep only Job A and Job B |
