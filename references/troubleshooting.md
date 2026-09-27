# Troubleshooting

Sources: [`docs/getting-started.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/getting-started.en.md)
(§Troubleshooting), [`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md)
(tips), [`docs/migrate.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/migrate.en.md), and
[`docs/configuration.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/configuration.en.md).

Start with `dsh-tui doctor` (environment, profile install, version alignment, API-key presence, config
files), then `/doctor` and `/status` inside the running TUI. When dsh exits unexpectedly, run
`dsh-tui safe` for read-only diagnostics, a profile plugin inventory and repair guidance.

## Symptom → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| `dsh-tui requires an interactive terminal` | stdout is not a TTY **and** the process was not started by the `dsh-tui` launcher. When both hold, the TUI frontend is skipped **silently** (no error; the host keeps booting) — the normal case when dsh-tui is only installed in a profile and a non-terminal host (Web/Tauri/GUI, piped or null stdout) starts the DSH composition | Start it directly in a terminal; do not redirect its main output. The error itself appears only when `dsh-tui` (or the standalone portable build) was explicitly launched without a TTY |
| `dsh` or `pnpm` cannot be found | the global npm bin directory is not on `PATH` | Put it on `PATH` and open a new terminal. `install.sh` checks both commands before installing |
| The TUI exits straight back to the shell with almost no error | the profile was installed by **pnpm 9**: the transitive `dsh-working-activity` is not hoisted where the loader can resolve it, the failed module resolution tears down the whole plugin tree, and the TUI prints the resume hint and exits (issue #60) | `npm install -g pnpm@latest`, then `dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui@latest` |
| `ERR_PNPM_IGNORED_BUILDS` during `dsh plugin` install | pnpm ≥11 blocks dependencies that carry install scripts by default (`@google/genai`, `protobufjs`); none of those scripts is needed at runtime | Add to the profile's `pnpm-workspace.yaml`: `allowBuilds: { '@google/genai': false, protobufjs: false }`. `/update` and `dsh-tui update` seed this automatically |
| Git URL install fails (`ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED`, `ERR_PNPM_WORKSPACE_PKG_NOT_FOUND` — **documented only in the Chinese page**) | Threefold block: `@dsh-std/*` are workspace dependencies, `vendor/dsh-std` is a git submodule, and pnpm ≥11 refuses `prepare` build scripts for git dependencies | Install the registry package instead: `dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui` |
| The model reports missing credentials | `DEEPSEEK_API_KEY` is not visible to the process | Confirm it is set in the **same shell** that starts `dsh`; also check `DEEPSEEK_BASE_URL` when using a custom endpoint |
| The activity row appears twice | `dsh-working-activity` was added separately to the profile | Keep the row inserted by the dsh-tui patch and remove the duplicate bundle entry |
| The TUI is misaligned or leaves terminal state behind | rendering or terminal-state bug | Run `/doctor`, record the terminal and mode, consult `keys-and-editing.md` and `architecture.md`. `DSH_TUI_RENDER_LOG` captures raw frames — **handle them as sensitive data** |
| "Re-running the install command changes nothing" | Without `@latest`, pnpm resolves within the version range already recorded in the profile's `package.json` (e.g. `^0.1.4`) | `dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui@latest`; confirm with the startup banner (`✦ dsh-TUI vX.Y.Z`) |
| Version-drift warning on the logo page | The dsh host is outside the verified compatibility range | Align it: `npm i -g @deepseek-ai/dsh@<version>` |
| An old global launcher cannot read the profile copy ("outdated launcher") | The launcher predates the profile format | `npm install -g --legacy-peer-deps @deepseek-harness-tui/dsh-tui@<version>` |
| `pnpm install --frozen-lockfile` fails on a fresh checkout | Submodules are empty | `git submodule update --init --recursive`, or re-clone with `--recurse-submodules` |
| `pnpm dev` / `pnpm tui` cannot find Harness packages | `scripts/run.ts` assumes a Harness monorepo `packages/*` layout | Set `DSH_TUI_DEV_WORKSPACE` to the Harness root, or prefer `pnpm dev` |
| A session in the manager is red with `held by pid <pid>` | Another TUI process holds that append-only session log | Wait for that process to exit; the row becomes enterable on its own (occupancy is re-read every 2 seconds) |
| Interrupt will not settle | The turn refuses to cancel | Press `Ctrl+C` / `Ctrl+D` again to force quit |
| Mid-turn commands do nothing | `/compact`, `/model`, `/restart` and `/update` are rejected while a turn runs | Press `Ctrl+C` first or wait for the turn to end |
| `/preset` will not change | The session already has messages (blank-only rule) | Start a `/new` session to use the new preset |
| `/update` reports "unavailable" | It only works when launched through `dsh --profile dsh-tui`; a source checkout gets a notice instead | Update through the profile command or `dsh-tui update` |
| Context pressure is amber/red | Context is ≥80% / ≥95% full | `/compact` (not available under the `minimal` preset) |
| Stale panels after resuming | The UI re-folds facts from the session log on resume | `Ctrl+L` reloads the session manager; the subagent panel re-folds on resume |
| Migration reports `unknown agent` | Wrong source name | The authoritative source list is what bare `dsh-tui migrate` prints |
| Migration reports `needs the profile's compiled copy` | The profile's compiled output is missing or too old | Run `dsh-tui update` first |
| Migration imports fewer files than the scan counted | The scan counts candidate files by name; the import additionally filters unreadable and empty conversations | A slightly lower number is expected |
| Migration finds nothing | The foreign agent's data directory is missing under the current home | Confirm the directory exists; set `GROK_HOME` when grok-build lives elsewhere |
| A fresh `DSH_HOME` first run reports **installation rejected** | Profile bootstrap hit a stale npm tarball | Upgrading dsh heals it, or use an existing profile meanwhile |

## Compatibility notes

| Item | Requirement |
|---|---|
| Node.js | `^22.19 \|\| >=24` (CI uses Node 24) |
| pnpm for profile installs | **10 or newer** (CI uses 11). pnpm 9 is incompatible — see the silent-exit row above |
| pnpm ≥ 11 | Blocks install-script dependencies by default → `ERR_PNPM_IGNORED_BUILDS` |
| DSH host | 0.1.7 is the current target (`/settings` writes the profile's `cordis.patch.yml`). Older hosts use `~/.dsh/settings.yaml`, `@deepseek-ai/dsh-agent-presets`, and directory presets under `$DSH_HOME/.agent-presets/` |
| `dsh-working-activity` | ≥ 0.5.0 required for the working line (the `workingActivity` projection replaced the old `activity/status` event outlet) |
| Preset ids | `ptc` (0.1.2) / `code` (legacy 0.1.1); both resolve across versions, and a legacy `code` preference resolves to `ptc` when the roster lacks `code` |
| Since 0.3 | Tools, planning, compaction and delegation are owned by the preset; `DSH_TUI_COMPACT_RATIO` / `DSH_TUI_COMPACT_RETAIN` and the old subagent-depth knobs are unused |
| Since 0.3.7 | `/resume` uses the JSONL session store shared with dsh web |
| Since 0.9.0 | `fullscreen` factory default is `true` (confirmed in `src/index.ts`) |
| Git URL installs | Not supported |

The verified engine version is `0.1.7-rc.2`; `ADAPTER.md` in the repository is the authoritative
compatibility list. The four primary docs state no `@deepseek-ai/schemastery` requirement and no
explicit DSH version range beyond "0.1.7" vs "older hosts".

## Diagnostic environment variables

All off by default; enable only what the issue needs.

| Variable | Purpose |
|---|---|
| `DSH_TUI_DEBUG=1` | dsh-tui diagnostics on stderr |
| `DSH_TUI_RENDER_LOG=<path>` | Raw ANSI frame capture. **May contain visible prompts, tool arguments and output — redact before sharing** |
| `DSH_TUI_DEBUG_REPAINTS=1` | Record repaint diagnostics |
| `DSH_TUI_COMMIT_LOG=1` | Record render-commit diagnostics |
| `DSH_TUI_ACCESSIBILITY=1` | Accessibility display paths (no animation, text-only images) |
| `DSH_TUI_TMUX_TRUECOLOR=1` | Enable the truecolor detection path in tmux |
| `DSH_TUI_TAB_STATUS=1` | Experimental terminal tab-status opt-in; no guarantee in every terminal |

Diagnostic output never changes session events or model routing.

## Reporting a rendering bug

1. Run `/doctor` and note the terminal, mode (`fullscreen` on/off) and version.
2. Consult `keys-and-editing.md` and `architecture.md` — several "bugs" are documented limitations
   (inline mode has no mouse reporting, tmux/screen falls back to text images, narrow terminals hide
   the whale below 64 columns).
3. Only then capture frames with `DSH_TUI_RENDER_LOG`, and review/redact before attaching them anywhere.
