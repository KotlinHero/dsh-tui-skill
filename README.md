# dsh-tui-skill

A standardized skill built from the documentation of
[**dsh-TUI**](https://github.com/ccch1mneyyy/dsh-TUI) — the interactive terminal UI plugin for
DeepSeek Harness (`@deepseek-harness-tui/dsh-tui`, commands `dsh-tui` / `dst`, profile `dsh-tui`).

This skill helps an AI agent — or any reader — install, operate, configure, theme, repair and reason
about dsh-TUI accurately, by distilling the project's `docs/` set into a compact, navigable reference.

## Contents

- `SKILL.md` — the entry point: what dsh-TUI is, the mental model, a routing table, the golden rules,
  and a quick start.
- `references/` — condensed knowledge per subject area:
  - `getting-started.md` — prerequisites, both install paths, profile lifecycle, CLI subcommands,
    on-disk layout, updating, safe mode, source builds, pnpm pitfalls.
  - `keys-and-editing.md` — the complete keymap by context, Esc/`Ctrl+C` ladders, vim mode, mouse,
    `/`+`@` completion, questionnaires and approvals, mode badges, terminal requirements.
  - `commands.md` — every slash command with arguments, effect and persistence, plus which commands are
    local, registry-provided or placeholders, and the CLI subcommands.
  - `sessions.md` — session lifecycle, the session manager, rewind, fork/tree, `/btw`, background
    sessions, storage, and `/migrate` from Claude Code / Codex / OMP / zcode / Grok Build.
  - `configuration.md` — `cordis.patch.yml`, every TUI config field, `/settings`, environment variables,
    agent presets, MCP, `/provider`, composition constraints.
  - `themes.md` — built-in palettes, `auto` and OSC 11 detection, custom theme JSON with the full color
    key list, color formats, validation, runtime plugin themes.
  - `integrations.md` — the VS Code companion extension and IDE selection channel, the built-in terminal
    paths, Herdr, the external injection channel, and image/terminal capability.
  - `architecture.md` — runtime mount chain, event-driven projection, the long-session performance
    model, the multi-TUI session-mount ledger, the security boundary, and the plugin seam tiers.
  - `troubleshooting.md` — symptom → cause → exact fix, a compatibility matrix, and the diagnostic
    environment variables.
  - `limitations.md` — hard limits, explicit prohibitions, configuration traps and security notes.

## Sources

Built from the English (`.en.md`) and Chinese documentation under
[`ccch1mneyyy/dsh-TUI/docs`](https://github.com/ccch1mneyyy/dsh-TUI/tree/main/docs), cross-checked
against the repository source where the two language versions disagree or a documented claim is stale.

- Docs snapshot: `main @ bbad0307706462220d9971c45e502bcbb2aeb2a5` (2026-09-27)
- Upstream release at snapshot time: `@deepseek-harness-tui/dsh-tui` `0.11.1`
- Verified locally against launcher `0.11.0`, profile `0.11.1`, DSH `0.1.7-rc.1`

Where a fact appears in only one language version, or the versions conflict, the reference file names
the source language in its "Divergences and gaps in the docs" section rather than silently choosing one.
Contributor-facing docs (`contributing`, `roadmap`, `community-management`, the safe-mode design drafts)
are deliberately out of scope.
