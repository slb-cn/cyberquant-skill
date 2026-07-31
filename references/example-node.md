# Node 接口请求示例模板

把某条 CyberQuant 数据接口（routeSlug）的调用**封装进你自己的 Node 程序**，做定期数据更新。

## 前置

- **Node.js ≥ 20**（用内置 `child_process`，无需额外 npm 依赖）。
- **`cyberquant-cli` 已安装**：本技能在「环境就绪检查」时会自动 `npm install -g cyberquant-cli`。若你是在技能之外单独使用，先跑 `npm install -g cyberquant-cli`。
- **配置已就绪**：`~/.cyberquant/config.json` 已含 `apiKey`（本技能会代为写入）。纯命令行环境首次可跑 `cyberquant-cli config set` 交互式写入。
- 路由与参数请先用 `cyberquant-cli` 或技能的「数据路由查询」确认（拿到真实的 `routeSlug` 与支持的参数名/类型）。

## 思路

不自己拼 HTTP，直接用 `child_process` 调本技能已装好的 `cyberquant-cli`：
- **单页查询** `query <slug>` —— **默认只返回第一页**；最简调用，首版示例用它。
- **全量查询** `query <slug> --all` —— 自动翻页拉全量；在单页命令后多加一个 `--all` 即可，需要时再用。
- **流式拉取** `stream <slug>` —— SSE 长连接一次拉完，**比分页查询更高效**，适合数据量较大的批量 / 全量拉取。

## 完整示例（保存为 `.mjs` 直接跑）

```javascript
// 依赖：Node.js ≥ 20；需已安装 cyberquant-cli（本技能环境就绪检查已自动 npm i -g 装好）。
// 思路：用 child_process 调 cyberquant-cli，不自己拼 HTTP。
//   定时触发：系统 cron，或 node-cron 包。
import { execFile, spawn } from 'node:child_process';
import { promisify } from 'node:util';
const run = promisify(execFile);

// { key: "v" | ["v1","v2"] }  ->  ["--key","v"] / ["--key","v1","--key","v2"]
// 多值/日期范围都用数组：tradeTime: ["2026-07-01","2026-07-31"] -> --tradeTime 2026-07-01 --tradeTime 2026-07-31
function toFlags(params) {
  const flags = [];
  for (const [k, v] of Object.entries(params)) {
    for (const item of (Array.isArray(v) ? v : [v])) flags.push(`--${k}`, String(item));
  }
  return flags;
}

// ① 单页查询（默认只返回第一页）—— 最简调用，结果落盘
async function query(routeSlug, params, outputFile) {
  const { stdout, stderr } = await run('cyberquant-cli', [
    'query', routeSlug, '--format', 'csv', '--output', outputFile, ...toFlags(params),
    // '--all',   // ⬆ 拉全量（自动翻页）：取消本行注释即可
  ]);
  if (stderr) console.error(stderr);
  console.log(stdout);                 // CLI 完成摘要（含行数）
}

// ② SSE 流式拉取 —— 长连接一次拉完，比 ① 分页更高效，适合较大数据量；边收边写文件
function stream(routeSlug, params, outputFile) {
  return new Promise((resolve, reject) => {
    const p = spawn('cyberquant-cli', [
      'stream', routeSlug, '--format', 'csv', '--output', outputFile, ...toFlags(params),
    ]);
    p.stdout.on('data', d => process.stdout.write(d));
    p.stderr.on('data', d => process.stderr.write(d));
    p.on('error', reject);
    p.on('exit', code => (code === 0 ? resolve() : reject(new Error(`exit ${code}`))));
  });
}

// 示例：查询平安银行（000001.SZ）最近 7 天的日 K 线
await query(
  'a-daily-bfq',
  { tradeDate: ['2026-07-25', '2026-07-31'], stockCode: '000001.SZ' },  // 最近 7 天，按需手动调整区间
  'daily-bfq.csv',
);
```

## 定时调度

- **系统 cron**（推荐，最省事）：每天收盘后跑一次上面的脚本。
  ```bash
  # crontab -e  —— 每天 16:30 跑一次（按需在 query 里加 --all 拉全量）
  30 16 * * 1-5 cd /path/to/job && /usr/bin/node update.mjs >> /path/to/job/cron.log 2>&1
  ```
- **node-cron 包**（进程内调度）：
  ```javascript
  import cron from 'node-cron';
  cron.schedule('30 16 * * 1-5', () => query('a-daily-bfq', { tradeDate: ['2026-07-25', '2026-07-31'], stockCode: '000001.SZ' }, 'daily-bfq.csv'));
  ```

## 备注

- **Windows**：`execFile('cyberquant-cli', ...)` 在 Windows 下需走 `.cmd`（给 `run` 传 `{ shell: true }`，或直接调 `cyberquant-cli.cmd`）；建议在 Git Bash / WSL 或 Linux / macOS 下跑定时任务。
- **stream 是长连接**：本平台数据均为盘后、按日更新，**没有实时数据**。② `stream` 只是把同一批盘后数据用长连接一次拉完、**比分页更高效**的拉取方式（适合较大数据量），并非「实时」；数据每日才更新一次、长连接起停也有开销，不要塞进短间隔 cron 反复起，按日调度即可。
- **输出文件**：`--output` 指定的文件会随拉取逐步增长；`stdout` 里是 CLI 的完成摘要（含行数），可据此判进度。
- 若想要 JSON 而非 CSV，把 `--format csv` 换成 `--format json`，输出文件后缀相应改 `.json`。
