**English** · [中文](./README.zh.md)

---

# dsh-remote

[![npm version](https://img.shields.io/npm/v/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![downloads](https://img.shields.io/npm/dw/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![license](https://img.shields.io/github/license/zhz1667/dsh-remote)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-7a3ef3)](https://github.com/topics/dsh-plugin)

Original project by [@flymysql](https://github.com/flymysql) · [Blog](https://gitpull.cn) ·
[Upstream Issues](https://github.com/flymysql/dsh-remote/issues) · [中文说明](./README.zh.md)

**This repository is a fork: `zhz1667/dsh-remote`, version `0.8.24`.** It exists to add one thing
the upstream release does not yet have — **compatibility with DSH `0.2.0-rc.2`** — because without it
the plugin is silently refused by the harness. Everything below the compatibility section is the
upstream plugin, unchanged.

![dsh-remote — make any SSH machine a real DSH workspace](docs/cover.png)

**Remote-work assistant for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (DSH).**

Manage several SSH machines, then pick a **remote workspace** (or a **local** one) and let the agent operate right there without leaving the harness — listing files, reading code, running builds & commands over the remote host, and keeping that remote directory mirrored into a real local workspace object.

The harness Web UI intentionally binds `127.0.0.1` (the CLI rejects `--host 0.0.0.0` for safety). This plugin goes the other way: **you connect out** to the machines you maintain, pick a workspace, and work in it through the normal DSH workspace + agent fs flows — no changes to `dsh-workspace` or the harness core.

---

## Why this version (0.8.24) — DSH 0.2.0-rc.2 compatibility

**Short version:** upstream `0.8.23` cannot run on DSH `0.2.0-rc.2`. The harness refuses to load it
and skips the whole bundle, so the plugin looks "installed but missing". `0.8.24` fixes the
declaration that caused that refusal. **No plugin logic changed.**

### The symptom

On DSH `0.2.0-rc.2` the plugin never appears, and boot prints:

```
dsh: skipping profile bundle "dsh-remote": Error: Plugin dsh-remote@0.8.21 is incompatible
with dsh 0.2.0-rc.2: peerDependencies {"@deepseek-ai/dsh-commands":"^0.1.0-rc.6",
"@deepseek-ai/dsh-host-webserver":"^0.1.0-rc.6","@deepseek-ai/dsh-tools":"^0.1.0-rc.6",
"@deepseek-ai/dsh-system-prompt":"^0.1.0-rc.6","@deepseek-ai/dsh-client-ui-renderer":"^0.1.2-rc.1",
"@deepseek-ai/dsh-client-locale":"^0.1.2-rc.1","@deepseek-ai/dsh-client-connection":"^0.1.2-rc.1"}.
Running it may cause crashes or data loss. …
```

### The cause

Before a profile imports a plugin, DSH compares **every `peerDependencies` entry named
`@deepseek-ai/dsh` or `@deepseek-ai/dsh-*`** against the single runtime version returned by
`getDshRuntimeVersion()`. The predicate is:

```js
semver.satisfies(runtimeVersion, requirement, { includePrerelease: true })
```

Two consequences that matter here:

- A caret range on a **0.x** version is locked to that minor line. `^0.1.0-rc.6` means
  `>=0.1.0-rc.6 <0.2.0` — so it can **never** match `0.2.0-rc.2`, no matter how many 0.1
  pre-releases ship.
- The check reads **peer declarations only**. `engines.dsh` is *not* consulted, so adding
  `dsh.engines.dsh` alone changes nothing.

(Implementation: `evaluatePluginCompatibility()` in `@deepseek-ai/dsh-app-boot`. Same rule is
documented in that package's README.)

### What 0.8.24 changes

| | 0.8.23 (upstream) | 0.8.24 (this fork) |
| --- | --- | --- |
| DSH peer ranges | `^0.1.0-rc.6` / `^0.1.2-rc.1` | **`^0.2.0-rc.1`** (matches `0.2.0-rc.2`) |
| `@deepseek-ai/cordis` | `^4.0.1` | `^4.0.4` (not part of the check; aligned with the ecosystem) |
| `@deepseek-ai/schemastery` | `^3.18.1` | `^3.18.4` |
| `dsh.engines.dsh` | absent | `>=0.2.0-rc.1` (documentation for humans and the plugin manager) |
| devDependencies | `0.1.0-rc.8` line | `0.2.0-rc.1` line, so tests run against the target runtime |
| Regression guard | — | **`test/compat.test.js`** — re-implements DSH's own predicate and fails if a
DSH peer range ever stops admitting the installed runtime version, plus a dedicated check that no
DSH peer range drifts back onto the 0.1 line |

The `optional` markers on `dsh-host-webserver` and `dsh-client-connection` are unchanged.

### What did **not** need changing (checked API by API)

| Plugin API usage | 0.1.0-rc.6 / 0.1.2-rc.1 | 0.2.0-rc.2 | Verdict |
| --- | --- | --- | --- |
| `defineTool` (`@deepseek-ai/dsh-tools`) | requires `output: { schema, render }` | identical (only optional `deferLoading` / `projectContent` added) | no change |
| `ctx.tools.register(t)` | same | same | no change |
| `commands.register({ name, description, handler })` | handler returns `{ kind: 'success' \| 'error', text }` | same | no change |
| `webServer.register(route)` | `WebRoute.kind` mandatory | `kind: 'exact' \| 'prefix'` still mandatory | already compliant — all 25 route registrations carry `kind: 'exact'` |
| `systemPrompt.section({ name, order, text })` | `text` receives `AssembleContext` | same, and `assembleContextFor(agent, signal)` still returns `{ agent, scope: agent, signal? }` | the `## Remote workspace` prompt section keeps working |

### Verification (reproducible)

```bash
npm ci
node --test        # 211 tests, 211 pass — run against real @deepseek-ai/* 0.2.0-rc.2 packages
node check.mjs     # static framework-constraint gate
```

- `node --test` → **211/211 pass**. `test/upload.test.js` and `test/http-transport.test.js` really
  `import lib/index.js`, so the plugin module is loaded against 0.2.0-rc.2 dependencies.
- Calling DSH's **own** `evaluatePluginCompatibility()` (`@deepseek-ai/dsh-app-boot@0.2.0-rc.2`,
  `getDshRuntimeVersion()` = `0.2.0-rc.2`): the `git HEAD` manifest → **INCOMPATIBLE** (byte-for-byte
  the error above); this working tree → **COMPATIBLE**.
- `node check.mjs` → `OK: no framework-constraint violations.`

### Version support

| DSH runtime | Use |
| --- | --- |
| `0.2.0-rc.1` / `0.2.0-rc.2` | **this fork (0.8.24)** |
| `0.1.x` | upstream [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote) (≤ 0.8.23) |

If you must keep the upstream package on `0.2.0-rc.2` unrebuilt, DSH offers an explicit per-version
exemption (`dsh plugin allow-version`, or the plugin manager) — the error message above says so. That
is an acceptance of risk, not a fix; the fix is the peer-range update in this fork.

---

## Screen previews

Settings → **远程工作区** — a multi-machine SSH registry (add / edit / delete / set-current, password stored locally):

<img src="docs/ui-settings-panel.png" alt="dsh-remote settings — multi-machine registry (light theme, host scrubbed)" width="720"/>

The native **"Add workspace" / "Select workspace"** flow — a centered modal, two tabs, opens on **本机 (local)**; switch to **远程 (remote)**:

- **远程** — a **machine `<select>`**, a path field that **auto-prefills `/` and live-completes** directories (picking one immediately reveals its next level, OS/VSCode-style), plus a **浏览…** floating browser that fills the field without committing — you review, edit, then **设为远程工作区**.

Real capture (host scrubbed to a placeholder):

<img src="docs/ui-picker-panel.png" alt="dsh-remote workspace picker — real dialog; 本机 (local) tab; 远程 machine select + prefilled root path + autocomplete" width="720"/>

---

## Features

### Machines & connections

- **Multi-machine SSH** — save any number of hosts (`host`/`port`/`user` + **private key** or **password**). Passwords are stored locally and never shown back in the UI. Switch with one click in Settings. Per-machine **passphrase / host-key mode / SSH agent / keyboard-interactive (OTP) / proxy jump (bastion)** and an **optional OS-keychain password** (`加密保存密码` — macOS Keychain / Windows DPAPI / Linux secret-tool).
- **`~/.ssh/config` aliases (resolved live, never copied)** — a machine can be saved as just a **Host alias** (`useSshConfig`): hostname/user/port/key/jump host are read from `~/.ssh/config` **at every connect**, so editing that file takes effect immediately and there is nothing to re-import; the registry stores **no copy** of those values (the key stays a path reference, its content is never read). Full OpenSSH semantics: multi-alias `Host a b`, `*`/`?` wildcards, `!` negation, `Include` (globbed, relative to `~/.ssh`), trailing-`\` continuations and ssh_config(5)'s *first-obtained-value-wins*. In Settings, **Import from ~/.ssh/config** saves an alias in one click (or **Copy fields** materialises a normal machine), the alias list and machine rows show **alias → what it actually resolves to**, and anything the plugin cannot honour (`ProxyJump` with several hops, `ProxyCommand`) is surfaced as a warning instead of silently degrading.
- **Host-key verification (TOFU)** — every SSH connect verifies the host key (`hostKeyMode: accept-new`): first connect records it, a later CHANGE is rejected as a possible man-in-the-middle. `verify` also refuses hosts never seen before; `off` disables it. Stored at `$DSH_HOME/remote-workspaces/known_hosts.json`; reset with `/remote forget-key`.
- **Connection health** — a **「测试连接」** button validates host/user/key/password (with per-category error hints: auth / network / host key / timeout) before you save a machine; latency is cached on the machine record.
- **Port forwarding panel** — create/start/stop/remove **local** (`127.0.0.1:port → remote`) and **reverse** (`remote → local`) tunnels in the Settings page or via `rw_forward`; definitions persist, auto-restart on reconnect when enabled, all tunnels stop on disconnect.

### Workspaces

- **Two-tab workspace picker** (fills the native "Add workspace" flow):
  - **本机 / Local** — opens the **native OS folder chooser** over the host (macOS `osascript` / Linux `zenity`→`kdialog` / **Windows `FolderBrowserDialog`**), or lets you type a local path → adopted directly as a normal DSH local workspace.
  - **远程 / Remote** — the picker is a **centered modal**. Pick a **machine** → on Windows hosts the root shows a **"This PC" drive view** (`C:\`, `D:\`, `E:\`… instead of the Git Bash MSYS root) and the path field live **autocompletes** directories (accepts `C:\Users\…` or `/c/Users/…` — Windows paths are rewritten to the Git Bash form underneath); selecting a directory immediately lists its next level. A **浏览…** floating browser (Windows-aware breadcrumb `此电脑 / C:\ / Users / dev`, drive rows, size + mtime, dirs first, follows symlinks) fills the field without committing; the **回上一级** button works at any depth (even when the browser was opened at the path bar's value). **最近 workspaces** quick-pick, **`~` 主目录** shortcut and **新建目录** are one click away. On confirm it creates a **real local mirror** under `$DSH_HOME/remote-workspaces/<host>-<user>-<port>/<base>` that passes `fs.realpath` → the harness adopts it as a real workspace while dsh-remote keeps it synced over SFTP.
- **Remote `@` completion (issue #39)** — in a remote session `@` lists the **remote** tree (read live over SFTP, not the local mirror): directories drill down, a slash-free query fuzzy-matches the whole tree, and candidates are **workspace-relative paths** (`@src/main.c`) exactly like a local session. The `rw_*` tools accept those relative paths and resolve them against the remote workspace root. The index is bounded (entries/directories/deadline + cache + failure breaker) and **falls back to the local mirror when the host is unreachable** — never a silent empty list. Local sessions are untouched.
- **Sidebar remote editing** — the remote file tab is **editable**: click **编辑** → edit → **保存到远程** with an mtime optimistic lock (409 + "重新读取" on concurrent change). File ops are **session-bound** (v0.8.19): the explorer sends `sessionId` so two conversations on different hosts do not share the active-machine pool. The explorer rows show file sizes and have a **right-click menu** (下载到本地镜像 / 重命名 / 删除 / 新建目录).
- **Data lives under the harness home** — machines + mirrors follow `$DSH_HOME`; pre-0.6 data under `~/.dsh/remote-workspaces` is migrated automatically on first run.

### Cross-platform remotes

- **Git Bash default terminal (Windows remotes)** — the remote platform is auto-detected (`cmd /c ver`, plus an `uname -s` MINGW/MSYS probe as fallback); on Windows the plugin locates Git Bash (`config.shell` can pin a path or `native` disables wrapping) and pipes every command to `bash -s` over the exec channel, so quoting/backslash escaping is never an issue regardless of the SSH default shell. `rw_exec` runs with a Git Bash cwd (`/c/Users/…` form). `/dsh-remote/status`, `rw_info` and the 测试连接 button report the detected platform + shell.
- **Windows path auto-conversion** — typing `C:\Users\dev\project` (or `C:/…`, `/c/…`, `/C:/…`) is normalized underneath to the Git Bash form `/c/Users/dev/project` for shell commands, while workspaces are stored and shown Windows-style (`C:\Users\dev\project`). All model tools accept and report both forms; SFTP access uses the Win32-OpenSSH `/D:/…` form (see `toSftpPath`).
- **Cross-platform remotes** — all file access is SFTP-protocol-level (no shell dependency), so Linux/macOS/Windows remotes all work for listing, reading, writing, searching and syncing.

### Agent tools & sync

- **Model tools** — 20 tools, all Windows/POSIX portable via SFTP: `rw_info`, `rw_connect` (with `save`), `rw_pick_workspace`, `rw_list_dir` (size+mtime), `rw_stat`, `rw_read_file` (encoding-aware: utf-8/gbk), `rw_write_file`, **`rw_edit`** (literal replace + mtime optimistic lock), `rw_append`, `rw_mkdir`, `rw_remove` (recursive, bounded), `rw_move`, `rw_exec` (pty/env), **`rw_search`** (SFTP tree walk — works on Windows too, honors ignore rules, context lines), `rw_download`/`rw_upload` (streaming fastGet/fastPut + size caps), **`rw_forward`** (SSH tunnels), `rw_sync`, `rw_push`, `rw_disconnect`.
- **Bidirectional SFTP sync, conflict-aware** — `rw_sync` (remote → mirror) and `rw_push` (mirror → remote) are **three-way** (remote vs local vs last-synced snapshot): files changed on both sides are **reported as conflicts and never silently overwritten** (`force=true` overrides). Defaults are **depth 8 / 2000 files**; hitting a cap is reported as **`TRUNCATED`**. Both support **dry-run**, **background tasks**, and honor **gitignore-style ignore rules**.
- **Async long tasks** — `rw_sync`/`rw_push` with `async: true` return a `taskId`; progress/result/cancel via `/dsh-remote/task` (single-flight queue).
- **Command audit log** — every `rw_exec`/write/remove/move/forward is appended to `$DSH_HOME/remote-workspaces/audit.log` (time · user@host · op · exit code · command); the Settings page shows the last 30.
- The active `user@host:/path` is injected into every system prompt (plus active forwards).

### Integration

- **No official `dsh-workspace` core is modified** — everything is delivered as a normal plugin (directory-flow holes filled by the client half at `priority -100`).
- **Official Desktop compatibility (experimental, unreleased)** — a compatibility path for the official DeepSeek Harness Desktop, exercised against the Host transport:
  - The SSH settings and directory picker use `/api/dsh-remote/*` over the Desktop's `dsh-app:` carrier. Exact Fetch routes are registered on `ctx.connection.fetch`; the carrier retains ownership of authentication.
  - A native **Remote Files** entry uses `sidebarRightTabs` and the keyed `sidebar.right.pane.tab` seat. It reuses the existing explorer/editor and gives remote files their own session-scoped resource addresses, rather than sending remote paths to the local Files viewer.
  - `dsh-better-sidebar` is not bundled. Web hosts may install it separately; official Desktop uses the native right-sidebar integration instead.
  - Since **v0.8.19**, sidebar `/ls` `/read` `/write` `/fs` resolve the session's mirror binding (same path as `rw_*`) when the client sends `sessionId`. Two sessions on different hosts no longer share the active-machine pool for file ops.
  - Official Desktop's native file-tab GUI, failed/cancelled dialogs, non-macOS hosts, and a full legacy Web UI pass are still experimental. Desktop's package installer may also require an explicit policy for the optional `ssh2` / `cpu-features` build scripts; this change does not loosen an application's build allowlist or automatically approve dependency scripts.

---

## Install

### From this fork (DSH 0.2.0-rc.2)

```bash
# straight from the fork's GitHub ref
dsh plugin --profile web add github:zhz1667/dsh-remote

# or from a local checkout of this repo (dev iteration; pack first to avoid a symlinked copy)
npm pack --pack-destination /tmp
dsh plugin --profile web add file:/tmp/dsh-remote-0.8.24.tgz

# sanity check — should print nothing about "skipping profile bundle"
dsh --profile web --dump-config | grep dsh-remote
```

> Prefer the tarball over `add /path/to/repo`: a symlinked install resolves
> `@deepseek-ai/dsh-tools` / `schemastery` from the plugin's own `node_modules` and can end up with
> two copies of a host package in one process.

### Published Web bundle (upstream npm, 0.1.x line)

```bash
dsh plugin add dsh-remote            # add the bundle
```

Since **v0.8.18**, `dsh-remote` installs and mounts only itself. The Web sidebar
([dsh-better-sidebar](https://www.npmjs.com/package/dsh-better-sidebar)) is optional and is no longer a dependency or an automatically mounted row. This keeps the SSH tools and settings UI independent from a particular sidebar implementation.

To add the optional Web remote-file explorer/editor, install both bundles:

```bash
dsh plugin add dsh-remote
dsh plugin add dsh-better-sidebar
```

When the standalone sidebar service is present, `dsh-remote` discovers it dynamically and registers its remote explorer/editor tabs. Without it, all `rw_*` tools, the settings UI, sync, audit log, and port forwarding continue to work. Official Desktop uses its native right-sidebar seats and does not need `dsh-better-sidebar`.

> **Upgrading from 0.7.2–0.8.17:** upgrading to 0.8.18 removes the embedded
> sidebar dependency and mount. Install `dsh-better-sidebar` separately only if
> you still want that Web UI. Any old profile override for
> `id: dsh-remote-sidebar` can be removed because that row no longer exists.

---

## Quick start

1. **Add a machine** — Settings → 远程工作区 → add host/port/user + key or password → (optional) set it current.
2. **Open a workspace** — click **Add workspace** in the sidebar / conversation:
   - **本机** → system folder chooser (or type a local path) → local workspace.
     On hosts without a usable OS dialog (DSH Desktop's browse backend, headless
     SSH hosts without zenity/kdialog) the in-app directory browser pops up
     instead — breadcrumbs, Windows drive switch, new-folder, pick-and-fill.
   - **远程** → choose the machine → browse to a remote directory (or type `/path`) → "设为远程工作区" ⇒ a local mirror workspace is created and adopted.
3. **Work with the agent** — treat it like any workspace:
   - `rw_list_dir(path?)`/`rw_read_file` — inspect remote files
   - `rw_write_file(path, content)` / `rw_edit(path, old, new)` — create / patch a remote file directly
   - `rw_stat(path)` / `rw_mkdir(path)` / `rw_remove(path, recursive?)` / `rw_move(path, dest)` — manage remote paths
   - `rw_search(pattern, path?)` — grep remote files (SFTP walk, Windows OK)
   - `rw_exec(command, cwd?, pty?)` — run remote shell commands (defaults to the workspace dir)
   - `rw_forward(listenPort, targetHost?, targetPort?)` — open an SSH tunnel
   - `rw_sync(dryRun?/force?/async?)` / `rw_push(dryRun?/force?/async?)` — conflict-aware mirror pull/push

## CLI defaults (optional)

Provide a default machine in `cordis.patch.yml`:

```yaml
# Example only — use values for your own machine.
- id: dsh-remote
  name: dsh-remote
  config:
    host: 203.0.113.10   # or your real host / hostname
    port: 22
    username: dev
    privateKeyPath: ~/.ssh/id_rsa
    # or password: '…'
    workspace: ~/project
```

If `host` is empty the plugin starts disconnected and you configure machines in the UI.

## CLI quick reference

Installing and driving DSH may live in different shells, so both the `dsh` binary and the `npx` form are shown. Always tell DSH **which profile** to use with `--profile <name>` (usually `web`).

```bash
# install the bundle into a profile (npm is pulled by pnpm; recommended)
dsh plugin --profile web add dsh-remote
# same but when `dsh` is not on PATH (e.g. Windows PowerShell inside a repo)
npx --yes @deepseek-ai/dsh plugin --profile web add dsh-remote

# confirm it is installed
dsh plugin --profile web list
npx --yes @deepseek-ai/dsh plugin --profile web list

# start the web surface (reload profile; the plugin activates on boot)
dsh --profile web
npx --yes @deepseek-ai/dsh --profile web   # http://127.0.0.1:3080

# use a local checkout instead of the npm version (dev iteration)
npx --yes @deepseek-ai/dsh plugin --profile web add /path/to/dsh-remote
npx --yes @deepseek-ai/dsh plugin --profile web remove dsh-remote   # back to release
```

After a successful start, `Settings → 远程工作区` appears and the "Add workspace" flow gains the 本机 / 远程 tabs (screenshots above).

## Development (sandbox, not product)

Iterate **in the sandbox**, never by hand-editing a product profile — the
product profile is re-managed by the plugin manager and reverts hand-deployed
files on reinstall. Use the helper script:

```bash
scripts/dev-run.sh --restart   # start / restart the isolated sandbox
scripts/dev-run.sh --stop      # stop it
scripts/dev-run.sh --status    # is it running?
```

- Runs its own DSH instance (`dev-harness/harness` inside this repo) with the
  plugin copied in from `lib/` — it boots through the same `bin.js web --patch`
  path as the desktop app, so the sandbox reproduces the product boot behavior.
- The sandbox web UI serves on `http://127.0.0.1:50599` and the plugin routes
  are live immediately (e.g. `GET /dsh-remote/machines`).
- **Host-half changes** (`lib/index.js`) need a sandbox restart (`--restart`);
  **client-half changes** (`lib/client.js`) need a page refresh.
- Node ESM resolves dependencies from the importing file's real path, so the
  script **copies** `lib/` (hardlink copy, `cp -al`) into the sandbox profile
  instead of symlinking — a symlink breaks `@deepseek-ai/*` resolution.
- Run `node check.mjs` (static framework-constraint gate: command-name regex, …)
  before every commit; `scripts/boot-smoke.sh` boots an isolated instance to prove
  the plugin still starts.
- **On Windows, keep LF in the working tree** (`git config core.autocrlf false`).
  `test/i18n.test.js` parses dictionary text out of `lib/client.js` with literal
  LF patterns, so a CRLF checkout fails that test for a reason unrelated to any
  code change.
- Full rules live in `scripts/dev-standards.md` (command names, cordis service
  access via `ctx.get()` only, optional framework services may never register,
  verify third-party callback contracts against the real runtime, …).

Deploying to a product profile is a separate, explicit action (`./sync.sh`)
and should be done only when you intend to release.

## Configuration

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `host` | string | `''` | default SSH host (else start disconnected) |
| `port` | int | `22` | default SSH port |
| `username` | string | `''` | default SSH user |
| `password` | string | `''` | default SSH password (non-empty overrides key) |
| `privateKeyPath` | string | `''` | private key path (used only when explicitly provided) |
| `passphrase` | string | `''` | passphrase for an encrypted private key |
| `workspace` | string | `''` | default remote workspace path |
| `shell` | string | `''` | remote command terminal strategy: `''`=auto-detect (Git Bash on Windows remotes), `'git-bash'`=prefer Git Bash, `'native'`=never wrap, anything else=explicit bash.exe path (e.g. `C:\Program Files\Git\bin\bash.exe`) |
| `commandTimeoutMs` | int | 20000 | per remote command timeout |
| `connectTimeoutMs` | int | 15000 | SSH connect timeout |
| `maxFileBytes` | int | 52428800 | skip mirroring/reading files larger than this (0 = no cap) |
| `hostKeyMode` | string | `accept-new` | host-key policy: `accept-new` (TOFU), `verify` (reject unknown hosts), `off` (skip) |
| `useAgent` | bool | `false` | authenticate via the OpenSSH agent (`SSH_AUTH_SOCK`) |
| `keyboardInteractive` | bool | `false` | allow keyboard-interactive auth (OTP/MFA) with the configured password |
| `proxy` | object | — | jump host: `{ host, port?, username?, password?, privateKeyPath? }` |
| `autoPush` | bool | `false` | auto-push edited mirror files back to the remote (watcher, debounced) |
| `auditLog` | bool | `true` | append executed commands to `$DSH_HOME/remote-workspaces/audit.log` |
| `encoding` | string | `utf-8` | text encoding for remote file reads/writes (e.g. `gbk`) |
| `fileReference` | bool | `true` | remote `@` completion: in a remote session `@` lists the **remote** tree over SFTP (issue #39); off → only the local mirror |
| `fileReferenceMaxResults` | int | `20` | max `@` candidates rendered for one query |
| `fileReferenceMaxEntries` | int | `3000` | max entries retained in one remote workspace's `@` index |
| `fileReferenceExcludedDirectories` | string[] | `[.git, node_modules, dist, build, out, coverage, target, .next, .nuxt, .turbo, .venv, __pycache__, .pytest_cache, .mypy_cache, .gradle]` | directory basenames the remote `@` traversal skips |
| `fileReferenceTimeoutMs` | int | `4000` | wall-clock budget for one remote `@` index pass (on expiry the partial index answers rather than making the caret wait) |

## FAQ / troubleshooting

**Plugin missing after upgrading DSH (0.2.x)** — see [Why this version](#why-this-version-0824--dsh-020-rc2-compatibility). A plugin whose DSH peer ranges do not admit the runtime is skipped at boot; install this fork (or rebuild with `^0.2.0-rc.1` peer ranges) and re-run `dsh --profile web --dump-config` to confirm no `skipping profile bundle` line remains.

**`@` lists remote files but the built-in read tool cannot open them** — the harness's own file tools see the session's **local mirror** (`$DSH_HOME/remote-workspaces/…`), which stays empty until `rw_sync` downloads it. Read remote files with `rw_read_file` / the sidebar remote tab: `@src/main.c` in a remote session means `<remote workspace>/src/main.c`, and every `rw_*` tool resolves such a relative path against the remote workspace root. Seeing nothing at all? The remote `@` index falls back to the mirror when the host is unreachable, and the settings page's 测试连接 shows why.

**Host key 变了 / 提示可能中间人** — 主机重装过或密钥更换过：`/remote-forget-key`（或设置页 → 机器 → 重新信任），下次连接重新记录。

**连接报"认证失败"** — 检查用户名/密码/私钥路径；私钥加密了要填 Passphrase；公司机器要求 OTP/动态码时勾选 keyboard-interactive。

**连不上内网机器** — 走跳板机：机器表单里填「跳板机」主机（也可以先把它本身配成一台机器）。主机不可达类错误会给出分类提示。

**rw_sync/rw_push 报冲突** — 远端和本地都改过同一个文件时会跳过并列出冲突（绝不静默覆盖）。处理：手动合并后重新同步，或用 `force=true` 以一边为准。

**Windows 远程** — 列表/读写/搜索/同步全部走 SFTP 协议，不依赖 POSIX shell；中文文件用 `encoding=gbk` 读。

**镜像里没有某个目录** — 默认 ignore 规则（`.git`、`node_modules`、`target` 等）会跳过；在 `$DSH_HOME/remote-workspaces/.dsh-remote-ignore` 加 `!` 之外的条目即可调整（gitignore 语法）。

**侧边栏远程文件保存失败（409）** — 远端文件在你打开后已被改动，重新读取后再编辑（mtime 乐观锁保护）。

**密码怎么加密保存** — 机器表单勾选「加密保存密码」：macOS 用系统钥匙串（security），Windows 用 DPAPI，Linux 需要 secret-tool（libsecret）；后端不可用时自动回退明文。

## Safety

Giving the plugin a machine's credentials lets the agent run **shell commands as your user** on that host. Only add machines you trust. Passwords are saved on the local machine file (or the OS keychain when enabled); treat it as sensitive (you may lock file ACLs). Every executed command is recorded in the audit log when `auditLog` is on — review it from the Settings page.

## License

MIT — same license as the upstream project. Original work © [@flymysql](https://github.com/flymysql);
this fork's changes © [@zhz1667](https://github.com/zhz1667).

## Contributing

Fixes for the DSH `0.2.0-rc.2` compatibility layer are welcome as issues/PRs on
[this fork](https://github.com/zhz1667/dsh-remote). Substantive plugin changes belong upstream: see
[CONTRIBUTING.md](./CONTRIBUTING.md) and the
[upstream Discussions](https://github.com/flymysql/dsh-remote/discussions) /
[upstream Issues](https://github.com/flymysql/dsh-remote/issues).

Thanks to everyone who has landed a change upstream (merged PRs in parentheses):

[@dahaipeng](https://github.com/dahaipeng) (#31) ·
[@YiHui-Liu](https://github.com/YiHui-Liu) (#28) ·
[@nekomona](https://github.com/nekomona) (#24) ·
[FoolishWiser](https://github.com/FoolishWiser) (#17) ·
[@jace1cch](https://github.com/jace1cch) (#16) ·
[@Minggle](https://github.com/Minggle) (#10) ·
[4FMTWRV](https://github.com/4FMTWRV) (#6) ·
[glzhangzhi](https://github.com/glzhangzhi) (per-session SSH pool fix)

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) — the fork's `0.8.24` entry documents the DSH 0.2.0-rc.2 change.
