# Keys, editing, mouse and prompts

Sources: [`docs/interaction.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/interaction.en.md) and
[`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md), cross-checked
against the Chinese originals `docs/interaction.md` / `docs/user-guide.md` (ZH-only facts are marked).

`⌘` may replace `Ctrl` on macOS for the bindings that carry it (`⌘V`, `⌘O`, `⌘R`, `⌘T`, `⌘L`, `⌘Enter`, …),
except `Ctrl+C` / `Ctrl+D`, which stay on `Ctrl`. `⌘` needs the extended keyboard protocol
(iTerm2 / kitty / WezTerm / ghostty / tmux); Terminal.app must use `Ctrl`.

## Sending while the model is working

| Key | Behavior |
|---|---|
| `Enter` | Idle: send. **Working: steer** — inject the text at the next step boundary, no interrupt. Menu open: confirm |
| `Tab` | Complete a `/` command or `@` file. **Working: follow-up** — queue after the current turn |
| `Ctrl+Enter` | Interrupt the running turn and send the input immediately |
| `Shift+Enter` / `Ctrl+J` | Newline at the caret; `Ctrl+J` is the fallback when the terminal cannot report Shift; macOS Terminal.app uses `Option+Enter` |
| `Alt/Option+Up` | Pull the last undelivered message back into the editor (no interrupt) |
| `Esc` with pending messages | Interrupt the turn and re-send the pending message now |
| `↑` on a queued message | Withdraws that queued entry, same as `Alt+Up` (once the running turn claimed it, only a notice appears) |

## Global keys

| Key | Behavior |
|---|---|
| `Shift+Tab` | Cycle session modes (default → plan → full-access); mounted third-party permission presets follow in registry order |
| `Ctrl+O` | Expand/collapse transcript detail: full reasoning, tool arguments and output. Also unfolds the long-line fold |
| `Ctrl+P` | Toggle the loaded-context panel shown at startup |
| `Ctrl+T` | Open the trajectory/trace scene (same as `/trace`); `q`/`Esc` returns |
| `Ctrl+R` | Input-history search; repeat or `↓` for the next result; `Enter` fills the input |
| `Ctrl+L` | Clear and force a physical terminal redraw (also reloads the session manager) |
| `?` | Shortcut and command help when the input is empty |
| `Ctrl+Shift+E` | Expand the fullscreen draft editor (or click `⛶` at the end of the input row) |
| `Ctrl+G` | Edit the input in `$VISUAL` → `$EDITOR`; `:cq`/non-zero exit keeps the draft. With neither set the TUI asks you to configure one (no `vi` fallback) |
| `←` on empty input | Background this session and open the session manager (same as `/bg`) |
| `Shift+Up` | Enter message selection (`↑/↓` move, `Enter` expands one row, `Esc` exits) |
| `PgUp` / `PgDn` | Page the fullscreen transcript (inline mode leaves paging to the terminal) |

Remappable in `/settings` → `dsh-tui` → `Shortcuts`: paste, history search, external editor,
`Ctrl+O/T/P/R/L`, subagent dashboard, show-all, todo fold. Enter combos like `alt+v`, comma-separate
several, leave blank to restore defaults; saves apply live and clashing combos are rejected.
Deployments can pin them with `shortcuts.<action>` in `cordis.yml`.

## Interrupt / exit ladders

`Ctrl+C`:

1. Working → interrupt. If the interrupt will not settle, press again to force-exit.
2. Idle with a **selection** in the prompt → copy the selection (selection is kept for editing).
3. Idle with non-empty input → clear it.
4. Idle, empty input → press twice to exit.

`Ctrl+D` follows the same ladder minus the copy step. `Esc` unwinds layer by layer:

```text
close help -> close the image preview -> close the command menu -> close the file menu (current @ token only)
-> with a selection in the input: clear only the selection (text untouched)
-> interrupt the turn and redeliver pending messages -> clear non-empty input
-> double-press on empty input = rewind (within 3 s)
```

With an image preview open, `Esc` / `Ctrl+C` / `Enter` close it and other keys are not passed through.
In the `Ctrl+O` transcript view, `/` opens full-session search instead of command completion.

## Editing keys

| Key | Behavior |
|---|---|
| `←` / `→` | Move by character; with a selection, collapse to the corresponding edge |
| `Ctrl+←` / `Ctrl+→` | Move by word |
| `Home` / `End` | Start / end of the current logical line (`Ctrl+E` also moves to line end) |
| `Ctrl+A` | Opens the **subagent dashboard** (`Mod+A` inside the editor still moves to line start) |
| `Ctrl+U` / `Ctrl+K` | Delete before / after the caret |
| `Ctrl+W` | Delete the preceding word |
| `Backspace` / `Delete` | Delete the character before / after the caret; with a selection, delete the whole selection |
| `Up` / `Down` | Move between lines when multiline; otherwise browse input history (last 200 entries, kept across restarts) |
| `Ctrl+V` / `Alt+V` | Paste clipboard text or files; images become durable attachments. Use `Alt+V` when the terminal intercepts `Ctrl+V` |
| Typing | Replaces an active selection; the caret lands after the inserted text |
| `Ctrl+Shift+E` | Fullscreen draft editor: line numbers, current-line highlight, live line/char stats; `Enter` inserts a newline, `Ctrl+Enter` sends, `Esc` collapses and keeps the draft |

Long drafts: pasting ≥6 lines or ≥600 characters folds into a chip; click it or `Ctrl+O` to expand —
`Enter` always submits the full text.

### vim editing mode (`/vim`)

`/vim` toggles vim editing for the prompt (per-session, not persisted). The prompt shows an `INSERT`
badge; `Esc` switches to `NORMAL`.

| NORMAL keys | Action |
|---|---|
| `h` `l` `j` `k` | Move left / right / up / down |
| `0` `^` `$` | Line start / first non-blank / line end |
| `w` `b` | Word forward / back |
| `x` `X` | Delete character under / before the caret |
| `dd` `d$` `d0` `d^` `dw` | Delete line / to end / to start / to first non-blank / word |
| `u` | Undo |
| `i` `I` `a` `A` `o` `O` | Back to INSERT at the usual positions |
| `Enter` | Send |

Unrecognized keys are ignored; `Esc` does nothing in NORMAL; clear with `Ctrl+C` or `dd`.

## Completion

**Slash commands** — `/` opens the menu; `Tab` completes. The menu merges local commands, the DSH
registry, and skill commands (`/name`). An unknown command is sent to the model as an ordinary
message rather than rejected.

**`@` files** — `@` triggers at any position in the message:

- Keep typing to filter; `Tab`/`Enter` accept; directories drill down; `Esc` closes only the current
  `@` token's menu.
- Plain fragments use fuzzy subsequence matching with prefix/boundary and short-path boosts.
- Path-shaped queries (`@src/`, `@./`, `@../`, `@~/`, `@D:\`, anything with a separator) read only that
  directory rather than scanning the repository.
- PNG/JPEG/WebP/GIF become durable image attachments; text files and directory listings attach as text.
- Async candidate refresh keeps the currently selected candidate selected.

`/workspace ` completion offers built-in and plugin-contributed subcommands (plugin aliases match too).
A workspace target may be an absolute path, a path relative to the current workspace, or a `file://`
URL; `dsh-tui .`, `dsh-tui ../project` and `dsh-tui file:///path/to/project` accept the same targets.

## Mouse (fullscreen mode)

Mouse behavior belongs to `fullscreen: true`. Inline mode enables no mouse reporting — with one ZH
caveat: `interaction.md` says the full-screen trajectory scene still enables mouse tracking in inline
mode, which `interaction.en.md` does not state (see the divergences note below).

| Action | Effect |
|---|---|
| Left-drag | Select text and copy on release (OSC 52 plus system clipboard tools as fallback), then auto-clear |
| Double / triple click | Select word / line, copy immediately |
| Wheel | Scroll the message list; with a text selection, pan the selection with the content |
| In the input box | Drag / `Shift`+click / double-click build a selection; `Backspace`/`Delete` delete it, typing replaces it, `←/→` collapse it, `Esc` clears only the selection, `Ctrl+C` copies it |
| Single click | Message row: expand/collapse. Hyperlink: open browser. Sticky header / "load earlier" / "↓ N new": load or jump |
| Hover truncated content | After ~600 ms a popover shows the full content; no popover while dragging a selection |
| Keyboard extend | With a selection, `Shift+←/→/↑/↓/Home/End` extends or shrinks it across lines |
| Scrollbar track | Draggable with `scrollGutter: scrollbar`; `Shift`/`Alt`/`Ctrl`+drag is still text selection |

## Questionnaires, approvals and plan review

These panels temporarily own the keyboard.

| Panel | Keys |
|---|---|
| `ask_user_question` questionnaire | `↑/↓` move, `Space` toggle a multi-select option, `Tab` jump to the custom-answer line, `Enter` submit. `Esc` from question 2 returns to the previous question keeping the draft; `Esc` on question 1 cancels the whole batch (the model receives `ASK_CANCELLED`); `Ctrl+C` cancels from any question. `Ctrl+K` folds the panel — the ask keeps waiting, and while folded `Esc`/`Ctrl+C` expand first |
| Tool approval | `↑/↓` move, `1` allow once, `2` deny, `Enter` submit; `Esc`/`Ctrl+C` deny |
| Plan review | `↑/↓` move, `1`/`2` quick pick, typing fills feedback, `Enter` submits, `Esc` interrupts. **Approving must carry no feedback text** |

Details:

- The questionnaire's **last row is a free-input line**: typing directly on an option row submits that
  option's label together with your custom text, no `Tab` needed.
- Batched questions and concurrent subagent questions are shown one at a time in FIFO order, and a
  compact Q&A summary is folded into the local transcript afterwards.
- **Approval wins over a questionnaire** when both are pending; an approval from a background session
  is labeled with that session.
- The approval waterfall allows only allow-once / deny — there is no "always allow" and no feedback
  channel in the tool-approval seam (documented only in the Chinese `architecture.md`).

## Mode badges and permission policy

`Shift+Tab` cycles the configured session modes; the badge shows the active one. The built-in trio is
default (workspace-write + approval) → plan (read-only) → full-access (danger-full-access). After
approving a plan or `/plan off`, the sandbox and approval policy return to their pre-plan state.
Third-party permission presets appended to the cycle are owned by the host registry (ZH-only detail).

## Terminal requirements

- Interactive TTY required. Windows Terminal recommended: ≥110 columns, monospace, TrueColor.
- Images need Kitty graphics or Sixel (auto-detected, Kitty preferred); tmux/screen, non-TTY and
  accessibility mode show text only, and **sending images to the model is unaffected**.
  `DSH_TUI_IMAGE_PROTOCOL=auto|kitty|sixel|none` overrides the protocol;
  `DSH_TUI_DISABLE_TERMINAL_IMAGES=1` forces previews off; `DSH_TUI_DISABLE_MOUSE` disables mouse
  handling in fullscreen mode.
- `/doctor` checks the environment; `/terminal-setup` gives terminal advice.

## Divergences and gaps in the docs

- **Mouse in inline mode** — `interaction.en.md` states that all mouse behavior applies only under
  `fullscreen: true` and inline enables no mouse reporting; `interaction.md` adds that the full-screen
  trajectory scene still tracks the mouse in inline mode. Verify in a real terminal before relying on it.
- **ZH-only**: the external injection channel (editor integration) — a local injection socket at
  `~/.dsh-tui/inject/<sessionId>.sock` (Windows pipe `\\.\pipe\dsh-tui-inject-<sessionId>`),
  `~/.dsh-tui/inject/servers.json`, and newline-delimited JSON `prompt.append` / `command.execute`
  messages (implementation `src/dsh-adapter/inject-channel.ts`). The English page omits it entirely.
- **EN-only**: the macOS `⌘` block; the background-sessions section (`←` on empty input, the footer
  agent count, the approval panel naming its originating session).
- `#L12-14`-style file:line references and questionnaire/approval timeouts are documented in neither language.
