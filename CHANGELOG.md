# Changelog

本文件记录 cyberquant-skill 的版本演进。版本号遵循语义化版本（SemVer）。

## 0.2.0

### 新增

- **第 6 项能力「接口请求示例代码」（路径 D）**：当用户想把某接口集成进自己程序做定期更新时，技能复用路由发现拿到 `routeSlug` + 参数，生成可直接运行的 **Node** 与 **Python** 两版示例；语言不明确时反问用户确认。
- 示例模板分两个独立文件存放（各自维护、互不影响）：`references/example-node.md`（封装本技能已自动安装的 `cyberquant-cli`，不重写 HTTP 层）、`references/example-python.md`（用官方 [`cyberquant`](https://pypi.org/project/cyberquant/) 包，附 `pip install cyberquant` 安装说明）。
- **示例默认「单页查询 + 注释拉全量」**：首版只给「一次调用 + 打印结果」——分页默认只查第一页，用一行注释写明拉全量的方式（Node `query` 加 `--all`、Python `client.query` 改 `client.query_all`）；日期参数默认拉**最近 7 天**并注释「按需手动调整区间」；不自加数据累积 / 合并 / 增量回补 / 全量回填 / 本地缓存等业务逻辑（需要时由用户基于示例追问再演进）。数据量较大、想要更高效的批量 / 全量拉取时再给 SSE 流式段。
- **SSE `stream` 定位为「更高效的批量拉取」**：本平台数据均为盘后、按日更新、**无实时数据**——`stream`（SSE）是长连接一次流式拉取、**比分页查询更高效**的拉取方式（适合数据量较大），并非「实时连续数据」，按日调度即可、勿塞进短间隔调度反复起。

### 文档对齐

- README.md / CHANGELOG.md 此前落后于 SKILL.md（仍为 3 项），现补齐「字段定位」「账户与权限查询」两项，三处文档统一为 **6 项**能力。

## 0.1.0

首发版本。

### 新增

- 作为 CyberQuant 数据能力的统一入口（Claude Code 技能）。
- **第 0 步首次自检与引导安装**（幂等）：检查并安装 `cyberquant-cli`、注册 `cyberquant-mcp`（用户级 `-s user`）、检查并配置 apiKey（调用 MCP `configure` 工具）。
- **第 1 步能力告知**：数据路由查询、数据分析、数据导出。
- **第 2 步意图路由**：
  - 分析数据 → MCP `list_routes → get_route_detail → query_data`（CSV，pageSize 上限 1000，不自动翻页）。
  - 导出数据 → CLI `cyberquant-cli stream ... --output`（固定 stream 方式），含 MCP params → CLI flags 参数翻译规则。

### 变更

- **导出去除分页参数**：stream（SSE）为长连接流式拉取，CLI flags 不再传 `--pageSize`/`--page` 等分页参数（数据自动流式拉取至结束）。
- **导出改为后台异步执行**：`cyberquant-cli stream` 改用 Bash `run_in_background: true` 在独立线程执行——启动后**立即回复用户「数据导出中」**（提示关注输出文件内容变化），**后台任务完成后再回复「导出完成」**（给出文件路径与行数），不再阻塞对话、不再同步等待。
