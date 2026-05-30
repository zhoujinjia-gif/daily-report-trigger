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
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. 勾选 `repo` scope（全仓读写权限）
4. 生成 token，复制后添加到本仓库 Secrets
5. ⚠️ 必须使用 Classic PAT，Fine-grained PAT 的 Actions 权限不兼容 dispatch API
