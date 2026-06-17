# cyberquant-skill

> 一个 [Claude Code](https://docs.claude.com/en/docs/claude-code) 技能（Skill），作为 **CyberQuant 数据共享API服务平台**的统一入口：自动安装并配置 [cyberquant-mcp](https://www.npmjs.com/package/cyberquant-mcp) 与 [cyberquant-cli](https://www.npmjs.com/package/cyberquant-cli)，之后根据你的自然语言意图自动路由——**查询/分析数据走 MCP，导出数据到文件走 CLI（stream 方式）**。

## 它能做什么

装上这个技能后，在 Claude Code 对话里直接用自然语言提需求即可，技能会自动完成「安装 → 配置 → 路由」全流程，提供三项能力：

| 能力 | 说明 | 走哪个工具 |
|---|---|---|
| **数据路由查询** | 看有哪些数据、某条路由支持哪些参数 | MCP `list_routes` / `get_route_detail` |
| **数据分析** | 对话内即时查看/分析（小数据，返回 CSV） | MCP `query_data` |
| **数据导出** | 把数据流式导出成文件（大数据/留档） | CLI `cyberquant-cli stream` |

## 前置要求

- **Node.js ≥ 20**
- 可访问 npm 公网（首次会从 npm 拉取 cyberquant-mcp / cyberquant-cli）
- 一个CyberQuant**API Key**（格式 `sk_live_xxx`；登录 [https://quant.cyberspace2077.com/](https://quant.cyberspace2077.com/) 获取，无账号可联系微信号 **lghxt520** 申请。首次使用时技能也会向你索取并代为写入）

## 安装（一行命令）

把本仓库 clone 到 Claude Code 的 skills 目录即可。仓库根目录就是技能目录，clone 完直接可用：

```bash
# 全局可用（所有项目都能用，推荐）
git clone git@github.com:slb-cn/cyberquant-skill.git ~/.claude/skills/cyberquant-skill
```

```bash
# 或：仅当前项目可用（clone 到该项目的 .claude/skills/ 下）
git clone git@github.com:slb-cn/cyberquant-skill.git .claude/skills/cyberquant-skill
```

安装后**重启一次 Claude Code 会话**，让技能被加载。

## 首次使用

在对话里直接提需求，技能会自动：

1. 检查并安装 `cyberquant-cli`（缺失时 `npm install -g`）；
2. 检查并注册 `cyberquant-mcp`（缺失时 `claude mcp add cyberquant-mcp -s user -- npx -y cyberquant-mcp`，**首次注册需重启会话生效**）；
3. 检查 `~/.cyberquant/config.json` 是否已配 apiKey，没有就向你索取并写入；
4. 告知你三项能力，然后按你的意图路由。

> 全新机器上首次注册 MCP 后，技能会提示你重启一次会话——这是 Claude Code 加载新 MCP 工具的固有机制，重启后即可正常使用。

## 用法示例

```
/cyberquant-skill 帮我看看都有哪些数据路由
/cyberquant-skill 分析一下平安银行（000001.SZ）最近一周的日K线数据
/cyberquant-skill 把平安银行最近一个月的日K线导出成 csv 文件
```

- **分析**类需求 → 技能依次调用 `list_routes → get_route_detail → query_data`，在对话里给出 CSV 结果。
- **导出**类需求 → 技能先用 MCP 定位路由与参数，再执行 `cyberquant-cli stream <route> --format csv --output <文件>`，完成后告诉你文件路径和行数。

## 配置文件

两个工具共用同一份配置：`~/.cyberquant/config.json`

```json
{
  "endpoint": "https://api.cyberspace2077.com",
  "apiKey": "sk_live_xxx",
  "mcp": { "pageSize": 200, "timeout": 30000 }
}
```

- `endpoint`：API Gateway 地址，一般无需修改。
- `apiKey`：首次使用时技能会代为写入，也可手动编辑此文件。
- `mcp.pageSize`：MCP 单次查询条数，默认 200，**上限 1000**。

## 关键约束

- MCP 单次查询 `pageSize` 上限 **1000**；返回 **CSV**；**不自动翻页**（数据多时会引导你缩小范围）。
- **导出固定走 `stream`** 方式。

## 更新

```bash
cd ~/.claude/skills/cyberquant-skill && git pull
```

更新后重启会话即可生效。版本变更见 [CHANGELOG.md](./CHANGELOG.md)。

## 相关项目

- [cyberquant-mcp](https://www.npmjs.com/package/cyberquant-mcp) — MCP Server，对话内查询/分析
- [cyberquant-cli](https://www.npmjs.com/package/cyberquant-cli) — 命令行，数据导出

## License

MIT
