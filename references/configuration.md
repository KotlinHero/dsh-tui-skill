# Configuration

Sources: [`docs/configuration.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/configuration.en.md) and
[`docs/user-guide.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/user-guide.en.md)
(§Settings), plus `docs/getting-started.en.md`.

## Where configuration lives

```text
$DSH_HOME/profiles/dsh-tui/cordis.patch.yml
```

`DSH_HOME` defaults to `~/.dsh`. The file is a top-level YAML array and may use the `!!js` expressions
DSH supports.

Profile startup layers, in order:

1. `dsh-base`
2. Installed bundles
3. The package's own `cordis.patch.yml`
4. **The user patch (applied last)**

A user configuration normally overrides an existing row **by `id`**; use `insert` only for a genuinely
new service.

> **`config` blocks are replaced, not deep-merged.** When you override a row you must repeat every key
> you want to keep active.

Two settings surfaces:

- **`/settings`** (in-TUI editor) writes plugin Config fields to the active profile's
  `cordis.patch.yml` on DSH 0.1.7. Older hosts still use `~/.dsh/settings.yaml`; that file is not the
  new settings entry point. Language and layout preferences update live; **`fullscreen` and
  `terminalImages` need `/restart`**. Namespaces not declared as TUI blocks are listed read-only.
- **Hand-edited profile config** for the startup-level keys that `/settings` does not expose:
  `provider`, `model`, `cwd`, `workspace`, `preset`, `sessionId`, `modes`, and `effort`.

The root `cordis.yml` in the repository is a bare-composition example. A normal profile installation
uses `cordis.patch.yml` — **do not copy the root configuration into the profile**.

## Complete override example

```yaml
- id: dsh-tui
  config:
    provider: deepseek-official
    model: deepseek-flash
    # Prefer leaving cwd unset — the default resolves to the git worktree root
    # containing the launch directory. To pin a workspace use an absolute path
    # (e.g. cwd: /repo/packages/app), NOT `!!js process.cwd()` (issue #96).
    effort: max
    activity: true
    activityFrames: moon8
    contextBar: true
    fullscreen: false
    terminalImages: true
    preset: !!js process.env.DSH_TUI_PRESET ?? undefined
    workspace: !!js process.env.DSH_TUI_WORKSPACE_TARGET ?? undefined
    sessionId: !!js process.env.DSH_TUI_RESUME_SESSION ?? undefined
```

## TUI config fields

| Field | Default / source | Meaning |
|---|---|---|
| `provider` | Harness `agentDefaultModel`; bare compositions fall back to `deepseek-official` | Model route. `provider` **and** `model` must both be set to form an explicit route |
| `model` | Harness `agentDefaultModel`; bare compositions fall back to `deepseek-flash` | Startup model; `/model` switches through a session fork |
| `cwd` | Git worktree root containing the launch directory (`process.cwd()` outside any worktree) | TUI-side session workspace: agent meta, `@` completion/mention expansion, `/resume` filtering, status line. Resuming a session adopts that session's persisted cwd. The bash/fs-policy/sandbox roots are owned by the composition layer (default: the launch directory) and may differ |
| `workspace` | unset | Startup workspace target: local path, `file://` URL, or plugin-provided URI; takes precedence over `cwd` |
| `effort` | normally `max` in the bundle | Reasoning effort for every request (validated against the model's levels; an invalid level silently falls back to the adapter default). Precedence: `/settings` `effortDefault` (`auto` defers) > this field > persisted `/effort` choice > model default |
| `effortDefault` | unset | Default reasoning effort for new sessions; `auto` defers to `effort` |
| `whale` / `whaleIdle` | `true` / `true` | Header whale and welcome-page idle animation |
| `minimal` | `false` | Reduce header decoration and colors |
| `modes` | built-in trio | `Shift+Tab` session-mode cycle; defaults to default → plan → full-access |
| `activity` | `true` | Show the live activity row |
| `activityFrames` | `moon8` | Activity animation preset; `/activity` changes it at runtime. A legacy saved `claude` is read as `moon8` |
| `contextBar` | `true` | Segmented context-usage bar below the input. Both this and `/settings → statusBar.contextBar` must be on for it to render |
| `fullscreen` | `true` (factory default since 0.9.0) | `true` uses the alternate screen, app scrolling and mouse selection; `false` uses inline mode. Requires `/restart` |
| `terminalImages` | `true` | Allow previews in supported terminals; `false` keeps text metadata and skips image probing and preview decoding. Requires `/restart` |
| `preset` | roster default `standard` | Agent preset for new sessions; explicit configuration wins over the persisted preference |
| `sessionId` | unset | Session to resume, normally injected by the Windows `--resume` launcher |

## `/settings` fields

Changes save automatically; `Esc` exits.

| Field | Notes |
|---|---|
| `lang` | UI language zh/en (locked while `DSH_TUI_LANG` is pinned) |
| `fullscreen` | Fullscreen mode (default on), effective after `/restart` |
| `terminalImages` | Terminal image preview (default on, needs terminal support), effective after `/restart`. Off shows text only and skips preview decoding; **sending images to the model is unaffected** |
| `whale` | Pixel whale header (default on); three intro animations are picked per launch; `/deepseek` re-rolls |
| `whaleIdle` | Welcome idle animation (default on); freezes after the first task |
| `diffLayout` | Edit/Write diff layout: auto (two columns ≥110 cols) / split / unified |
| `thinkingFold` | Thinking block: preview (2–3 lines, folded when settled) / full |
| `effortDefault` | Default reasoning effort: auto / off / low / high / max. When the model lacks that level it drops one level and shows a notice |
| `smoothStreaming` | Smooth streaming (default on) at ~30 fps; replay/history stays direct |
| `toolBackground` | Tool-card background emphasis: none / subtle / strong |
| `mermaidDiagrams` | Render ```` ```mermaid ```` blocks as character diagrams (default on); applies immediately |
| `scrollGutter` | Transcript gutter: timeline (default) / scrollbar / hidden; applies immediately |
| `pageMargin` | Page margin from all four edges: none / slim / normal (default) / roomy, or custom `NxM` (**cap 8x4**); applies immediately |
| `foldTerminalCommand` | Fold multi-line terminal commands on tool cards (default off); `Ctrl+O` or click expands |
| `expandEditor` | Fullscreen draft editor (default on); `Ctrl+Shift+E` or `⛶`; off hides both entries |
| `recapOnOpen` | Auto recap divider when opening/resuming a session (default on; setting name documented ZH-only) |
| `statusBar.*` | Every status-bar toggle: `compact`, `model`, `thinking`, `cwd`, `contextUsage`, `cache`, `tokens`, `cost`, `tps`, `gitBranch`, `sessionTitle`, `sessionId`, `mode`, `contextBar`, `activity`, `trajectory`. **`statusBar.sessionId` is a bottom-bar display toggle, unrelated to the cordis startup `sessionId`** |

Default-on status fields: `compact`, `model`, `thinking`, `cwd`, `contextUsage`, `cache`, `cost`,
`goal`, `contextBar`. Default-off: `tokens`, `tps`, `gitBranch`, `sessionTitle`, `sessionId`, `mode`,
`activity`, `trajectory`.

## Terminal images: precedence and force-off

- `/settings → Terminal image previews` overrides `config.terminalImages`.
- Without a saved choice the config value applies (default on).
- Enabling still requires Kitty graphics support and a display mode that allows image rendering.
- `DSH_TUI_DISABLE_TERMINAL_IMAGES=1` **always** forces previews off; the settings list shows it
  separately as "Image previews (forced off)".
- Disabled previews read and decode nothing and send no rendering commands. **Sending images to the
  model is unaffected** in every case.

## Environment variables

| Variable | Purpose |
|---|---|
| `VISUAL` / `EDITOR` | External editor for `Ctrl+G` (`VISUAL` wins; arguments like `code --wait` are allowed; with neither set the TUI asks you to configure one — no `vi` fallback) |
| `DEEPSEEK_API_KEY` | Required DeepSeek credential |
| `DEEPSEEK_BASE_URL` | Override the compatible DeepSeek API endpoint |
| `DSH_HOME` | Harness home (profiles, sessions, credentials, attachments); defaults to `~/.dsh` |
| `DSH_TUI_PERSONA` | Override the Agent persona injected by the composition |
| `DSH_TUI_PRESET` | Override the default Agent preset for new sessions |
| `DSH_TUI_THEME` | Pin a built-in (`auto`/`light`/`dark`/`dark-ansi`), static, or plugin theme ahead of the persisted selection |
| `DSH_TUI_DISABLE_MOUSE` | Temporarily disable mouse handling in fullscreen mode |
| `DSH_TUI_DISABLE_TERMINAL_IMAGES` | `1` forces probing, preview reads/decoding and image rendering off, overriding config and `/settings` |
| `DSH_TUI_IMAGE_PROTOCOL` | `auto` (default), `kitty`, `sixel`, `none`; overrides protocol selection **without bypassing** the preview preference, disable switch, non-fullscreen, accessibility or multiplexer guards |
| `DSH_TUI_RESUME_SESSION` | Resume a session at startup, normally set by a launcher |
| `DSH_TUI_WORKSPACE_TARGET` | Workspace path or URI resolved at startup, normally set by `dsh-tui <target>` |
| `DSH_TUI_SESSION_ROOT` | Override the JSONL session root; profile default `$DSH_HOME/sessions`, bare `cordis.yml` default `~/.dsh-tui/sessions` |
| `DSH_PERMISSION_MODE` | Override the non-Windows sandbox policy, e.g. `workspace-write` or `danger-full-access` |
| `DSH_TUI_WORKSPACE` | Working directory used by the Windows `dsh-tui.cmd` launcher |
| `DSH_TUI_DEBUG` | Enable dsh-tui diagnostics on stderr |
| `DSH_TUI_RENDER_LOG` | File path for raw ANSI frame capture. **May capture visible prompts, tool arguments and output — redact before sharing** |
| `DSH_TUI_LANG`, `DSH_TUI_LIANGSHEN_BASH_PATH`, `DSH_TUI_ACCESSIBILITY`, `DSH_TUI_DEV_ROOT`, `DSH_TUI_DEV_WORKSPACE`, `GROK_HOME` | See the named sections below and in `sessions.md` |

The old `CC_TUI_*` and `DSH_CC_*` names are no longer read; use the `DSH_TUI_*` prefix.

### Diagnostic-only variables (all off by default)

| Variable | Purpose |
|---|---|
| `DSH_TUI_DEBUG_REPAINTS=1` | Record repaint diagnostics |
| `DSH_TUI_COMMIT_LOG=1` | Record render-commit diagnostics |
| `DSH_TUI_ACCESSIBILITY=1` | Enable accessibility display paths (no animation or graphics preview) |
| `DSH_TUI_TMUX_TRUECOLOR=1` | Enable the truecolor detection path in tmux |
| `DSH_TUI_TAB_STATUS=1` | Experimental terminal tab-status opt-in, no guarantee in every terminal |

Diagnostic output never changes session events or model routing — enable only what the issue needs.

## Preference files (`~/.dsh-tui/`)

All are best-effort and fall back cleanly on a bad file:

| File | Holds |
|---|---|
| `theme.json` | `/theme` choice |
| `model.json` | `/model` choice, surviving restart and `/new` |
| `agent-preset.json` | Default agent preset |
| `effort.json` | Persisted `/effort` choice |
| `working-activity.json` | `/activity` frames choice |
| `lang.json` | `/lang` choice |
| `trajectory.json` | Trajectory preferences |
| `resume.txt` / `last-used.json` | Resume marker (read by the Windows launcher) |
| `home.json` | Session-manager "seen" marker |
| `session-pins.json` | Pinned sessions in the session manager |
| `session-mounts.json` | Occupancy ledger for concurrent TUI instances (`architecture.md`) |
| `themes/<name>.json` | Custom themes (`themes.md`) |

## Agent presets

Each session composes its model-visible tools and prompt through the official preset registry:
`@deepseek-ai/dsh-agent-preset-registry` on 0.1.7, `@deepseek-ai/dsh-agent-presets` on older hosts.

| ID | Name | Capability |
|---|---|---|
| `standard` | Standard (default) | Editing, shell, search, skills, planning, goals, subagents, workflows |
| `ptc` (0.1.2) / `code` (legacy 0.1.1) | PTC | Standard plus the PTC SDK presentation for composing operations in TypeScript; both names resolve compatibly across versions |
| `minimal` | Minimal | Persistent Bash and `str_replace_editor` only, **without compaction** |
| `cordis` | Creation | Standard plus runtime inspection and plugin-experimentation tools |
| `liangshen` | Liangshen mode | Minimal's two-tool surface first for root and delegated agents, the full catalog after the first tool call, and a fresh anchor after compaction |

Selecting and switching:

- `/preset` opens the picker, `/preset <id>` selects, `/preset status` reports.
- Picker names/descriptions come from registry declarations (`preset.yml` on older hosts); under
  `/lang en` the built-ins show localized English names.
- A blank session switches in place. Once a conversation has started, the official blank-only rule
  stores the choice as the default for `/new` or the next launch.
- Precedence: explicit `config.preset` or `DSH_TUI_PRESET` → persisted preference → roster default `standard`.
- A legacy `code` preference resolves to `ptc` when the roster no longer provides `code`, then
  migrates; rc rosters keep their real `code` id and session logs are never rewritten.
- Resuming a session restores the preset recorded in its log and does not overwrite it with the current default.

**Liangshen mode** ships with dsh-tui. On Windows its first-round `bash` runs an auto-discovered Git
Bash: the installation tree of a `git.exe` on PATH, then conventional/Scoop roots, then bare `bash` on
PATH — it never accepts the System32 WSL launcher. Pin it with `DSH_TUI_LIANGSHEN_BASH_PATH` (the pin
is the only candidate; a miss warns and exposes the full tool catalog on the first round).

**Custom presets**: on 0.1.7 declare `@deepseek-ai/dsh-agent-preset` through a profile/bundle with
`id`, `name` and `plugins`; the TUI no longer scans directories itself. Older hosts still discover
`$DSH_HOME/.agent-presets/<name>/agent.cordis.yml`. Since 0.3, tools, planning, compaction and
delegation are owned by the preset — the old `DSH_TUI_COMPACT_RATIO`, `DSH_TUI_COMPACT_RETAIN` and
subagent-depth knobs are gone.

## MCP servers

`@deepseek-ai/dsh-mcp-client` supports stdio and streamable HTTP; mounted tools register as
`mcp__<server>__<tool>` and enter the model tool set automatically. Insert servers in the user
`cordis.patch.yml`:

```yaml
- insert:
    - id: mcp-context7
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        transport: stdio
        serverName: context7
        command: npx
        args: ['-y', '@upstash/context7-mcp']

    - id: mcp-remote
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        transport: streamable-http
        serverName: remote
        url: https://example.com/mcp
        headers:
          Authorization: !!js process.env.MCP_TOKEN
```

Run `/mcp` to inspect connected servers and tool counts. Field reference:
<https://deepseek-harness.github.io/deepseek-harness/reference/config-catalog#deepseek-ai-dsh-mcp-client>.

## Providers at runtime (`/provider`)

- Add, edit or delete model providers without a restart, from built-in catalog routes or custom API endpoints.
- **Only providers written by the user settings layer can be edited or deleted**; inherited ones cannot be removed.
- Keys are written to `~/.dsh/.credentials.yaml` (mode `0600`) and render as `••••••`. Only
  non-environment keys are stored, and a key shared with another provider is kept on delete.

| Artifact | Location |
|---|---|
| Provider profile | `llm-pi-ai.providers.<route>` in the active profile config on 0.1.7, or `~/.dsh/settings.yaml` on older hosts; the route registers on write and unregisters on delete |
| API key | `~/.dsh/.credentials.yaml` (mode `0600`), referenced as `<ROUTE>_API_KEY` |

With the bundled `dsh-auth` plugin mounted, the add branch also offers **subscription sign-in (OAuth)**
for ChatGPT / Claude / Grok via browser or device-code flow (no API key); `/auth status|login|logout`
shares the same source. Custom endpoints need a route name, API key, baseURL and protocol
(`openai-completions` / `openai-responses` / `anthropic-messages`); after adding, run `/model` to switch.

## Live activity row

`dsh-working-activity` is installed with the package and inserted by its patch. The working line reads
the plugin's `workingActivity` session projection, which needs `dsh-working-activity` ≥ 0.5.0 (older
versions produce no working line). Tune it by overriding the existing id only:

```yaml
- id: working-activity
  config:
    publishIntervalMs: 500
```

**Do not insert a second row and do not separately run `dsh plugin ... add dsh-working-activity`** for
the same profile — the activity row would mount twice.

## Composition constraints

- `user-interaction` normally comes from `dsh-base`; the plugin creates a fallback in a bare
  composition, but the profile patch must not insert a duplicate.
- Mount the core `subagent` service before manually inserting a subagent provider.
- A custom `plan-mode` override requires a non-empty `section`.
- Profile mode uses the base JSONL persistence row rooted at the shared `~/.dsh/sessions`, so TUI and
  web read the same history. `DSH_TUI_SESSION_ROOT` always names a JSONL root.

## Divergences and gaps in the docs

- The `fullscreen` factory default is stated in the Chinese `architecture.md` and implied by
  `configuration.en.md`; `src/index.ts` confirms `Schema.boolean().default(true)` for both `fullscreen`
  and `terminalImages`.
- `configuration.md` (ZH) references the user guide for `/provider` menu details; the English page omits
  the cross-reference.
- `~/.dsh-tui/model.json` surviving restart and `/new` is documented only in the English page.
- No `@deepseek-ai/schemastery` or DSH host-version range appears in these files; the repository's
  `ADAPTER.md` is the authoritative compatibility list.
