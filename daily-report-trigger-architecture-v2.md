# Daily Report Trigger — Architecture v2

## 项目概述 / Overview

**daily-report-trigger** 是三个投资报告仓库的统一调度中心。每个美股交易日 21:15 UTC（北京时间次日 05:15），自动触发 A 股日报、美股日报，并在月末最后一个交易日额外触发全市场月报。

**v2 核心变更**（相对 v1 的双 cron 方案）：
- 单 cron 替代双 cron，全年覆盖夏令时/冬令时
- 新增月报触发（原 v1 月报自管 cron，现已整合）
- 三层去重防双发：coordinator cache → workflow 无 cron → Python SQLite
- 支持 workflow_dispatch 手动选择报告类型 + force 模式

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

**为什么 21:15 一个 cron 就够了？**
夏令时美股 20:00 UTC 收盘 → 21:15 已收盘 1h15m；冬令时 21:00 UTC 收盘 → 21:15 刚收盘 15m。两个时区都能确保数据完整。

---

## 2. 目录结构 / File Tree

```
daily-report-trigger/
├── .github/
│   └── workflows/
│       └── coordinator.yml          # 核心调度 workflow（唯一的 workflow 文件）
├── README.md                        # 用户文档
├── trigger-v2-prompt.md             # v2 重设计 spec（存档）
└── daily-report-trigger-architecture-v2.md  # 本文件
```

---

## 3. 目标仓库 / Target Repositories

| 仓库名 | 事件类型 | 触发条件 | 发送内容 |
|------|------|------|------|
| `A-Share-report` | `send_report` | 每个美股交易日 | A股持仓日报 |
| `US-Equity-report` | `send_report` | 每个美股交易日 | 美股持仓日报 |
| `monthly-full-market-report` | `send_report` | 月末最后交易日 | 全市场月结单 |

所有目标仓库均属于同一 GitHub 账号 `zhoujinjia-gif`，使用同一个 `REPO_PAT` 进行 dispatch。

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

**输出**：
- `should_run` — 是否继续后续 job
- `today` — UTC 日期字符串 `"2026-05-30"`

**关键细节**：
- outputs 引用 `steps.decide.outputs.*`（不是 `steps.check.*`，后者是 cache restore step 无此输出）
- force 模式完全绕过 cache 检查

### 4.4 Job 2: check-market（市场日判断）

**前置条件**：`needs.dedup-check.outputs.should_run == 'true'`

**目的**：判断今天是否为美股交易日，以及是否为月末最后交易日

**机制**：内嵌 Python 脚本检查：
1. force 模式 → 直接返回 `is_trading_day=true, is_month_end=true`（用于手动测试月报）
2. 周末（weekday >= 5）→ `is_trading_day=false`
3. 美股假期检查（2026-2027 哈希表）→ `is_trading_day=false`
4. 月末判断：扫描未来 3 天，若无同月交易日 → `is_month_end=true`

**Force 变量传递**：
```yaml
env:
  FORCE: ${{ github.event.inputs.force }}
```
cron 触发时 `github.event.inputs.force` 为空 → `FORCE=""` → force 不走。

### 4.5 Job 3-5: 并行 Dispatch

三个 dispatch job 并行执行，各自向目标仓库发送 `repository_dispatch` 事件。

**Dispatch 方式**：使用 `actions/github-script@v7` 调用 GitHub REST API：
```javascript
await github.rest.repos.createDispatchEvent({
  owner: 'zhoujinjia-gif',     // 从 context.repo.owner 获取
  repo: 'A-Share-report',      // 硬编码仓库名
  event_type: 'send_report',   // 统一事件类型
  client_payload: {
    date: today,                // 传递日期用于各仓库日志
    force: force,               // 传递 force 标记
    source: 'coordinator'       // 来源标记
  }
});
```

**认证**：使用 `github-token: ${{ secrets.REPO_PAT }}`

**⚠️ REPO_PAT 要求**：
- 必须使用 **Classic PAT**（Fine-grained PAT 的 Actions 权限不兼容 dispatch API）
- 必须勾选 `repo` scope（全仓读写权限）
- 创建方法：GitHub Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic) → 勾选 `repo` → 生成

**Dispatch job 条件**：

| Job | 附加条件 |
|------|------|
| `dispatch-a-share` | `reports` 包含 `a_share` |
| `dispatch-us-equity` | `reports` 包含 `us_equity` |
| `dispatch-monthly` | `is_month_end == 'true'` 且 `reports` 包含 `monthly` |

**默认行为**：`reports` 默认为 `a_share,us_equity`；手动触发时可改为仅单个报告或加 `monthly`。

### 4.6 Job 6: mark-dispatched（写入去重标记）

**前置条件**：`always() && needs.dedup-check.outputs.should_run == 'true'`（即使 dispatch 部分失败也写标记）

**机制**：创建本地文件 + `actions/cache/save@v4` 持久化：
```
mkdir -p .coordinator-dedup
echo "2026-05-30" > .coordinator-dedup/last-dispatch.txt
# 缓存 key: coordinator-dispatched-2026-05-30
```

**缓存有效期**：GitHub Actions cache 默认保留 7 天，远超 24 小时去重需求。

---

## 5. 三层防重发体系

```
第一层 ─ Coordinator cache (协调器层)
  │ 同一天内 coordinator 第二次触发 → dedup-check 返回 should_run=false
  │ 全部 dispatch job 跳过
  │
第二层 ─ Workflow 无自有 cron (报告仓库层)
  │ 三个报告仓库的 workflow 均已删除 schedule/cron 触发
  │ 仅响应 repository_dispatch 和 workflow_dispatch
  │ 不可能被自有 cron 重复触发
  │
第三层 ─ Python SQLite 去重 (代码层，最终防线)
  │ 每个报告仓库的 main.py 在 run() 入口处调用 _dedup_check()
  │ 查询 report.db 中 23 小时内有 email_sent=1 的记录
  │ 存在 → 跳过，打印 "⏭ [DEDUP] 已在过去 23 小时内成功发送报告"
  │ force=True → 打印 "⚡ [FORCE] 跳过去重检查" 并继续
```

---

## 6. 报告仓库如何接收 Dispatch

### 6.1 Workflow on 声明

三个报告仓库的 workflow 文件：

```yaml
on:
  repository_dispatch:
    types: [send_report]
  workflow_dispatch:
    inputs:
      force:
        default: 'false'
      dry_run:
        default: 'false'
```

**注意**：月报原来使用 `types: [send_monthly_report]`，已统一改为 `send_report`。

### 6.2 client_payload 传递

Coordinator 发送的 `client_payload`：
```json
{
  "date": "2026-05-30",
  "force": true,
  "source": "coordinator",
  "is_month_end": true
}
```

报告仓库 workflow 读取并转为命令行参数：
```yaml
env:
  DISPATCH_DATE: ${{ github.event.client_payload.date }}
  DISPATCH_FORCE: ${{ github.event.client_payload.force }}
run: |
  if [ "${{ github.event.inputs.force }}" = "true" ] || [ "$DISPATCH_FORCE" = "true" ]; then
    FORCE_FLAG="--force"
  fi
  if [ "${{ github.event.inputs.dry_run }}" = "true" ]; then
    DRY_FLAG="--dry-run"
  fi
  python main.py run $FORCE_FLAG $DRY_FLAG
```

### 6.3 Python 去重入口

三个仓库的 `main.py` 均在 `run()` 入口处有统一的去重逻辑：

```python
DB_PATH = ROOT / "report.db"

def _dedup_check(db_path, window_hours=23):
    """检查 23 小时内是否已成功发送报告"""
    from datetime import datetime, timedelta
    threshold = (datetime.utcnow() - timedelta(hours=window_hours)).isoformat()
    conn = sqlite3.connect(str(db_path))
    conn.execute("""CREATE TABLE IF NOT EXISTS reports (...)""")
    row = conn.execute(
        "SELECT id FROM reports WHERE email_sent=1 AND timestamp_utc > ? LIMIT 1",
        (threshold,)
    ).fetchone()
    return row is not None

def run(force=False, dry_run=False):
    if not force and _dedup_check(DB_PATH):
        print("⏭ [DEDUP] 已在过去 23 小时内成功发送报告")
        return 0
    # ... 继续生成并发送
```

---

## 7. 手动触发指南

### 7.1 通过 Coordinator（推荐）

`https://github.com/zhoujinjia-gif/daily-report-trigger/actions/workflows/coordinator.yml`

| 场景 | force | reports | 效果 |
|------|:---:|------|------|
| 非交易日在测试 | `true` | 默认 | 发送 A股 + 美股日报 |
| 只测 A 股 | `true` | `a_share` | 只发 A 股 |
| 测试月报 | `true` | `a_share,us_equity,monthly` | 三报全发 |
| 正常工作日补发 | `false` | 默认 | 如有今日缓存则跳过 |

### 7.2 直接触发单个报告仓库

`https://github.com/zhoujinjia-gif/<repo>/actions`

在对应仓库的 Actions 页面 → Run workflow → `force: true` → Run

**适用场景**：coordinator 故障时的应急手段。

---

## 8. 故障排查

### 所有 dispatch job 被跳过

**症状**：dedup-check ✅ 但 check-market 和 dispatch job 全是灰色跳过

**原因**：`should_run=false` — 今日已 dispatch

**修复**：workflow_dispatch 时 force 选 `true`

---

### dispatch job 执行但报 403

**症状**：`Error: Resource not accessible by personal access token`，`x-accepted-github-permissions: contents=write`

**原因**：`REPO_PAT` 权限不够，可能是 Fine-grained PAT 的 Actions scope 不兼容

**修复**：使用 **Classic PAT**（勾选 `repo`），替换 Secrets 中的 `REPO_PAT`

---

### check-market 显示 is_trading_day=false

**原因**：当天是周末或假期

**修复**：需要测试时使用 `force: true`

---

### 报告仓库收到 dispatch 但没发邮件

**检查步骤**：
1. 报告仓库 Actions 中看 run 日志
2. 搜索 `DEDUP` — 是否被 Python 层拦截
3. 搜索 `trading day` — 是否被交易日检查拦截
4. 搜索 `SMTP` — 是邮件发送失败

---

### 月报未在月末触发

**可能原因**：
1. `check-market` 的月末计算有误（假期列表不完整）
2. `reports` 参数不包含 `monthly`

**修复**：检查 coordinator run 的 check-market 日志，确认 `Trading day: True, Month end: True`

---

## 9. Secrets 配置

| 仓库 | Secret | 用途 | 类型 |
|------|------|------|------|
| `daily-report-trigger` | `REPO_PAT` | 调用 GitHub API dispatch | Classic PAT，repo scope |
| `A-Share-report` | `A_QQ_EMAIL` | QQ 邮箱 | SMTP 发件人 |
| `A-Share-report` | `A_QQ_SMTP_CODE` | QQ SMTP 授权码 | SMTP 认证 |
| `A-Share-report` | `A_RECIPIENT_EMAIL` | 收件邮箱 | 报告接收人 |
| `US-Equity-report` | `QQ_EMAIL` / `QQ_SMTP_CODE` / `RECIPIENT_EMAIL` | 同上 | 同上 |
| `US-Equity-report` | `ALPHA_VANTAGE_KEY` | 美股行情 | 免费注册 alphavantage.co |
| `US-Equity-report` | `EXCHANGE_RATE_KEY` | USD/CNY 汇率 | 免费注册 exchangerate-api.com |
| `monthly-full-market-report` | 同上 5 个 | — | — |

---

## 10. 版本记录

| 版本 | 日期 | 变更 |
|------|------|------|
| v1 | 2026-05-22 | 双 cron + curl 手动 dispatch，仅触发 A股+美股 |
| v2 | 2026-05-30 | 单 cron coordinator、新增月报调度、三层去重、workflow_dispatch 选择性触发、Classic PAT 认证 |

---

*本文件由 Claude Code 生成，用于 AI 理解触发机制架构*
