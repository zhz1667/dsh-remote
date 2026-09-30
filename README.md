[中文](./README.zh.md) · **English**

---

# dsh-remote

[![npm](https://img.shields.io/npm/v/dsh-remote)](https://www.npmjs.com/package/dsh-remote)
[![license](https://img.shields.io/github/license/zhz1667/dsh-remote)](LICENSE)
[![dsh-plugin](https://img.shields.io/badge/topic-dsh--plugin-7a3ef3)](https://github.com/topics/dsh-plugin)

Work on a remote SSH machine **from inside** [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)
(DSH) — connect to a server, pick a folder there as your workspace, and let the agent read, write,
search, sync, and run commands on it as if it were local.

> **This repository (`zhz1667/dsh-remote`) is a fork of
> [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote).** The only reason it exists is the
> DSH `0.2.0-rc.2` compatibility fix described in the very next section. Original author:
> [@flymysql](https://github.com/flymysql) · MIT licensed.

---

## Why this version exists (0.8.24 → DSH 0.2.0-rc.2)

**In one sentence:** the upstream release declared peer ranges that DSH `0.2.0-rc.2` refuses to
load, so the plugin was silently skipped at boot; this fork corrects those ranges so it installs and
runs. **No plugin feature or logic was changed** — only version metadata.

### What you see

Install the plugin on DSH `0.2.0-rc.2` and it simply never appears. On boot the harness logs
something like:

```
dsh: skipping profile bundle "dsh-remote": Error: Plugin dsh-remote@0.8.21 is incompatible
with dsh 0.2.0-rc.2: peerDependencies {"@deepseek-ai/dsh-commands":"^0.1.0-rc.6",
"@deepseek-ai/dsh-host-webserver":"^0.1.0-rc.6","@deepseek-ai/dsh-tools":"^0.1.0-rc.6",
"@deepseek-ai/dsh-system-prompt":"^0.1.0-rc.6","@deepseek-ai/dsh-client-ui-renderer":"^0.1.2-rc.1",
"@deepseek-ai/dsh-client-locale":"^0.1.2-rc.1","@deepseek-ai/dsh-client-connection":"^0.1.2-rc.1"}.
Running it may cause crashes or data loss. … uninstall and reinstall a compatible version.
```

The settings panel is gone, the workspace picker loses its "远程" tab, the `rw_*` tools don't exist —
because the whole bundle was dropped before it could load.

### Why it happens

DSH gates every plugin on its `package.json` **before** importing it. It takes the single runtime
version from `getDshRuntimeVersion()` and asks, for each `peerDependencies` entry named
`@deepseek-ai/dsh` or `@deepseek-ai/dsh-*`:

```js
semver.satisfies(runtimeVersion, range, { includePrerelease: true })
```

Two facts make `0.1.x → 0.2.0` a hard wall:

- **A caret range on a `0.x` version is frozen to that minor.** `^0.1.0-rc.6` expands to
  `>=0.1.0-rc.6 <0.2.0`, so it can *never* match `0.2.0-rc.2` — no matter how many 0.1 pre-releases
  ship afterward.
- **Only peer ranges are checked.** `engines.dsh` is never read, so declaring `>=0.2.0-rc.1` there
  changes nothing on its own.

The rule lives in `@deepseek-ai/dsh-app-boot` (`evaluatePluginCompatibility()`) and is documented in
that package's README. The one-line mental model: *a 0.x caret range means "this minor line only."*

### The fix

Move every `@deepseek-ai/dsh-*` peer range from the `0.1` line onto `^0.2.0-rc.1`, which does admit
`0.2.0-rc.2`. Alongside that:

- `@deepseek-ai/cordis` peer → `^4.0.4` and `@deepseek-ai/schemastery` dep → `^3.18.4` (aligned with
  the current ecosystem; these are not part of the compatibility check).
- `dsh.engines.dsh: ">=0.2.0-rc.1"` added for humans and the plugin manager (not consulted by the
  gate).
- devDependencies moved onto the `0.2.0-rc` line so the test suite runs against the target runtime.
- **`test/compat.test.js`** added — a regression guard that re-implements DSH's exact predicate and
  fails if any DSH peer range ever stops admitting the installed runtime version, or drifts back to
  the `0.1` line.

### What we deliberately did *not* touch

Every framework API the plugin uses is unchanged between the 0.1 and 0.2 lines, so the source stayed
put:

| Plugin usage | Still valid in 0.2.0-rc.2? |
| --- | --- |
| `defineTool()` with `output: { schema, render }` | Yes — signature identical. |
| `commands.register()` returning `{ kind, text }` | Yes. |
| `webServer.register()` with `kind: 'exact'` | Yes — all 25 routes already set it. |
| `systemPrompt.section()` receiving `AssembleContext` | Yes — `assembleContextFor()` still returns `{ agent, scope, signal? }`. |

### How it was verified

```bash
npm ci
npm test          # 211 tests, 211 pass, against the real @deepseek-ai/* 0.2.0-rc.2 packages
node check.mjs    # static framework-constraint gate
```

- `npm test` → **211/211 pass**. Two of the suites import `lib/index.js` for real, so the plugin
  module is genuinely loaded on 0.2.0-rc.2 dependencies.
- DSH's own `evaluatePluginCompatibility()` (`@deepseek-ai/dsh-app-boot@0.2.0-rc.2`,
  `getDshRuntimeVersion()` → `0.2.0-rc.2`): the old manifest reports **INCOMPATIBLE** (the exact
  error above); this one reports **COMPATIBLE**.
- `node check.mjs` → `OK: no framework-constraint violations.`

### Which version to use

| DSH runtime | dsh-remote |
| --- | --- |
| `0.2.0-rc.1` / `0.2.0-rc.2` | **this fork (`0.8.24`)** |
| `0.1.x` | upstream [flymysql/dsh-remote](https://github.com/flymysql/dsh-remote), ≤ `0.8.23` |

> If you are pinned to the upstream package on `0.2.0-rc.2` and cannot rebuild it, DSH offers a
> per-version exemption (`dsh plugin allow-version` — also mentioned in the error). That accepts the
> risk; it is not a fix. The fix is the peer-range update in this fork.

---

## Features

- **Reach any SSH host** — password, private key (with passphrase), OpenSSH agent, keyboard-interactive
  OTP/MFA, and jump hosts (bastions) are all supported. Save as many machines as you like and switch
  the active one with a click.
- **`~/.ssh/config` by reference** — a machine can be stored as just a `Host` alias. Hostname, user,
  port, key, and jump host are read from that file on **every** connect, so edits take effect instantly
  and the plugin never keeps a stale copy. Full OpenSSH matching is honored (multi-host `Host a b`,
  wildcards, `!` negation, `Include`, line continuations, first-value-wins). One click imports an
  alias from the settings panel.
- **Host-key safety (TOFU)** — every connect verifies the host key. First sight is recorded; a later
  *change* is rejected as a possible interception. `verify` mode also refuses unknown hosts.
- **Secret handling** — per-machine passwords can be stored in the OS keychain (macOS Keychain /
  Windows DPAPI / Linux secret-tool) instead of the registry file, with a transparent fallback.
- **Remote workspace, real workspace** — choosing a remote folder materializes a local mirror under
  the harness home and registers it as a genuine workspace, so everything the agent does on files
  keeps working. The mirror path is derived from host+user+port+basename, and it is idempotent.
- **Windows remotes are first-class** — the platform is auto-detected and commands are piped through
  Git Bash (`bash -s`), so a Windows host behaves like a POSIX one (`/c/Users/...`). You can type
  `C:\Users\me\project` anywhere and it is translated underneath. File access is SFTP-level, so
  listing/reading/writing/searching/syncing never depends on a POSIX shell.
- **20 agent tools** (`rw_*`) covering connect, browse, read, write, search, exec, tunnels, and
  three-way sync — cross-platform and encoding-aware (`utf-8`, `gbk`, …).
- **Conflict-aware mirror sync** — `rw_sync` (pull) and `rw_push` (push) are three-way. A file edited
  on both sides is reported as a conflict, never silently clobbered. Honors ignore rules, has
  dry-run, bounds, and background execution.
- **Remote `@` completion** — in a remote session, `@` lists the *remote* tree (read over SFTP) as
  workspace-relative paths, with fuzzy search and directory drill-down. Bounded and cached, and it
  falls back to the local mirror if the host is unreachable.
- **Port forwarding** — create local and reverse SSH tunnels from the settings panel or the
  `rw_forward` tool. Definitions persist and can auto-restore on reconnect.
- **Command audit log** — every remote command, write, delete, move, and tunnel is appended to
  `audit.log` under the harness home, reviewable in the UI.
- **No core changes** — the plugin ships entirely through the normal bundle mechanism; `dsh-workspace`
  and the harness core are untouched.
- **Desktop support (experimental)** — settings and the picker also work over the Desktop
  `dsh-app:` carrier, and a native right-sidebar "remote files" surface reuses the standard
  explorer/editor instead of hijacking the local file viewer.

### Screenshots

Settings → **远程工作区** (multi-machine registry, passwords never shown back):

<img src="docs/ui-settings-panel.png" alt="dsh-remote settings — multi-machine SSH registry" width="720"/>

The workspace picker — a two-tab modal that opens on 本机 (local); switch to 远程 (remote) to pick a
machine, live-autocomplete a path, or browse:

<img src="docs/ui-picker-panel.png" alt="dsh-remote workspace picker — local and remote tabs with path autocomplete" width="720"/>

---

## Install

Requires DSH `0.2.0-rc.1` or newer.

```bash
# from this fork (recommended for DSH 0.2.x)
dsh plugin --profile web add github:zhz1667/dsh-remote

# from a local checkout — pack first so the plugin resolves against the host's
# @deepseek-ai/* instead of a second copy inside the repo
npm pack --pack-destination /tmp
dsh plugin --profile web add file:/tmp/dsh-remote-0.8.24.tgz

# sanity check: no "skipping profile bundle" line should mention dsh-remote
dsh --profile web --dump-config | grep dsh-remote

# start the web surface (the plugin activates on boot)
dsh --profile web          # http://127.0.0.1:3080
```

> Prefer the `.tgz` over `add /path/to/repo`: a symlinked checkout can pull in a duplicate
> `@deepseek-ai/dsh-tools` / `schemastery` in the same process.

Running the DSH CLI through `npx` works identically when `dsh` is not on `PATH`:
`npx --yes @deepseek-ai/dsh plugin --profile web add …`.

> **Optional Web file explorer.** The remote file browser/editor UI in the web build is optional and
> is **not** a dependency. Install `dsh-better-sidebar` alongside if you want that surface; without
> it, the `rw_*` tools, settings, sync, audit log, and port forwarding all still work, and official
> Desktop uses its native right sidebar instead.

---

## Quick start

1. **Register a machine** — Settings → 远程工作区 → add host / port / user and a key or password →
   optionally set it active. (Or point the CLI at an `~/.ssh/config` alias.)
2. **Open a remote workspace** — use the sidebar's "add workspace" flow → **远程** tab → pick the
   machine → choose a directory (type, autocomplete, or browse) → confirm. A local mirror is created
   and adopted as a real workspace.
3. **Work** — treat it like any workspace. The agent gains the `rw_*` tools, and you can type `@` to
   pull in remote paths.

### Everyday tools

| Goal | Tool |
| --- | --- |
| Inspect the host / connect / choose a workspace | `rw_info` · `rw_connect` · `rw_pick_workspace` |
| Browse and read | `rw_list_dir` · `rw_stat` · `rw_read_file` |
| Create and change files | `rw_write_file` · `rw_edit` (optimistic lock) · `rw_append` |
| Move things around | `rw_mkdir` · `rw_move` · `rw_remove` |
| Run commands | `rw_exec` (pty / env / cwd, with timeouts) |
| Search | `rw_search` (SFTP walk, honors ignore rules) |
| Transfer | `rw_download` · `rw_upload` (streaming, size-capped) |
| Network | `rw_forward` (local & reverse SSH tunnels) |
| Sync | `rw_sync` (pull) · `rw_push` (push) — three-way, conflict-aware |
| Housekeeping | `rw_disconnect` |

### Slash commands

| Command | Does |
| --- | --- |
| `/remote` | Open the remote workspace controls (status, pick, machines, forwards, audit). |
| `/remote-forget-key` | Forget a stored host key (e.g. after a host rebuild). |
| `/remote-ignore` | Show or edit the mirror ignore rules. |

### Optional CLI default

You can pre-seed a default machine in `cordis.patch.yml`:

```yaml
- id: dsh-remote
  name: dsh-remote
  config:
    host: 203.0.113.10   # your real host
    port: 22
    username: dev
    privateKeyPath: ~/.ssh/id_rsa
    # or: password: '…'
    workspace: ~/project
```

Leave `host` empty to start disconnected and configure everything in the UI.

---

## Configuration

All keys live in the bundle config; they can be set in the UI, in `cordis.patch.yml`, or via CLI
overrides. Defaults shown.

### Connection

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `host` | string | `''` | Default SSH host (empty → starts disconnected). |
| `port` | int | `22` | Default SSH port. |
| `username` | string | `''` | Default SSH user. |
| `password` | string | `''` | Password login (takes precedence over the key when set). |
| `privateKeyPath` | string | `''` | Private-key path; used only when explicitly given. |
| `passphrase` | string | `''` | Passphrase for an encrypted key. |
| `proxy` | object | all empty | Jump host: `{ host, port, username, password, privateKeyPath }`. Empty `host` = no jump. |
| `useAgent` | bool | `false` | Authenticate via the OpenSSH agent (`SSH_AUTH_SOCK`). |
| `keyboardInteractive` | bool | `false` | Allow keyboard-interactive (OTP/MFA) using `password`. |
| `hostKeyMode` | string | `accept-new` | `accept-new` (record then verify) · `verify` (reject unknown) · `off`. |

### Transfers and limits

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `commandTimeoutMs` | int | `20000` | Per-command timeout. |
| `connectTimeoutMs` | int | `15000` | SSH connect timeout. |
| `maxOutputChars` | int | `200000` | Hard cap on collected output per call. |
| `maxFileBytes` | int | `52428800` | Skip mirroring/reading files larger than this (0 = no cap). |
| `encoding` | string | `utf-8` | Text encoding for remote reads/writes (e.g. `gbk`). |

### Workspace and shell

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `workspace` | string | `''` | Default remote workspace path. |
| `shell` | string | `''` | Terminal strategy: `''`=auto (Windows remotes use Git Bash) · `git-bash`=prefer it · `native`=never wrap · anything else=explicit `bash.exe` path. |
| `autoPush` | bool | `false` | Auto-push edited mirror files back to the remote (debounced watcher). |
| `auditLog` | bool | `true` | Append executed commands to `audit.log`. |

### Remote `@` completion

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `fileReference` | bool | `true` | In a remote session, list the remote tree for `@`. |
| `fileReferenceMaxResults` | int | `20` | Candidates rendered per query. |
| `fileReferenceMaxEntries` | int | `3000` | Max entries kept in one workspace's index. |
| `fileReferenceExcludedDirectories` | string[] | see below | Directory basenames the traversal skips. |
| `fileReferenceTimeoutMs` | int | `4000` | Wall-clock budget per index pass (partial index answers on expiry). |

Default excluded directories: `.git`, `node_modules`, `dist`, `build`, `out`, `coverage`, `target`,
`.next`, `.nuxt`, `.turbo`, `.venv`, `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.gradle`.

### Updates

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `updateMode` | string | `manual` | `manual` (check on demand) · `auto` (check on load + on a timer) · `off`. |
| `updateCheckIntervalMs` | int | `21600000` (6 h) | Auto-mode poll interval for a newer npm release. |

### Ignore rules

Sync and remote search honor a gitignore-**style** file at
`$DSH_HOME/remote-workspaces/.dsh-remote-ignore` (edit it with `/remote-ignore`). It is a deliberate
subset — no surprises:

- Blank lines and `#` comments are skipped.
- `!` negation is **not** supported.
- Trailing `/` matches directories only.
- A pattern containing `/` is anchored at the workspace root; otherwise it matches a basename at any
  depth.
- `*` (not `/`), `**` (including `/`), and `?` (one char, not `/`) are supported; `[abc]` character
  classes are not.

---

## HTTP surface

The plugin registers 25 routes under `/dsh-remote/*` on the host web server (plus four fs routes
reused by the optional sidebar), e.g. `status`, `machines`, `connect`, `test-connect`, `forwards`,
`audit`, `ssh-config`, `task`/`tasks`, and the `update-*` trio. It also mounts a small set over
`ctx.connection.fetch` under `/api/dsh-remote/*` for the Desktop carrier. This section is
informational; the agent uses the tools above.

---

## Development

```bash
npm ci
npm test
node check.mjs
```

- `node check.mjs` is the static gate (command-name regex, route prefix, …) — run it before every
  commit.
- `scripts/boot-smoke.sh` boots an isolated DSH instance to prove the plugin still starts.
- `scripts/dev-run.sh --restart` spins up a sandbox harness on `http://127.0.0.1:50599` with the
  plugin wired in; host-side edits (`lib/index.js`) need a restart, client-side edits
  (`lib/client.js`) only need a page refresh.
- Iterate in the sandbox — never hand-edit a product profile; it's re-managed by the plugin manager.
  `./sync.sh` deploys to a profile deliberately, for releases only.
- Full rules: `scripts/dev-standards.md`.
- On Windows keep the checkout LF (`git config core.autocrlf false`); `test/i18n.test.js` parses
  dictionary literals out of `lib/client.js` and a CRLF checkout breaks it for reasons unrelated to
  any code change.

---

## Safety

Adding a machine hands the agent that host's credentials, so it can run shell commands as your user
there. Add only hosts you trust. Passwords live in the machine registry on your machine (or the OS
keychain when enabled) — treat that file as sensitive. With `auditLog` on, every remote command is
recorded to `audit.log`; review it from the settings panel.

---

## License

MIT. Original work © [@flymysql](https://github.com/flymysql); this fork's DSH 0.2.0-rc.2
compatibility change © [@zhz1667](https://github.com/zhz1667). See [LICENSE](./LICENSE), and
[CHANGELOG.md](./CHANGELOG.md) for history.
