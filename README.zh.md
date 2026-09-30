[English](./README.md) · **中文**

---

# dsh-remote

[![npm version](https://img.shields.io/npm/v/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![downloads](https://img.shields.io/npm/dw/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![license](https://img.shields.io/github/license/zhz1667/dsh-remote)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-7a3ef3)](https://github.com/topics/dsh-plugin)

原作者 [@flymysql](https://github.com/flymysql) · [博客](https://gitpull.cn) ·
[上游 Issues](https://github.com/flymysql/dsh-remote/issues) · [English](./README.md)

**本仓库是 fork：`zhz1667/dsh-remote`，版本 `0.8.24`。** 它只为补上一件上游还没做的事——
**兼容 DSH `0.2.0-rc.2`**。没有这一项，插件会被 DSH 静默拒绝加载。兼容性一节之后的全部内容
都是上游插件本身，未作改动。

![dsh-remote —— 把任意 SSH 机器变成真正的 DSH 工作区](docs/cover.png)

**为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）打造的远程工作助手。**

维护多台 SSH 机器，然后在「选择工作区」时选一个**远程工作区**（或**本地工作区**），Agent 就能在不离开
harness 的情况下直接操作——列文件、读代码、在远程主机上跑构建/命令，并把远程目录镜像成一个真实的
本地工作区对象。

DSH 的 Web 界面刻意只监听 `127.0.0.1`（CLI 为安全拒绝 `--host 0.0.0.0`）。本插件反过来：**由你主动
连出**到你维护的机器，选一个工作区，然后通过 DSH 原生的工作区 + 文件流来工作——**不改动
`dsh-workspace` 核心**。

---

## 为什么有这个版本（0.8.24）—— 适配 DSH 0.2.0-rc.2

**一句话**：上游 `0.8.23` 在 DSH `0.2.0-rc.2` 上**跑不起来**——harness 会拒绝加载并跳过整个 bundle，
表现为"装了却找不到插件"。`0.8.24` 只修正了导致这次拒绝的那处声明，**插件逻辑一行未改**。

### 现象

在 DSH `0.2.0-rc.2` 上插件始终不出现，启动时打印：

```
dsh: skipping profile bundle "dsh-remote": Error: Plugin dsh-remote@0.8.21 is incompatible
with dsh 0.2.0-rc.2: peerDependencies {"@deepseek-ai/dsh-commands":"^0.1.0-rc.6",
"@deepseek-ai/dsh-host-webserver":"^0.1.0-rc.6","@deepseek-ai/dsh-tools":"^0.1.0-rc.6",
"@deepseek-ai/dsh-system-prompt":"^0.1.0-rc.6","@deepseek-ai/dsh-client-ui-renderer":"^0.1.2-rc.1",
"@deepseek-ai/dsh-client-locale":"^0.1.2-rc.1","@deepseek-ai/dsh-client-connection":"^0.1.2-rc.1"}.
Running it may cause crashes or data loss. …
```

### 根因

DSH 在 profile 导入插件前，会把插件 `package.json` 里**所有名为 `@deepseek-ai/dsh` 或
`@deepseek-ai/dsh-*` 的 `peerDependencies`** 与 `getDshRuntimeVersion()` 返回的唯一运行时版本逐一比对。
判定谓词是：

```js
semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })
```

由此有两条关键结论：

- **0.x 上的 caret 范围锁死 minor**。`^0.1.0-rc.6` 等价于 `>=0.1.0-rc.6 <0.2.0`，因此**永远**匹配不上
  `0.2.0-rc.2`——无论 0.1 线再发多少个预发布版。
- **该校验只看 peer 声明**，`engines.dsh` **不参与**。所以单加 `dsh.engines.dsh` 不会有任何作用。

（实现见 `@deepseek-ai/dsh-app-boot` 的 `evaluatePluginCompatibility()`；同一规则也写在该包 README 中。）

### 0.8.24 改了什么

| | 0.8.23（上游） | 0.8.24（本 fork） |
| --- | --- | --- |
| DSH peer 范围 | `^0.1.0-rc.6` / `^0.1.2-rc.1` | **`^0.2.0-rc.1`**（可匹配 `0.2.0-rc.2`） |
| `@deepseek-ai/cordis` | `^4.0.1` | `^4.0.4`（不参与该校验，仅对齐生态） |
| `@deepseek-ai/schemastery` | `^3.18.1` | `^3.18.4` |
| `dsh.engines.dsh` | 无 | `>=0.2.0-rc.1`（给人看、给插件管理器看，**不参与**校验） |
| devDependencies | `0.1.0-rc.8` 线 | `0.2.0-rc.1` 线，使测试跑在目标运行时上 |
| 回归保护 | — | **`test/compat.test.js`**：用与 DSH **相同**的谓词断言每个 DSH peer 范围都接受当前运行时版本，并单独拦截"范围又漂回 0.1 线"这种退化 |

`dsh-host-webserver` 与 `dsh-client-connection` 的 `optional` 标记保持不变。

### 代码层为什么不用改（逐项核对）

| 插件用到的 API | 0.1.0-rc.6 / 0.1.2-rc.1 | 0.2.0-rc.2 | 结论 |
| --- | --- | --- | --- |
| `defineTool`（`@deepseek-ai/dsh-tools`） | 强制 `output: { schema, render }` | 同构（仅新增可选 `deferLoading` / `projectContent`） | 无需改 |
| `ctx.tools.register(t)` | 同 | 同 | 无需改 |
| `commands.register({ name, description, handler })` | handler 返回 `{ kind: 'success' \| 'error', text }` | 同 | 无需改 |
| `webServer.register(route)` | `WebRoute.kind` 必填 | `kind: 'exact' \| 'prefix'` 仍必填 | 早已合规——本插件 25 处路由注册均已带 `kind: 'exact'` |
| `systemPrompt.section({ name, order, text })` | `text` 收 `AssembleContext` | 同；且 `assembleContextFor(agent, signal)` 仍返回 `{ agent, scope: agent, signal? }` | `## Remote workspace` 提示词段照旧生效 |

### 验证（可复现）

```bash
npm ci
node --test        # 211 项测试，211 通过 —— 跑在真实的 @deepseek-ai/* 0.2.0-rc.2 上
node check.mjs     # 静态框架约束闸门
```

- `node --test` → **211 / 211 通过**。其中 `test/upload.test.js`、`test/http-transport.test.js` 会真实
  `import lib/index.js`，即插件模块确实是在 0.2.0-rc.2 依赖上加载过的。
- 直接调用 **DSH 自己的** `evaluatePluginCompatibility()`（`@deepseek-ai/dsh-app-boot@0.2.0-rc.2`，
  `getDshRuntimeVersion()` = `0.2.0-rc.2`）：`git HEAD` 的清单 → **INCOMPATIBLE**（与上面的报错逐字一致）；
  本工作区 → **COMPATIBLE**。
- `node check.mjs` → `OK: no framework-constraint violations.`

### 版本支持

| DSH 运行时 | 用哪个 |
| --- | --- |
| `0.2.0-rc.1` / `0.2.0-rc.2` | **本 fork（0.8.24）** |
| `0.1.x` | 上游 [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote)（≤ 0.8.23） |

如果你一定要在 `0.2.0-rc.2` 上继续用上游那份未重建的包，DSH 提供了按精确版本的豁免
（`dsh plugin allow-version` 或插件管理器，上面的报错里也写了）。那是**自认风险**，不是修复；
真正的修复就是本 fork 里的 peer 范围更新。

---

## 界面预览

设置 → **远程工作区** —— 多机 SSH 列表（增/删/改/设为当前，密码本地保存、不回显）：

<img src="docs/ui-settings-panel.png" alt="dsh-remote 设置页 — 多机列表（浅色主题，主机已打码）" width="720"/>

原生 **「Add workspace / 选择工作区」** 流程 —— **居中弹窗**、两个 tab，**默认落在「本机」**；切到 **「远程」**：

- **远程** —— 一个**机器下拉**；路径输入框**自动预填 `/` 并实时补全目录**（点选一个目录后**立即列出它的下一级**，像系统/VSCode 逐级选目录）；另外有**「浏览…」浮窗**，选中仅回填到输入框（不直接提交），你复核 / 修改后点「设为远程工作区」。

真实截图（机器已打码为占位）：

<img src="docs/ui-picker-panel.png" alt="dsh-remote 工作区选择 — 真实弹窗；默认本机 tab；远程：机器下拉 + 预填根路径 + 自动补全" width="720"/>

---

## 功能

### 机器与连接

- **多机 SSH** —— 保存任意多台主机（`host`/`port`/`user` + **私钥**或**密码**）。密码本地保存、界面永不回显。设置页一键切换。每台机器可单独配置 **Passphrase / 主机密钥模式 / SSH agent / keyboard-interactive（OTP 动态码）/ 跳板机（ProxyJump）**，以及**可选的系统钥匙串密码**（`加密保存密码` —— macOS Keychain / Windows DPAPI / Linux secret-tool）。
- **`~/.ssh/config` 别名（实时解析，绝不复制）** —— 机器可以只存一个 **Host 别名**（`useSshConfig`）：主机名/用户/端口/私钥/跳板机都在**每次连接时**从 `~/.ssh/config` 读取，改配置文件立刻生效、无需重新导入；注册表里**不存这些值的副本**（私钥只留路径引用，从不读取其内容）。完整 OpenSSH 语义：多别名 `Host a b`、`*`/`?` 通配、`!` 取反、`Include`（支持 glob，相对 `~/.ssh`）、行尾 `\` 续行，以及 ssh_config(5) 的 *first-obtained-value-wins*。设置页里 **从 ~/.ssh/config 导入** 一键存为别名（或 **复制字段** 物化成普通机器），别名列表与机器行会显示 **别名 → 实际解析结果**；插件无法兑现的项（多跳 `ProxyJump`、`ProxyCommand`）会显式告警，而不是静默降级。
- **主机密钥校验（TOFU）** —— 每次 SSH 连接都校验主机密钥（`hostKeyMode: accept-new`）：首次记录，之后若**变更**则拒绝，防中间人。`verify` 连从未见过的机器也拒绝；`off` 关闭。存于 `$DSH_HOME/remote-workspaces/known_hosts.json`，用 `/remote forget-key` 重置。
- **连接健康检查** —— **「测试连接」** 按钮在保存前校验 host/user/私钥/密码，并按类别给出错误提示（认证 / 网络 / 主机密钥 / 超时）；测得延迟缓存在机器记录上。
- **端口转发面板** —— 在设置页或通过 `rw_forward` 创建/启动/停止/删除 **本地**（`127.0.0.1:port → 远程`）与 **反向**（`远程 → 本地`）隧道；定义持久化，启用时重连自动恢复，断开连接时全部停止。

### 工作区

- **双 tab 工作区选择器**（接管原生「添加工作区」流程）：
  - **本机 / Local** —— 打开**系统原生文件夹对话框**（macOS `osascript` / Linux `zenity`→`kdialog` / **Windows `FolderBrowserDialog`**），或直接输入本地路径 → 作为普通 DSH 本地工作区收编。
  - **远程 / Remote** —— 居中弹窗。选**机器**后，Windows 远程的根目录会显示 **"此电脑" 盘符视图**（`C:\`、`D:\`、`E:\`…，而不是 Git Bash 的 MSYS 根），路径框**实时补全**目录（`C:\Users\…` 或 `/c/Users/…` 都收，Windows 路径在底层重写为 Git Bash 形式）；选中目录立刻列出下一级。**浏览…** 浮层浏览器（Windows 感知面包屑 `此电脑 / C:\ / Users / dev`、盘符行、大小 + mtime、目录优先、跟随符号链接）只填字段不提交；**回上一级** 在任意深度可用（即使浏览器是在路径栏当前值上打开的）。**最近 workspaces** 快选、**`~` 主目录** 快捷方式、**新建目录** 都是一键。确认后在 `$DSH_HOME/remote-workspaces/<host>-<user>-<port>/<base>` 下创建**真实本地镜像**，能通过 `fs.realpath` → harness 把它当作真工作区收编，而 dsh-remote 负责用 SFTP 持续同步。
- **远程 `@` 补全（issue #39）** —— 远程会话里 `@` 列出**远程**目录树（走 SFTP 实时读取，不是本地镜像）：目录可逐级深入，不含斜杠的查询会在整棵树上模糊匹配，候选是**工作区相对路径**（`@src/main.c`），与本地会话完全一致。`rw_*` 工具接受这类相对路径并按远程工作区根解析。索引有界（条目数/目录数/截止时间 + 缓存 + 失败熔断），且**主机不可达时回退到本地镜像**——绝不会静默返回空列表。本地会话不受影响。
- **侧边栏远程编辑** —— 远程文件 tab **可编辑**：点 **编辑** → 改 → **保存到远程**，带 mtime 乐观锁（并发修改时 409 + "重新读取"）。文件操作**按会话绑定**（v0.8.19）：浏览器会带上 `sessionId`，因此不同主机上的两个会话不会共用当前机器连接池。浏览器行会显示文件大小，并有**右键菜单**（下载到本地镜像 / 重命名 / 删除 / 新建目录）。
- **数据都在 harness home 下** —— 机器与镜像跟随 `$DSH_HOME`；0.6 之前放在 `~/.dsh/remote-workspaces` 的数据会在首次运行时自动迁移。

### 跨平台远程

- **Windows 远程默认走 Git Bash** —— 自动探测远程平台（`cmd /c ver`，辅以 `uname -s` 的 MINGW/MSYS 探测）；Windows 上插件定位 Git Bash（`config.shell` 可指定路径，或 `native` 关闭包装），并把每条命令通过 exec 通道喂给 `bash -s`，因此无论 SSH 默认 shell 是什么，引号/反斜杠转义都不会出问题。`rw_exec` 以 Git Bash 风格的工作目录（`/c/Users/…`）运行。`/dsh-remote/status`、`rw_info` 与「测试连接」都会报告探测到的平台 + shell。
- **Windows 路径自动转换** —— 输入 `C:\Users\dev\project`（或 `C:/…`、`/c/…`、`/C:/…`）会在底层规范化为 Git Bash 形式 `/c/Users/dev/project` 供 shell 使用，而工作区仍按 Windows 风格存储与展示（`C:\Users\dev\project`）。所有模型工具都接受并回显两种形式；SFTP 访问使用 Win32-OpenSSH 的 `/D:/…` 形式（见 `toSftpPath`）。
- **跨平台远程** —— 所有文件访问都停在 SFTP 协议层（不依赖 shell），因此 Linux/macOS/Windows 远程的列目录、读写、搜索、同步都一样可用。

### Agent 工具与同步

- **模型工具** —— 20 个工具，全部通过 SFTP 实现 Windows/POSIX 双平台可移植：`rw_info`、`rw_connect`（带 `save`）、`rw_pick_workspace`、`rw_list_dir`（大小+mtime）、`rw_stat`、`rw_read_file`（感知编码：utf-8/gbk）、`rw_write_file`、**`rw_edit`**（字面替换 + mtime 乐观锁）、`rw_append`、`rw_mkdir`、`rw_remove`（递归、有界）、`rw_move`、`rw_exec`（pty/env）、**`rw_search`**（SFTP 遍历——Windows 也能用，遵守 ignore 规则，支持上下文行）、`rw_download`/`rw_upload`（流式 fastGet/fastPut + 大小上限）、**`rw_forward`**（SSH 隧道）、`rw_sync`、`rw_push`、`rw_disconnect`。
- **双向 SFTP 同步，冲突可感知** —— `rw_sync`（远程 → 镜像）与 `rw_push`（镜像 → 远程）都是**三方**（远程 vs 本地 vs 上次同步快照）：两边都改过的文件会被**列为冲突并绝不静默覆盖**（`force=true` 可强制）。默认 **深度 8 / 2000 文件**；触顶会明确报告 **`TRUNCATED`**。两者都支持 **dry-run**、**后台任务**，并遵守 **gitignore 风格 ignore 规则**。
- **长任务异步化** —— `rw_sync`/`rw_push` 传 `async: true` 会返回 `taskId`；进度/结果/取消通过 `/dsh-remote/task`（单飞队列）。
- **命令审计日志** —— 每次 `rw_exec`/写/删/移动/转发都会追加到 `$DSH_HOME/remote-workspaces/audit.log`（时间 · user@host · 操作 · 退出码 · 命令）；设置页展示最近 30 条。
- 当前生效的 `user@host:/path` 会被注入每一份系统提示词（连同活动中的端口转发）。

### 集成方式

- **不改官方 `dsh-workspace` 核心** —— 一切都以普通插件形式交付（目录流程的缺口由客户端半侧以 `priority -100` 补齐）。
- **官方 Desktop 兼容适配（实验性，尚未发布）** —— 以 `0.1.5-rc.2` Host 通信协议验证，不修改 Harness 核心：
  - 通过 `ctx.connection.fetch` 注册 `/api/dsh-remote/*`，由 Desktop 的 `dsh-app:` 通道承载请求，鉴权仍由宿主负责，**不启动 Web Server**。
  - 通过 `sidebarRightTabs` 和 `sidebar.right.pane.tab` 提供原生「远程文件」入口，复用原来的文件树与编辑器，不把远端路径传给本地文件预览器。
  - `dsh-better-sidebar` 不再内置；Web 版可以单独安装，官方 Desktop 则使用原生右侧栏集成。
  - **v0.8.19** 起侧栏 `/ls` `/read` `/write` `/fs` 在请求带 `sessionId` 时按该会话的镜像绑定选机（与 `rw_*` 同一套），不再落到「当前机器」连接池；宿主侧测试覆盖双机会话路由与编辑 409/重读/保存。
  - 已验证 Host 启动、IPC 请求、真实 SSH 的只读连接/目录列表/文本读取，以及设置页和测试 SSH 配置的导入。原生文件标签的完整 GUI、失败/取消交互、非 macOS 宿主以及旧 Web 版完整 UI 回归仍属实验性。
  - Desktop 安装器还可能要求明确配置 `ssh2` / `cpu-features` 可选构建脚本策略。隔离验证中禁用了这些可选脚本；本改动不放宽应用的构建白名单，也不自动批准脚本。

---

## 安装

### 从本 fork 安装（DSH 0.2.0-rc.2）

```bash
# 直接装 GitHub ref
dsh plugin --profile web add github:zhz1667/dsh-remote

# 或从本地检出安装（改代码即测；建议先打包，避免符号链接式安装）
npm pack --pack-destination /tmp
dsh plugin --profile web add file:/tmp/dsh-remote-0.8.24.tgz

# 自检 —— 不应再出现 "skipping profile bundle"
dsh --profile web --dump-config | grep dsh-remote
```

> 建议用 tarball，而不是 `add /path/to/repo`：符号链接式安装会让
> `@deepseek-ai/dsh-tools` / `schemastery` 从插件自己的 `node_modules` 解析，
> 同一进程里可能出现宿主包的两份副本。

### 上游 npm 发布版（0.1.x 线）

```bash
dsh plugin add dsh-remote            # 安装 bundle
```

自 **v0.8.18** 起，`dsh-remote` 只安装并挂载自己。Web 侧边栏
（[dsh-better-sidebar](https://www.npmjs.com/package/dsh-better-sidebar)）可选，既不再是依赖，也不再自动挂载。
这样 SSH 工具与设置界面就不依赖某一种侧边栏实现。

若要额外的 Web 远程文件浏览器/编辑器，两个 bundle 都装：

```bash
dsh plugin add dsh-remote
dsh plugin add dsh-better-sidebar
```

当独立的侧边栏服务存在时，`dsh-remote` 会动态发现它并注册远程浏览器/编辑器 tab。没有它，
全部 `rw_*` 工具、设置界面、同步、审计日志、端口转发照常工作。官方 Desktop 用原生右侧栏座位，
不需要 `dsh-better-sidebar`。

> **从 0.7.2–0.8.17 升级：** 升到 0.8.18 会移除内嵌的侧边栏依赖与挂载。只有仍想要那套 Web UI 时才
> 单独安装 `dsh-better-sidebar`。旧的 `id: dsh-remote-sidebar` profile 覆盖项可以删掉，因为那一行
> 已不存在。

---

## 快速上手

1. **添加机器** —— 设置 → 远程工作区 → 填 host/port/user + 私钥或密码 →（可选）设为当前。
2. **打开工作区** —— 在侧边栏/会话里点 **添加工作区**：
   - **本机** → 系统文件夹对话框（或直接输入本地路径）→ 本地工作区。
     在没有可用系统对话框的宿主上（DSH Desktop 的浏览后端、没装 zenity/kdialog 的无头 SSH 主机），
     会弹出应用内目录浏览器——面包屑、Windows 盘符切换、新建目录、选完回填。
   - **远程** → 选机器 → 浏览到远程目录（或输入 `/path`）→ 「设为远程工作区」⇒ 创建并收编本地镜像工作区。
3. **和 agent 一起干活** —— 当作普通工作区使用：
   - `rw_list_dir(path?)`/`rw_read_file` —— 查看远程文件
   - `rw_write_file(path, content)` / `rw_edit(path, old, new)` —— 直接创建 / 打补丁
   - `rw_stat(path)` / `rw_mkdir(path)` / `rw_remove(path, recursive?)` / `rw_move(path, dest)` —— 管理远程路径
   - `rw_search(pattern, path?)` —— 在远程文件里 grep（SFTP 遍历，Windows 也行）
   - `rw_exec(command, cwd?, pty?)` —— 跑远程 shell 命令（默认在工作区目录）
   - `rw_forward(listenPort, targetHost?, targetPort?)` —— 开 SSH 隧道
   - `rw_sync(dryRun?/force?/async?)` / `rw_push(dryRun?/force?/async?)` —— 冲突可感知的镜像拉取/推送

## 可选：CLI 默认机

在 `cordis.patch.yml` 里提供一台默认机器：

```yaml
# 示例：请换成你自己的机器
- id: dsh-remote
  name: dsh-remote
  config:
    host: 203.0.113.10   # 或你真实的主机 / 域名
    port: 22
    username: dev
    privateKeyPath: ~/.ssh/id_rsa
    # 或者 password: '…'
    workspace: ~/project
```

`host` 为空时插件以未连接状态启动，机器都在 UI 里配置。

## 常用命令（安装 / 查看 / 启动）

安装与启动 DSH 可能在不同 shell 里，所以 `dsh` 二进制与 `npx` 两种形式都给出。务必用
`--profile <name>` 指明**哪个 profile**（通常是 `web`）。

```bash
# 安装（从 npm 拉到 profile，pnpm 由 DSH 调用；推荐）
dsh plugin --profile web add dsh-remote
# 同一效果：当 `dsh` 不在 PATH 时（例如仓库里的 Windows PowerShell）
npx --yes @deepseek-ai/dsh plugin --profile web add dsh-remote

# 确认已装
dsh plugin --profile web list
npx --yes @deepseek-ai/dsh plugin --profile web list

# 启动 web 界面（重载 profile，新插件在启动时生效）
dsh --profile web
npx --yes @deepseek-ai/dsh --profile web   # http://127.0.0.1:3080

# 迭代用本地源码替换 npm 版（便于改 dsh 插件代码后即测）
npx --yes @deepseek-ai/dsh plugin --profile web add /path/to/dsh-remote
npx --yes @deepseek-ai/dsh plugin --profile web remove dsh-remote   # 回到发布版
```

启动成功后，`设置 → 远程工作区` 会出现，「添加工作区」流程也会多出 本机 / 远程 两个 tab（见上方截图）。

## 开发（沙箱优先，勿改产品）

**在沙箱里迭代**，不要手工改产品 profile —— 产品 profile 由插件管理器重新接管，
重新安装时会覆盖手工放进去的文件。用辅助脚本：

```bash
scripts/dev-run.sh --restart   # 启动 / 重启隔离沙箱
scripts/dev-run.sh --stop      # 停止
scripts/dev-run.sh --status    # 是否在运行
```

- 它在本仓库内跑自己的 DSH 实例（`dev-harness/harness`），插件从 `lib/` 复制进去；
  走的是与桌面端相同的 `bin.js web --patch` 路径，所以沙箱能复现产品启动行为。
- 沙箱 Web UI 在 `http://127.0.0.1:50599`，插件路由立即生效（例如 `GET /dsh-remote/machines`）。
- **宿主半侧改动**（`lib/index.js`）需要重启沙箱（`--restart`）；**客户端半侧改动**（`lib/client.js`）
  刷新页面即可。
- Node ESM 按导入文件的**真实路径**解析依赖，所以脚本是把 `lib/` **复制**（硬链接复制 `cp -al`）
  进沙箱 profile 而不是做符号链接——符号链接会破坏 `@deepseek-ai/*` 的解析。
- 每次提交前跑 `node check.mjs`（静态框架约束闸门：命令名正则等）；`scripts/boot-smoke.sh`
  会启动一个隔离实例，证明插件仍能启动。
- **Windows 上请让工作区保持 LF**（`git config core.autocrlf false`）。`test/i18n.test.js`
  用字面 LF 模式从 `lib/client.js` 里抽取字典文本，CRLF 检出会让该测试失败，且与代码改动无关。
- 完整规则见 `scripts/dev-standards.md`（命令命名、cordis 服务只能经 `ctx.get()` 访问、
  可选框架服务永不注册、第三方回调契约必须对照真实运行时验证……）。

部署到产品 profile 是另一个明确动作（`./sync.sh`），只在你准备发布时执行。

## 配置

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `host` | string | `''` | 默认 SSH 主机（为空则未连接启动） |
| `port` | int | `22` | 默认 SSH 端口 |
| `username` | string | `''` | 默认 SSH 用户 |
| `password` | string | `''` | 默认 SSH 密码（非空时覆盖私钥） |
| `privateKeyPath` | string | `''` | 私钥路径（仅在显式提供时使用） |
| `passphrase` | string | `''` | 加密私钥的 Passphrase |
| `workspace` | string | `''` | 默认远程工作区路径 |
| `shell` | string | `''` | 远程命令终端策略：`''`=自动探测（Windows 远程用 Git Bash）、`'git-bash'`=优先 Git Bash、`'native'`=不包装、其它值=显式 bash.exe 路径（如 `C:\Program Files\Git\bin\bash.exe`） |
| `commandTimeoutMs` | int | 20000 | 单条远程命令超时 |
| `connectTimeoutMs` | int | 15000 | SSH 连接超时 |
| `maxFileBytes` | int | 52428800 | 超过此大小的文件跳过镜像/读取（0 = 不限制） |
| `hostKeyMode` | string | `accept-new` | 主机密钥策略：`accept-new`（TOFU）、`verify`（拒绝未知主机）、`off`（跳过） |
| `useAgent` | bool | `false` | 用 OpenSSH agent（`SSH_AUTH_SOCK`）认证 |
| `keyboardInteractive` | bool | `false` | 允许用已配置密码做 keyboard-interactive（OTP/MFA） |
| `proxy` | object | — | 跳板机：`{ host, port?, username?, password?, privateKeyPath? }` |
| `autoPush` | bool | `false` | 镜像文件被编辑后自动推回远程（监听 + 防抖） |
| `auditLog` | bool | `true` | 执行的命令写入 `$DSH_HOME/remote-workspaces/audit.log` |
| `encoding` | string | `utf-8` | 远程文件读写的文本编码（如 `gbk`） |
| `fileReference` | bool | `true` | 远程 `@` 补全：远程会话里 `@` 走 SFTP 列**远程**树（issue #39）；关闭则只看本地镜像 |
| `fileReferenceMaxResults` | int | `20` | 单次查询最多渲染多少个 `@` 候选 |
| `fileReferenceMaxEntries` | int | `3000` | 单个远程工作区 `@` 索引最多保留多少条目 |
| `fileReferenceExcludedDirectories` | string[] | `[.git, node_modules, dist, build, out, coverage, target, .next, .nuxt, .turbo, .venv, __pycache__, .pytest_cache, .mypy_cache, .gradle]` | 远程 `@` 遍历跳过的目录名 |
| `fileReferenceTimeoutMs` | int | `4000` | 单次远程 `@` 建索引的墙钟预算（超时用已得的部分索引作答，不让光标干等） |

## 常见问题 / 排错

**升级 DSH 到 0.2.x 后插件不见了** —— 见上文「为什么有这个版本」。DSH peer 范围不接受当前运行时版本的插件会在启动时被跳过；装本 fork（或用 `^0.2.0-rc.1` 重新构建），再跑一次 `dsh --profile web --dump-config` 确认不再有 `skipping profile bundle`。

**`@` 能列出远程文件，但内置读取工具打不开** —— harness 自带的文件工具看到的是会话的**本地镜像**（`$DSH_HOME/remote-workspaces/…`），在 `rw_sync` 下载之前它是空的。读远程文件请用 `rw_read_file` / 侧边栏远程 tab：远程会话里的 `@src/main.c` 意思是 `<远程工作区>/src/main.c`，每个 `rw_*` 工具都会把这类相对路径按远程工作区根解析。完全看不到东西？主机不可达时远程 `@` 索引会回退到镜像，设置页的「测试连接」会告诉你原因。

**Host key 变了 / 提示可能中间人** —— 主机重装过或密钥更换过：`/remote-forget-key`（或设置页 → 机器 → 重新信任），下次连接重新记录。

**连接报"认证失败"** —— 检查用户名/密码/私钥路径；私钥加密了要填 Passphrase；公司机器要求 OTP/动态码时勾选 keyboard-interactive。

**连不上内网机器** —— 走跳板机：机器表单里填「跳板机」主机（也可以先把它本身配成一台机器）。主机不可达类错误会给出分类提示。

**rw_sync/rw_push 报冲突** —— 远端和本地都改过同一个文件时会跳过并列出冲突（绝不静默覆盖）。处理：手动合并后重新同步，或用 `force=true` 以一边为准。

**Windows 远程** —— 列表/读写/搜索/同步全部走 SFTP 协议，不依赖 POSIX shell；中文文件用 `encoding=gbk` 读。

**镜像里没有某个目录** —— 默认 ignore 规则（`.git`、`node_modules`、`target` 等）会跳过；在 `$DSH_HOME/remote-workspaces/.dsh-remote-ignore` 加 `!` 之外的条目即可调整（gitignore 语法）。

**侧边栏远程文件保存失败（409）** —— 远端文件在你打开后已被改动，重新读取后再编辑（mtime 乐观锁保护）。

**密码怎么加密保存** —— 机器表单勾选「加密保存密码」：macOS 用系统钥匙串（security），Windows 用 DPAPI，Linux 需要 secret-tool（libsecret）；后端不可用时自动回退明文。

## 安全提醒

把某台机器的凭据交给插件，就等于允许 agent 以**你的身份**在那台主机上执行 **shell 命令**。
只添加你信任的机器。密码保存在本机文件里（启用后存系统钥匙串）；请当作敏感数据对待
（可以额外收紧文件权限）。`auditLog` 开启时每条执行的命令都会记入审计日志——请在设置页定期查看。

## License

MIT —— 与上游同许可。原始作品 © [@flymysql](https://github.com/flymysql)；本 fork 的改动 © [@zhz1667](https://github.com/zhz1667)。

## 参与贡献

针对 DSH `0.2.0-rc.2` 兼容层的修复，欢迎提到[本 fork](https://github.com/zhz1667/dsh-remote)的 issue/PR。
插件本身的实质性改动请走上游：见 [CONTRIBUTING.md](./CONTRIBUTING.md)，以及
[上游 Discussions](https://github.com/flymysql/dsh-remote/discussions) /
[上游 Issues](https://github.com/flymysql/dsh-remote/issues)。

感谢所有向上游贡献过改动的人（括号内为已合并 PR）：

[@dahaipeng](https://github.com/dahaipeng) (#31) ·
[@YiHui-Liu](https://github.com/YiHui-Liu) (#28) ·
[@nekomona](https://github.com/nekomona) (#24) ·
[FoolishWiser](https://github.com/FoolishWiser) (#17) ·
[@jace1cch](https://github.com/jace1cch) (#16) ·
[@Minggle](https://github.com/Minggle) (#10) ·
[4FMTWRV](https://github.com/4FMTWRV) (#6) ·
[glzhangzhi](https://github.com/glzhangzhi)（按会话隔离 SSH 连接池修复）

## 变更记录

见 [CHANGELOG.md](./CHANGELOG.md) —— 本 fork 的 `0.8.24` 条目记录了这次 DSH 0.2.0-rc.2 适配。
