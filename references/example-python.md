# Python 接口请求示例模板

把某条 CyberQuant 数据接口（routeSlug）的调用**集成进你自己的 Python 程序**，做定期数据更新。使用官方 [`cyberquant`](https://pypi.org/project/cyberquant/) 包的 Python API（无需自己拼 HTTP）。

## 安装与配置

```bash
# 安装（二选一）
pip install cyberquant          # 或：pipx install cyberquant（隔离环境，推荐）
```

- 包主页：<https://pypi.org/project/cyberquant/>
- 要求 Python ≥ 3.10。
- **首次配置**（写入 `~/.cyberquant/config.json`，与 cyberquant-cli / cyberquant-mcp 共用）：
  ```bash
  cyberquant config set         # 交互式输入 API 端点与 API Key
  ```
  若本技能已帮你配过（`~/.cyberquant/config.json` 已含 `apiKey`），则跳过此步。

## 思路

用官方包的公开 API：
- **单页查询** `ApiClient.query(slug, params)` —— **默认只返回第一页**（默认 pageSize 50）；最简调用，首版示例用它。
- **全量查询** `ApiClient.query_all(slug, params)` —— 自动翻页拉全量；在单页示例基础上换一个方法名即可，需要时再用。
- **流式拉取** `connect_sse(...)` —— SSE 长连接一次拉完，**比分页查询更高效**，适合数据量较大的批量 / 全量拉取。

## 完整示例（直接跑）

```python
# 安装：pip install cyberquant   （或 pipx install cyberquant）
# 包主页：https://pypi.org/project/cyberquant/
# 首次配置（写入 ~/.cyberquant/config.json）：cyberquant config set
from cyberquant.lib.config import require_config
from cyberquant.lib.api_client import ApiClient

cfg = require_config()

# ① 单页查询（默认只返回第一页）—— 最简调用，首版示例用它
with ApiClient(cfg) as client:
    res = client.query("<routeSlug>", {
        "<参数1>": "<值1>",
        # date 范围用两元素数组："<日期参数>": ["2026-07-25", "2026-07-31"],  # 最近 7 天，按需手动调整区间
        # "pageSize": 1000,   # 可选，默认 50，上限 1000
    })
print(res["data"])            # 当前页数据
# ⬆ 拉全量（自动翻页）改用：rows = client.query_all("<routeSlug>", { ... })，再 print(rows)

# ② SSE 流式拉取 —— 长连接一次拉完，比 ① 分页更高效，适合较大数据量；边收边落盘
from cyberquant.lib.sse_client import connect_sse
from cyberquant.lib.formatter import create_stream_writer
writer = create_stream_writer("csv", "output.csv", cfg)
writer.initialize()
connect_sse(cfg, "<routeSlug>", {"<参数>": "<值>"},
            on_data=lambda d: writer.write_record(d))
writer.finalize()
```

## 真实示例：查询平安银行最近 7 天日 K 线

```python
from cyberquant.lib.config import require_config
from cyberquant.lib.api_client import ApiClient

cfg = require_config()

with ApiClient(cfg) as client:
    res = client.query("a-daily-bfq", {
        "tradeDate": ["2026-07-25", "2026-07-31"],   # 最近 7 天，闭区间，按需手动调整
        "stockCode": "000001.SZ",
    })
# 拉全量（自动翻页）改用：rows = client.query_all("a-daily-bfq", { ... })
rows = res["data"]
print(f"共 {len(rows)} 条，首条：{rows[0] if rows else None}")
```

## 定时调度

- **系统 cron**（推荐）：
  ```bash
  # crontab -e  —— 每天 16:30 跑一次（按需在脚本里把 query 改成 query_all 拉全量）
  30 16 * * 1-5 cd /path/to/job && /usr/bin/python3 update.py >> /path/to/job/cron.log 2>&1
  ```
- **APScheduler / schedule**（进程内调度）：
  ```python
  from apscheduler.schedulers.blocking import BlockingScheduler
  sched = BlockingScheduler()
  sched.add_job(lambda: update_daily(), 'cron', day_of_week='mon-fri', hour=16, minute=30)
  sched.start()
  ```

## 备注

- `client.query` **默认只返回第一页**（单页，返回 `{success, data, meta:{pagination:{hasMore, nextCursor}}}`，默认 pageSize 50、上限 1000）；`client.query_all` 在此基础上自动按 `nextCursor` 翻页、返回完整 `list`——换一个方法名即可从「第一页」升级到「全量」。
- **stream 是长连接**：本平台数据均为盘后、按日更新，**没有实时数据**。② `connect_sse` 只是把同一批盘后数据用长连接一次拉完、**比分页更高效**的拉取方式（适合较大数据量），并非「实时」；数据每日才更新一次、长连接起停也有开销，不要塞进短间隔调度反复起，按日调度即可。
- 路由与参数请先用 `cyberquant` CLI（`cyberquant query ...`）或技能的「数据路由查询」确认真实的 `routeSlug` 与支持的参数名/类型。
