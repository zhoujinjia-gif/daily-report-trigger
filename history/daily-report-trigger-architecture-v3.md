# Daily Report Trigger — Architecture v3

## 项目概述 / Overview

**daily-report-trigger** 是三个投资报告仓库的统一调度中心。由外部 cron 服务（cron-job.org）在每个美股交易日收盘后准时触发，自动生成 A 股日报、美股日报，并在月末最后一个交易日额外触发全市场月报。

**v3 核心变更**（相对 v2）：
1. 触发器从 GitHub schedule 迁移到 **cron-job.org 外部 cron**（GitHub schedule 不可靠）
2. 单触发器处理全部报告：日报 + 月末月报统一由 `auto_trigger` 事件触发
3. 文件重命名为 `scheduler.yml`，触发器简化为 `repository_dispatch: [auto_trigger]` + `workflow_dispatch`
4. 三层去重：coordinator cache / 报告仓库无自有 cron / Python SQLite 23h
5. 手动触发永不写缓存（`mark-dispatched` 仅 `repository_dispatch` 事件触发）
6. `is_month_end` 扫描当月全部剩余天数

---

## 1. 触发时机 / Cron Schedule

### 主触发器：cron-job.org（双市场独立触发）

| 市场 | 事件类型 | Cron | 时区 | 收盘后 |
|------|---------|------|------|--------|
| A 股 | `trigger_a_share` | `30 7 * * 1-5` | UTC | 30 min（15:00 收盘 → 15:30 触发） |
| 美股 | `trigger_us_equity` | `30 16 * * 1-5` | America/New_York | 30 min（16:00 收盘 → 16:30 触发） |

| 市场 | EDT (夏季) 触发 | EST (冬季) 触发 | 北京时间 |
|------|----------------|----------------|---------|
| A 股 | 07:30 UTC | 07:30 UTC | 15:30 |
| 美股 | 20:30 UTC | 21:30 UTC | 04:30 / 05:30 |

**去重缓存按市场隔离，A 股和美股互不影响。**

---

## 2. 目录结构 / File Tree

```
daily-report-trigger/
├── .github/workflows/
│   └── scheduler.yml                      # 核心调度 workflow（唯一文件）
├── README.md                              # 英文说明
├── README-CN.md                           # 中文说明
└── history/                               # 存档
    ├── daily-report-trigger-v2-prompt.md
    ├── daily-report-trigger-architecture-v2.md
    ├── daily-report-trigger-v3-prompt.md          # v3 需求 spec
    ├── daily-report-trigger-architecture-v3.md    # 本文件
    └── coordinator-v3.yml                         # v3 workflow 备份
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
  repository_dispatch:              # ★ 主触发器：外部 cron（cron-job.org）
    types: [auto_trigger]
  workflow_dispatch:                # 手动触发（备用）
    inputs:
      force:
        type: choice
        options: ['false', 'true']
        default: 'false'
      reports:
        default: 'a_share,us_equity'
```

**不再使用 GitHub 内置 schedule**（已被证明不可靠，触发时间随机）。外部 cron-job.org 在 `America/New_York` 时区准时触发，自动处理冬令时/夏令时切换。

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
2. 用 `actions/cache/restore@v5` 查找 key = `coordinator-dispatched-<TODAY>` 的缓存
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
| v3.1 | 2026-06-02 | 本地目录去日期前缀（US-Equity-report / A-Share-report / monthly-full-market-report），cache action @v4→@v5 |
| v3.2 | 2026-06-03 | **三重修复**：(1) `trigger.yml`→`coordinator.yml` 重命名导致 schedule 失效；(2) dispatch 条件添加 `github.event_name == 'schedule'`；(3) `is_month_end` 扫描范围 `range(1,4)`→`range(1,32)` 修复 8 次误判；(4) 添加 Juneteenth 假期 |
| v3.3 | 2026-06-04 | **force 污染修复**：`mark-dispatched` 仅 `repository_dispatch` 写缓存，手动触发不写 |
| v4.0 | 2026-06-05 | **外部 cron + 双市场分时触发**：A 股 `trigger_a_share` 07:30 UTC（收盘后 30min），美股 `trigger_us_equity` 16:30 ET（收盘后 30min，DST 自动）；去重缓存按市场隔离 |

---

## ⚠️ 11. 已知陷阱

### 陷阱 A：Workflow 重命名导致 Schedule 失效

**现象**：`workflow_dispatch` 正常，但 `schedule` 从不触发（run list 中 0 次 schedule 事件）。

**根因**：workflow 文件从 `trigger.yml` 重命名为 `coordinator.yml` 后，GitHub Actions 未自动重新注册 schedule。空提交（0 文件变更）无法唤醒。

**修复方法**：
1. 删除 `on.schedule` 块 → push
2. 加回 `on.schedule` 块 → push
3. 两次 push 让 GitHub 将 schedule 视为新注册

### 陷阱 B：schedule 事件下 dispatch 条件失败

**现象**：schedule 触发后，所有 dispatch Job 被 SKIP。

**根因**：dispatch Job 条件依赖 `github.event.inputs.reports`，但 `schedule` 事件中 `github.event.inputs` 为空对象 `{}`，`github.event.inputs.reports` 为 null。`null == ''` 为 false，`contains(null, ...)` 为 false → 条件永远不满足。

**修复**：每个 dispatch Job 的 `if:` 条件添加 `github.event_name == 'schedule' ||`：

```yaml
if: |
  needs.check-market.outputs.is_trading_day == 'true' &&
  (github.event_name == 'schedule' || github.event.inputs.reports == '' || contains(github.event.inputs.reports, 'a_share'))
```

这样 schedule 事件自动 dispatch 全部默认报告（A股+美股），workflow_dispatch 行为不变。

---

*本文件由 Claude Code 生成，用于 AI 理解触发机制架构*
