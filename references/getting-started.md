# Installation, launch and profile lifecycle

Sources: [`docs/getting-started.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/getting-started.en.md),
[`docs/configuration.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/configuration.en.md),
[`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md), and the
Chinese originals `docs/getting-started.md`, which carry two facts the English page omits (marked ZH-only below).

## Prerequisites

| Requirement | Detail |
|---|---|
| Node.js | `^22.19 \|\| >=24` (CI uses Node 24) |
| DSH CLI | `@deepseek-ai/dsh`, installed globally |
| pnpm | **10 or newer** (CI uses 11). `dsh plugin` delegates profile installation to pnpm; pnpm 9 hoists transitive dependencies differently, leaves `dsh-working-activity` unresolvable inside the profile, and the TUI exits right after startup with almost no error output (issue #60) |
| Terminal | An interactive TTY. `dsh-tui` cannot start with stdout redirected |
| Credentials | `DEEPSEEK_API_KEY`; set `DEEPSEEK_BASE_URL` too when using a compatible custom endpoint |

Never commit a real credential. A normal profile launch reads the environment variable directly.

```powershell
$env:DEEPSEEK_API_KEY = 'your-key'
```

## Install

Two equivalent paths. The **global path (ZH-only in the docs: `getting-started.md`)** installs the
launcher, which auto-initializes the profile on first run:

```sh
npm install -g @deepseek-ai/dsh @deepseek-harness-tui/dsh-tui
# pnpm if missing: npm install -g pnpm   (or: corepack enable pnpm)
dsh-tui
```

The **manual path** does the profile step explicitly:

```sh
npm install -g @deepseek-ai/dsh
npm install -g pnpm                       # or: corepack enable pnpm
dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui
dsh --profile dsh-tui                     # equivalent to `dsh-tui`
```

From a repository checkout, `sh install.sh` only checks for `dsh` and `pnpm` and then runs the profile
plugin command — it copies no source files and needs no local build.

### What the first profile install does

1. Initializes `$DSH_HOME/profiles/dsh-tui/`. When `DSH_HOME` is unset the root is normally `~/.dsh`.
2. Uses `@deepseek-ai/dsh-base` as the first profile bundle.
3. Installs `@deepseek-harness-tui/dsh-tui` inside the profile with pnpm.
4. Reads the package's `dsh.bundle.patch` metadata and adds its `cordis.patch.yml` as a composition layer.

Startup order matters when diagnosing a broken composition:

```text
dsh-base -> other bundles -> @deepseek-harness-tui/dsh-tui patch -> user profile patch
```

`dsh-working-activity` is already a dependency of the package and is inserted by the `dsh-tui` patch.
**Do not add `dsh-working-activity` separately to the same profile** or the activity row may mount twice.

## Start the TUI

```sh
dsh --profile dsh-tui      # or: dsh-tui   /   dst
```

The process starts in the current directory, which is also the Agent's default workspace — change into
the target project before starting it. `dsh-tui <path|URL>` starts with a specific workspace target.

On Windows the checkout also ships:

```bat
dsh-tui.cmd
dsh-tui.cmd --resume
```

- `--resume` reads `%USERPROFILE%\.dsh-tui\resume.txt` and restores the session last selected by the TUI.
- `DSH_TUI_WORKSPACE` overrides the working directory used by the batch launcher.

On first normal launch (no `--resume`, no workspace, no prompt) the TUI opens the **session manager**
so you can pick a workspace; `~/.dsh-tui/home.json` records that it has been seen so later launches go
straight to chat.

## CLI subcommands

`dsh-tui help` (or `dst help`) prints the full usage; the `dst` alias accepts the same commands.

| Command | Purpose |
|---|---|
| `dsh-tui update` | Update the profile to the latest release and align the launcher (same install logic as `/update`, without restarting into the TUI) |
| `dsh-tui doctor` | Environment checks: dsh/pnpm, profile install and version alignment, API-key presence (state only, never the value), config file presence |
| `dsh-tui safe` | Safe mode: read-only diagnostics, inventory, repair guidance |
| `dsh-tui safe --rescue` | Additionally create/verify the clean rescue profile (exit 0 when ready, 1 when refused) |
| `dsh-tui version` | Launcher and profile versions (`--version` / `-v` equivalent) |
| `dsh-tui help` | Usage (`--help` / `-h` equivalent) |
| `dsh-tui --resume [id]`, `-c`, `--continue` | Resume the last (or a specific) session |
| `dsh-tui <path\|URL>` | Start with that workspace target |

`help` and `version` work even when dsh is missing or the profile is not initialized; **every other
argument is forwarded verbatim to `dsh --profile dsh-tui`**.

## On-disk locations

The skill keeps these two roots distinct — neither substitutes for the other:

| Path | Holds |
|---|---|
| `$DSH_HOME` (default `~/.dsh`) | profiles, sessions, credentials, attachments |
| `$DSH_HOME/profiles/dsh-tui/` | the installed profile: `package.json`, `cordis.yml`, `cordis.patch.yml`, `pnpm-lock.yaml`, `node_modules` |
| `$DSH_HOME/profiles/dsh-tui/cordis.patch.yml` | **the user override file** — edit this by hand for anything `/settings` does not expose |
| `$DSH_HOME/profiles/dsh-tui-safe/` | rescue profile created by `dsh-tui safe --rescue` |
| `$DSH_HOME/sessions/` | shared JSONL session store (`DSH_TUI_SESSION_ROOT` overrides; a bare `cordis.yml` composition defaults to `~/.dsh-tui/sessions/`) |
| `$DSH_HOME/.credentials.yaml` | API keys written by `/provider` (mode `0600`) |
| `~/.dsh-tui/` | **TUI data directory, independent of `$DSH_HOME`**: theme/model/preset/effort/language preferences, `resume.txt`, `session-pins.json`, `session-mounts.json`, `themes/` |

## Update

```sh
# profile only (also available in-TUI as /update, which restarts and resumes)
dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui@latest
```

- **Without `@latest`**, pnpm resolves inside the version range already recorded in the profile's
  `package.json` (for example `^0.1.4`), so re-running the plain install command can appear to change
  nothing. That is the usual cause of "the update did nothing".
- The startup banner shows the running version (`✦ dsh-TUI vX.Y.Z`).
- Your `cordis.patch.yml` override layer survives updates untouched.
- The in-TUI `/update` updates the **profile**; when you launched through the global `dsh-tui`
  command, align the launcher as well (ZH-only guidance in `getting-started.md`):
  `npm install -g @deepseek-harness-tui/dsh-tui@latest` (or `pnpm add -g …`).
- Session storage has moved between versions (since 0.3.7 `/resume` uses the JSONL store shared with
  dsh web). Older sessions missing from the list after a major update is expected; the data is not deleted.

### pnpm install-script blocks and foreign-platform natives

If `dsh plugin` fails with `ERR_PNPM_IGNORED_BUILDS` (pnpm ≥11 blocks dependencies that carry install
scripts by default, e.g. `@google/genai` and `protobufjs` — none of those scripts is needed at
runtime), add to the profile's `pnpm-workspace.yaml`:

```yaml
allowBuilds:
  '@google/genai': false
  protobufjs: false
```

`/update` and `dsh-tui update` seed this automatically. Updates also maintain
`ignoredOptionalDependencies` for foreign-platform `@img/sharp-*` natives (about 200 MB of downloads
avoided); the list is recomputed for the running platform on every update, and entries a user added
outside those platform tables are left alone.

## Safe mode

When dsh exits unexpectedly, safe mode provides read-only environment diagnostics, a profile plugin
inventory and repair guidance.

- **Two entries**: run `dsh-tui safe` manually, or accept the prompt shown after dsh exits with a
  non-zero code. The prompt appears only in interactive terminals; scripts and pipes get a single hint
  line and keep the exit code. It covers only a non-zero exit of the final dsh child process, not a
  startup hang (a spawn failure counts as exit code 1).
- **Read-only**, with two exceptions: retry normal startup, and create/reuse the rescue profile, which
  writes only under `$DSH_HOME/profiles/dsh-tui-safe/`. (Every dsh launch also writes
  `$DSH_HOME/profiles/node_modules` fallback links and the pnpm global store — not introduced by safe mode.)
- **The rescue profile must prove clean or it refuses to start**: the candidate directory must be a
  recognizable profile; the existing profile's root manifest must not declare third-party plugins;
  `$DSH_HOME/cordis.patch.yml` (home layer) must not exist; `dsh-tui-safe/cordis.patch.yml` must have no
  entries (a default "comments + `[]`" does not count). Before deleting or rebuilding it, safe mode
  checks the top-level entries by name and shape and lists anything unexpected — it never silently
  deletes files.
- **Repair commands are listed, not run**:
  - `dsh plugin --profile dsh-tui remove <third-party plugin>`
  - `dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui@<version>`
  - `dsh-tui doctor`
- With an unreadable or too-old profile copy, upgrade the launcher first:
  `npm install -g --legacy-peer-deps @deepseek-harness-tui/dsh-tui@<version>`.

## Migrating from the former `dsh-cc-tui` package

Earlier releases used the unscoped `dsh-cc-tui` package, a `cc-tui` profile, `CC_TUI_*` / `DSH_CC_*`
environment variables and a `~/.dsh-cc` data directory. The current release **no longer reads the old
names and does not migrate data automatically**:

```sh
dsh plugin --profile dsh-tui add @deepseek-harness-tui/dsh-tui
dsh --profile dsh-tui
```

Then copy themes, configuration and history files from `~/.dsh-cc` into `~/.dsh-tui` yourself. Once the
new profile works, `$DSH_HOME/profiles/cc-tui` and the old data directory are leftovers. **Do not add
both packages to the same profile.** This is a different operation from `/migrate`, which imports
conversation history from other coding agents (`sessions.md`).

## Remove / uninstall

The docs give no full profile-removal procedure. What is documented is per-plugin removal inside a
profile:

```sh
dsh plugin --profile dsh-tui remove <plugin>
```

## Develop from source

```sh
git clone --recurse-submodules https://github.com/ccch1mneyyy/dsh-TUI.git
cd dsh-TUI
pnpm install --frozen-lockfile
pnpm build
pnpm smoke
```

- Three submodules exist and two are required to install: `vendor/dsh-std` (its `packages/*` are
  workspace packages in `pnpm-workspace.yaml`) and `dsh-auth` (pulled in via `link:`). Without
  `--recurse-submodules`, `pnpm install --frozen-lockfile` fails; fix an existing clone with
  `git submodule update --init --recursive`.
- `pnpm build` cleans `lib/`, compiles `src/` into `lib/types/` and runs the build gates.
- **Git URL installs are not supported** (workspace deps / submodule / pnpm ≥11 prepare allowlist);
  the ZH troubleshooting section adds the codes `ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED` and
  `ERR_PNPM_WORKSPACE_PKG_NOT_FOUND`.
- After the first setup or a model/key change: `pnpm dev:copy-config` (copies only
  `~/.dsh/settings.yaml` and `~/.dsh/.credentials.yaml`; mode `0600` on Unix). Then `pnpm dev` builds,
  packs, installs in isolation and launches, using isolated `HOME`, `DSH_HOME` and session directories
  under `$XDG_CACHE_HOME/dsh-tui-dev` (Unix) or `%LOCALAPPDATA%\dsh-tui-dev` (Windows), overridable with
  `DSH_TUI_DEV_ROOT`. `pnpm dev:test` verifies build/pack/install without launching.
- `pnpm tui` runs `scripts/run.ts`, which expects a Harness monorepo `packages/*` layout by default;
  a standalone checkout must set `DSH_TUI_DEV_WORKSPACE` to the Harness root. Prefer `pnpm dev` to test
  only this repository.

## Divergences and gaps in the docs

- **ZH-only** (`getting-started.md`): the global install path, the launcher-alignment update steps, and
  the "in-TUI `/update` only updates the profile" note. `user-guide.en.md` corroborates the global path.
- **EN-only** (`configuration.en.md`): `~/.dsh-tui/model.json` surviving restart and `/new`.
- No supported-DSH-host version range exists in these files beyond "DSH 0.1.7" vs "older hosts"; the
  verified engine is `0.1.7-rc.2` and `ADAPTER.md` in the repository is the authoritative compatibility list.
- A gap, not a divergence: the docs describe no complete profile uninstall.
