# Daily Report Trigger — Architecture v4

> 最后更新：2026-06-06

## 设计原则

- **双 cron，双市场**：A 股和美股各自独立 cron，各用自己的时区
- **两层去重**：coordinator 同日缓存 + Python 层 23h SQLite，简单可靠
- **手动不干扰自动**：手动触发用独立缓存 key，不写缓存，不对自动触发造成任何影响

## 系统全景

```
cron-job.org
  ├─ Job A: 30 7 * * 1-5 UTC           → POST {"event_type":"trigger_a_share"}
  └─ Job B: 30 16 * * 1-5 America/NY   → POST {"event_type":"trigger_us_equity"}
              │
              ▼  GitHub API (repository_dispatch)
              │
              ▼
  ┌─────────────────────────────────────────────────────────┐
  │  daily-report-trigger / scheduler.yml                   │
  │                                                         │
  │  dedup-check ──→ check-market ──→ dispatch-a-share     │
  │       │                           dispatch-us-equity    │
  │       │                           dispatch-monthly *    │
  │       └── mark-dispatched                               │
  │            (* 仅月末 + 美股触发时)                         │
  └──────┬──────────────────────────────────────┬───────────┘
         │ repository_dispatch: send_report     │
         ▼                                      ▼
  ┌──────────────┐                      ┌──────────────┐
  │ A-Share-report│                      │US-Equity-report│
  │  main.py     │                      │  main.py      │
  │  → 23h dedup │                      │  → 23h dedup  │
  │  → fetch data│                      │  → fetch data │
  │  → send email│                      │  → send email │
  └──────────────┘                      └──────────────┘
                                              │ (月末)
                                              ▼
                                     ┌─────────────────┐
                                     │monthly-report   │
                                     │  main.py        │
                                     │  → 23h dedup    │
                                     │  → full report  │
                                     │  → send email   │
                                     └─────────────────┘
```

## 触发时间

| 报告 | Cron | 时区 | 实际时间 |
|------|------|------|---------|
| A 股日报 | `30 7 * * 1-5` | UTC | 15:30 北京时间（收盘 15:00 + 30min） |
| 美股日报 | `30 16 * * 1-5` | America/New_York | 16:30 ET（收盘 16:00 + 30min，DST 自动） |
| 月报 | 跟美股 | 同上 | 仅月末最后一个美股交易日 |

## 去重机制（两层）

### 第一层：Coordinator 缓存

- 位置：scheduler.yml 的 `dedup-check` job
- 机制：GitHub Actions cache，key = `coordinator-<event_type>-<DATE>`
- 窗口：同一天（按 UTC 日期）
- 作用：cron-job.org 偶发重试时阻止重复 dispatch

### 第二层：Python SQLite

- 位置：各 `main.py` 的 `_dedup_check()` 函数
- 机制：查询 report.db，检查 23 小时内是否有 `email_sent=1` 的记录
- 窗口：23 小时
- 作用：最终防线，阻止重复发邮件

## 手动触发隔离

| 触发方式 | 缓存 key | 写缓存？ | 对 cron 影响 |
|---------|---------|----------|------------|
| cron-job.org `trigger_a_share` | `coordinator-trigger_a_share-<DATE>` | ✅ 是 | — |
| cron-job.org `trigger_us_equity` | `coordinator-trigger_us_equity-<DATE>` | ✅ 是 | — |
| GitHub `workflow_dispatch` | `coordinator-manual-<DATE>` | ❌ 否 | 无 |

- 手动触发用独立 key `coordinator-manual-<DATE>`，不会命中 cron 的缓存
- `mark-dispatched` job 只在 `repository_dispatch` 事件时写缓存，手动触发不写
- 手动 `force=true` 跳过 coordinator 缓存检查和 Python 23h 去重

## 文件结构

```
daily-report-trigger/
├── .github/workflows/scheduler.yml   # 唯一 workflow（6 个 job）
├── README.md                         # 英文
├── README-CN.md                      # 中文
└── history/                          # 设计文档
    └── daily-report-trigger-architecture-v4.md

A-Share-report/
├── .github/workflows/daily_report.yml  # 3 步：checkout → install → run
├── main.py                             # Python 主编排（含 23h 去重）
├── trading_calendar.py                 # A 股交易日历
├── data_sources.py                     # 新浪 + 东方财富数据
├── report_builder.py                   # HTML 生成 + SMTP 发送
├── position_manager.py                 # 持仓管理
├── positions.json                      # 持仓数据
├── templates/report.html               # Jinja2 模板
└── report.db                           # SQLite（去重记录）

US-Equity-report/
├── .github/workflows/daily_report.yml  # 同上
├── main.py                             # 含美股 16:00 ET 收盘检查
├── trading_calendar.py                 # NYSE 交易日历
├── data_sources.py                     # Yahoo/Alpha Vantage/FMP
├── report_builder.py
├── position_manager.py
├── positions.json
└── report.db

monthly-full-market-report/
├── .github/workflows/monthly_report.yml
├── main.py                             # 全市场月报编排
├── positions.py                        # Python 配置格式
├── providers/                          # 数据源
├── core/                               # 计算引擎
├── charts/                             # 图表生成
└── templates/report.html
```

## 故障排查

| 现象 | 检查 |
|------|------|
| 没收到报告 | cron-job.org 执行历史 → daily-report-trigger Actions → 下游 report Actions |
| cron 日志 204 但 workflow 灰色跳过 | `dedup-check` 缓存命中（当天已触发过），正常行为 |
| 手动触发正常，cron 不触发 | cron-job.org 时区/PAT 是否正确 |
| 收到重复邮件 | Python 层 dedup 是否报 DB 错误，check report Actions log |
