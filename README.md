# codegraph-mcp

把 [codegraph](https://www.npmjs.com/package/@colbymchenry/codegraph) 的 stdio MCP 服务器接进
DeepSeek Harness（下称 DSH）的**纯配置型 bundle**。

它本身没有运行时代码：不带 `src`、不声明任何依赖，只靠 `cordis.patch.yml` 让 DSH 内置的
`@deepseek-ai/dsh-mcp-client` 启动 `codegraph serve --mcp`，于是会话里就多出 8 个
`mcp__codegraph__*` 工具（`codegraph_explore`、`codegraph_search`、`codegraph_node`、
`callers`、`callees`、`impact`、`files`、`status`）。

> 设计要点一：**不写死任何机器路径**。`command` 与 `args` 里的 `!!js` 表达式在插件加载时求值，
> 依次查看 `CODEGRAPH_NODE` / `CODEGRAPH_JS` 环境变量、探测常见全局 npm 根，找到 codegraph 平台包里的
> `node.exe` 与 `lib/dist/bin/codegraph.js`。Windows / macOS / Linux 都能用，换电脑不用改配置。
>
> 设计要点二：**不写死项目路径**。服务器以「无默认项目」模式启动，每次调用都由调用方显式传
> `projectPath`，所以在同一个 DSH 里可以分析任意多个已建索引的目录，换项目不用改插件。

---

## 目录结构

```
codegraph-mcp/
├── package.json          # bundle 清单：dsh.bundle.patch、exports（locale/icon）、files
├── cordis.patch.yml      # 唯一真正干活的文件：insert 一条 dsh-mcp-client 条目
├── icon.svg              # 插件管理页显示的图标（64×64，四节点图谱）
└── locale/
    ├── en.json           # 英文标题/描述
    └── zh.json           # 中文标题/描述
```

各文件职责：

| 文件 | 作用 | 是否必需 |
| --- | --- | --- |
| `package.json` | DSH 靠 `dsh.bundle.patch` 找到补丁文件；`exports` 让 GUI 能读到 `locale/*.json` 与 `./icon`；`files` 是打包白名单 | 必需 |
| `cordis.patch.yml` | 声明一个 `@deepseek-ai/dsh-mcp-client` 实例；`command`/`args` 里的 `!!js` 表达式在加载时探测本机 codegraph 的位置，**不含任何机器路径** | 必需 |
| `icon.svg` | 插件卡片图标；也可删掉并去掉 `exports["./icon"]` | 可选 |
| `locale/en.json`、`locale/zh.json` | 卡片和详情页的标题与描述，按界面语言切换 | 可选 |

`package.json` 的关键字段：

```json
{
  "name": "dsh-codegraph-mcp",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    "./locale/*.json": "./locale/*.json",
    "./icon": "./icon.svg"
  },
  "files": ["cordis.patch.yml", "locale/*.json", "icon.svg", "LICENSE", "README.md"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

- `name` 会作为安装后的包名出现在 profile 里（发布到 npm 用的就是这个 `dsh-codegraph-mcp`）。
- 想让插件限定 DSH 版本，可加 `"engines": { "dsh": ">=0.2.0" }`；不满足时安装会被拒绝并提示用
  `dsh plugin allow-version` 显式豁免。注意预发布版（例如 `0.2.1-rc.5`）要写含预发布的范围才能匹配上。

---

## 前置条件

1. **DSH**（桌面版即可），运行时需自带 `@deepseek-ai/dsh-mcp-client`（`0.2.x` 的桌面版已内置；
   本仓库在 `@deepseek-ai/dsh` 0.2.1-rc.5 上验证）。
2. **codegraph** 已安装并可用：
   ```powershell
   npm install -g @colbymchenry/codegraph
   codegraph --version      # 本仓库在 1.6.2 上验证
   ```
3. **目标项目已建索引**（这一步不能省）：
   ```powershell
   cd D:\path\to\your-project
   codegraph init
   ```
   `codegraph init` 会在项目根生成 `.codegraph/`（SQLite 库 + 自带 `.gitignore`）。
   注意 `codegraph sync` **不会**为没有索引的目录新建库，它只会向上找最近的祖先索引。

---

## 安装

### 方式一：从 npm 安装（推荐）

1. DSH 侧边栏 → **插件** → **添加插件**，填包名 `dsh-codegraph-mcp`（或命令行 `dsh plugin add dsh-codegraph-mcp`）。
2. 安装完成后点 **立即启用**，然后**重启 DSH**。

### 方式二：本地目录（开发/自用）

1. 把本仓库放到任意固定目录，**之后不要移动**（见「注意事项」中的 junction 说明）。
2. 打开 DSH 侧边栏 → **插件** → **添加插件**（或点旁边的箭头 → **安装第三方插件**）。
3. 在输入框里填该目录的**绝对路径**，例如 `D:\Code\dsh-codegraph-mcp`，回车或点「安装」。
4. 安装完成后点 **立即启用**，然后**重启 DSH**。

安装器会把这份目录以 `link:`（Windows 上是 junction）挂进
`<DSH_HOME>\profiles\<profile>\node_modules\`，并把包名加进该 profile 的
`package.json` 的 `dsh.profile.bundles`。因为是链接安装，**改了本仓库的文件后重启 DSH 就生效**，
不需要重新安装。

### 方式三：Git 仓库

同一入口选择 Git 地址即可（`https://github.com/dupengair/dsh-codegraph-mcp.git`）。
本 bundle 无依赖，安装器不需要额外跑构建。

### 卸载

插件页里对该包执行**移除**，然后重启 DSH。目标项目里的 `.codegraph/` 不受影响，可单独保留或删除。

---

## 使用

重启 DSH 后，任意会话（任意工作区）都有这 8 个工具：

| 工具（公开名） | 用途 |
| --- | --- |
| `mcp__codegraph__codegraph_explore` | 主力工具。给自然语言问题或符号名，一次返回相关符号的**带行号源码** + 调用路径 + 影响面 |
| `mcp__codegraph__codegraph_search` | 按名字查符号 |
| `mcp__codegraph__codegraph_node` | 取单个符号的定义与调用者/被调用者 |
| `mcp__codegraph__codegraph_callers` | 谁调用它 |
| `mcp__codegraph__codegraph_callees` | 它调用谁 |
| `mcp__codegraph__codegraph_impact` | 改动影响面 |
| `mcp__codegraph__codegraph_files` | 索引内的文件树 |
| `mcp__codegraph__codegraph_status` | 索引状态 |

### 必须传 `projectPath`

本配置**没有默认项目**（`args` 里没有 `--path`，也没设 `cwd`），所以 codegraph 会把自己当成
「按项目工作」的网关：

- 每个工具的 `inputSchema` 都把 `projectPath` 标成**必填**（`tools/list` 里可见）；
- `initialize` 阶段下发的服务端说明也会明确要求按项目查询；
- 不传 `projectPath` 时不会瞎猜，而是明确报错——本仓库实测原文：

  ```
  No CodeGraph project is loaded for this session.
  Searched for a .codegraph/ directory starting from: C:\Users\<you>\.dsh\profiles\desktop
  Either the server root has no index of its own (e.g. a monorepo where only sub-projects are indexed),
  or the MCP client launched the server outside your project without reporting the workspace root.
  Either way, target the project explicitly:
    • Pass projectPath to the tool call, e.g. projectPath: "/absolute/path/to/your/project"
    • Or add --path to the server's MCP config args: ["serve", "--mcp", "--path", "/absolute/path/to/your/project"]
  ```

  注意第一行里的 `Searched for … starting from:` 是 **DSH 的 profile 目录**（子进程继承宿主
  cwd），不是你的项目——这正是「没有默认项目」的来源。
- 传一个没索引的路径会返回：`The project at <path> isn't indexed with codegraph (no .codegraph/ directory found ...)`。

如果你更想要「固定默认项目 + 偶尔用 projectPath 切」，在 `args` 里补一对 `--path <项目绝对路径>`
（或设 `cwd` 到项目根）即可；本仓库刻意不这么做。

调用示例（在会话里就是自然语言，下面只是等价示意）：

```json
{ "query": "mutateElement renderScene", "projectPath": "/path/to/your-project" }
```

实际用法：

- 想在某个仓库上用 codegraph，在会话里说一句「这个项目在 `D:\Work\foo`」，后续调用都会带上它；
- 换项目就换 `projectPath`，**同一会话里可以并行查多个项目**（各自只需 `codegraph init` 过一次）；
- 路径缺失或有歧义时，助手会先问你要路径，而不是猜一个。

> `.codegraph/` 会沿目录**向上**解析，所以对一个大目录（例如某个工作区根）init 一次，
> 其所有子目录都能查；只有子目录**自身是独立 git 仓库**时会被拒绝（报 nested repository），
> 那里需要单独 init。

### 为什么打开了全部 8 个工具

codegraph 默认只暴露 `codegraph_explore`（它把其余工具的能力基本覆盖了）。本仓库通过
`env.CODEGRAPH_MCP_TOOLS` 打开了全部 8 个：

```yaml
env:
  CODEGRAPH_MCP_TOOLS: 'explore,node,search,callers,callees,impact,files,status'
```

想回到默认的最小面，把这一行删掉即可；值是逗号分隔的短名（`codegraph_` 前缀可写可不写）。

---

## 配置逐项说明

```yaml
- insert:
    - id: codegraph-mcp
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: codegraph
        transport: stdio
        command: 'C:/Users/<you>/AppData/Roaming/npm/node_modules/@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-win32-x64/node.exe'
        args:
          - --liftoff-only
          - --disable-warning=ExperimentalWarning
          - 'C:/Users/<you>/AppData/Roaming/npm/node_modules/@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-win32-x64/lib/dist/bin/codegraph.js'
          - serve
          - --mcp
        env:
          CODEGRAPH_MCP_TOOLS: 'explore,node,search,callers,callees,impact,files,status'
        failOnStartupError: true
        toolCallTimeoutMs: 120000
```

| 字段 | 说明 |
| --- | --- |
| `serverName` | 工具名里的命名空间，即 `mcp__<serverName>__<tool>`。取值限 `[A-Za-z0-9_-]{1,32}`，同一作用域内唯一；重名会报 `serverName "x" is already in use` |
| `transport` | `stdio`（本仓库）或 `streamable-http` |
| `command` / `args` | 直接 spawn 的命令，**不走 shell**。所以必须写可执行文件路径，不能指望 `.cmd` shim |
| `env` | 追加给子进程的环境变量。DSH 会剔除名字匹配 `/KEY\|PASSWORD\|SECRET\|TOKEN/i` 的变量以及全部 `DSH_*`，敏感信息不要指望从这里传 |
| `cwd` | 可选。本配置不设，子进程继承宿主工作目录（profile 目录，里面没有 `.codegraph/`，因此进入「无默认项目」模式） |
| `failOnStartupError` | 默认 `false`；这里设 `true`，让命令路径写错、握手失败之类的问题在启动时直接暴露，而不是静默变成「没有工具」 |
| `toolCallTimeoutMs` | 默认 `60000`；首次索引较大项目的查询会慢，这里放宽到 120 秒 |
| `maxInstructionBytes` | 服务端 instructions 的截断上限，默认 32768，未改动 |

---

## 找不到 codegraph 时

插件加载时会自己探测，正常情况下不用配任何路径。探测顺序：

1. 环境变量 `CODEGRAPH_NODE`（`command`）与 `CODEGRAPH_JS`（args 里的 CLI 入口）；
2. 常见全局 npm 根里的 `@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-<平台>-<架构>`：
   `%APPDATA%\npm\node_modules`、`<node 安装目录>\..\..\lib\node_modules`、
   `/usr/local/lib/node_modules`、`/opt/homebrew/lib/node_modules`、`/usr/lib/node_modules`、
   `~/.npm-global/lib/node_modules`；
3. 都没有就返回空串 —— MCP 启动失败，DSH 的插件/日志里能看到 `spawn` 相关报错。

这时按情况处理：

- **没装 codegraph**：`npm install -g @colbymchenry/codegraph`，重启 DSH。
- **装在非常规位置**（pnpm、nvm、自定义 prefix）：在 patch 里给 `env` 补两条显式路径，重启 DSH：
  ```yaml
  env:
    CODEGRAPH_MCP_TOOLS: 'explore,node,search,callers,callees,impact,files,status'
    CODEGRAPH_NODE: 'C:/path/to/node.exe'
    CODEGRAPH_JS: 'C:/path/to/codegraph.js'
  ```
  （`npm root -g` 可查全局根；Windows 上是 `<全局根>/@colbymchenry/codegraph/node_modules/@colbymchenry/codegraph-win32-x64/`。）

> 为什么要用内置 `node.exe` 而不是 `codegraph.cmd`：Node 24 在 Windows 上 spawn `.cmd`
> 会抛 `EINVAL`（`spawn EPERM`/`EINVAL`），而 MCP 子进程是直接 spawn、不经 shell。
> codegraph 自己的 npm shim 也是靠内置 node 绕开这个问题的。

---

## 注意事项

- **探测靠「常见全局 npm 根」的启发式。** 用 pnpm 全局装、或用自定义 prefix 时可能探不到，
  用上面的 `CODEGRAPH_NODE` / `CODEGRAPH_JS` 显式指定即可（也是后续版本要做成界面配置的原因）。
- **不要移动或删除本目录。** profile 里存的是 `link:`/junction 指向它；搬走之后插件会加载失败，
  需要重新安装（或改 junction）。
- **删除或重建索引前先退出 DSH。** 运行中的 server 会占用 `.codegraph/codegraph.db`，
  目录删不掉、也重建不了，`codegraph init` 会卡在锁上。
- **改完 `cordis.patch.yml` 必须重启 DSH**；索引变化（改代码、重新 init）不需要重启，watcher 会
  实时拾取。
- 一个 `codegraph serve` 进程服务整台 DSH 的所有会话、所有工作区，是**单例**，不要为了「多开」
  加第二个同 `serverName` 的条目。
- `.codegraph/` 里有 `codegraph.db`、`daemon.pid`、`writer.pid` 等运行时文件，**不要提交到 git**；
  init 时自带的 `.gitignore` 已经处理了这一点。
- 索引对内存和磁盘有成本：一个中等仓库大约十几 MB 级（实测 `dsh-fish`：190 文件、2870 节点、
  12287 条边、13.1 MB）。
- 首次索引或超大项目查询可能超时；必要时调大 `toolCallTimeoutMs`。
- MCP 的 `projectPath` 接受绝对路径，也接受相对「该目录」的路径；建议始终传绝对路径。
- 工具名带命名空间，改名 `serverName` 会同时改变所有工具名，注意别和已有的 MCP 服务器冲突。

---

## 发布前检查清单（给作者）

- [x] `cordis.patch.yml` 已改成运行时探测，仓库里不含任何本机路径。
- [x] `package.json`：name `dsh-codegraph-mcp`、version `0.1.0`、无 `private`、补 repository/homepage/bugs/author。
- [x] `LICENSE`（MIT）。
- [ ] 确认 `icon.svg` 与两份 `locale/*.json` 是最终文案。
- [ ] `npm publish` 前 `npm pack --dry-run` 核对文件清单。
- [ ] 在干净机器上验证一遍：安装 → 重启 → 对一个已 `codegraph init` 的项目发
      `codegraph_explore`（带 `projectPath`）→ 确认返回源码而非报错。

---

## License

MIT，见 [LICENSE](./LICENSE)。
