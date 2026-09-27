# Slash commands

Source: [`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md) (§Command reference) and
[`docs/interaction.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/interaction.en.md) (§Slash commands),
cross-checked against `src/commands.ts` (`LOCAL_COMMANDS`) at commit `bbad03077`.

## How the command menu is composed

The `/` menu merges three sources:

1. **Local commands** — declared by dsh-TUI in `src/commands.ts`. `LOCAL_COMMANDS` holds **57** entries
   (the docs' "built-in commands (50)" is an approximation), plus two aliases (`/background` → `/bg`,
   `/quit` and `/q` → `/exit`) and one hidden entry (`/deepseek`).
2. **DSH registry commands** — merged at runtime from the host command registry
   (`/plan`, `/goal`, `/permission`, `/feedback`, and anything a mounted plugin registers).
   For a name both sides declare, the **registry handler wins**; local commands win the collision
   filter only against the skill catalog.
3. **Skill catalog** — every discovered direct-call skill joins completion as `/<skill-name>`
   (completion only; it is not listed in the `/help` menu).

`/lang en|zh` switches the UI language *and* the command descriptions, resolved at render time, so a
language switch repaints them immediately.

Typing rules:

- Every command supports **Tab completion**. For commands with arguments, type `/command ` (with the
  trailing space) and then press `Tab`.
- **An unknown command is not an error** — the line is sent to the model as an ordinary message.
- Argument parsing splits on the same rule as the registry: `/plan off` dispatches `plan` with the
  raw input ` off`.

## 1. Session and conversation

| Command | Args | Effect |
|---|---|---|
| `/new` | — | Start a new session. No confirmation; the previous session is persisted and stays reachable via `/resume`. Also clears the resume marker. |
| `/resume` | — | Open the session manager (workspace + session columns, live filter, pins, cross-workspace). Switching sessions only *parks* the current one — a running turn keeps going. |
| `/home` | — | The same session manager, workspace view. |
| `/agentview` | — | The same session manager, hosted-session state. |
| `/bg` | — | Send this session to the background and open the session manager. Alias: `/background`. |
| `/workspace` | `resume` \| `rename <name>` \| `open <path\|URI>` | Manage workspaces: `resume` opens the workspace picker, `rename` renames the current workspace, `open` opens a workspace and starts a fresh session. A target may be an absolute path, a path relative to the current workspace, or a `file://` URL; other URI schemes come from optional plugins. |
| `/tree` | — | Session family tree: hover to preview, click to rewind / fork / switch branch. |
| `/fork` | — | Copy the current session into a resumable clone (the original is unaffected). |
| `/rename` | `[title]` | Rename the current session. With no argument it shows the current title and usage. |
| `/recap` | — | Recent-activity summary (one line) plus a suggested title; press `a` or click to apply the title. |
| `/rewind` | — | Open the rewind selector — the same screen as double-pressing `Esc` on an empty input. |
| `/clear` | — | Clear the *view* (expand/selection state). Never touches the session log. |
| `/compact` | — | Manual DSH compaction. Rejected mid-turn; unavailable under the `minimal` preset. |
| `/export` | — | Export the session as Markdown to the current session cwd (full log, including thinking and tool-call sections). |
| `/btw` | `<question>` | Side question: one turn, **no tools**, does not interrupt the main turn, not written to history. |
| `/trace` | — | Open the trace scene (same as `Ctrl+T`). |
| `/restart` | — | Restart the process and resume this session. Rejected mid-turn. |
| `/exit` | — | Exit dsh-TUI. Aliases: `/quit`, `/q`. |

## 2. Status and diagnostics

| Command | Args | Effect |
|---|---|---|
| `/context` | — | Loaded-context detail: instructions, runtime context, skills, tools. |
| `/status` | — | Session overview: model + effort, working/idle, session id, cwd + git branch, tokens, cache hit rate, context percentage, title. |
| `/cost` | — | Token usage and cache hit rate. DSH provides **no cost metering**; the bottom-bar cost figure is an estimate and the platform bill is authoritative. |
| `/tokens` | — | Token usage plus context percentage. |
| `/balance` | — | DeepSeek account balance (free read-only API): summary line, hover for detail, click to refresh. |
| `/config` | — | Configuration sources: `cordis.patch.yml` path, launch method, model routing. |
| `/doctor` | — | Environment check: Node/platform, API key, model routing, cwd, context window, session storage, plugin host. |
| `/migrate` | `[agent] [--dry-run]` | Import conversation history from another coding agent. See `sessions.md`. |
| `/init` | — | Create `AGENTS.md` in the working directory (reports created / exists / failed). |
| `/agents` | — | Subagent list for this session. |
| `/jobs` | — | Background-task panel: status, runtime, exit code; `↑/↓` select, `k` stop. While it is open, `Esc` closes **only the panel** — close it before pressing `Ctrl+C`. |
| `/settings` | — | Open the plugin settings editor. See `configuration.md`. |
| `/help` | — | Shortcut and command help (also reachable with `?` on an empty input). |
| `/tips` | — | Usage tips panel (`↑/↓` scroll, `Esc` close). The startup hint line advertises it. |
| `/plugins` | `check <dsh-plugin.json path>` | Plugin diagnostics: trust banner, host descriptor, authorization matrix, ledger. `check` validates a manifest and reports compatibility. |

## 3. Model, display and preferences

| Command | Args | Effect |
|---|---|---|
| `/model` | — | Model selector. **Switching forks the session** (history kept, only routing changes; the old session stays in `/resume`). Rejected mid-turn. Persisted to `~/.dsh-tui/model.json`. |
| `/effort` | `[status\|<id>]` | Reasoning effort. No argument opens a slider (`←/→` adjusts live); `status` reports the level; `<id>` sets it. Persisted to `~/.dsh-tui/effort.json`. |
| `/thinking` | — | Toggle the extended-thinking display (thinking expands item by item while streaming). |
| `/activity` | `[frames <name>\|status]` | Working-status animation. No argument opens the selector; `frames <name>` sets it directly (including `random`). Default `moon8`. Persisted to `~/.dsh-tui/working-activity.json`. |
| `/preset` | `[<id>\|status]` | Agent preset. **Cannot switch a session that already has messages** (blank-only). See `configuration.md`. Persisted to `~/.dsh-tui/agent-preset.json`. |
| `/theme` | `[<name>\|status]` | Theme. No argument opens the picker; `status` shows the current theme (and, under `auto`, the resolved OSC 11 result). See `themes.md`. |
| `/color` | `[<name>\|status\|reset]` | Session accent color. No argument opens the palette (`↑/↓` pick, `Enter` apply). Names: `red` `orange` `yellow` `green` `blue` `purple` `pink` `cyan`. Saved per session. |
| `/lang` | `en\|zh\|status` | Hot-switch the UI language. Precedence: `DSH_TUI_LANG` > profile config > persisted. |
| `/vim` | — | Toggle vim editing mode for the prompt. Per-session, not persisted. |
| `/terminal-setup` | — | Terminal setup advice (Windows Terminal ≥110 columns, paste keys). |

## 4. Account, policy and extensions

| Command | Args | Effect |
|---|---|---|
| `/provider` | — | Interactive provider wizard: add / edit / delete routes, or — with the bundled `dsh-auth` plugin mounted — **OAuth subscription sign-in** for ChatGPT / Claude / Grok (no API key). See `configuration.md`. |
| `/login` | — | Credential status: source, store writability, base URL. |
| `/logout` | — | Logout notes (for an environment-sourced key, delete the variable and restart). |
| `/permission` | `[<preset>\|status]` | View or switch the permission preset and policy; no argument opens the selector. |
| `/add-dir` | — | File-policy scope notes (rooted at the working directory). |
| `/hooks` | — | Placeholder: notes when DSH hooks are not mounted in the composition. |
| `/mcp` | — | MCP connection state, tools grouped as `mcp__<server>__<tool>`; shows a `cordis.patch.yml` snippet when nothing is configured. See `configuration.md`. |
| `/skills` | — | Skill-catalog selector (name, source, blurb). `Enter` fills a direct-call skill as `/name `. dsh-TUI itself ships no generic skills. |
| `/update` | — | Update dsh-TUI and restart to resume the session. Rejected mid-turn and **only works when launched via `dsh --profile dsh-tui`**. |

## 5. Placeholder commands

| Command | Status |
|---|---|
| `/connect` | Placeholder — DSH has no remote-connection mechanism yet. |
| `/hooks` | Placeholder unless DSH hooks are mounted. |

## 6. Registry commands (host-provided)

| Command | Effect |
|---|---|
| `/plan` | `[off\|message]` plan mode; `/plan off` exits. |
| `/goal` | Set or view the session goal. |
| `/feedback` | Submit usage feedback. |
| `/permission` | View/switch presets from the DSH `permissionPresets` registry (third-party presets in registry order). |

These come from the DSH command registry: dsh-TUI only merges them into the menu, completes and
dispatches them. Anything else the active composition registers appears in the same menu.

## 7. CLI subcommands (not slash commands)

| Command | Purpose |
|---|---|
| `dsh-tui update` | Update the profile to the latest release and align the launcher. Same install logic as `/update`, without restarting into the TUI. |
| `dsh-tui doctor` | Environment checks: dsh/pnpm, profile install and version alignment, whether the API key is set (state only, never the value), config file presence. |
| `dsh-tui safe` | Safe mode: read-only diagnostics, plugin inventory, repair guidance. `safe --rescue` also creates/verifies the clean rescue profile. |
| `dsh-tui version` | Launcher and profile versions (`--version` / `-v` are equivalent). |
| `dsh-tui help` | Usage (`--help` / `-h` are equivalent). |
| `dsh-tui migrate [agent] [--dry-run]` | Session import from the command line (same logic as `/migrate`). |
| `dsh-tui --resume [id]`, `-c`, `--continue` | Resume the last (or a specific) session. |

`help` and `version` work even when `dsh` is missing or the profile is not initialized; **every other
argument is forwarded verbatim to `dsh --profile dsh-tui`**.

## Divergences and gaps in the docs

- `getting-started.en.md` documents only `update` / `doctor` / `safe` / `version` / `help`; the
  `--resume [id]` and `-c, --continue` options are documented in `user-guide.en.md` and confirmed by
  running `dsh-tui help` against launcher `0.11.0`.
- The "built-in commands (50)" figure in `user-guide.en.md` does not match `src/commands.ts`; the
  count stated above is the source-verified one. `/tips` and the hidden `/deepseek` are described in
  prose but appear in no command table.
- `/find` (and `Ctrl+Shift+F`) is **not** a dsh-TUI command: it belongs to the third-party
  `dsh-tui-find` plugin listed in `docs/links.md`.
- Neither language documents timeouts for questionnaires/approvals, and the `#L12-14`-style
  file:line reference syntax mentioned on the project README is absent from `docs/interaction.*`.
