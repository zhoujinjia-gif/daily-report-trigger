# 每日报告调度中心（Daily Report Coordinator）

由 **cron-job.org** 外部 cron 服务在每个美股交易日收盘后准时触发，自动发送 A 股日报、美股日报，月末自动发送全市场月结单。

## 管理的报告

| 报告 | 目标仓库 | 频率 |
|------|---------|------|
| A股持仓日报 | `A-Share-report` | 每个美股交易日 |
| 美股持仓日报 | `US-Equity-report` | 每个美股交易日 |
| 全市场月结单 | `monthly-full-market-report` | 每月最后一个美股交易日 |

## 触发时间

由 cron-job.org 在 **America/New_York** 时区执行 `43 16 * * 1-5`：

| 季节 | ET 时间 | UTC | 北京时间 |
|------|--------|-----|---------|
| 夏令时 EDT (3-11月) | 16:43 | 20:43 | 次日 04:43 |
| 冬令时 EST (11-3月) | 16:43 | 21:43 | 次日 05:43 |

**一个 cron，全年自动跟随美股时区，收盘后约 43 分钟触发。**

> 不再使用 GitHub 内置 schedule（已被验证触发时间随机，不可靠）。

## 系统架构

```
cron-job.org (America/New_York, 43 16 * * 1-5)
        │  POST repository_dispatch: auto_trigger
        ▼
  scheduler.yml（6 个 Job）
        │
        ├─ 去重检查 ──────────── 同日缓存判断
        ├─ 交易日判断 ────────── 美股日历 + 月末检测
        ├─ 派发 A股日报 ────────► A-Share-report
        ├─ 派发 美股日报 ───────► US-Equity-report
        ├─ 派发 月结单 ─────────► monthly-full-market-report（仅月末）
        └─ 写入去重标记 ──────── 防止同日重复派发（仅 auto_trigger 写）
```

## 防重发机制（三层防护）

| 层级 | 位置 | 机制 |
|------|------|------|
| 第一层 | coordinator cache | GitHub Actions 缓存，同日只 dispatch 一次 |
| 第二层 | 报告仓库 workflow | 无自有 cron，仅响应 dispatch 事件 |
| 第三层 | Python SQLite | `main.py` 入口查询 23 小时内是否已发送 |

## 手动触发

GitHub Actions → Daily Report Coordinator → Run workflow：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `force` | `false` | 设为 `true` 跳过所有去重和交易日检查 |
| `reports` | `a_share,us_equity` | 逗号分隔：`a_share`, `us_equity`, `monthly` |

手动触发**不会写去重缓存**，不影响自动触发。

**常用场景：**
- 周末测试日报：`force=true`，reports 保持默认
- 测试月报：`force=true`，reports 设为 `a_share,us_equity,monthly`
- 只测 A 股：`force=true`，reports 设为 `a_share`

## 部署配置

### 本仓库所需 Secret

| Secret | 用途 |
|--------|------|
| `REPO_PAT` | 用于跨仓库 dispatch 的 Classic PAT（repo scope） |

### 外部 Cron 配置

cron-job.org → Job ID: `7741669`

| 配置项 | 值 |
|--------|-----|
| URL | `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches` |
| Method | POST |
| Body | `{"event_type":"auto_trigger"}` |
| Cron | `43 16 * * 1-5`（America/New_York 时区） |

### 目标仓库所需 Secret

三个报告仓库各自需要 SMTP 凭据和 API 密钥，详见各仓库的 README。

## 文件结构

```
daily-report-trigger/
├── .github/workflows/scheduler.yml     # 唯一 workflow 文件
├── README.md                           # 英文说明
├── README-CN.md                        # 本文件（中文说明）
└── history/                            # 设计文档存档
    ├── daily-report-trigger-architecture-v3.md
    ├── daily-report-trigger-architecture-v2.md
    └── coordinator-v3.yml              # v3 workflow 备份
```

## 故障排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| 没收到报告 | cron-job.org 未触发 | 检查 cron-job.org 执行历史 |
| 所有 Job 灰色跳过 | 今日已 dispatch | 手动触发时 `force=true` |
| dispatch 报 403 | PAT 是 Fine-grained 而非 Classic | 重新创建 Classic PAT，勾选 `repo` scope |
| `is_trading_day=false` | 周末或美股假期 | 测试用 `force=true` |
| 月报未触发 | 非月末最后交易日 | 查看 check-market Job 日志确认 `is_month_end` |
| dispatch 成功但没收到邮件 | 报告仓库 Python 层去重拦截 | 检查报告仓库 Actions 日志，搜索 `DEDUP` |

## 相关链接

- 架构详解：`history/daily-report-trigger-architecture-v3.md`
- 各报告仓库的 Actions 页面可直接手动触发（应急备用）
