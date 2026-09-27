# Integrations: VS Code, Herdr, editors and external injection

Sources: [`docs/vscode.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/vscode.en.md),
[`docs/getting-started.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/getting-started.en.md),
[`docs/interaction.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/interaction.en.md) and
[`docs/architecture.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/architecture.en.md).

## Two ways to run in VS Code

| Option | What it is |
|---|---|
| **1. `dsh-tui-vscode` companion extension** (recommended) | The TUI runs in a real VS Code integrated terminal in a new column beside the editor. Multiple concurrent sessions, a sidebar session history, one-click start, resume-last and specific-session resume. Publisher `baobaolaodie`, on the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=baobaolaodie.dsh-tui-vscode) |
| **2. The built-in integrated terminal** | Run `dsh-tui` directly: zero install, seconds to start |

The extension uses `createTerminal` with the CLI inside it — **no webview and no xterm emulation** — and
"does not touch the TUI's rendering core: it only hosts it and adds editor integration".

`dsh-tui` (this repository) and `dsh-tui-vscode` (the extension) are **versioned and released
independently**. The selection channel requires an extension speaking protocol v2
(**`dsh-tui-vscode` ≥ 0.7.0**); with an older extension the feature stays silently disabled and
everything else is unaffected.

### Extension features

- Editor-title button, activity-bar whale icon, command-palette entries.
- A `DeepSeek` integrated terminal opened with `ViewColumn.Beside` — it never takes the current column.
- **One independent terminal and process per session**; existing sessions keep running in their own terminals.
- Sidebar session history grouped by project, with refresh and specific-session resume.
- Environment injection for `DSH_TUI_LANG`, `$VISUAL`, `$DSH_HOME`, a target session id, and
  `DSH_TUI_IDE_PORT` / `DSH_TUI_IDE_TOKEN` for the selection channel.
- Per-session termination when its terminal closes, or after double `Ctrl+C` inside the TUI.
- While sessions run, a bottom-left status-bar item (`dsh-tui`) appears; clicking it starts a new
  session (`dsh-tui-vscode.open`).

### Extension commands

| Command ID | Title | Action |
|---|---|---|
| `dsh-tui-vscode.open` | Open panel | Start a new session (the editor-title button uses this entry point) |
| `dsh-tui-vscode.start` | Start new session | Start a new session |
| `dsh-tui-vscode.resume` | Resume last session | Resume via `--resume` |
| `dsh-tui-vscode.focus` | Focus session panel | Focus the most recent terminal, else start one |
| `dsh-tui-vscode.kill` | Terminate session | Send Ctrl+C to the most recent terminal |
| `dsh-tui-vscode.refreshSessions` | Refresh sessions | Manually refresh the sidebar |
| `dsh-tui-vscode.resumeSession` | Resume session | Resume a specific session (sidebar click) |
| `dsh-tui-vscode.insertAtMention` | Insert @-mention | With editor focus press `Ctrl+Alt+K` (macOS `Cmd+Alt+K`) or use the editor context menu: inserts the current file/selection as `@absolute/path Lstart-end` into the dsh-tui input (whole file when nothing is selected; falls back to the clipboard when no session is running) |

### Extension settings

| Key | Default | Description |
|---|---|---|
| `dsh-tui-vscode.command` | `dsh-tui` | Launch command (resolved to an absolute path against the host `PATH`) |
| `dsh-tui-vscode.extraArgs` | `[]` | Extra CLI args, e.g. `["--lang","en"]` |
| `dsh-tui-vscode.lang` | `""` | `""` / `zh` / `en`, exported as `DSH_TUI_LANG` |
| `dsh-tui-vscode.injectEditor` | `true` | Export `$VISUAL` when unset (Chinese page: when neither `$VISUAL` nor `$EDITOR` is set) |
| `dsh-tui-vscode.editorCommand` | `code -w` | Value exported as `$VISUAL` |
| `dsh-tui-vscode.dshHome` | `""` | `$DSH_HOME` override (empty = inherit) |

Prerequisites: VS Code ≥ 1.90 and a global `dsh` CLI plus `dsh-tui` (0.7.0+ recommended);
`DEEPSEEK_API_KEY` for running models. Install the extension from the Marketplace
(`Ctrl+Shift+X`, search `dsh-tui`) or from source:
`git clone https://github.com/baobaolaodie/dsh-tui-vscode.git`, `npm install`,
`npm run package && code --install-extension dsh-tui-vscode-<version>.vsix --force` (or `npm run install:local`).

## IDE selection channel

- With a build that includes the channel, the extension runs a **loopback WebSocket server** and writes
  a lock file (directory `0700`, file `0600`).
- Extension-launched sessions connect through injected environment variables at startup. Manually
  launched sessions (tmux, SSH) discover it by scanning the lock — **only windows whose workspace covers
  the session directory are ever dialed**; with no match the integration stays silently disabled and a
  foreign project is never connected.
- Selecting code instantly shows a `⧉ N lines selected` badge under the TUI prompt; it disappears when
  the selection clears.
- Submitting attaches only the selected lines, with a `⧉ Selected N lines from <relative path>`
  indicator above the user bubble (rebuilt after a restart).
- Pushes carry the editor buffer's **own** text, so unsaved edits are attached exactly as seen.
  Oversized selections are capped and marked with the same policy as `@`-mentions.
- Without an IDE or after a disconnect everything degrades silently.
- Handshake: the TUI sends `ide/hello` (token + `protocolVersion`); the extension answers
  `ide/hello_ack` (protocolVersion + workspace folders) **only when the token validates**. A wrong token
  is silently dropped; `selection_changed` then carries absolute **0-based inclusive** line numbers plus
  the buffer's own selection text. **Both ends must ship the same protocol version.**

## Running in the built-in terminal (or any xterm.js host)

VS Code, Cursor and code-server get dedicated compatibility paths under `TERM_PROGRAM=vscode`:

- Truecolor, OSC 8 links (VS Code renders them clickable), OSC 52 clipboard (VS Code prompts for
  permission the first time), synchronized output and smooth draining.
- Streaming Markdown, tool cards, scrolling and double-Esc time travel behave exactly as in a
  standalone terminal.
- Resume options: `dsh-tui --resume` (last), `dsh-tui --resume <id>` or `--resume=<id>` (since 0.7.0);
  `-c` / `--continue` equals `--resume`.
- `Ctrl+G` uses `$VISUAL`/`$EDITOR`; to edit inside VS Code export `code -w` through
  `terminal.integrated.env.<platform>` in `settings.json`, or let the companion extension export it.
- UI language defaults to Chinese; set `"DSH_TUI_LANG": "en"` for English.

Known differences in the integrated terminal: wheel and drag selection belong to the integrated
terminal ("copy on release" becomes an OS-level copy); the extended keyboard protocol is decided by
xterm.js and may differ from kitty/WezTerm; OSC 52 clipboard triggers VS Code's own permission prompt.
Acceptance baseline: any rendering change should be walked through inside the VS Code integrated
terminal in **both inline and fullscreen modes at narrow widths** — startup, resize, scroll, input,
cancel, clean exit.

## Sidebar data and title precedence

Sidebar session data comes from the session logs under `~/.dsh/sessions` (zstd JSONL), the dsh-storage
ledger `~/.dsh/storages/session_projcache.json`, and the TUI's `~/.dsh-tui/last-used.json`.

Title precedence: log `session/title` event → storage-ledger title → first user message → "unnamed
session". Items group by project (cwd short name), most recently active first, and within a group most
recently used first; auto-refresh watches the session directories.

Limitations of the sidebar: for logs without a `session` header the project name comes from decoding
the cwd-encoded group directory, which is **lossy for hyphenated project names** (the real cwd is in the
item tooltip); specific-session resume requires this profile's `cordis.patch.yml` (dsh-tui 0.7.0+).

## Herdr

Run `dsh-tui` directly in a [Herdr](https://herdr.dev) pane with no extra setup. dsh-TUI reports
`idle` / `working` / `blocked` through Herdr's local integration API — questionnaires and tool approvals
count as `blocked` — and stays completely inactive outside Herdr.

## External injection channel (editor integration)

Documented **only in the Chinese `interaction.md`**, and implemented in
`src/dsh-adapter/inject-channel.ts` (with the regression `scripts/verify-inject-channel.mjs`):

- A local injection socket per session: `~/.dsh-tui/inject/<sessionId>.sock`, or on Windows the named
  pipe `\\.\pipe\dsh-tui-inject-<sessionId>`.
- A registry of endpoints in `~/.dsh-tui/inject/servers.json`.
- Newline-delimited JSON messages, e.g. `prompt.append` and `command.execute`, so an editor or script
  can push text into the prompt or run a command in the running TUI.
- The English documentation omits this channel entirely — treat the details above as ZH-sourced and
  verify against `docs/interaction.md` (and the source) before relying on them.

## Images and terminal capability

- Thumbnails and full previews need Kitty graphics or Sixel (auto-detected, Kitty preferred).
- `DSH_TUI_IMAGE_PROTOCOL=auto|kitty|sixel|none` overrides protocol selection without bypassing the
  preview preference, the disable switch, non-fullscreen, accessibility or multiplexer guards;
  `DSH_TUI_DISABLE_TERMINAL_IMAGES=1` forces previews off.
- tmux/screen, non-TTY and accessibility sessions show text only. **Sending images to the model is
  unaffected in every case.**
- Disabling previews never changes what is attached to a prompt: staged images are real image blocks
  and the prompt never contains base64.

## Divergences in the docs

- `injectEditor` is described as "Export `$VISUAL` when unset" in English and "when neither `$VISUAL`
  nor `$EDITOR` is set" in Chinese; the editor entry point is an "editor-title button" (EN) vs an
  "editor tab-bar button" (ZH).
- The external injection channel exists only in the Chinese interaction page.
- The English sidebar ordering is explicit about two levels; the Chinese summary only says
  "most recently active/used".
