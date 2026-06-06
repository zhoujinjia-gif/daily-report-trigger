# 每日报告调度中心

由 cron-job.org 外部 cron 在各自市场收盘后准时触发，自动发送 A 股日报、美股日报，月末自动跟发月报。

## 管理的报告

| 报告 | 仓库 | 频率 |
|------|------|------|
| A股持仓日报 | `A-Share-report` | 每个 A 股交易日 |
| 美股持仓日报 | `US-Equity-report` | 每个美股交易日 |
| 全市场月报 | `monthly-full-market-report` | 每月最后一个美股交易日（跟美股日报一起发） |

## 触发时间

| 报告 | Cron | 时区 | 实际时间 |
|------|------|------|---------|
| A股日报 | `30 7 * * 1-5` | **UTC** | 15:30 北京时间（收盘 + 30min） |
| 美股日报 + 月报 | `30 16 * * 1-5` | **America/New_York** | 16:30 ET（收盘 + 30min，DST 自动） |

## 去重机制（两层）

| 层 | 位置 | 机制 |
|----|------|------|
| Coordinator 缓存 | scheduler.yml `dedup-check` | GitHub Actions cache，按事件类型 + 日期隔离 |
| Python SQLite | 各 `main.py` | 23 小时内 `email_sent=1` 则跳过 |

## 手动触发（不影响自动 cron）

GitHub → [Actions](https://github.com/zhoujinjia-gif/daily-report-trigger/actions) → Run workflow：

| 参数 | 默认 | 说明 |
|------|------|------|
| `force` | `false` | `true` 跳过所有去重和交易日检查 |
| `reports` | `a_share,us_equity` | 逗号分隔，加 `monthly` 测月报 |

手动触发用独立缓存 key（`coordinator-manual-<DATE>`），不写缓存，不会干扰 cron 自动触发。

## 文件结构

```
daily-report-trigger/
├── .github/workflows/scheduler.yml
├── README.md / README-CN.md
└── history/
    └── daily-report-trigger-architecture-v4.md
```

## cron-job.org 配置

两个 Job，都 POST 到 `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches`：

| | Job A（A股） | Job B（美股） |
|---|---|---|
| Method | POST | POST |
| Headers | `Authorization: token ghp_xxx` | 同 |
| | `Content-Type: application/json` | 同 |
| Body | `{"event_type":"trigger_a_share"}` | `{"event_type":"trigger_us_equity"}` |
| Cron | `30 7 * * 1-5` | `30 16 * * 1-5` |
| 时区 | **UTC** | **America/New_York** |

## 故障排查

| 现象 | 看哪里 |
|------|--------|
| 没收到报告 | cron-job.org → daily-report-trigger Actions → 下游 report Actions |
| coordinator 显示 204 但跳过 | dedup 缓存命中（当天已触发），正常 |
| 收到重复邮件 | 报告仓库 Actions 日志搜 `DEDUP` |
