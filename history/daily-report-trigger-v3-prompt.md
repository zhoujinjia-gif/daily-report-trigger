# daily-report-trigger-v3-prompt.md

# Daily Report Trigger v3 — 确认实施 + history/ 归档

放置于: daily-report-trigger/history/daily-report-trigger-v3-prompt.md
注意: history/ 中已有 v2 文件，本次只新建 v3 文件，不删除或覆盖 v2 文件。
完成后新建 daily-report-trigger-architecture-v3.md 存入同目录。

-----

## 0. v3 的目标

v2 已定义了完整的 coordinator 架构（单 cron 21:15 UTC + 三层去重 + 月末判断）。
v3 的任务是：

1. 确认 coordinator.yml 已按 v2 架构正确实施（逐项对照检查）
1. 若有遗漏，按本文补全
1. 建立 history/ 归档目录，存入 v3 文档
1. 写好 architecture-v3.md 供 AI 理解

-----

## 1. 执行前准备

```bash
cd /path/to/daily-report-trigger
mkdir -p history

# 确认已有文件不被删除
ls history/   # v2 文件应完好

# 查看现有 workflow
cat .github/workflows/coordinator.yml
```

-----

## 2. 对照检查清单（v2 架构 → 实际文件）

逐项确认 coordinator.yml 是否包含以下内容，缺失则补充：

### 2a. Cron 触发

```yaml
# 应只有这一个 cron，不能有其他 schedule
on:
  schedule:
    - cron: '15 21 * * 1-5'
  workflow_dispatch:
    inputs:
      force:
        type: choice
        options: ['false', 'true']
        default: 'false'
      reports:
        default: 'a_share,us_equity'
```

检查项：

- ☐ 只有一个 cron（无双 cron 旧版）
- ☐ workflow_dispatch 含 force 和 reports 两个 input

### 2b. Job 结构

检查是否有以下 6 个 job（名称可略有差异）：

- ☐ `dedup-check` — 使用 actions/cache 检查今日是否已 dispatch
- ☐ `check-market` — Python 脚本判断交易日 + 月末
- ☐ `dispatch-a-share` — 向 A-Share-report 发 repository_dispatch
- ☐ `dispatch-us-equity` — 向 US-Equity-report 发 repository_dispatch
- ☐ `dispatch-monthly` — 向 monthly-full-market-report 发（仅月末）
- ☐ `mark-dispatched` — 写入去重缓存

### 2c. dedup-check job 关键细节

```yaml
# cache key 格式
key: coordinator-dispatched-${{ steps.date.outputs.TODAY }}

# outputs 必须引用 steps.decide.outputs（不是 steps.check.outputs）
outputs:
  should_run: ${{ steps.decide.outputs.should_run }}
  today: ${{ steps.decide.outputs.today }}
```

检查项：

- ☐ cache key 含日期变量
- ☐ outputs 来自正确的 step id

### 2d. check-market Python 脚本关键细节

```python
# force 模式下直接返回 true/true
FORCE = os.environ.get('FORCE', '')
if FORCE == 'true':
    # is_trading_day=true, is_month_end=true
    # （方便手动测试月报）

# 月末判断：扫描未来3天，无同月交易日则为月末
for i in range(1, 4):
    future = today + timedelta(days=i)
    if future.month != today.month: break
    if future.weekday() < 5 and future not in US_HOLIDAYS:
        has_future = True; break
is_month_end = not has_future
```

检查项：

- ☐ force 模式绕过所有检查
- ☐ 月末判断逻辑正确（检查未来3天）
- ☐ US_HOLIDAYS 包含 2026 和 2027 年主要假期

### 2e. Dispatch job 认证

```yaml
# 必须使用 secrets.REPO_PAT，必须是 Classic PAT（repo scope）
github-token: ${{ secrets.REPO_PAT }}
```

检查项：

- ☐ 使用 REPO_PAT 而非 GITHUB_TOKEN
- ☐ PAT 是 Classic 类型（不是 Fine-grained）
- ☐ PAT 勾选了 repo scope

### 2f. 仓库名称

```javascript
// 确认这三行仓库名与实际 GitHub 仓库名完全一致（大小写敏感）
repo: 'A-Share-report'
repo: 'US-Equity-report'
repo: 'monthly-full-market-report'
```

如实际仓库名不同，更新为正确名称。

### 2g. mark-dispatched job 条件

```yaml
# 必须用 always() 确保即使 dispatch job 失败也写入标记
if: always() && needs.dedup-check.outputs.should_run == 'true'
```

检查项：

- ☐ 条件包含 always()

-----

## 3. 若 coordinator.yml 不存在或严重缺失

若文件完全不存在，或与 v2 架构差异太大，完整创建：

```bash
mkdir -p .github/workflows
cat > .github/workflows/coordinator.yml << 'YAML_EOF'
name: Daily Report Coordinator

on:
  schedule:
    - cron: '15 21 * * 1-5'
  workflow_dispatch:
    inputs:
      force:
        description: '强制运行（跳过去重和交易日检查）'
        required: false
        default: 'false'
        type: choice
        options: ['false', 'true']
      reports:
        description: '发送哪些报告（逗号分隔: a_share,us_equity,monthly）'
        required: false
        default: 'a_share,us_equity'

jobs:
  dedup-check:
    runs-on: ubuntu-latest
    outputs:
      should_run: ${{ steps.decide.outputs.should_run }}
      today: ${{ steps.decide.outputs.today }}
    steps:
      - name: Get date
        id: date
        run: echo "TODAY=$(date -u +%Y-%m-%d)" >> $GITHUB_OUTPUT

      - name: Restore dedup cache
        id: cache
        uses: actions/cache/restore@v4
        with:
          path: .coordinator-dedup
          key: coordinator-dispatched-${{ steps.date.outputs.TODAY }}

      - name: Decide whether to run
        id: decide
        run: |
          FORCE="${{ github.event.inputs.force }}"
          CACHE_HIT="${{ steps.cache.outputs.cache-hit }}"
          TODAY="${{ steps.date.outputs.TODAY }}"
          if [ "$FORCE" = "true" ]; then
            echo "should_run=true" >> $GITHUB_OUTPUT
            echo "today=$TODAY"   >> $GITHUB_OUTPUT
            echo "✅ Force mode"
          elif [ "$CACHE_HIT" = "true" ]; then
            echo "should_run=false" >> $GITHUB_OUTPUT
            echo "today=$TODAY"     >> $GITHUB_OUTPUT
            echo "⏭ Already dispatched today"
          else
            echo "should_run=true" >> $GITHUB_OUTPUT
            echo "today=$TODAY"    >> $GITHUB_OUTPUT
            echo "✅ First run today"
          fi

  check-market:
    runs-on: ubuntu-latest
    needs: dedup-check
    if: needs.dedup-check.outputs.should_run == 'true'
    outputs:
      is_trading_day: ${{ steps.market.outputs.is_trading_day }}
      is_month_end:   ${{ steps.market.outputs.is_month_end }}
    steps:
      - name: Market check
        id: market
        env:
          FORCE: ${{ github.event.inputs.force }}
        run: |
          python3 << 'PYEOF'
          import os, sys
          from datetime import date, timedelta

          FORCE = os.environ.get('FORCE','') == 'true'
          if FORCE:
              print("is_trading_day=true"); print("is_month_end=true")
              os.system("echo 'is_trading_day=true' >> $GITHUB_OUTPUT")
              os.system("echo 'is_month_end=true'   >> $GITHUB_OUTPUT")
              print("⚡ Force: skipping market checks"); sys.exit(0)

          today = date.today()
          if today.weekday() >= 5:
              os.system("echo 'is_trading_day=false' >> $GITHUB_OUTPUT")
              os.system("echo 'is_month_end=false'   >> $GITHUB_OUTPUT")
              print("Weekend, not trading"); sys.exit(0)

          US_HOLIDAYS = {
              date(2026,1,1), date(2026,1,19), date(2026,2,16),
              date(2026,4,3), date(2026,5,25), date(2026,7,3),
              date(2026,9,7), date(2026,11,26), date(2026,12,25),
              date(2027,1,1), date(2027,1,18), date(2027,2,15),
              date(2027,3,26), date(2027,5,31), date(2027,7,5),
              date(2027,9,6), date(2027,11,25), date(2027,12,24),
          }

          is_trading = today not in US_HOLIDAYS
          is_month_end = False
          if is_trading:
              has_future = False
              for i in range(1, 4):
                  f = today + timedelta(days=i)
                  if f.month != today.month: break
                  if f.weekday() < 5 and f not in US_HOLIDAYS:
                      has_future = True; break
              is_month_end = not has_future

          os.system(f"echo 'is_trading_day={'true' if is_trading else 'false'}' >> $GITHUB_OUTPUT")
          os.system(f"echo 'is_month_end={'true' if is_month_end else 'false'}'   >> $GITHUB_OUTPUT")
          print(f"Trading: {is_trading}, Month end: {is_month_end}")
          PYEOF

  dispatch-a-share:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      (github.event.inputs.reports == '' ||
       contains(github.event.inputs.reports, 'a_share'))
    steps:
      - name: Dispatch A-Share report
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_PAT }}
          script: |
            const today = '${{ needs.dedup-check.outputs.today }}';
            const force = '${{ github.event.inputs.force }}' === 'true';
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'A-Share-report',
              event_type: 'send_report',
              client_payload: { date: today, force, source: 'coordinator' }
            });
            console.log(`✅ A-Share dispatched for ${today}`);

  dispatch-us-equity:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      (github.event.inputs.reports == '' ||
       contains(github.event.inputs.reports, 'us_equity'))
    steps:
      - name: Dispatch US Equity report
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.REPO_PAT }}
          script: |
            const today = '${{ needs.dedup-check.outputs.today }}';
            const force = '${{ github.event.inputs.force }}' === 'true';
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'US-Equity-report',
              event_type: 'send_report',
              client_payload: { date: today, force, source: 'coordinator' }
            });
            console.log(`✅ US-Equity dispatched for ${today}`);

  dispatch-monthly:
    runs-on: ubuntu-latest
    needs: [dedup-check, check-market]
    if: |
      needs.check-market.outputs.is_trading_day == 'true' &&
      needs.check-market.outputs.is_month_end   == 'true' &&
      (github.event.inputs.reports == '' ||
       contains(github.event.inputs.reports, 'monthly'))
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
              repo: 'monthly-full-market-report',
              event_type: 'send_report',
              client_payload: { date: today, force, source: 'coordinator', is_month_end: true }
            });
            console.log(`✅ Monthly dispatched for ${today} (MONTH END)`);

  mark-dispatched:
    runs-on: ubuntu-latest
    needs: [dedup-check, dispatch-a-share, dispatch-us-equity]
    if: always() && needs.dedup-check.outputs.should_run == 'true'
    steps:
      - name: Create dedup marker
        run: |
          mkdir -p .coordinator-dedup
          echo "${{ needs.dedup-check.outputs.today }}" > .coordinator-dedup/last.txt
      - name: Save dedup cache
        uses: actions/cache/save@v4
        with:
          path: .coordinator-dedup
          key: coordinator-dispatched-${{ needs.dedup-check.outputs.today }}
YAML_EOF
echo "✅ coordinator.yml created"
```

-----

## 4. 确认各报告仓库 workflow 的 on: 声明

在三个报告仓库中，各自的 `.github/workflows/daily_report.yml`（或同名）的 `on:` 块：

```yaml
# 必须只有这两个触发方式，不能有 schedule/cron
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

若发现仍有 `schedule:` / `cron:` 字段，删除它。

-----

## 5. 确认三个报告仓库的 Python 去重逻辑

三个仓库的 `main.py` 的 `run()` 函数入口处，确认含以下结构：

```python
def run(force=False, dry_run=False):
    if not force and _dedup_check(DB_PATH):
        print("⏭ [DEDUP] 已在过去 23 小时内成功发送报告，跳过")
        return 0
    # ...

def _dedup_check(db_path, window_hours=23):
    import sqlite3
    from datetime import datetime, timedelta
    threshold = (datetime.utcnow() - timedelta(hours=window_hours)).isoformat()
    try:
        conn = sqlite3.connect(str(db_path))
        conn.execute("""CREATE TABLE IF NOT EXISTS reports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp_utc TEXT NOT NULL,
            email_sent INTEGER DEFAULT 0,
            is_trading_day INTEGER DEFAULT 1,
            run_duration_ms INTEGER,
            error_message TEXT
        )""")
        conn.commit()
        row = conn.execute(
            "SELECT id FROM reports WHERE email_sent=1 AND timestamp_utc>? LIMIT 1",
            (threshold,)
        ).fetchone()
        conn.close()
        return row is not None
    except Exception as e:
        print(f"⚠ dedup check error: {e}"); return False
```

若缺失，在各报告仓库的 main.py 中添加。

-----

## 6. Secrets 确认清单

|仓库                        |Secret 名                                 |是否已配置|
|--------------------------|-----------------------------------------|:---:|
|daily-report-trigger      |REPO_PAT（Classic PAT, repo scope）        |请确认  |
|A-Share-report            |A_QQ_EMAIL                               |请确认  |
|A-Share-report            |A_QQ_SMTP_CODE                           |请确认  |
|A-Share-report            |A_RECIPIENT_EMAIL                        |请确认  |
|US-Equity-report          |QQ_EMAIL / QQ_SMTP_CODE / RECIPIENT_EMAIL|请确认  |
|US-Equity-report          |ALPHA_VANTAGE_KEY                        |请确认  |
|US-Equity-report          |EXCHANGE_RATE_KEY                        |请确认  |
|monthly-full-market-report|同上所有                                     |请确认  |

若 REPO_PAT 是 Fine-grained PAT（会导致 403），需重新创建 Classic PAT：
GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
→ Generate new token → 勾选 repo scope → 复制 → 更新 Secrets

-----

## 7. 端到端测试

```bash
# GitHub UI: daily-report-trigger → Actions → Daily Report Coordinator
# → Run workflow → force=true → reports=a_share
# 观察 Jobs 执行情况

# 验证清单:
# ☐ dedup-check job ✅ 绿色（first run today）
# ☐ check-market job ✅ 绿色（is_trading_day=true，因为 force=true）
# ☐ dispatch-a-share job ✅ 绿色
# ☐ A-Share-report 仓库 Actions 中出现新的 workflow run
# ☐ 收到 A股 邮件一封（不是两封）
# ☐ 再次手动触发（不加 force）→ dedup-check 返回 should_run=false，所有 dispatch 跳过
```

-----

## 8. 存档 history/ 目录（新建，不覆盖 v2）

```bash
# 确认不覆盖已有文件
ls history/  # 应包含 v2 相关文件

# 本文件已通过 agent 操作存入:
# history/daily-report-trigger-v3-prompt.md   ← 本文件

# 测试通过后，将 coordinator.yml 备份到 history/
cp .github/workflows/coordinator.yml history/coordinator-v3.yml
echo "✅ coordinator v3 备份到 history/"
```

-----

## 9. 写入 architecture-v3 文档（新建）

```bash
# 新建，不覆盖 v2
touch history/daily-report-trigger-architecture-v3.md
```

在 `history/daily-report-trigger-architecture-v3.md` 中写入完整内容。
基于已有的 v2 架构文档（daily-report-trigger-architecture-v2.md）全文，
更新以下部分：

v3 核心变更:

```
1. 修复双发 bug：确认单 cron (15 21 * * 1-5)，删除所有旧 schedule
2. 确认三层去重：coordinator cache / 报告仓库无 cron / Python SQLite
3. 确认 Classic PAT（非 Fine-grained）用于 dispatch
4. 增加月报 dispatch-monthly job（条件: is_month_end=true）
5. check-market Python 脚本 force 模式优先逻辑
6. mark-dispatched job 使用 always() 条件
7. history/ 目录归档（v2 文件保留，v3 新增）
8. coordinator.yml 备份到 history/coordinator-v3.yml
```

版本记录表增加一行:
| v3 | 2026-05-31 | 实施确认 + dedup强化 + history/归档 + architecture-v3.md |
更新readme和readme-cn，确认没问题后更新推送github
-----

## 10. 最终文件清单

```
daily-report-trigger/
├── .github/workflows/coordinator.yml    ← 核心（已存在或重建）
├── README.md                            ← 已存在，可更新版本号
└── history/
    ├── [v2 已有文件保持不变]            ← 不删除不修改
    ├── daily-report-trigger-v3-prompt.md          ← 本文件（新建）
    ├── coordinator-v3.yml                          ← 步骤 8 备份（新建）
    └── daily-report-trigger-architecture-v3.md    ← 步骤 9 写入（新建）
```
