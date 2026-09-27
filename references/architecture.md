# Architecture, performance and the mount runtime

Sources: [`docs/architecture.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/architecture.en.md),
[`docs/session-mount-runtime.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/session-mount-runtime.en.md),
and [`docs/plugins.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/plugins.en.md), cross-checked
against the Chinese originals. This file is for "why does it behave this way" and for changing the TUI —
it is not needed to *use* it.

## Runtime path

```text
Cordis profile
  -> src/index.ts            (plugin contract and Schema; keep the entry small and lazy)
  -> src/dsh-adapter/plugin.ts  (TTY guard, services, Agent create/resume, React mount, cleanup funnel)
  -> DSH Agent / session / tool services
  -> src/dsh-adapter/channel.ts (session/event -> Channel)
  -> src/screens/Chat.tsx    (keyboard and mode orchestration)
  -> src/components/*        (views)
  -> src/ui.ts               (themed Box/Text, render, selection, scroll)
  -> src/ink/* + Yoga        (layout, terminal protocol, differential output)
  -> ANSI terminal
```

Module ownership, briefly:

| Module | Owns |
|---|---|
| `src/index.ts` | Cordis plugin name, injection declaration, config interface, Schema |
| `src/dsh-adapter/plugin.ts` | TTY guard, service assembly, Agent create/resume, React mount, the single cleanup funnel |
| `src/dsh-adapter/channel.ts` | Channel composition root: options/services, owner/binding, specialist wiring, install, start/release |
| `src/screens/Chat.tsx` | Modal precedence, global keys, scroll/search/selection state, slash dispatch |
| `src/components/` | Views and design-system primitives — no Agent or session source of truth |
| `src/ui.ts` | Themed primitives: render, selection, scroll |
| `src/theme.ts`, `src/themeCatalog.ts` | Theme resolution and catalog ordering |
| `src/dsh-adapter/themes.ts` | The `ctx.tuiThemes` seam, registration lifecycle, host facade |
| `src/ink/`, `src/native-ts/yoga-layout/` | Renderer, terminal protocol, events, selection, Yoga bridge, pure-TS layout |
| `cordis.patch.yml` | Profile bundle layer, service rows, overrides, mount ordering |
| `src/workspaces.ts` | Local-path fallback and the generic workspace-provider registry (contains no provider protocol) |

Composition rules stated as one-way dependencies:

- An uninstalled or released action **fails explicitly**; it never pretends success with a no-op.
- Do not duplicate DSH Agent/session/tool services in a component — connect through an existing
  service, registry, or channel seam.
- Workspace and theme extensions are one-way: the TUI publishes structural interfaces; plugins register
  URIs, metadata and executors or complete palettes, and the host owns validation, ordering, rendering
  and lifecycle.
- Protocol parsing and external connections belong to the plugin: removing a plugin must leave local
  workspaces and session flows free of missing config, placeholders or fallback branches.

## The session log is the source of truth

- Conversation truth lives in DSH `session/event` records, not in a React-local array: initial replay,
  incremental streaming, assistant/reasoning/tool association, rewind boundaries, and reconstruction
  after resume, export, compact and fork.
- The Channel keeps a TUI-sized **projection**. Past the window, older rows fold into short previews;
  the complete content stays in the log and is restorable from events.
- Tool results are associated by **`callId`**, never by array position.
- There is exactly **one projector**: `channel/projection.ts`.
- `workingActivity` is a *session projection published by the external `dsh-working-activity` plugin*;
  the TUI only reads it, derives nothing in-process, and writes no UI state to the shared log.
- `/permission` adapters read the real service contract `current(session)` through the session-projection
  seam, with a compatibility fallback for the older event-log shape.

## Rendering and long-session performance

- **Differential output**: each frame writes only screen changes; capability detection selects
  synchronized output, cursor and Windows Terminal paths.
- **Message virtualization**: off-screen rows use the last measured fixed-height placeholder and leave
  the full layout subtree.
- **Replay coalescing**: consecutive token chunks merge during history replay.
- **Bounded caches** for transcript, render nodes and measurement; removing a bound requires measured evidence.
- **Zero-allocation hot paths**: the `visibleRows` pipeline is memoized on rows identity, length and a
  streaming-bit fingerprint — no per-tick array or Map allocations.
- **Global LRU caches** for `wrapText` and markdown tokens reuse measurements across mounts.
- **Framed backfill**: the main screen mounts the tail window first and backfills history in frames.
- **Landing anchor**: `/resume` asserts that the newest message's last row is visible and reachable;
  long-session restores skip the splash animation.
- **Display-cell width** (not `string.length`) drives layout: ANSI escapes, combining marks, emoji and
  East Asian wide characters.
- Diagnostics must never print to an active TUI's stdout — use stderr (`DSH_TUI_DEBUG`) or
  `DSH_TUI_RENDER_LOG`.
- Changing `src/ink/` or Yoga requires running the CI questionnaire/tool-card regressions plus the
  affected scroll, resize, copy-on-select or PTY harness.

Documented image budgets (English page carries the fullest statement):

- A full preview allows a **2048-pixel edge** and at most **2,097,152 pixels (8 MiB RGBA)**.
- Thumbnails decode at **384**; ordinary plugin images **1024 / 4 MiB**; frames **16 MiB**; Sixel output **4 MiB**.
- Each inspection keeps **one encoded original up to 64 MiB** with a **64-megapixel** input limit and no
  full-size JS RGBA cache; cropping precedes nearest-neighbor zoom.
- A stale `[Image #N]` placeholder (evicted past **128 staged images**, or invalidated by a session
  switch) warns on click, on submit and as a slash-command argument.
- Previews fade the surrounding conversation halfway toward the terminal background (or use the faint
  attribute for uncoloured text) and restore it on close.
- Sixel thumbnails are cropped to the visible transcript while scrolling; opening a preview withdraws
  background thumbnails and closing restores them from cache.

## Inline vs fullscreen

| Mode | Ownership |
|---|---|
| Inline | Content stays on the main screen; the terminal emulator owns scrollback and native text selection |
| Fullscreen | `AlternateScreen`: the TUI owns scrolling, mouse selection, OSC 52 copy and screen restoration |

Both modes share the Channel and React views but use different terminal protocol paths. Any change to
input, scrolling, mouse, cursor, resize or cleanup must be verified in **both** modes, especially narrow
terminals and Windows ConPTY. The factory default is fullscreen (`src/index.ts`).

## Multi-TUI: the session mount runtime

A **session** is one running conversation (on disk an append-only event log). A **process** is one TUI
terminal. The web UI and the GUI host all sessions per machine; the TUI is the terminal process, so one
machine can run several TUIs at once.

One TUI process = one terminal + one **mount set** = the attached session plus every **parked** session.
Parking is neither pausing nor "save and close": a parked session is alive and an in-flight turn keeps
generating; the terminal just does not show its transcript.

Three rules:

1. Switching a session changes what you look at, not what is running.
2. A session is **occupied** when another live process has it mounted — two processes driving one
   append-only log would interleave events and corrupt the transcript. Occupied sessions appear red with
   `held by pid <pid>` and cannot be entered until that process exits.
3. A parked session is not occupied, so the same process keeps full ownership when you return to it.

The on-disk ledger:

| Artifact | Detail |
|---|---|
| `~/.dsh-tui/session-mounts.json` | Mount-set ledger: processes × the session ids they have mounted |
| `~/.dsh-tui/session-mounts.lock` | Lock guarding read-modify-write |
| Publish cadence | `PUBLISH_INTERVAL_MS = 15 s`, one small lock-guarded write; the ledger is rewritten wholesale, so it cannot grow unbounded |
| Screen refresh | The session screen's own **2 s** tick re-reads occupancy and live status with `setState` only |
| Session listing | `listSessions()` once when the screen opens, plus manual `Ctrl+L` |
| Liveness witness | The **pid**, checked against live processes; dead records are filtered in memory on read and pruned by the next write |
| Permissions | The ledger's directory is `0700` and files are `0600` |
| Timers | Every timer is `.unref()`d, is never the reason a process cannot exit, and is cleaned up through `ctx.effect` / `owner.own` |

Check and claim are one step, so two processes cannot both believe they own a session. The read path
never writes the file. Live status comes from the channel's existing agent-view projection snapshot
rather than a new subscription. `--resume` establishes a **cross-process identity** for the session it
restores, so a later terminal that resumes the same id sees the first one as its holder.

## Persistence locations

| Path | Contents |
|---|---|
| `~/.dsh/sessions/` | Shared JSONL session events for profile TUI and web |
| `~/.dsh-tui/sessions/` | JSONL session events for direct `cordis.yml` runs |
| `~/.dsh-tui/resume.txt` | Recent session id used by the Windows launcher and the exit hint |
| `~/.dsh-tui/last-used.json` | `/resume` recency metadata |
| `~/.dsh-tui/session-mounts.json` + `.lock` | Occupancy ledger (above) |
| `~/.dsh-tui/themes/` | User theme JSON files (runtime plugin themes never write here) |

Preference files are optional state: malformed or missing files fall back silently rather than blocking
startup. The data directory is `~/.dsh-tui`; early releases used `~/.dsh-cc` and there is no automatic
migration. `DSH_TUI_SESSION_ROOT` overrides the JSONL root in either composition.

## Permissions and the security boundary

- dsh-TUI provides **no separate sandbox**. Effective capability comes from the DSH services mounted by
  `cordis.patch.yml`.
- It implements the tool-level approval UI (a local panel answering the `approval/request` waterfall)
  while `/permission` preset switching comes from the dsh-base `permission-presets` row. The
  documented waterfall allows **allow-once or deny only** — the protocol has no "always allow" and no
  feedback channel (Chinese `architecture.md` only).
- Documented platform defaults: non-Windows `DSH_PERMISSION_MODE` defaults to `workspace-write` with the
  approval policy normally `ask`; Windows has no usable local sandbox backend in the current
  composition, so it uses `danger-full-access` with `never` approval to match the terminal trust model.
  Treat these as composition-owned, not TUI-owned.
- `/permission` reads the mounted `permissionPresets` registry in declared order; third-party presets
  join the picker, Tab completion and the `Shift+Tab` cycle. `custom` is a projection of current state,
  never a selectable target. Switching prefers the official `/permission <preset>` command and falls
  back to the service's own write path `set(session, preset)`; it writes real `permission/preset`,
  `sandbox/mode` and `approval/policy` events and confirms by readback. With neither path available it
  fails loudly instead of sending the input to the model, and a mounted service that is empty,
  inconsistent, broken or unsafe **fails closed**.
- Exiting plan mode restores the pre-plan atoms first, then the durable preset identity from before
  plan mode while the registry still offers it.
- MCP servers, shell, filesystem tools and custom presets expand what the model can access and are
  code-execution surfaces in the same policy domain. **Inspect the active profile patch before running
  in an untrusted repository — the visual TUI alone does not describe the effective policy.**
- Clipboard images are exported through a `0600` temporary file in a `0700` private directory, stored
  in the Harness attachment library and rendered as `[Image #N]`; the temporary export is then deleted,
  so the prompt contains neither a path nor base64. The mount ledger's directory is `0700` and files are `0600`.

## Honest evidence status (from the docs)

- The documented L4 batch had **not** manually exercised a real TTY in inline/fullscreen mode, at narrow
  width, or on Windows ConPTY; live model integration still needs a target-terminal check.
- L4 passed one independent local review, the compile/build/package gates, all **58 channel-ui checks**
  and **three CI renderer regressions** — this is not real-TTY or long-term-memory stress evidence.
- There is **no automated full-flow suite requiring real model credentials**; CI uses headless rendering
  and fake services.

## Plugin seams

The authoritative admission and development guide lives outside this repository:
[Terminal Interaction Ecosystem Plugin Admission and Development Guide](https://github.com/T-Auto/dsh-ecosystem-spec/blob/main/docs/plugin-admission-and-development.md).

| Tier | Seams |
|---|---|
| Stable candidates (shape frozen; a breaking change is deprecated with a warning first) | 6 settings sections · 8 fullscreen scenes · 10 managed dialogs · 11 status line · 12 keyboard shortcuts · 13 entry renderers |
| Experimental (may move with dsh-std / the admission spec) | 9 decision events · toast notifications (`ctx.tuiToast`) |
| Follows upstream (stability governed by Cordis / DSH) | 1 session events · 2 official prompt slots · 3 skill packaging · 4 themes · 5 system-prompt sections · 7 profile composition |

Also relevant:

- `@deepseek-harness-tui/dsh-tui/api` is an experimental public surface; the `test-utils` subpath and
  `ctx.tuiPluginHost.grants.corrupt` were removed in the adapter layering refactor (#705), and `grants`
  narrowed to `HostGrantFacade`.
- The repository's own maintainer skills live under `.agents/skills/` and are not part of the shipped product.

## Divergences and gaps in the docs

- `architecture.md` (ZH) is the only place stating the fullscreen factory default; the English page
  omits it (and its "see below" pointer dangles). `src/index.ts` confirms it.
- The English `architecture` page alone carries the detailed image-budget material and the modal-gallery
  rules; `architecture.md` alone carries the `ctx.approval` seam and the allow-once/deny waterfall.
- Precision differences: "2,097,152 pixels" (EN) vs "about 2.1 megapixels" (ZH); explicit two-level
  sidebar ordering (EN) vs "most recently active/used" (ZH).
- `session-mount-runtime.en.md` says "the model that survived is the overview's" without naming the
  screen; the Chinese page names `/agentview`.
- `tuiShortcuts` and `tuiStatus` are documented in `interaction.*` rather than `plugins.*`.
