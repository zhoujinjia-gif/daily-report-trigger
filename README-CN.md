# 每日报告调度中心（Daily Report Coordinator）

由 **cron-job.org** 外部 cron 服务在每个市场收盘后准时触发，自动发送 A 股日报、美股日报，月末自动发送全市场月结单。

## 管理的报告

| 报告 | 目标仓库 | 频率 |
|------|---------|------|
| A股持仓日报 | `A-Share-report` | 每个 A 股交易日 |
| 美股持仓日报 | `US-Equity-report` | 每个美股交易日 |
| 全市场月结单 | `monthly-full-market-report` | 每月最后一个美股交易日 |

## 触发时间

由 cron-job.org 配置 **两个独立的 cron 任务**：

| 报告 | Cron 表达式 | 时区 | 触发事件 | 说明 |
|------|-----------|------|---------|------|
| A股日报 | `30 7 * * 1-5` | **UTC** | `trigger_a_share` | 每天 07:30 UTC = **15:30 北京时间**（A股 15:00 收盘 + 30min） |
| 美股日报 | `30 16 * * 1-5` | **America/New_York** | `trigger_us_equity` | 每天 16:30 ET（美股 16:00 收盘 + 30min，DST 自动调整） |

**A 股和美股各一个 cron，各在各自市场收盘后约 30 分钟触发，全年自动跟随对应时区。**

> 不再使用 GitHub 内置 schedule（已验证触发时间随机，不可靠）。

## 系统架构

```
cron-job.org
    ├─ Job A: 30 7 * * 1-5 UTC → POST trigger_a_share
    └─ Job B: 30 16 * * 1-5 America/New_York → POST trigger_us_equity
                │
                ▼
          GitHub API: repository_dispatch
                │
                ▼
          scheduler.yml（6 个 Job）
                │
                ├─ 去重检查 ──────────── 按事件类型隔离的缓存判断
                ├─ 交易日判断 ────────── 根据触发来源判断对应市场
                ├─ 派发 A股日报 ────────► A-Share-report
                ├─ 派发 美股日报 ───────► US-Equity-report
                ├─ 派发 月结单 ─────────► monthly-full-market-report（仅月末 + 美股触发时）
                └─ 写入去重标记 ──────── 防止同日重复派发（仅 auto 触发写）
```

## 防重发机制（三层防护）

| 层级 | 位置 | 机制 |
|------|------|------|
| 第一层 | coordinator cache | GitHub Actions 缓存，按事件类型隔离，同日只 dispatch 一次 |
| 第二层 | 报告仓库 workflow | 12h GitHub Actions 运行历史检查，防止 coordinator 双发 |
| 第三层 | Python SQLite | `main.py` 入口查询 23 小时内是否已发送 |

## 手动触发

GitHub Actions → Daily Report Coordinator → Run workflow：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `force` | `false` | 设为 `true` 跳过所有去重和交易日检查 |
| `reports` | `a_share,us_equity` | 逗号分隔：`a_share`, `us_equity`, `monthly` |

手动触发**不会写去重缓存**，不影响自动触发。

**常用场景：**
- 周末测试：`force=true`，reports 保持默认
- 测试月报：`force=true`，reports 设为 `a_share,us_equity,monthly`
- 只测 A 股：`force=true`，reports 设为 `a_share`

## 部署配置

### 本仓库所需 Secret

| Secret | 用途 |
|--------|------|
| `REPO_PAT` | 用于跨仓库 dispatch 的 Classic PAT（repo scope） |

### 外部 Cron 配置（cron-job.org）

需要配置 **两个** cron 任务：

#### Job A：A 股日报触发

| 配置项 | 值 |
|--------|-----|
| URL | `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches` |
| Method | POST |
| Headers | `Authorization: token ghp_xxxx`（你的 Classic PAT） |
| Body | `{"event_type":"trigger_a_share"}` |
| Cron | `30 7 * * 1-5`（UTC 时区） |

#### Job B：美股日报触发

| 配置项 | 值 |
|--------|-----|
| URL | `https://api.github.com/repos/zhoujinjia-gif/daily-report-trigger/dispatches` |
| Method | POST |
| Headers | `Authorization: token ghp_xxxx`（你的 Classic PAT） |
| Body | `{"event_type":"trigger_us_equity"}` |
| Cron | `30 16 * * 1-5`（America/New_York 时区） |

> ⚠️ 务必删除所有旧 cron 任务（例如发送 `auto_trigger` 事件的旧任务），避免重复触发。

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
| 没收到任何报告 | cron-job.org 未触发 | 检查 cron-job.org 两个 Job 的执行历史 |
| A 股报告在开盘前就发了 | cron 任务时区/时间配置错误 | 确认 Job A 为 `30 7 * * 1-5` **UTC 时区** |
| 所有 Job 灰色跳过 | 今日已 dispatch（缓存命中） | 手动触发时 `force=true` |
| dispatch 报 403 | PAT 是 Fine-grained 而非 Classic | 重新创建 Classic PAT，勾选 `repo` scope |
| `is_trading_day=false` | 周末或美股假期 | 测试用 `force=true` |
| 月报未触发 | 非月末最后交易日 | 查看 check-market Job 日志确认 `is_month_end` |
| dispatch 成功但没收到邮件 | 报告仓库 Python 层去重拦截 | 检查报告仓库 Actions 日志，搜索 `DEDUP` |
| 一天收到多份日报 | cron-job.org 有僵尸任务 | 检查并删除 cron-job.org 上所有旧 cron 任务 |

## 相关链接

- 架构详解：`history/daily-report-trigger-architecture-v3.md`
- 各报告仓库的 Actions 页面可直接手动触发（应急备用）
