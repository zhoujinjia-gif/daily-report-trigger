# Agent Prompt: 触发机制优化 — 稳定双报告 + 月报调度系统

> 执行者为 Claude Code Agent。
> **本文件修改两类仓库**：(1) `daily-report-trigger` 触发器仓库，(2) 三个报告仓库各自的 workflow + dedup 逻辑。
> 必须先清理旧文件再写入新文件。

---

## 0. 问题诊断与设计方案

### 当前问题

1. **双发邮件**：两个 cron 时间（夏令时 + 冬令时 各一个）都会 dispatch，若两个都成功就会触发两次
2. **不稳定**：dispatch 失败没有重试，或者报告仓库收到 dispatch 后没有严格去重
3. **逻辑分散**：触发器和报告各自管时间，容易错位

### 根因

```
cron 20:02 UTC ──► dispatch ──► 报告仓库运行 ──► 发送邮件
cron 21:02 UTC ──► dispatch ──► 报告仓库再次运行 ──► 再次发送 ← 问题所在
```

### 解决方案

**三层防护**：
1. **触发器层**：只保留单一 cron（`15 21 * * 1-5` UTC），自然覆盖夏冬令时
2. **协调器层**：GitHub Actions job 级别去重（同一天只 dispatch 一次）  
3. **报告仓库层**：SQLite 去重（每个报告 23 小时内不重复发送），这是最终防线

### 时间计算

| 模式 | UTC offset | 美股收盘（ET 16:00）= UTC | 21:15 UTC = 北京时间 |
|------|-----------|--------------------------|---------------------|
| 夏令时（EDT，UTC-4） | -4 | 20:00 UTC | 次日 05:15 北京 |
| 冬令时（EST，UTC-5） | -5 | 21:00 UTC | 次日 05:15 北京 |

**结论**：`21:15 UTC` 一个 cron = 夏令时收盘后 1h15m，冬令时收盘后 15min。全年有效，无需双 cron。

---

## 1. Step 1 — 清理 daily-report-trigger 仓库

**在 daily-report-trigger 仓库根目录执行**：

```bash
# 删除所有现有 workflow 文件
rm -f .github/workflows/*.yml
rm -f .github/workflows/*.yaml

# 确认清空
ls .github/workflows/   # 应为空

# 删除任何旧的辅助脚本（若有）
rm -f trigger.py trigger.sh dispatch.py
```

---

## 2. Step 2 — 创建新 coordinator workflow

在 `daily-report-trigger/.github/workflows/` 目录下创建 `coordinator.yml`：

```yaml
# daily-report-trigger/.github/workflows/coordinator.yml
# 统一触发器 — 每个美股交易日 21:15 UTC（北京时间次日 05:15）发送双日报 + 月底月报

name: Daily Report Coordinator

on:
  schedule:
    # 21:15 UTC，周一至五
    # 夏令时: 美股收盘(20:00 UTC)后 75 分钟
    # 冬令时: 美股收盘(21:00 UTC)后 15 分钟
    - cron: '15 21 * * 1-5'

  workflow_dispatch:
    inputs:
      force:
        description: '强制运行（跳过所有去重检查）'
        required: false
        default: 'false'
        type: choice
        options: ['false', 'true']
      reports:
        description: '发送哪些报告（逗号分隔: a_share,us_equity,monthly）'
        required: false
        default: 'a_share,us_equity'

env:
  TZ: 'Asia/Shanghai'

jobs:
  # ── Job 1: 去重检查 ──
  dedup-check:
    runs-on: ubuntu-latest
    outputs:
      should_run: ${{ steps.check.outputs.should_run }}
      today: ${{ steps.check.outputs.today }}
    steps:
      - name: Get today's date
        id: date
        run: echo "TODAY=$(date -u +%Y-%m-%d)" >> $GITHUB_OUTPUT

      - name: Check dedup cache
        id: check
        uses: actions/cache/restore@v4
        with:
          path: .coordinator-dedup
          key: coordinator-dispatched-${{ steps.date.outputs.TODAY }}

      - name: Set run decision
        id: decide
        run: |
          FORCE="${{ github.event.inputs.force }}"
          CACHE_HIT="${{ steps.check.outputs.cache-hit }}"
          TODAY="${{ steps.date.outputs.TODAY }}"

          if [ "$FORCE" = "true" ]; then
            echo "should_run=true" >> $GITHUB_OUTPUT
            echo "✅ Force mode: will run"
          elif [ "$CACHE_HIT" = "true" ]; then
            echo "should_run=false" >> $GITHUB_OUTPUT
            echo "⏭ Already dispatched today ($TODAY), skipping"
          else
            echo "should_run=true" >> $GITHUB_OUTPUT
            echo "✅ First run today ($TODAY), will dispatch"
          fi
          echo "today=$TODAY" >> $GITHUB_OUTPUT

  # ── Job 2: 市场日检查 ──
  check-market:
    runs-on: ubuntu-latest
    needs: dedup-check
    if: needs.dedup-check.outputs.should_run == 'true'
    outputs:
      is_trading_day: ${{ steps.market.outputs.is_trading_day }}
      is_month_end: ${{ steps.market.outputs.is_month_end }}
    steps:
      - name: Check US market status
        id: market
        run: |
          python3 << 'EOF'
          import sys
          from datetime import datetime, date, timedelta
          import os

          today = date.today()
          weekday = today.weekday()  # 0=Monday, 6=Sunday

          # 周末不是交易日
          if weekday >= 5:
              print("Not a trading day (weekend)")
              os.system("echo 'is_trading_day=false' >> $GITHUB_OUTPUT")
              os.system("echo 'is_month_end=false' >> $GITHUB_OUTPUT")
              sys.exit(0)

          # 美股主要假期（非完整列表，但覆盖主要节假日）
          US_HOLIDAYS_2026 = {
              date(2026, 1, 1),   # 元旦
              date(2026, 1, 19),  # MLK Day
              date(2026, 2, 16),  # Presidents Day
              date(2026, 4, 3),   # Good Friday
              date(2026, 5, 25),  # Memorial Day
              date(2026, 7, 3),   # Independence Day observed
              date(2026, 9, 7),   # Labor Day
              date(2026, 11, 26), # Thanksgiving
              date(2026, 11, 27), # Black Friday (half day, treat as holiday)
              date(2026, 12, 25), # Christmas
          }
          US_HOLIDAYS_2027 = {
              date(2027, 1, 1),
              date(2027, 1, 18),
              date(2027, 2, 15),
              date(2027, 3, 26),  # Good Friday
              date(2027, 5, 31),  # Memorial Day
              date(2027, 7, 5),   # Independence Day observed
              date(2027, 9, 6),   # Labor Day
              date(2027, 11, 25), # Thanksgiving
              date(2027, 12, 24), # Christmas observed
          }
          US_HOLIDAYS = US_HOLIDAYS_2026 | US_HOLIDAYS_2027

          is_trading = today not in US_HOLIDAYS

          # 是否为月末最后交易日
          is_month_end = False
          if is_trading:
              # 检查接下来 3 天是否还有同月交易日
              has_future_trading = False
              for i in range(1, 4):
                  future = today + timedelta(days=i)
                  if future.month != today.month:
                      break  # 已进入下月
                  if future.weekday() < 5 and future not in US_HOLIDAYS:
                      has_future_trading = True
                      break
              is_month_end = not has_future_trading

          import os
          os.system(f"echo 'is_trading_day={'true' if is_trading else 'false'}' >> $GITHUB_OUTPUT")
          os.system(f"echo 'is_month_end={'true' if is_month_end else 'false'}' >> $GITHUB_OUTPUT")
          print(f"Trading day: {is_trading}, Month end: {is_month_end}")
          EOF

  # ── Job 3: 发送 A股日报 ──
  dispatch-a-share:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      (github.event.inputs.reports == '' || contains(github.event.inputs.reports, 'a_share'))
    steps:
      - name: Dispatch A-Share daily report
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_PAT }}
          script: |
            const today = '${{ needs.dedup-check.outputs.today }}';
            const force = '${{ github.event.inputs.force }}' === 'true';
            
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'A-Share-report',          // ← 确认为你的实际仓库名
              event_type: 'send_report',
              client_payload: {
                date: today,
                force: force,
                source: 'coordinator'
              }
            });
            
            console.log(`✅ A-Share report dispatched for ${today}`);

  # ── Job 4: 发送美股日报 ──
  dispatch-us-equity:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      (github.event.inputs.reports == '' || contains(github.event.inputs.reports, 'us_equity'))
    steps:
      - name: Dispatch US Equity daily report
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_PAT }}
          script: |
            const today = '${{ needs.dedup-check.outputs.today }}';
            const force = '${{ github.event.inputs.force }}' === 'true';
            
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'US-Equity-report',        // ← 确认为你的实际仓库名
              event_type: 'send_report',
              client_payload: {
                date: today,
                force: force,
                source: 'coordinator'
              }
            });
            
            console.log(`✅ US Equity report dispatched for ${today}`);

  # ── Job 5: 发送月报（仅月末）──
  dispatch-monthly:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      needs.check-market.outputs.is_month_end == 'true' &&
      (github.event.inputs.reports == '' || contains(github.event.inputs.reports, 'monthly'))
    steps:
      - name: Dispatch Monthly report
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_PAT }}
          script: |
            const today = '${{ needs.dedup-check.outputs.today }}';
            const force = '${{ github.event.inputs.force }}' === 'true';
            
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'monthly-full-market-report', // ← 确认为你的实际仓库名
              event_type: 'send_report',
              client_payload: {
                date: today,
                force: force,
                source: 'coordinator',
                is_month_end: true
              }
            });
            
            console.log(`✅ Monthly report dispatched for ${today} (MONTH END)`);

  # ── Job 6: 写入去重缓存（仅成功后）──
  mark-dispatched:
    runs-on: ubuntu-latest
    needs: [dedup-check, dispatch-a-share, dispatch-us-equity]
    if: always() && needs.dedup-check.outputs.should_run == 'true'
    steps:
      - name: Create dedup marker
        run: |
          mkdir -p .coordinator-dedup
          echo "${{ needs.dedup-check.outputs.today }}" > .coordinator-dedup/last-dispatch.txt

      - name: Save dedup cache
        uses: actions/cache/save@v4
        with:
          path: .coordinator-dedup
          key: coordinator-dispatched-${{ needs.dedup-check.outputs.today }}
```

---

## 3. Step 3 — 修改每个报告仓库的 workflow

**对 A股报告、美股报告、月报三个仓库分别执行以下操作**：

### 3a. 清理旧的 cron

打开各仓库的 `.github/workflows/daily_report.yml`（或同名文件），**删除 schedule/cron 触发**，只保留 `repository_dispatch` 和 `workflow_dispatch`：

```yaml
# 替换 on: 部分为以下内容（移除所有 schedule）：

on:
  repository_dispatch:
    types: [send_report]
  workflow_dispatch:
    inputs:
      force:
        description: '强制运行'
        required: false
        default: 'false'
      dry_run:
        description: 'Dry run (不发送邮件)'
        required: false
        default: 'false'

# jobs 部分保持原有结构，只修改 run 命令参数传递：
# 将 force 和 dry_run 从 event.inputs 或 client_payload 中读取
```

**修改 jobs.run.steps 中的 run 命令**，确保同时支持两种触发方式：

```yaml
- name: Run report
  env:
    # 邮件凭据（保持原有）
    QQ_EMAIL: ${{ secrets.A_QQ_EMAIL }}         # A股用 A_ 前缀
    QQ_SMTP_CODE: ${{ secrets.A_QQ_SMTP_CODE }}
    RECIPIENT_EMAIL: ${{ secrets.A_RECIPIENT_EMAIL }}
    # 触发来源
    DISPATCH_DATE: ${{ github.event.client_payload.date }}
    DISPATCH_FORCE: ${{ github.event.client_payload.force }}
  run: |
    FORCE_FLAG=""
    DRY_FLAG=""

    # 支持 workflow_dispatch 手动参数
    if [ "${{ github.event.inputs.force }}" = "true" ] || [ "$DISPATCH_FORCE" = "true" ]; then
      FORCE_FLAG="--force"
    fi
    if [ "${{ github.event.inputs.dry_run }}" = "true" ]; then
      DRY_FLAG="--dry-run"
    fi

    python main.py run $FORCE_FLAG $DRY_FLAG
```

### 3b. 在各报告仓库的 main.py 添加 23 小时 SQLite 去重（最终防线）

在 `run()` 函数最开始处（`is_trading_day` 检查之前）插入：

```python
def run(force=False, dry_run=False):
    """主编排流程"""

    # ══ 去重检查（最终防线，防止 coordinator 双发）══
    if not force:
        if _dedup_check(DB_PATH):
            print("⏭ [DEDUP] 已在过去 23 小时内成功发送报告，本次跳过。")
            print("   如需强制运行，使用 --force 参数。")
            return
    else:
        print("⚡ [FORCE] 跳过去重检查，强制执行。")

    # ... 原有逻辑继续 ...


def _dedup_check(db_path, window_hours=23):
    """
    检查过去 window_hours 小时内是否已成功发送报告。
    返回 True 表示"已发送，应跳过"。
    """
    import sqlite3
    from datetime import datetime, timedelta

    threshold = (datetime.utcnow() - timedelta(hours=window_hours)).isoformat()

    try:
        conn = sqlite3.connect(db_path)
        # 确保 reports 表已创建（有些项目可能还没运行过）
        conn.execute("""
            CREATE TABLE IF NOT EXISTS reports (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp_utc TEXT NOT NULL,
                email_sent INTEGER DEFAULT 0,
                is_trading_day INTEGER DEFAULT 1,
                run_duration_ms INTEGER,
                error_message TEXT
            )
        """)
        conn.commit()

        row = conn.execute(
            """SELECT id, timestamp_utc FROM reports
               WHERE email_sent = 1
               AND timestamp_utc > ?
               ORDER BY timestamp_utc DESC
               LIMIT 1""",
            (threshold,)
        ).fetchone()
        conn.close()

        if row:
            print(f"   上次发送时间(UTC): {row[1]}")
            return True  # 已发送，跳过
        return False  # 未发送，可以运行

    except Exception as e:
        print(f"⚠ 去重检查出错（将继续运行）: {e}")
        return False  # 出错时允许运行，宁可多发一次也不漏发
```

---

## 4. Step 4 — 更新 README.md（daily-report-trigger 仓库）

将 `daily-report-trigger/README.md` 替换为：

```markdown
# Daily Report Coordinator

每个美股交易日 21:15 UTC（北京时间次日 05:15）自动触发三个报告仓库。

## 触发时机

| 报告 | 触发条件 |
|------|---------|
| A股持仓日报 | 每个美股交易日 |
| 美股持仓日报 | 每个美股交易日 |
| 全市场月结单 | 每月最后一个美股交易日 |

## Cron 时间

```
15 21 * * 1-5  (UTC)
= 北京时间 次日 05:15
= 夏令时(EDT): 美股收盘后 75 分钟
= 冬令时(EST): 美股收盘后 15 分钟
```

## 手动触发

在 GitHub Actions → Workflows → Daily Report Coordinator → Run workflow
- `force`: 是否跳过去重检查
- `reports`: 发送哪些报告（默认: a_share,us_equity）

## 防重发机制（三层）

1. **coordinator 层**：GitHub Actions cache，同日只 dispatch 一次
2. **报告仓库 workflow 层**：无自有 cron，只接收 dispatch
3. **Python 代码层**：SQLite 记录，23 小时内不重复发送

## 所需 Secrets

在本仓库 Settings → Secrets → Actions 配置：

| Secret | 说明 |
|--------|------|
| `REPO_PAT` | 有权限 dispatch 其他仓库的 Personal Access Token |

### REPO_PAT 创建方法
1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. 选择 Resource owner 为你的账号
3. Repository access：选择 A-Share-report、US-Equity-report、monthly-full-market-report
4. Permissions → Repository permissions → Actions → Read and write
5. 生成 token，复制后添加到本仓库 Secrets
```

---

## 5. Step 5 — 验证各仓库 Secrets 配置

确认以下 Secrets 在各仓库正确配置：

**daily-report-trigger 仓库**：
- `REPO_PAT`：Fine-grained PAT，有 Actions write 权限

**A-Share-report 仓库**：
- `A_QQ_EMAIL`
- `A_QQ_SMTP_CODE`
- `A_RECIPIENT_EMAIL`

**US-Equity-report 仓库**：
- `A_QQ_EMAIL`（或独立的 US_ 前缀版本）
- `A_QQ_SMTP_CODE`
- `A_RECIPIENT_EMAIL`
- `EXCHANGE_RATE_KEY`（汇率 API，若使用 ExchangeRate-API）
- `ALPHA_VANTAGE_KEY`（若使用 Alpha Vantage，免费注册 alphavantage.co）

**monthly-full-market-report 仓库**：
- 同上所有 Secrets

---

## 6. Step 6 — 测试运行

```bash
# 方法 1：GitHub UI
# daily-report-trigger → Actions → Daily Report Coordinator
# → Run workflow → force=true → reports=a_share

# 方法 2：GitHub CLI（若已安装）
gh workflow run coordinator.yml \
  --repo YOUR_USERNAME/daily-report-trigger \
  --field force=true \
  --field reports=a_share

# 验证结果：
# 1. coordinator 的 check-market job 通过
# 2. dispatch-a-share job 成功，可在 A-Share-report 的 Actions 看到新触发
# 3. 再次手动触发（不加 force）→ 应被 dedup 拦截，看到 "Already dispatched" 日志
# 4. 等收到邮件，确认只收到一封
```

---

## 7. 用户手动操作清单（需你自己完成）

以下步骤需要**你本人在 GitHub 网页上操作**，Agent 无法代劳：

### 7a. 创建 REPO_PAT

1. 访问 https://github.com/settings/personal-access-tokens/new
2. Token name: `daily-report-coordinator`
3. Expiration: 1 year（或 No expiration）
4. Repository access: Only selected repositories
   → 选中: `daily-report-trigger`, `A-Share-report`, `US-Equity-report`, `monthly-full-market-report`
5. Permissions → Repository permissions:
   - **Actions**: Read and write ✅
   - Contents: Read（可选，用于读取文件）
6. 点击 Generate token → 复制 token（只显示一次）
7. 前往 `daily-report-trigger` → Settings → Secrets and variables → Actions
   → New repository secret → Name: `REPO_PAT` → 粘贴 token

### 7b. 仓库名确认

在 `coordinator.yml` 中找到以下三行，**确认仓库名与你 GitHub 上的实际名称一致**：
```yaml
repo: 'A-Share-report'             # ← 改为你的实际仓库名
repo: 'US-Equity-report'           # ← 改为你的实际仓库名
repo: 'monthly-full-market-report' # ← 改为你的实际仓库名
```

### 7c. Alpha Vantage（美股数据，若还没有）

1. 访问 https://www.alphavantage.co/support/#api-key
2. 免费注册，获取 API key
3. 在 US-Equity-report 仓库 → Settings → Secrets → `ALPHA_VANTAGE_KEY`

### 7d. 汇率 API（若还没有）

1. 访问 https://www.exchangerate-api.com （免费 1500次/月）
2. 注册获取 API key
3. 在 US-Equity-report 和 monthly 仓库 Secrets 中添加 `EXCHANGE_RATE_KEY`

---

## 8. 运行时序图

```
21:15 UTC 每个工作日
    │
    ▼
coordinator.yml 启动
    │
    ├─ Job: dedup-check
    │    检查今日是否已 dispatch
    │    已 dispatch → 全流程跳过（不再触发报告）
    │    未 dispatch → 继续
    │
    ├─ Job: check-market
    │    是否为美股交易日？
    │    是否为月末最后交易日？
    │
    ├─ Job: dispatch-a-share    ──► A-Share-report
    │    （并行）                     Python dedup 二次确认
    │                                 发送邮件
    │
    ├─ Job: dispatch-us-equity  ──► US-Equity-report
    │    （并行）                     Python dedup 二次确认
    │                                 发送邮件
    │
    ├─ Job: dispatch-monthly    ──► monthly-full-market-report
    │    （仅月末）                   Python dedup 二次确认
    │                                 发送月报邮件
    │
    └─ Job: mark-dispatched
         写入今日 dedup 缓存
         下次 cron 触发时自动跳过
```

---

## 9. 常见问题

**Q：手动 workflow_dispatch 也会被去重拦截？**
A：在 coordinator 中选 `force=true` 即可跳过。在报告仓库单独触发时加 `--force` 参数。

**Q：cron 在 GitHub Actions 上有延迟怎么办？**
A：GitHub 的 cron 有时延迟 10-30 分钟，属正常现象。21:15 UTC 触发可能实际在 21:30-21:45 执行。对于收到邮件的时间（北京次日清晨）没有影响。

**Q：如何确认触发链路正常？**
A：`daily-report-trigger → Actions` 中看到绿色 ✅，然后在各报告仓库的 `Actions` 中能看到新的 workflow run。

**Q：月报没有在月末触发？**
A：检查 `check-market` job 的日志，确认 `is_month_end=true`。若计算有误，可在 `coordinator.yml` 的 `US_HOLIDAYS` 中补充当年假期。
