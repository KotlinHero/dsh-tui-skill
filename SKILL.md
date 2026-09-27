---
name: dsh-tui-skill
description: Operate, configure and troubleshoot dsh-TUI — the interactive terminal UI plugin for DeepSeek Harness (`@deepseek-harness-tui/dsh-tui`, commands `dsh-tui` / `dst`, profile `dsh-tui`). Use when installing, updating or repairing dsh-TUI (npm/profile install, `dsh-tui update` / `doctor` / `safe --rescue`, version or pnpm alignment, silent startup exits, `ERR_PNPM_*`), when explaining what a key or slash command does (the full keymap, `/settings`, session manager, `/resume`, `/compact`, `/rewind`, `/fork`, `/tree`, `/btw`, `/migrate`, `/preset`, `/model`, `/provider`, `/permission`), when editing `$DSH_HOME/profiles/dsh-tui/cordis.patch.yml` or any `DSH_TUI_*` option, when authoring or debugging themes under `~/.dsh-tui/themes/`, when importing sessions from Claude Code / Codex / OMP / zcode / Grok Build, when a session shows as held by another terminal or the TUI misaligns, and when reasoning about the runtime (Cordis mount chain, session projections, the multi-TUI mount ledger, plugin seams) or the VS Code companion extension `dsh-tui-vscode`.
---

# dsh-TUI

dsh-TUI is an **interactive terminal UI plugin for DeepSeek Harness** — not a separate agent and not a
Harness fork. It mounts into a DSH profile as a pure plugin (with no core changes), ships a pixel-whale
header, streaming Markdown, tool cards, `/` and `@` completion, image previews, mermaid diagrams, a
timeline rail, live context/TPS meters, one session manager for every session, and a thin VS Code
companion extension. Package `@deepseek-harness-tui/dsh-tui`; commands `dsh-tui` and the alias `dst`;
profile name `dsh-tui`.

Two consequences shape everything:

1. **It is a front end.** Conversation truth lives in the DSH session log and effective capability comes
   from the mounted DSH services. The TUI owns rendering, keys and session surfacing — not the sandbox,
   the model route or the tool set.
2. **Configuration is profile composition.** The real knobs are rows in
   `$DSH_HOME/profiles/dsh-tui/cordis.patch.yml`; `/settings` is a friendly editor for a subset of them.

Docs snapshot for this skill: `ccch1mneyyy/dsh-TUI` **`main @ bbad0307706462220d9971c45e502bcbb2aeb2a5`**
(2026-09-27, release `0.11.1`); the English pages under `docs/*.en.md` are the primary source and the
Chinese originals are cross-checked where they add or contradict facts. Verified on this machine against
launcher `0.11.0` and profile `0.11.1` on DSH `0.1.7-rc.1`.

## Read the right reference

All paths are relative to this skill's base directory. Read the file **before** making non-trivial
changes to a dsh-TUI install.

| Task / question | Read |
|---|---|
| Install, launch, profile lifecycle, CLI subcommands, safe mode, updating, source builds, pnpm pitfalls | `references/getting-started.md` |
| Which key does what — keys, vim mode, mouse, completion, questionnaires/approvals, mode badges | `references/keys-and-editing.md` |
| Every slash command with its arguments, effect and persistence; local vs registry vs placeholder | `references/commands.md` |
| Sessions, workspace rail, rewind, fork/tree, `/btw`, backgrounding, session storage, `/migrate` | `references/sessions.md` |
| `cordis.patch.yml`, every TUI config field, `/settings`, env vars, presets, MCP, `/provider` | `references/configuration.md` |
| Built-in themes, `auto`/OSC 11, custom theme JSON and keys, plugin themes | `references/themes.md` |
| VS Code extension and selection channel, Herdr, editor injection, image/terminal capability | `references/integrations.md` |
| Runtime architecture, performance model, multi-TUI mount ledger, security boundary, plugin seams | `references/architecture.md` |
| A broken install or a misbehaving TUI: symptom → cause → fix, compatibility matrix, diagnostics | `references/troubleshooting.md` |
| Hard limits, "do not do X", configuration traps, security notes | `references/limitations.md` |

## Golden rules

- **Edit the profile patch, not the package.** User configuration lives in
  `$DSH_HOME/profiles/dsh-tui/cordis.patch.yml` (a top-level YAML array; `!!js` works). Override an
  existing row **by `id`**; use `insert` only for a genuinely new service.
- **A row override replaces its whole `config` block** — it is not deep-merged. Repeat every key you
  want to keep.
- **On DSH 0.1.7 `/settings` writes that same profile file.** `~/.dsh/settings.yaml` is the legacy user
  layer, not the settings entry point.
- **Prefer leaving `cwd` unset** (it resolves to the git worktree root containing the launch directory).
  Never use `!!js process.cwd()` — it pins the workspace to the launch subdirectory (issue #96).
- **`/restart` is required for `fullscreen` and `terminalImages`**; most other preferences apply live and
  `/reload` re-reads preference files.
- **Never add `dsh-working-activity` to the profile separately.** It ships as a dependency and is
  inserted by the dsh-tui patch; adding it again mounts a duplicate activity row.
- **`/preset` is blank-session-only.** Once a conversation has messages, the choice becomes the default
  for the next `/new` instead of switching in place.
- **Unknown slash commands are sent to the model as ordinary messages**, not rejected.
- **Two directories, never interchangeable:** `$DSH_HOME` (`~/.dsh`) holds profiles, sessions and
  credentials; `~/.dsh-tui` holds TUI preferences, themes, pins and the session-mount ledger.
- **One append-only log per session.** A session held by another live terminal is red with
  `held by pid <pid>` and cannot be entered — do not try to work around it, wait for that process to exit.

## Quick start

```sh
# install (the launcher auto-initializes the profile on first run; pnpm 10+ required)
npm install -g @deepseek-ai/dsh @deepseek-harness-tui/dsh-tui
dsh-tui                       # or: dst, or dsh --profile dsh-tui

# environment and repair
dsh-tui doctor                # environment / profile / key checks
dsh-tui safe                  # read-only diagnostics after an unexpected exit
dsh-tui update                # or in-TUI /update (profile only)

# update without @latest can stay pinned to the recorded range
dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui@latest
```

Inside the TUI: `/` for commands, `?` for help, `Shift+Tab` for modes, `Esc` `Esc` to rewind,
`Ctrl+O` for full tool/reasoning detail, `/resume` for the session manager, `/settings` for the
preference editor.

## Getting the facts right

- **Command surface:** the authoritative list is `src/commands.ts` (`LOCAL_COMMANDS`, 57 entries plus
  aliases and one hidden command) merged at runtime with DSH registry commands such as `/plan`, `/goal`,
  `/permission` and `/feedback`. `references/commands.md` states which is which.
- **Docs vs source:** where the English and Chinese pages disagree, or where a claim looks stale, the
  skill states both and cites the pinned source. Each reference file ends with a short
  "Divergences and gaps in the docs" note.
- **Not part of dsh-TUI:** `/find` belongs to the third-party `dsh-tui-find` plugin; the VS Code
  extension is versioned separately; authoring *new* plugins is governed by the external
  [plugin admission and development guide](https://github.com/T-Auto/dsh-ecosystem-spec/blob/main/docs/plugin-admission-and-development.md).
- **Sensitive data:** `DSH_TUI_RENDER_LOG` frames and session exports can contain prompts, tool
  arguments and output. Review and redact before sharing anywhere.
