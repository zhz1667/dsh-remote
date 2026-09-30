**中文** · [English](./README.md)

---

# dsh-remote

[![npm](https://img.shields.io/npm/v/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![license](https://img.shields.io/github/license/zhz1667/dsh-remote)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-7a3ef3)](https://github.com/topics/dsh-plugin)

在 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）**内部**就能操作远程 SSH
机器——连上服务器，把那边的某个目录选成工作区，然后让 agent 像对待本地目录一样读取、修改、搜索、
同步文件和执行命令。

> **本仓库（`zhz1667/dsh-remote`）是 [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote)
> 的 fork。** 它存在的唯一理由，就是紧随其后的那一节所讲的 DSH `0.2.0-rc.2` 兼容修复。
> 原作者：[@flymysql](https://github.com/flymysql) · MIT 许可。

---

## 为什么有这个版本（0.8.24 → DSH 0.2.0-rc.2）

**一句话：** 上游版本声明的 peer 范围不被 DSH `0.2.0-rc.2` 接受，导致插件在启动时被整个静默跳过；
本 fork 修正了这些范围，插件得以正常安装运行。**没有改动任何插件功能或逻辑**——只改了版本元数据。

### 你会看到什么

在 DSH `0.2.0-rc.2` 上装好插件，它却始终不出现。启动时 harness 会打印类似：

```
dsh: skipping profile bundle "dsh-remote": Error: Plugin dsh-remote@0.8.21 is incompatible
with dsh 0.2.0-rc.2: peerDependencies {"@deepseek-ai/dsh-commands":"^0.1.0-rc.6",
"@deepseek-ai/dsh-host-webserver":"^0.1.0-rc.6","@deepseek-ai/dsh-tools":"^0.1.0-rc.6",
"@deepseek-ai/dsh-system-prompt":"^0.1.0-rc.6","@deepseek-ai/dsh-client-ui-renderer":"^0.1.2-rc.1",
"@deepseek-ai/dsh-client-locale":"^0.1.2-rc.1","@deepseek-ai/dsh-client-connection":"^0.1.2-rc.1"}.
Running it may cause crashes or data loss. … 请卸载并重新安装与当前 DSH 兼容的版本。
```

设置面板没了，工作区选择器少了「远程」tab，`rw_*` 工具也不存在——因为整个 bundle 在加载前就被丢弃了。

### 为什么会这样

DSH 在导入插件**之前**就用它的 `package.json` 做一道闸门。它取出 `getDshRuntimeVersion()` 返回的
唯一运行时版本，然后对每一个名为 `@deepseek-ai/dsh` 或 `@deepseek-ai/dsh-*` 的
`peerDependencies` 条目问一句：

```js
semver.satisfies(runtimeVersion, range, { includePrerelease: true })
```

两个事实让 `0.1.x → 0.2.0` 变成一道硬墙：

- **0.x 版本上的 caret 范围被锁死在那条 minor 上。** `^0.1.0-rc.6` 展开为
  `>=0.1.0-rc.6 <0.2.0`，所以它**永远**匹配不上 `0.2.0-rc.2`——之后 0.1 线再发多少个预发布版都无济于事。
- **只检查 peer 范围。** `engines.dsh` 从不参与判断，所以单在那里写 `>=0.2.0-rc.1` 毫无作用。

这条规则在 `@deepseek-ai/dsh-app-boot`（`evaluatePluginCompatibility()`）里实现，该包 README 有记载。
一句话记住它：*0.x 上的 caret 范围 =「仅限这条 minor 线」。*

### 修复

把所有 `@deepseek-ai/dsh-*` 的 peer 范围从 `0.1` 线挪到 `^0.2.0-rc.1`——后者能接受 `0.2.0-rc.2`。同时：

- `@deepseek-ai/cordis` peer → `^4.0.4`、`@deepseek-ai/schemastery` 依赖 → `^3.18.4`（对齐当前生态；
  这两项不参与兼容校验）。
- 新增 `dsh.engines.dsh: ">=0.2.0-rc.1"`，用于人和插件管理器阅读（闸门不看它）。
- devDependencies 挪到 `0.2.0-rc` 线，让测试跑在目标运行时上。
- 新增 **`test/compat.test.js`** —— 一道回归防线：复刻 DSH 自己的判定谓词，一旦有 DSH peer 范围不再接受
  当前运行时版本、或又漂回 `0.1` 线就让测试失败。

### 我们刻意没动的部分

插件用到的每个框架 API 在 0.1 与 0.2 两条线之间都没变，所以源码原样保留：

| 插件的用法 | 在 0.2.0-rc.2 上仍有效？ |
| --- | --- |
| `defineTool()` 配 `output: { schema, render }` | 是——签名完全一致。 |
| `commands.register()` 返回 `{ kind, text }` | 是。 |
| `webServer.register()` 带 `kind: 'exact'` | 是——25 个路由本来就都写了。 |
| `systemPrompt.section()` 收到 `AssembleContext` | 是——`assembleContextFor()` 仍返回 `{ agent, scope, signal? }`。 |

### 是怎么验证的

```bash
npm ci
npm test          # 211 项测试，211 通过，跑在真实的 @deepseek-ai/* 0.2.0-rc.2 上
node check.mjs    # 静态框架约束闸门
```

- `npm test` → **211/211 通过**。其中两个测试套件会真实 `import lib/index.js`，即插件模块确实在
  0.2.0-rc.2 依赖上加载过。
- 直接用 DSH 自己的 `evaluatePluginCompatibility()`（`@deepseek-ai/dsh-app-boot@0.2.0-rc.2`，
  `getDshRuntimeVersion()` → `0.2.0-rc.2`）：旧清单报 **INCOMPATIBLE**（就是上面那段报错），
  新清单报 **COMPATIBLE**。
- `node check.mjs` → `OK: no framework-constraint violations.`

### 该用哪个版本

| DSH 运行时 | dsh-remote |
| --- | --- |
| `0.2.0-rc.1` / `0.2.0-rc.2` | **本 fork（`0.8.24`）** |
| `0.1.x` | 上游 [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote)，≤ `0.8.23` |

> 如果你被钉在上游那份包、跑在 `0.2.0-rc.2` 上又无法重建，DSH 提供了按版本的豁免
> （`dsh plugin allow-version`，报错里也提到了）。那是自认风险，不是修复；
> 真正的修复就是本 fork 里的 peer 范围更新。

---

## 功能

- **连上任意 SSH 主机** —— 密码、私钥（含 Passphrase）、OpenSSH agent、keyboard-interactive 的
  OTP/MFA、跳板机全都支持。可以保存任意多台机器，一键切换当前机器。
- **按引用使用 `~/.ssh/config`** —— 一台机器只需存一个 `Host` 别名。主机名、用户、端口、私钥、跳板机
  都在**每次连接时**从该文件读取，改完立刻生效，插件不保留任何副本。完整遵循 OpenSSH 匹配语义
  （多主机 `Host a b`、通配、`!` 取反、`Include`、行续行、first-value-wins）。设置页可一键导入别名。
- **主机密钥安全（TOFU）** —— 每次连接都校验主机密钥。首次见到即记录，之后一旦**变更**就按可能的
  拦截而拒绝。`verify` 模式连从未见过的机器也拒绝。
- **密码保管** —— 每台机器的密码可存入系统钥匙串（macOS Keychain / Windows DPAPI / Linux
  secret-tool），而不是注册表文件，并带回退到明文的透明提示。
- **远程工作区，也是真工作区** —— 选一个远程目录，会在 harness home 下物化出一个本地镜像并注册为
  真正的工作区，agent 对文件的操作照常可用。镜像路径由 host+user+port+目录名推出，且是幂等的。
- **Windows 远程是一等公民** —— 自动探测平台，命令统一走 Git Bash（`bash -s`）管道，于是 Windows
  主机表现得像 POSIX 主机（`/c/Users/...`）。任何地方都可以输入 `C:\Users\me\project`，底层自动转换。
  文件访问停在 SFTP 层，列目录/读写/搜索/同步从不依赖 POSIX shell。
- **20 个 agent 工具**（`rw_*`）—— 覆盖连接、浏览、读取、写入、搜索、执行命令、隧道和三方同步，
  跨平台且感知编码（`utf-8`、`gbk` 等）。
- **冲突可感知的镜像同步** —— `rw_sync`（拉取）与 `rw_push`（推送）是三方的。两边都改过的文件会被
  报为冲突，绝不静默覆盖。支持忽略规则、dry-run、规模上限与后台执行。
- **远程 `@` 补全** —— 远程会话里 `@` 列出*远程*目录树（走 SFTP 读取），以工作区相对路径呈现，
  支持模糊搜索与逐级下钻。有界、可缓存，主机不可达时回退到本地镜像。
- **端口转发** —— 在设置面板或通过 `rw_forward` 工具创建本地与反向 SSH 隧道。定义持久化，可选重连
  自动恢复。
- **命令审计日志** —— 每次远程命令、写入、删除、移动和隧道都会追加到 harness home 下的 `audit.log`，
  可在界面里复核。
- **不动核心** —— 插件完全通过正常的 bundle 机制交付；`dsh-workspace` 与 harness 核心未被触碰。
- **Desktop 支持（实验性）** —— 设置页与选择器也能走 Desktop 的 `dsh-app:` 载体；原生右侧栏的「远程文件」
  复用标准浏览器/编辑器，而不是劫持本地文件查看器。

### 界面预览

设置 → **远程工作区**（多机注册表，密码永不回显）：

<img src="docs/ui-settings-panel.png" alt="dsh-remote 设置页 —— 多机 SSH 注册表" width="720"/>

工作区选择器 —— 双 tab 弹窗，默认落在「本机」；切到「远程」选机器、实时补全路径或浏览：

<img src="docs/ui-picker-panel.png" alt="dsh-remote 工作区选择器 —— 本机与远程 tab，含路径自动补全" width="720"/>

---

## 安装

需要 DSH `0.2.0-rc.1` 或更新版本。

```bash
# 从本 fork 安装（DSH 0.2.x 推荐）
dsh plugin --profile web add github:zhz1667/dsh-remote

# 从本地检出安装 —— 先打包，让插件解析到宿主的 @deepseek-ai/*，
# 而不是仓库里夹带的第二份副本
npm pack --pack-destination /tmp
dsh plugin --profile web add file:/tmp/dsh-remote-0.8.24.tgz

# 自检：不应再有提到 dsh-remote 的 "skipping profile bundle"
dsh --profile web --dump-config | grep dsh-remote

# 启动 web 界面（插件在启动时激活）
dsh --profile web          # http://127.0.0.1:3080
```

> 建议用 `.tgz` 而不是 `add /path/to/repo`：符号链接式的检出可能在同一进程里带进第二份
> `@deepseek-ai/dsh-tools` / `schemastery`。

`dsh` 不在 `PATH` 时，用 `npx` 完全等价：`npx --yes @deepseek-ai/dsh plugin --profile web add …`。

> **可选的 Web 文件浏览器。** Web 版里的远程文件浏览/编辑界面是可选的，**不是**依赖。想要那套界面就另外
> 装 `dsh-better-sidebar`；不装的话 `rw_*` 工具、设置、同步、审计日志、端口转发照常工作，
> 官方 Desktop 则使用其原生右侧栏。

---

## 快速上手

1. **登记机器** —— 设置 → 远程工作区 → 填 host / port / user 和私钥或密码 →（可选）设为当前。
   （也可以直接给一个 `~/.ssh/config` 别名。）
2. **打开远程工作区** —— 用侧边栏的「添加工作区」流程 → **远程** tab → 选机器 → 选目录
   （输入、自动补全或浏览）→ 确认。本地镜像会被创建并收编为真正的工作区。
3. **开工** —— 当作普通工作区使用。agent 会获得 `rw_*` 工具，你也可以输入 `@` 唤出远程路径。

### 常用工具

| 目标 | 工具 |
| --- | --- |
| 查看主机 / 连接 / 选工作区 | `rw_info` · `rw_connect` · `rw_pick_workspace` |
| 浏览与读取 | `rw_list_dir` · `rw_stat` · `rw_read_file` |
| 新建与修改文件 | `rw_write_file` · `rw_edit`（乐观锁） · `rw_append` |
| 移动文件 | `rw_mkdir` · `rw_move` · `rw_remove` |
| 执行命令 | `rw_exec`（pty / env / cwd，带超时） |
| 搜索 | `rw_search`（SFTP 遍历，遵守 ignore 规则） |
| 传输 | `rw_download` · `rw_upload`（流式，有大小上限） |
| 网络 | `rw_forward`（本地与反向 SSH 隧道） |
| 同步 | `rw_sync`（拉取） · `rw_push`（推送）—— 三方、冲突可感知 |
| 收尾 | `rw_disconnect` |

### 斜杠命令

| 命令 | 作用 |
| --- | --- |
| `/remote` | 打开远程工作区控制（状态、选目录、机器、转发、审计）。 |
| `/remote-forget-key` | 忘掉已记录的主机密钥（例如主机重装之后）。 |
| `/remote-ignore` | 查看或编辑镜像的忽略规则。 |

### 可选的 CLI 默认机

也可以在 `cordis.patch.yml` 里预置一台默认机器：

```yaml
- id: dsh-remote
  name: dsh-remote
  config:
    host: 203.0.113.10   # 换成你真实的主机
    port: 22
    username: dev
    privateKeyPath: ~/.ssh/id_rsa
    # 或者：password: '…'
    workspace: ~/project
```

`host` 留空则以未连接状态启动，机器都在 UI 里配置。

---

## 配置

所有键都在 bundle 配置里，可通过 UI、`cordis.patch.yml` 或 CLI 覆盖设置。下表给出默认值。

### 连接

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `host` | string | `''` | 默认 SSH 主机（为空则未连接启动）。 |
| `port` | int | `22` | 默认 SSH 端口。 |
| `username` | string | `''` | 默认 SSH 用户。 |
| `password` | string | `''` | 密码登录（设置后优先于私钥）。 |
| `privateKeyPath` | string | `''` | 私钥路径；仅在显式提供时使用。 |
| `passphrase` | string | `''` | 加密私钥的 Passphrase。 |
| `proxy` | object | 全为空 | 跳板机：`{ host, port, username, password, privateKeyPath }`。`host` 为空即不跳转。 |
| `useAgent` | bool | `false` | 用 OpenSSH agent（`SSH_AUTH_SOCK`）认证。 |
| `keyboardInteractive` | bool | `false` | 允许用 `password` 做 keyboard-interactive（OTP/MFA）。 |
| `hostKeyMode` | string | `accept-new` | `accept-new`（先记录后校验） · `verify`（拒绝未见过的主机） · `off`。 |

### 传输与限制

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `commandTimeoutMs` | int | `20000` | 单条命令超时。 |
| `connectTimeoutMs` | int | `15000` | SSH 连接超时。 |
| `maxOutputChars` | int | `200000` | 单次调用收集输出的硬上限。 |
| `maxFileBytes` | int | `52428800` | 超过此大小的文件跳过镜像/读取（0 = 不限制）。 |
| `encoding` | string | `utf-8` | 远程读写的文本编码（如 `gbk`）。 |

### 工作区与 shell

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `workspace` | string | `''` | 默认远程工作区路径。 |
| `shell` | string | `''` | 终端策略：`''`=自动（Windows 远程用 Git Bash） · `git-bash`=优先它 · `native`=不包装 · 其它值=显式 `bash.exe` 路径。 |
| `autoPush` | bool | `false` | 镜像文件被编辑后自动推回远程（防抖监听）。 |
| `auditLog` | bool | `true` | 把执行的命令写入 `audit.log`。 |

### 远程 `@` 补全

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `fileReference` | bool | `true` | 远程会话里 `@` 列出远程目录树。 |
| `fileReferenceMaxResults` | int | `20` | 每次查询渲染的候选数。 |
| `fileReferenceMaxEntries` | int | `3000` | 单个工作区索引保留的最大条目数。 |
| `fileReferenceExcludedDirectories` | string[] | 见下 | 遍历时跳过的目录名。 |
| `fileReferenceTimeoutMs` | int | `4000` | 单次建索引的墙钟预算（超时用已有的部分索引作答）。 |

默认排除目录：`.git`、`node_modules`、`dist`、`build`、`out`、`coverage`、`target`、`.next`、
`.nuxt`、`.turbo`、`.venv`、`__pycache__`、`.pytest_cache`、`.mypy_cache`、`.gradle`。

### 更新

| 键 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `updateMode` | string | `manual` | `manual`（按需检查） · `auto`（加载时 + 定时检查） · `off`。 |
| `updateCheckIntervalMs` | int | `21600000`（6 小时） | auto 模式轮询 npm 新版本的间隔。 |

### 忽略规则

同步与远程搜索遵循 `$DSH_HOME/remote-workspaces/.dsh-remote-ignore` 里一个 gitignore **风格**的文件
（用 `/remote-ignore` 编辑）。它是一个刻意收窄的子集——不搞意外：

- 跳过空行与 `#` 注释。
- **不支持** `!` 取反。
- 行尾 `/` 只匹配目录。
- 含 `/` 的模式锚定在工作区根；否则匹配任意深度的同名目录。
- 支持 `*`（不含 `/`）、`**`（含 `/`）、`?`（一个字符，不含 `/`）；**不支持** `[abc]` 字符类。

---

## HTTP 接口

插件在宿主 web server 上注册了 25 个 `/dsh-remote/*` 路由（外加可选侧边栏复用的 4 个 fs 路由），
例如 `status`、`machines`、`connect`、`test-connect`、`forwards`、`audit`、`ssh-config`、
`task`/`tasks` 以及 `update-*` 三件套。它还通过 `ctx.connection.fetch` 在 `/api/dsh-remote/*` 下挂了一组
给 Desktop 载体用。本节仅供参考；agent 用的是上面的工具。

---

## 开发

```bash
npm ci
npm test
node check.mjs
```

- `node check.mjs` 是静态闸门（命令名正则、路由前缀等）——每次提交前都跑。
- `scripts/boot-smoke.sh` 会启动一个隔离的 DSH 实例，证明插件仍能启动。
- `scripts/dev-run.sh --restart` 起一个沙箱 harness（`http://127.0.0.1:50599`）并接上插件；
  宿主侧改动（`lib/index.js`）需要重启，客户端侧改动（`lib/client.js`）只需刷新页面。
- **在沙箱里迭代**——不要手工改产品 profile，它由插件管理器重新接管。`./sync.sh` 是面向发布的
  有意部署动作。
- 完整规则见 `scripts/dev-standards.md`。
- Windows 上请让检出保持 LF（`git config core.autocrlf false`）；`test/i18n.test.js` 从
  `lib/client.js` 里解析字典字面量，CRLF 检出会让它因与代码改动无关的原因失败。

---

## 安全提醒

登记一台机器，等于把那台主机的凭据交给 agent，让它能以你的身份在那里执行 shell 命令。只登记你信任的
主机。密码保存在你本机的机器注册表文件里（启用后存系统钥匙串）——请把它当敏感文件对待。`auditLog`
开启时每条远程命令都会记到 `audit.log`，可在设置面板里复核。

---

## License

MIT。原始作品 © [@flymysql](https://github.com/flymysql)；本 fork 的 DSH 0.2.0-rc.2 兼容改动
© [@zhz1667](https://github.com/zhz1667)。见 [LICENSE](./LICENSE)，历史见
[CHANGELOG.md](./CHANGELOG.md)。
