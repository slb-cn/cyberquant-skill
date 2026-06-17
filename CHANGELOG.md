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
