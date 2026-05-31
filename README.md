# Daily Report Coordinator

Unified scheduler that dispatches stock reports every US trading day at 21:15 UTC (05:15 Beijing time next morning).

## Reports Managed

| Report | Repo | Frequency |
|--------|------|-----------|
| A-Share Daily | `A-Share-report` | Every US trading day |
| US Equity Daily | `US-Equity-report` | Every US trading day |
| Full-Market Monthly | `monthly-full-market-report` | Last US trading day of month |

## Cron Schedule

```
15 21 * * 1-5  (UTC)
```

| Timezone | Local Time | Context |
|----------|-----------|---------|
| UTC | 21:15 | Trigger time |
| Beijing (UTC+8) | 05:15 next day | Reports arrive before wake-up |
| EDT (UTC-4, summer) | 17:15 | ~75 min after US market close |
| EST (UTC-5, winter) | 16:15 | ~15 min after US market close |

Single cron covers both DST periods — no need for dual schedules.

## Architecture

```
cron / workflow_dispatch
        │
        ▼
  coordinator.yml
        │
        ├─ dedup-check      (cache-based, same-day guard)
        ├─ check-market     (US trading day + month-end detection)
        ├─ dispatch-a-share ──► A-Share-report
        ├─ dispatch-us-equity ─► US-Equity-report
        ├─ dispatch-monthly ──► monthly-full-market-report  (month-end only)
        └─ mark-dispatched  (write cache)
```

## Deduplication (3 layers)

1. **Coordinator cache** — GitHub Actions cache prevents double dispatch on same day
2. **No self-cron** — Report repos have no schedule triggers, respond only to dispatch
3. **Python SQLite** — Each report's `main.py` checks 23h window before sending

## Manual Trigger

Go to Actions → Daily Report Coordinator → Run workflow:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `force` | `false` | Skip all dedup checks |
| `reports` | `a_share,us_equity` | Comma-separated: `a_share`, `us_equity`, `monthly` |

**Examples:**
- Test on weekend: `force=true`, reports=default
- Test monthly report: `force=true`, reports=`a_share,us_equity,monthly`
- Test A-share only: `force=true`, reports=`a_share`

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

### Target Repo Secrets

Each report repo needs its own secrets (SMTP credentials, API keys) — see their respective READMEs.

## File Structure

```
daily-report-trigger/
├── .github/workflows/coordinator.yml   # Single workflow file
├── README.md
├── README-CN.md
└── history/                            # Design docs & specs
    ├── daily-report-trigger-architecture-v2.md
    ├── daily-report-trigger-architecture-v3.md
    ├── daily-report-trigger-v2-prompt.md
    ├── daily-report-trigger-v3-prompt.md
    └── coordinator-v3.yml              # v3 workflow backup
```

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| All jobs skipped | Already dispatched today | Use `force=true` |
| 403 on dispatch | PAT is Fine-grained, not Classic | Recreate as Classic PAT with `repo` scope |
| `is_trading_day=false` | Weekend or holiday | Use `force=true` for testing |
| Monthly not triggered | Not month-end | Check check-market logs |
| Dispatch succeeded but no email | Report repo Python dedup blocked it | Check report repo Actions logs |
