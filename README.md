# 实践课 2 · 小组组织建仓与 Pi Agent 源码阅读

| 项目 | 记录 |
| --- | --- |
| GitHub 账号 | `LSX-item` |
| 小组组织 | `MangDui` |
| 分析版本 | `c3cc91632d0a1950fb122605b190233f2aa0006e` |

## 一、小组组织建仓

小组将课程仓库 fork 到 `MangDui` 组织，保留了 `pi-hunnu-2026` 这一仓库名。仓库页标题下方显示了 fork 的来源。组织成员页显示 `LSX-item` 已加入小组，可以在统一的组织仓库中参与后续开发。

- [小组代码仓库](https://github.com/MangDui/pi-hunnu-2026)
- [课程源仓库](https://github.com/hunnuOpenSourceSoftware2026/pi-hunnu-2026)

![小组仓库及其 fork 来源](pi-agent-practice-02/s0-org-repo.png)

![MangDui 组织成员页，红框标记 LSX-item](pi-agent-practice-02/s0-org-members.png)

## 二、下载源码并核对版本

使用 PowerShell 将小组仓库的 `main` 分支下载到 `D:\pi-hunnu-2026`。第一次连接中断，改用 HTTP/1.1 后克隆成功。

```powershell
git -c http.version=HTTP/1.1 clone --branch main https://github.com/MangDui/pi-hunnu-2026.git D:/pi-hunnu-2026
Set-Location 'D:\pi-hunnu-2026'
```

实际 clone 地址：`https://github.com/MangDui/pi-hunnu-2026.git`

| 核验命令 | 本地结果 |
| --- | --- |
| `Get-Location` | `D:\pi-hunnu-2026` |
| `git remote -v` | `origin` 的 fetch 和 push 均指向 `https://github.com/MangDui/pi-hunnu-2026.git` |
| `git branch --show-current` | `main` |
| `git rev-parse HEAD` | `c3cc91632d0a1950fb122605b190233f2aa0006e` |
| `git rev-parse origin/main` | 与本地 `HEAD` 相同 |
| `git status --short` | 无输出，工作区干净 |

S1 截图底部同时记录了 `packages` 下的 12 个源码目录。

![S1 本地仓库核验及源码目录](pi-agent-practice-02/s1-clone-info.png)

## 三、构建顺序与模块分析

根 `package.json` 用 npm workspaces 管理 `packages/*`、`apps/*` 和部分示例。根 `build` 脚本通过 `cd` 与 `&&` 串联 11 次子包构建，前一步失败会终止后续步骤。以下结果来自固定提交中的脚本和源码，属于静态分析。

```text
telemetry → schema → chord → tui → codemode → mcp → ai → durable → env → agent → coding-agent
```

`packages` 下还包含 `evals`。该包有自己的评估源码，根 `build` 没有调用它，其对 `ai` 和 `coding-agent` 的引用声明在 `devDependencies`。

下表中的“TS 编译”指 `tsc -p tsconfig.build.json`，将 `src/` 下的 TypeScript 编译到 `dist/`，生成 JavaScript、类型声明及映射文件。直接内部依赖以各包 `package.json` 的 `dependencies` 为准，按目录名列出。表中的相对路径均以源码仓库根目录为起点。

| 模块 | 主要职责 | build 与产物 | 直接内部依赖 | 源码依据 |
| --- | --- | --- | --- | --- |
| `telemetry` | 定义可跨后端使用的遥测接口，提供内存采集与空实现 | TS 编译 | 无 | `packages/telemetry/src/metrics.ts`、`packages/telemetry/src/memory.ts` |
| `schema` | 将 JSON Schema 校验规则编译成可复用的校验函数，也提供输入转换 | TS 编译 | 无 | `packages/schema/src/compiler.ts`、`packages/schema/src/coerce.ts` |
| `chord` | 提供跨进程对象运行时，组织文档、契约、存储及连接 | TS 编译 | `schema`、`telemetry` | `packages/chord/src/documents/`、`packages/chord/src/store/`、`packages/chord/src/link/` |
| `tui` | 处理终端输入、组件布局和差分绘制 | TS 编译。原生组件由单独的 `build:native:*` 脚本构建 | 无 | `packages/tui/src/tui.ts`、`packages/tui/package.json` |
| `codemode` | 在受限 JavaScript 运行环境中暴露宿主提供的工具 | TS 编译 | 无 | `packages/codemode/src/runtime/`、`packages/codemode/src/wasm.ts` |
| `mcp` | 建立 MCP 客户端连接，封装协议请求、传输和授权 | TS 编译 | 无 | `packages/mcp/src/client.ts`、`packages/mcp/src/transports/`、`packages/mcp/src/oauth/` |
| `ai` | 统一模型供应商的接口、流式响应和模型目录 | 先生成模型目录，再检查数据、执行 TS 编译，将供应商数据复制到 `dist/providers/data` | `schema`、`telemetry` | `packages/ai/catalog/generate.ts`、`packages/ai/scripts/check-model-data.ts`、`packages/ai/package.json` |
| `durable` | 管理需持久化的会话、任务与文档，封装其存储和恢复 | TS 编译 | `chord`、`ai`、`codemode`、`schema`、`telemetry` | `packages/durable/src/session/`、`packages/durable/src/storage/`、`packages/durable/src/tasks.ts` |
| `env` | 提供远程执行环境客户端，通过 SSH 连接远端守护进程并访问进程、终端等能力 | TS 编译。Rust 守护进程由独立的 `build:daemon` 命令构建 | `chord`、`durable` | `packages/env/src/remote-env.ts`、`packages/env/src/ssh.ts`、`packages/env/package.json` |
| `agent` | 管理代理状态和事件，协调模型响应、工具调用及后续轮次 | TS 编译 | `ai`、`schema` | `packages/agent/src/agent.ts`、`packages/agent/src/agent-loop.ts` |
| `coding-agent` | 把代理核心接入 CLI、交互界面、会话管理和内置编码工具 | `build:unbundled` 编译、设置 CLI/RPC 入口权限并复制主题等资源。随后用 esbuild 在 `dist/bundle/` 生成多个入口及共享分块 | `agent`、`ai`、`codemode`、`mcp`、`tui` | `packages/coding-agent/package.json`、`packages/coding-agent/src/cli.ts`、`scripts/build-coding-agent-bundle.mjs` |

## 四、模块层级与直接依赖

箭头由使用方指向被依赖方。模块框内列出它在 `packages/` 下直接依赖的子目录。第 1 层没有内部运行时依赖，其余模块的层级等于其直接内部依赖的最高层级加一。

| 层级 | 名称 | 模块 |
| --- | --- | --- |
| 第 4 层 | 应用与远程执行层 | `coding-agent`、`env` |
| 第 3 层 | 代理与持久化运行层 | `agent`、`durable` |
| 第 2 层 | 模型接口与对象运行层 | `ai`、`chord` |
| 第 1 层 | 基础与独立组件层 | `mcp`、`codemode`、`schema`、`telemetry`、`tui` |

![Pi Agent 模块依赖图](pi-agent-practice-02/module-dependencies.png)

[可编辑的 draw.io 图源](pi-agent-practice-02/module-dependencies.drawio)

## 五、三个问题

### 1. 根目录与子模块的 build 各读取什么配置？

在根目录运行 `npm run build` 时，npm 读取根 `package.json` 的 `scripts.build`，按固定顺序调度子包。进入子包后，npm 读取该包自己的 `package.json`。其中 `tsc -p tsconfig.build.json` 读取子包的编译配置，并通过 `extends` 继承根 `tsconfig.base.json` 的通用选项。例如 `coding-agent/tsconfig.build.json` 还将部分内部导入映射到上游包在 `dist` 中的声明文件。`ai` 在编译前生成并检查模型数据，`coding-agent` 在编译后还复制资源并打包。

### 2. 构建顺序和依赖有什么关系？

这版根脚本的顺序满足图中全部直接依赖的先后约束。例如 `coding-agent` 编译时会用到 `agent`、`ai` 和 `tui` 的声明，因此这些包先产出 `dist`。相邻模块可以没有依赖关系。例如 `chord` 后面紧跟 `tui`，后者的依赖声明没有 `chord`。把脚本中两个模块的位置连起来，无法据此得到一条依赖边。

### 3. 一条依赖路径中的模块如何分工？

`coding-agent → agent → ai → schema`

以模型发起一次工具调用为例，`coding-agent` 的 `src/core/agent-session.ts` 把代理核心接入编码会话。`agent` 的 `src/agent-loop.ts` 调用流式模型接口，组织模型回复和工具执行轮次。`ai` 统一供应商响应，并在 `src/utils/validation.ts` 中调用 `schema` 的 `compile` 生成工具参数校验器。`schema` 把声明的输入约束转换为可复用的校验逻辑。

## 六、重点分析模块

这次报告重点展开 `agent`。`src/agent.ts` 中的 `Agent` 对象维护运行状态并接收可替换的流式函数。`src/agent-loop.ts` 将这一状态和模型输入输出串到循环中，因此编码会话可以向它提供自己的工具，而代理循环仍负责推进下一轮。分析时把构建脚本、依赖声明和这一条具体调用路径放在一起，可以分别验证构建先后、包之间的引用和实际运行分工。

## 资料

- [课程任务说明](https://github.com/hunnuOpenSourceSoftware2026/assignments/blob/main/practice/02-Pi-Agent-fork.md)
- [本报告所用源码版本](https://github.com/MangDui/pi-hunnu-2026/tree/c3cc91632d0a1950fb122605b190233f2aa0006e)
- [报告结构参考](https://github.com/sakgratama-collab/pi-agent-practice/blob/main/practice2/README.md)
