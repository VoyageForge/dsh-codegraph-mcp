# @voyageforge/dsh-codegraph-mcp

把本机安装的 [CodeGraph](https://github.com/colbymchenry/codegraph) CLI 接成 **DSH（DeepSeek Harness）的 MCP 服务器**，让模型可以直接调用 `codegraph_explore`，而不必先跑一轮 grep / 读文件。

这是**连接**（工具本身）；配套的四个人格（`@voyageforge/dsh-csharp-preset` 等）里还写有**行为规则**（什么时候用、怎么判断工程有没有索引、CLI 缺失怎么办）。两者缺一不可：只有规则没有连接，模型会找不到工具；只有连接没有规则，工具可能被闲置。

## 前置条件

CodeGraph CLI **不由本 bundle 安装**，需要先自行装好：

```powershell
# Windows
irm https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.ps1 | iex
# 或已有 Node
npm i -g @colbymchenry/codegraph
```

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/colbymchenry/codegraph/main/install.sh | sh
# 或
npm i -g @colbymchenry/codegraph
```

然后给需要查询的工程建索引（每个工程一次，之后自动增量同步）：

```bash
cd your-project
codegraph init
```

## 安装到 DSH

```bash
dsh plugin --profile <profile> add git+https://github.com/VoyageForge/dsh-codegraph-mcp.git
```

或在 profile 的 `package.json` 里声明：

```json
{
  "dependencies": {
    "@voyageforge/dsh-codegraph-mcp": "git+https://github.com/VoyageForge/dsh-codegraph-mcp.git"
  },
  "dsh": {
    "profile": {
      "bundles": ["@voyageforge/dsh-codegraph-mcp"]
    }
  }
}
```

安装后重启 DSH（或新开会话），工具以 `mcp__codegraph__codegraph_explore` 出现。

## 它做了什么

在 profile 里插入一条 `@deepseek-ai/dsh-mcp-client` 条目，`serverName: codegraph`，用 stdio 启动 CodeGraph 的 MCP 模式。

关键设计：

| 决定 | 原因 |
|---|---|
| `command` 指向 CodeGraph 自带运行时的 `node.exe`，而非 `codegraph` | npm 只装出 `.cmd` / `.ps1` / 无扩展名 shim；DSH 以 `shell=false` 启动子进程，Node 24 起拒绝直接 spawn `.cmd`（EINVAL），无扩展名 shim 在 Windows 也不可执行 |
| 路径写成 `!!js` 表达式（从 `%APPDATA%` 推导） | 条目要能跨机器安装，不写死用户名 |
| `cwd: process.cwd()`，**不传 `--path`** | 跟随 DSH 打开的工程自动绑定索引；写死 `--path` 会让默认工程永远固定为一个 |
| 带 `--liftoff-only --disable-warning=ExperimentalWarning` | 前者规避 Node ≥ 22 上 tree-sitter WASM 的 Zone OOM，后者压掉 `node:sqlite` 告警以保持 stdio 干净 |

## 默认工程：`--path` 与 `projectPath` 的取舍（实测）

DSH 的 MCP 客户端**不声明 MCP roots**，而本 bundle 用 `cwd: process.cwd()`，所以服务器拿到的工作目录是 **DSH 宿主的启动目录**，不是某个会话的工作目录。实测（codegraph 1.6.2，宿主从 `F:\Projects\Web\deepseek-harness` 启动）不带 `projectPath` 调用时报：

```
No CodeGraph project is loaded for this session.
Searched for a .codegraph/ directory starting from: F:\Projects\Web\deepseek-harness
  • Pass projectPath to the tool call, e.g. projectPath: "/absolute/path/to/your/project"
  • Or add --path to the server's MCP config args: ["serve", "--mcp", "--path", "/absolute/path/to/your/project"]
```

两种应对方式：

| 方式 | 效果 | 代价 |
|---|---|---|
| **每次调用传 `projectPath`**（本 bundle 的取向） | 一个服务器可查任意多个已索引工程，条目保持跨机器通用 | 每次调用都要带参数；不带就报上面那条错 |
| 在 `args` 里**加 `--path <工程>`** | 该工程成为默认工程，调用无需参数 | **只能绑定一个工程**，且把机器专属路径写进了共享仓库，牺牲可移植性 |

配合 [`@voyageforge/dsh-csharp-preset`](https://github.com/VoyageForge/dsh-csharp-preset) 等预设使用时，预设的 persona 规则已经写明这一条——agent 会主动带上 `projectPath`，使用者通常无感。

如果你的布局固定、且几乎只在一个大工程里工作，想换默认绑定，注意**不要**在 profile 补丁层写 `config: { cwd: ... }` 之类的不完整覆盖：DSH 的 loader 对已有条目的 `config` 是**整体替换而非合并**，只写一个字段会清掉 `command` 与 `args`，连接直接失效（实测确认）。要么在本文件的 `args` 里加 `--path`，要么提供完整的 config。

## 跨平台

本文件目前只覆盖 **Windows**（从 `process.env.APPDATA` 推导路径）。macOS / Linux 上 `APPDATA` 不存在，需要把两条 `!!js` 改成对应位置，例如：

```yaml
command: !!js process.getBuiltinModule('node:path').join(process.env.HOME, '.npm-global/bin/codegraph')
```

（实际前缀以 `npm root -g` 的输出为准；Unix 平台可直接调用包内的 `bin/codegraph` 启动器，无需 `node.exe` 变通。）

## 注意

- 同一个 profile 里**只能有一条** `serverName: codegraph` 的条目；重复会因 `serverName` 已被占用而加载失败。
- 宿主从非工程目录启动时，服务器不绑定默认工程，但工具仍可用——按 CodeGraph 官方指引在调用时传 `projectPath`。
- 一个工程同时只允许一个 CodeGraph 写入者；多个 MCP 实例时，其余实例会代理到后台守护进程，不需要额外处理。

## License

内部工程，未声明开源许可证。
