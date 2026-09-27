# Limits, prohibitions and security boundary

Sources: [`docs/getting-started.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/getting-started.en.md),
[`docs/configuration.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/configuration.en.md),
[`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md),
[`docs/themes.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/themes.en.md),
[`docs/migrate.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/migrate.en.md), and
[`docs/architecture.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/architecture.en.md).

## Do not do these

| Don't | Why |
|---|---|
| Add `dsh-working-activity` separately to the `dsh-tui` profile | It already ships as a dependency and is inserted by the dsh-tui patch — duplicate activity rows may mount |
| Insert a second `working-activity` row | Override the existing id only |
| Insert a duplicate `user-interaction` row in the profile patch | It normally comes from `dsh-base`; the plugin only creates a fallback in a bare composition |
| Add both the old `dsh-cc-tui` and the current package to one profile | They are the same product under different names |
| Use `!!js process.cwd()` for `cwd` | It pins the workspace to the launch subdirectory (issue #96). Leave `cwd` unset or use an absolute path |
| Treat `~/.dsh/settings.yaml` as the settings entry point on DSH 0.1.7 | `/settings` writes the profile's `cordis.patch.yml`; the old user layer is legacy |
| Copy the root `cordis.yml` into the profile | It is a bare-composition example and may have a different service topology |
| Run `dsh-tui` with stdout redirected / without an interactive TTY | The TUI cannot render, and the explicit launcher then errors out |
| Attach `DSH_TUI_RENDER_LOG` captures to a public issue unreviewed | Frames may contain visible prompts, tool arguments and output |
| Commit a real credential | `DEEPSEEK_API_KEY` should come from the environment; never persist it in the repo |
| Rely on Git URL installs | Workspace dependencies, a git submodule and the pnpm ≥11 `prepare` allowlist all block it |
| Insert a subagent provider before mounting the core `subagent` service, or override `plan-mode` with an empty `section` | Both are composition contract requirements |

## Configuration traps

- Overriding a row **replaces the whole `config` block** — it is not deep-merged, so repeat every key
  that must stay active.
- The session-side `cwd` and the bash/fs-policy/sandbox roots may differ; the latter belong to the
  composition layer (default: the launch directory, governed by `dsh-base`).
- An invalid `effort` level **silently falls back** to the adapter default.
- `contextBar` renders only when **both** the config value and `/settings → statusBar.contextBar` are on.
- `fullscreen` and `terminalImages` changes need `/restart`; `/reload` does not apply them, and a running
  turn must finish (or be stopped with `Ctrl+C`) first.
- `DSH_TUI_IMAGE_PROTOCOL` does **not** bypass the preview preference, the disable switch,
  non-fullscreen, accessibility or multiplexer guards.
- The legacy `activityFrames` value `claude` reads as `moon8` and is no longer offered in the picker.
- Older `dsh-working-activity` (< 0.5.0) produces no working line at all.
- `statusBar.sessionId` is a display toggle, unrelated to the startup `sessionId`.
- Without `@latest`, updates may stay pinned to the range recorded in the profile `package.json`.
- A pnpm that does not support `ignoredOptionalDependencies` silently loses the sharp download saving.
- The safe-mode prompt appears only in interactive terminals, covers only a non-zero exit of the final
  dsh child (not a startup hang), and never silently deletes files.
- Every dsh launch writes `$DSH_HOME/profiles/node_modules` fallback links and the pnpm global store —
  that is not introduced by safe mode.
- `dsh-tui help` and `dsh-tui version` work without dsh or an initialized profile; every other argument
  is forwarded verbatim to `dsh --profile dsh-tui`.

## Session and UI limits

- `/compact`, `/model`, `/restart` and `/update` are **rejected mid-turn**.
- `/compact` is **unavailable under the `minimal` preset** (bash and editor only, no compaction).
- `/preset` cannot switch a session that already has messages; the choice becomes the default for the
  next `/new` or the next launch.
- Rewind cannot go past the first message, waits up to 30 s for a running turn to settle, and always
  forks a new session.
- The session the current terminal is attached to cannot be stopped from the session manager (only
  background sessions, with `Ctrl+X`).
- Removing a workspace from the rail removes the **entry only**; its directory and session logs survive.
- An unknown slash command is sent to the model as an ordinary message, not rejected.
- `/btw` is a single tool-less turn, is not written to history, and counts no tokens.
- Plan review approval must carry **no** feedback text; the questionnaire's last row is a free-input
  line; an approval always wins over a questionnaire.
- With `/jobs` open, `Esc` closes only that panel — close it before pressing `Ctrl+C` to interrupt.
- Thinking lines are **not** subject to the long-line fold.
- Disabling terminal images (config, `/settings`, or `DSH_TUI_DISABLE_TERMINAL_IMAGES=1`) never affects
  **sending** images to the model; tmux/screen, non-TTY and accessibility sessions behave the same way.
- The cost figure is an estimate (official models only) and the platform bill is authoritative; DSH
  provides no cost metering.
- `pageMargin` custom values cap at 8x4.
- Safe mode is read-only and only lists repair commands — you run them yourself.
- dsh-TUI ships no generic skills; `/skills` browses what DSH discovers.
- Theme names `auto`, the built-in names and `status` are reserved; a user theme named `auto` is
  shadowed by the pseudo-theme. Theme colors must be concrete values (no CSS variables, gradients or
  arbitrary CSS names) and a file with an invalid `base`, malformed JSON or a non-object `colors` is
  skipped as a whole.
- Migration never writes to the foreign agent's store, never migrates tool traffic or per-message
  original timestamps, skips files over 64 MB and malformed lines, and reports failures with exit
  code 1 without aborting the batch.
- "migrate" in dsh-TUI is the conversation importer, not the `dsh-cc-tui` package rename.

## Security boundary

- **No separate sandbox.** dsh-TUI runs inside the DSH composition and inherits the host's permission
  and sandbox policy. The documented non-Windows default policy is `workspace-write`; on Windows the
  composition's default is `danger-full-access` with approval `never` unless the composition overrides
  it, and `DSH_PERMISSION_MODE` overrides the non-Windows policy. (Platform defaults are stated only in
  the Chinese `architecture.md`; treat them as composition-owned, not TUI-owned.)
- The tool-approval waterfall offers **allow-once or deny only** — no "always allow" and no feedback
  channel (Chinese `architecture.md` only).
- Credentials: `/provider` writes only non-environment keys to `~/.dsh/.credentials.yaml` (mode `0600`),
  renders them masked, and keeps a key shared with another provider when one is deleted.
  `dsh-tui doctor` reports whether the API key is set — **state only, never the value**.
  `pnpm dev:copy-config` copies credentials into the isolated dev root (`0600` on Unix, OS-managed ACL
  on Windows).
- `DSH_TUI_RENDER_LOG` output is sensitive by nature. Review and redact before sharing.
- Anything the model can read is still governed by the host's file policy and the active preset; the
  TUI adds no separate access path. What the TUI sends to the model is the conversation plus the
  context the host composes.

## Divergences and gaps in the docs

- The Chinese `architecture.md` documents the `ctx.approval` seam and the allow-once/deny waterfall;
  the English page reduces this to "tool-level approval UI".
- `architecture.md:141` alone states the inline/fullscreen factory default; `src/index.ts` confirms
  `fullscreen` and `terminalImages` both default to `true`.
- The Git URL error codes appear only in the Chinese `getting-started.md`.
- Neither language documents a complete profile uninstall procedure.
