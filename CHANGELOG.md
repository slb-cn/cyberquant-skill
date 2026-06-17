# Changelog

本文件记录 cyberquant-skill 的版本演进。版本号遵循语义化版本（SemVer）。

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
