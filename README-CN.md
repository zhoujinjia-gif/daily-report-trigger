# 每日报告调度中心（Daily Report Coordinator）

每个美股交易日 21:15 UTC（北京时间次日 05:15），自动触发三份投资报告并邮件发送。

## 管理的报告

| 报告 | 目标仓库 | 频率 |
|------|---------|------|
| A股持仓日报 | `A-Share-report` | 每个美股交易日 |
| 美股持仓日报 | `US-Equity-report` | 每个美股交易日 |
| 全市场月结单 | `monthly-full-market-report` | 每月最后一个美股交易日 |

## Cron 时间

```
15 21 * * 1-5  (UTC)
```

| 时区 | 本地时间 | 说明 |
|------|---------|------|
| UTC | 21:15 | 统一触发时间 |
| 北京时间 (UTC+8) | 次日 05:15 | 醒来即可收到报告 |
| 夏令时 EDT (UTC-4) | 17:15 | 美股收盘后约 75 分钟 |
| 冬令时 EST (UTC-5) | 16:15 | 美股收盘后约 15 分钟 |

一个 cron 覆盖全年，无需区分夏令时/冬令时。

## 系统架构

```
定时触发 / 手动触发
        │
        ▼
  coordinator.yml（6 个 Job）
        │
        ├─ 去重检查 ──────────── 同日缓存判断
        ├─ 交易日判断 ────────── 美股日历 + 月末检测
        ├─ 派发 A股日报 ────────► A-Share-report
        ├─ 派发 美股日报 ───────► US-Equity-report
        ├─ 派发 月结单 ─────────► monthly-full-market-report（仅月末）
        └─ 写入去重标记 ──────── 防止同日重复派发
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

**常用场景：**
- 周末测试日报：`force=true`，reports 保持默认
- 测试月报：`force=true`，reports 设为 `a_share,us_equity,monthly`
- 只测 A 股：`force=true`，reports 设为 `a_share`

## 部署配置

### 本仓库所需 Secret

| Secret | 用途 |
|--------|------|
| `REPO_PAT` | 用于跨仓库 dispatch 的 Personal Access Token |

### REPO_PAT 创建步骤

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. 勾选 `repo` scope（全仓读写权限）
4. 生成后复制 token，添加到本仓库 Settings → Secrets → Actions，命名为 `REPO_PAT`

> ⚠️ **必须使用 Classic PAT**，Fine-grained PAT 的 Actions 权限不兼容 repository dispatch API。

### 目标仓库所需 Secret

三个报告仓库各自需要 SMTP 凭据和 API 密钥（QQ 邮箱授权码、汇率 API Key 等），详见各仓库的 README。

## 文件结构

```
daily-report-trigger/
├── .github/workflows/coordinator.yml   # 唯一 workflow 文件
├── README.md                           # 英文说明
├── README-CN.md                        # 本文件（中文说明）
└── history/                            # 设计文档存档
    ├── daily-report-trigger-architecture-v2.md   # 架构详解（面向 AI）
    └── daily-report-trigger-v2-prompt.md         # v2 重设计原始需求
```

## 故障排查

| 现象 | 可能原因 | 解决方法 |
|------|---------|---------|
| 所有 Job 灰色跳过 | 今日已 dispatch | 手动触发时 `force=true` |
| dispatch 报 403 | PAT 是 Fine-grained 而非 Classic | 重新创建 Classic PAT，勾选 `repo` scope |
| `is_trading_day=false` | 周末或美股假期 | 测试用 `force=true` |
| 月报未触发 | 非月末最后交易日 | 查看 check-market Job 日志确认 `is_month_end` |
| dispatch 成功但没收到邮件 | 报告仓库 Python 层去重拦截 | 检查报告仓库 Actions 日志，搜索 `DEDUP` |

## 相关链接

- 架构详解：`history/daily-report-trigger-architecture-v2.md`
- 各报告仓库的 Actions 页面可直接手动触发（应急备用）
