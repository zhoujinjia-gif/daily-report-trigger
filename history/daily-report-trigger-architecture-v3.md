# Daily Report Trigger — Architecture v3

## 项目概述 / Overview

**daily-report-trigger** 是三个投资报告仓库的统一调度中心。每个美股交易日 21:15 UTC（北京时间次日 05:15），自动触发 A 股日报、美股日报，并在月末最后一个交易日额外触发全市场月报。

**v3 核心变更**（相对 v2）：
1. 确认单 cron `15 21 * * 1-5`（无双 cron 旧版残留）
2. 确认三层去重完整：coordinator cache / 报告仓库无 cron / Python SQLite 23h
3. 确认 Classic PAT（非 Fine-grained）用于 dispatch（防 403）
4. check-market force 模式优先逻辑确认
5. mark-dispatched 使用 `always()` 条件确认
6. 确认三个报告仓库 workflow 均含 `repository_dispatch: [send_report]`
7. 确认三个报告仓库 Python `_dedup_check()` 入口完整
8. history/ 目录归档（v2 文件保留，v3 新增）
9. `coordinator.yml` 备份到 `history/coordinator-v3.yml`
10. README 更新（EN + CN）

---

## 1. 触发时机 / Cron Schedule

```
cron: '15 21 * * 1-5'  (UTC 时间)
```

| 时区 | 时间 | 说明 |
|------|------|------|
| UTC | 21:15 | 统一触发时间 |
| 北京时间 | 次日 05:15 | 用户醒来即可看到报告 |
| 夏令时 EDT (UTC-4) | 17:15 | 美股 16:00 收盘后 75 分钟 |
| 冬令时 EST (UTC-5) | 16:15 | 美股 16:00 收盘后 15 分钟 |

---

## 2. 目录结构 / File Tree

```
daily-report-trigger/
├── .github/workflows/
│   └── coordinator.yml                    # 核心调度 workflow（唯一文件）
├── README.md                              # 英文说明
├── README-CN.md                           # 中文说明
└── history/                               # 存档
    ├── daily-report-trigger-v2-prompt.md
    ├── daily-report-trigger-architecture-v2.md
    ├── daily-report-trigger-v3-prompt.md          # v3 需求 spec
    ├── daily-report-trigger-architecture-v3.md    # 本文件
    └── coordinator-v3.yml                         # ★ v3: coordinator 备份
```

---

## 3. 目标仓库 / Target Repositories

| 仓库名 | 事件类型 | 触发条件 | 发送内容 |
|------|------|------|------|
| `A-Share-report` | `send_report` | 每个美股交易日 | A股持仓日报 |
| `US-Equity-report` | `send_report` | 每个美股交易日 | 美股持仓日报 |
| `monthly-full-market-report` | `send_report` | 月末最后交易日 | 全市场月结单 |

所有目标仓库均属于同一 GitHub 账号，使用同一个 `REPO_PAT` 进行 dispatch（`context.repo.owner` 自动获取）。

---

## 4. 核心 Workflow — coordinator.yml

### 4.1 触发方式

```yaml
on:
  schedule:
    - cron: '15 21 * * 1-5'       # 自动触发（周一至五）
  workflow_dispatch:                # 手动触发
    inputs:
      force:                        # 跳过去重 + 交易日检查
        type: choice
        options: ['false', 'true']
        default: 'false'
      reports:                      # 选择性发送
        default: 'a_share,us_equity'
```

### 4.2 Job 依赖图

```
dedup-check ──► check-market ──┬── dispatch-a-share ────┐
                 (if should_run) ├── dispatch-us-equity ──┼── mark-dispatched
                                └── dispatch-monthly ────┘  (if should_run)
                                    (仅 is_month_end)
```

### 4.3 Job 1: dedup-check（去重检查）

**目的**：同日不重复 dispatch（第一层防护）

**机制**：
1. 获取今日 UTC 日期 → `steps.date.outputs.TODAY`
2. 用 `actions/cache/restore@v4` 查找 key = `coordinator-dispatched-<TODAY>` 的缓存
3. 根据 `cache-hit` 和 `force` 决定是否继续：

| force | cache-hit | 结果 | should_run |
|:---:|:---:|------|:---:|
| true | — | 强制执行 | `true` |
| false | true | 今日已发，跳过 | `false` |
| false | false | 今日首次，继续 | `true` |

**关键细节**：
- outputs 引用 `steps.decide.outputs.*`（不是 `steps.check.*`）
- cache restore step id = `cache`，decide step id = `decide`

### 4.4 Job 2: check-market（市场日判断）

**前置条件**：`needs.dedup-check.outputs.should_run == 'true'`

**机制**：内嵌 Python 脚本：
1. force 模式 → `is_trading_day=true, is_month_end=true`
2. 周末 → `false`
3. 美股假期检查（2026-2027 哈希表）
4. 月末判断：扫描未来 3 天，无同月交易日 → `is_month_end=true`

### 4.5 Job 3-5: 并行 Dispatch

```javascript
await github.rest.repos.createDispatchEvent({
  owner: context.repo.owner,        // 动态获取
  repo: 'A-Share-report',           // 硬编码仓库名
  event_type: 'send_report',
  client_payload: {
    date: today, force: force,
    source: 'coordinator'
  }
});
```

**认证**：`github-token: ${{ secrets.REPO_PAT }}`

**⚠️ REPO_PAT 要求**：
- 必须使用 **Classic PAT**（Fine-grained PAT 的 Actions 权限不兼容 dispatch API）
- 必须勾选 `repo` scope（全仓读写权限）

**Dispatch 条件**：

| Job | 附加条件 |
|------|------|
| `dispatch-a-share` | `reports` 包含 `a_share` |
| `dispatch-us-equity` | `reports` 包含 `us_equity` |
| `dispatch-monthly` | `is_month_end == 'true'` 且 `reports` 包含 `monthly` |

### 4.6 Job 6: mark-dispatched（写入去重标记）

```yaml
if: always() && needs.dedup-check.outputs.should_run == 'true'
```

使用 `always()` 确保即使 dispatch job 部分失败也写入标记，防止后续 cron 重复触发。

---

## 5. 三层防重发体系

```
第一层 ─ Coordinator cache (协调器层)
  │ 同一天内 coordinator 第二次触发 → dedup-check 返回 should_run=false
  │
第二层 ─ Workflow 无自有 cron (报告仓库层)
  │ 三个报告仓库的 workflow 均已删除 schedule/cron 触发
  │ 仅响应 repository_dispatch 和 workflow_dispatch
  │
第三层 ─ Python SQLite 去重 (代码层，最终防线)
  │ 每个报告仓库的 main.py 在 run() 入口处调用 _dedup_check()
  │ 查询 report.db 中 23 小时内有 email_sent=1 的记录
```

---

## 6. 报告仓库接收 Dispatch 确认

| 仓库 | workflow on: | Python dedup |
|------|:---:|:---:|
| A-Share-report | `repository_dispatch: [send_report]` + `workflow_dispatch` ✅ | `_dedup_check()` ✅ |
| US-Equity-report | `repository_dispatch: [update_position, send_report]` + `workflow_dispatch` ✅ | `_dedup_check()` ✅ |
| monthly-full-market-report | `repository_dispatch: [send_report]` + `workflow_dispatch` ✅ | `_dedup_check()` ✅ |

---

## 7. 手动触发指南

### 通过 Coordinator（推荐）

`https://github.com/<username>/daily-report-trigger/actions/workflows/coordinator.yml`

| 场景 | force | reports | 效果 |
|------|:---:|------|------|
| 非交易日测试 | `true` | 默认 | 发送 A股 + 美股日报 |
| 只测 A 股 | `true` | `a_share` | 只发 A 股 |
| 测试月报 | `true` | `a_share,us_equity,monthly` | 三报全发 |
| 工作日补发 | `false` | 默认 | 如有今日缓存则跳过 |

### 直接触发单个报告仓库

各报告仓库 Actions 页面 → Run workflow → `force: true`

---

## 8. Secrets 配置

| 仓库 | Secret | 用途 | 类型 |
|------|------|------|------|
| `daily-report-trigger` | `REPO_PAT` | 调用 GitHub API dispatch | Classic PAT，repo scope |
| `A-Share-report` | `A_QQ_EMAIL` / `A_QQ_SMTP_CODE` / `A_RECIPIENT_EMAIL` | SMTP | — |
| `US-Equity-report` | `QQ_EMAIL` / `QQ_SMTP_CODE` / `RECIPIENT_EMAIL` | SMTP | — |
| `US-Equity-report` | `ALPHA_VANTAGE_KEY` | 美股行情 | 免费 |
| `US-Equity-report` | `EXCHANGE_RATE_KEY` | USD/CNY 汇率 | 免费 |
| `monthly-full-market-report` | 同上 5 个 + `FMP_KEY` | — | — |

---

## 9. 故障排查

| 症状 | 原因 | 修复 |
|------|------|------|
| 所有 job 跳过 | 今日已 dispatch | `force=true` |
| dispatch 403 | Fine-grained PAT | 重建 Classic PAT，勾选 `repo` |
| `is_trading_day=false` | 周末/假期 | `force=true` |
| 月报未触发 | 非月末 | 查看 check-market 日志 `is_month_end` |
| dispatch 成功但无邮件 | Python dedup 拦截 | 查看报告仓库 Actions 日志 `DEDUP` |

---

## 10. 版本记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1 | 2026-05-22 | 双 cron + curl 手动 dispatch，仅 A股+美股 |
| v2 | 2026-05-30 | 单 cron coordinator、新增月报调度、三层去重、Classic PAT |
| v3 | 2026-05-31 | 实施确认 + 三层去重验证 + 报告仓库 workflow/Python dedup 确认 + history/归档 + architecture-v3.md |

---

*本文件由 Claude Code 生成，用于 AI 理解触发机制架构*
