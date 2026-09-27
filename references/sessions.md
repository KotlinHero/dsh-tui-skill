# Sessions, workspaces and migration

Sources: [`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md)
(§Session workflows), [`docs/interaction.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/interaction.en.md)
(§Session management screen, §Rewind, §Background sessions) and
[`docs/migrate.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/migrate.en.md).

## Session lifecycle

| Action | Command / key | Notes |
|---|---|---|
| New | `/new` | No confirmation. The previous session is persisted and stays reachable via `/resume`; the resume marker is cleared |
| Resume | `/resume`, `/home`, `/agentview`, `/bg`, or `⌸` at the input line start | All open the same session-management screen |
| Rename | `/rename <title>` | Persists immediately (writes a session/title event readable back in the manager) |
| Compact | `/compact` | Manual DSH compaction. **Rejected mid-turn** and **unavailable under the `minimal` preset**; the compaction point renders as a divider summary row |
| Export | `/export` | Markdown export of the full session log (thinking and tool sections) into the session cwd |
| Clear | `/clear` | Clears the view only — never the session log |
| Stop | session manager `Ctrl+X` | Stops the **background** session under the cursor. The session this terminal is attached to cannot be stopped; exit the TUI instead |
| Exit | `/exit`, `/quit`, `/q`, or double-press `Ctrl+C` / `Ctrl+D` when idle | Mid-work, press `Ctrl+C`/`Ctrl+D` again to force quit when the interrupt will not settle |

Command-line resume: `dsh-tui --resume` (last session), `dsh-tui --resume <id>` (specific session);
`-c` / `--continue` are equivalent. On Windows `dsh-tui.cmd --resume` reads
`~/.dsh-tui/resume.txt`.

## The session manager

One screen behind `/resume`, `/home`, `/agentview`, `/bg` and `⌸`: a workspace rail on the left and the
selected workspace's sessions on the right, each row carrying live state.

| Key | Action |
|---|---|
| `←` `→` | Choose which column owns the keyboard (rail / sessions); exactly one `❯` is on screen |
| `↑` `↓` / `PgUp` `PgDn` / wheel | Move within the rail or the session list (row 0 is the new-session card) |
| Type | Live-filter the selected workspace's sessions by title, directory, branch or model |
| `Enter` | Sessions column: enter the row under the cursor (row 0 starts a session in this workspace). Rail: open that workspace's action menu |
| `Ctrl+Enter` / `Ctrl+N` | Start a session in the workspace under the cursor |
| `Ctrl+X` | Stop the **background** session under the cursor |
| `Ctrl+L` | Re-read the workspace registry and the session listing |
| `Shift+Tab` | Rail: open the action menu of the workspace under the cursor |
| `Esc` | Dismiss a notice → clear the filter → leave the screen |

Row semantics:

- A workspace joins the rail by **starting dsh-tui in that directory**; there is no "add workspace"
  picker. The rail action menu has four entries: edit/select, new session, rename, and
  **remove from list** (removes the registration only — the directory and every session log survive).
- Live state comes from the agent-view projection: `working` / `needs input` / `idle` / `completed` /
  `failed` / `stopped`, with the attached session marked `current`.
- Clicking the in-row `★`/`☆` toggles the pin only (persisted in `~/.dsh-tui/session-pins.json`) and
  never enters the session.
- A session held by **another** TUI terminal is still listed but turns red with `held by pid <pid>`
  and cannot be entered: two processes driving one append-only log would interleave events and corrupt
  the transcript. Occupancy is re-read on the screen's own 2-second tick and clears when that process exits.
- **Switching sessions parks the current one — it does not stop it.** A turn in flight keeps running.
- Opening the screen does not lose an unsent draft: the composer unmounts with the screen and keeps
  text, caret position, staged image references, fold state, fullscreen-editor state and vim mode until
  `Esc` returns. A draft belongs to its session.
- An **"unregistered" group** appears when the workspace service is absent (bare composition) or a
  session's directory is not registered. Those sessions list and resume as usual; the group lives only
  inside this screen and is never written back to the registry.

Background sessions (`/bg`, or `←` on an empty prompt — the second item is **documented only in the
English page**):

- `/bg` moves the current session to the background (it keeps running), switches the terminal to a
  fresh session, and opens the session manager.
- While a background session waits on you, the prompt footer shows `← N agents`.
- A background session awaiting approval shows as **needs input** and the approval panel labels which
  session it comes from.

## Rewind / time travel

Double-press `Esc` on an empty input (or `/rewind`) opens the list of your own messages.

1. The TUI finds the start of the turn containing that message.
2. It creates a branch session through the DSH session fork.
3. It replays history before the boundary.
4. It restores the original message to the editor for revision and resubmission.

- The boundary is **before the turn** that contained the message, and you **cannot rewind past the
  first message**.
- If the model is working, the TUI cancels the turn and waits up to **30 seconds** for it to settle.
- The rewound branch is not a subagent: it records `parentSession` without `origin`, and keeps the
  current model route plus the session's own preset.
- Plugins can intervene through the `tui/rewind-prompt` decision event (veto with a reason, or offer
  extra rewind modes such as "rewind the conversation and restore files changed since"). The first
  option is always "Conversation only"; after a plugin mode runs, the plugin receives
  `tui/rewind-done` with the mode id and both session ids.

## Forking, `fork` tree and model switching

- `/fork` copies the current session into a resumable clone; the original is unaffected.
- `/tree` shows the session family tree with hover preview and click-to rewind / fork / switch branch.
- `/model` **switches by forking the session**: history is kept and only routing changes; the previous
  session stays in `/resume`. Rejected mid-turn. The choice persists in `~/.dsh-tui/model.json`.
- `/preset` cannot switch a session that already has messages; the choice becomes the default for the
  next `/new` (`configuration.md`).

## Side question: `/btw`

`/btw <question>` reuses the current context (system prompt plus existing history) for one
**tool-less** turn. The answer appears in a scrollable panel: `↑/↓` scroll, `c` copy, `Space`/`Enter`/`Esc`
close, `Esc` cancels while it is still waiting. The answer is **not written to history and counts no
tokens**, and the main turn is never interrupted.

## Message delivery while the model is working

| Input | Semantics |
|---|---|
| `Enter` | **Steer** — inject at the next step boundary, no interrupt |
| `Tab` | **Follow-up** — queue for after the current turn |
| `Ctrl+Enter` | **Interrupt** — cancel the turn and send now |
| `Alt+Up` | Recall the latest undelivered message |
| `↑` on a queued message | Withdraw the queued entry (once the running turn claimed it, only a notice appears) |
| `Esc` with pending messages | Interrupt and re-send the pending message now |

## Session storage

- Profile mode uses the base JSONL persistence row rooted at the shared `$DSH_HOME/sessions`
  (`~/.dsh/sessions` by default), so dsh-TUI and dsh web read the same history. A bare `cordis.yml`
  composition defaults to `~/.dsh-tui/sessions/`. `DSH_TUI_SESSION_ROOT` overrides and always names a
  JSONL root.
- Session storage has moved between releases: since 0.3.7 `/resume` uses the JSONL store shared with
  dsh web, so sessions missing from the list after a major update is expected — the data is not deleted.

## Importing sessions from other agents (`/migrate`)

```sh
dsh-tui migrate                    # list per-agent scannable file counts (writes nothing)
dsh-tui migrate claude-code        # import every Claude Code conversation
dsh-tui migrate codex --dry-run    # preview what would land, write nothing
```

In-TUI equivalent: `/migrate`.

- Bare `/migrate` opens a multi-select source picker — one row per agent (checkbox, scannable file
  count, an "active X min ago" badge, most recently active first). `Space` toggles, `a` selects
  all/none, `Enter` opens the confirmation layer (one line per checked source with its count, plus
  the note that existing sessions are skipped automatically and re-running is safe), `Enter` imports,
  `d` dry-runs, `Esc` returns. `Esc` closes the picker.
- `/migrate <agent>` goes straight to the same confirmation layer; `/migrate <agent> --dry-run`
  previews directly.
- The completion notification carries per-source real counters ("imported X · already present Y").
- The import runs in a **child process**, so the interface never freezes; results arrive through the
  notification flow and land in a local `/migrate` row. Both entry points share the same import logic
  and idempotency rules; the CLI stays direct for scripts.

Supported sources:

| Source | Local store | Notes |
|---|---|---|
| `claude-code` | `~/.claude/projects/` | Thinking traces preserved per turn |
| `codex` | `~/.codex/sessions/` | Model id carried forward from `turn_context`; encrypted reasoning stays unreadable and is not migrated |
| `omp` | `~/.omp/agent/sessions/` | DSH-lineage store, near-direct mapping |
| `zcode` | `~/.zcode/v2/sessions/` | Single-JSON-object format; roles, timestamps and cwd present |
| `grok-build` | `~/.grok/sessions/` (relocatable via `GROK_HOME`) | Reasoning sibling rows attach to the following assistant turn; `synthetic_reason` injected rows (system reminders, …) are not migrated |

Behaviour contract:

- **Read-only source.** Migration only reads the foreign agent's store. Artifacts are written through
  the official `JsonlSessionPersistence` into `$DSH_HOME/sessions`, so imported sessions are
  first-class: openable, continuable, rewindable.
- **Idempotent.** One deterministic UUID (v5) per source conversation; re-importing skips what is
  already present instead of stacking duplicates or rewriting existing logs. Both entry points derive
  identical session ids.
- **Structure preserved.** User/assistant messages and thinking traces are rebuilt turn by turn;
  multiple assistant messages inside one turn each get their own step.
- **Not migrated**: tool traffic (the contract is "re-read the conversation", not "resume the task")
  and per-message original timestamps (event times are import-time; the session start keeps the source record).
- **Robustness**: malformed lines, legal JSON `null` and files over **64 MB** are skipped safely. One
  failed conversation never aborts the batch; failures are listed and reported with **exit code 1**.

Measured reference (from the docs, real data): listing five sources over ~3000 files takes 0.7 s;
importing 60 zcode conversations 2.3 s; 163 claude-code conversations (~heavy thinking) 39 s;
728 codex + 1366 omp conversations about 2.5 minutes; a re-import is under 1 s because everything is
already present.

Afterwards `/resume` lists them in per-cwd directories under the official encoding rules, with titles
and start times readable, and thinking renders as collapsible reasoning blocks.

**Smart migration hint**: about 12 seconds after startup one background pass checks whether any source
saw file writes within the last 20 minutes and surfaces a single notification ("Just came from
&lt;agent&gt;? `/migrate` imports it quickly"). The scan is off the render path, fires at most once per
session, and while the hint is up `Enter` (with an empty prompt) jumps into the picker with that source
pre-checked.

## Divergences and gaps in the docs

- **EN-only**: the background-sessions section (`←` on an empty prompt, the `← N agents` footer, the
  approval panel naming its session); `/tree` and `/fork` appear in tables only in the English page.
- **Migration hint dismissal**: `migrate.en.md` says any other key dismisses the hint; `migrate.md`
  says other keys behave as usual and does not mention dismissal.
- **Silent scan source**: the English page generalizes ("a source with no data stays silent"); the
  Chinese page names `grok-build` specifically.
- **`GROK_HOME`**: the English page says to set it when grok-build lives elsewhere; the Chinese page
  requires it as an environment variable when relocating.
- "migrate" in this section is unrelated to the `dsh-cc-tui` package rename (`getting-started.md`).
