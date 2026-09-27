# Themes

Source: [`docs/themes.en.md`](https://github.com/ccch1mneyyy/dsh-TUI/blob/main/docs/themes.en.md)
(cross-checked against `docs/themes.md`).

## Built-in themes

dsh-TUI ships three **Gentle Mist Blue** palettes plus an `auto` pseudo-theme:

| Name | Purpose |
|---|---|
| `auto` | Pseudo-theme: follows the system/terminal background, resolving to `light` or `dark` |
| `light` | White panels, ink body text, mist-blue interaction color |
| `dark` | Dark-terminal adaptation with warm-gray text and soft blue accents |
| `dark-ansi` | Compatibility fallback using only the 16 ANSI colors |

- Without an explicit choice the TUI queries the terminal background with **OSC 11** and selects
  `light` or `dark`, falling back to **`dark`** when the terminal does not answer.
- Light-theme panels, tool cards and image previews use white (`#FFFFFF`) surfaces by default, with
  neutral borders on previews. **This never modifies the terminal's own background or wallpaper.**
- `auto` turns the one-shot startup detection into a standing choice:
  - It is a valid value for `/theme`, `DSH_TUI_THEME` and `~/.dsh-tui/theme.json`.
  - Selecting `auto` applies the last detected base immediately and re-queries OSC 11 in the background.
  - On terminals that follow the system theme, picking `auto` again (or restarting) catches up after a
    system light/dark switch.
  - `/theme status` shows which palette `auto` currently resolves to.
  - **A user theme named `auto` is shadowed by the built-in pseudo-theme** and is not listed in the picker.

Selection precedence:

```text
DSH_TUI_THEME
  > persisted choice in ~/.dsh-tui/theme.json
  > OSC 11 background detection
  > dark fallback
```

## Switching

- `/theme` opens the picker: `auto` and the built-ins come before static JSON themes and plugin themes.
- `/theme <name>` switches directly (static or runtime plugin theme).
- `/theme status` shows the current theme and persistence location.
- Confirming a choice hot-switches immediately and writes `~/.dsh-tui/theme.json`. **`DSH_TUI_THEME`,
  when set, still wins on the next launch.**

## Custom themes

Drop JSON files into `~/.dsh-tui/themes/`. Each file starts from one built-in palette and overrides a
subset of its colors:

```json
{
  "name": "sakura",
  "displayName": "Sakura",
  "base": "dark",
  "colors": {
    "accent": "#FF9EC7",
    "accentShimmer": "#FFC0D5",
    "activity": "#7DA1DE",
    "activityShimmer": "#ABC2EC",
    "mascotBody": "#D98A63",
    "inputBackground": "#000000",
    "permission": "#FFB3CC",
    "promptBorder": "#B08B99",
    "text": "#E8E6E0",
    "inactive": "#A99BA0",
    "subtle": "#8A7A80",
    "selectionBg": "#5C3A44",
    "success": "#9CC7A8",
    "error": "#E08591",
    "warning": "#E0C08A"
  }
}
```

| Field | Required | Meaning |
|---|---|---|
| `base` | Yes | `light`, `dark` or `dark-ansi`; source for every non-overridden color |
| `colors` | Yes | Partial override of the theme color keys |
| `name` | No | Theme id; defaults to the filename |
| `displayName` | No | Picker label; defaults to `name` |

When a file declares `name`, **its filename remains a loading alias**. The authoritative full key list
is the `Theme` type in `src/theme.ts`.

### Color keys

| Purpose | Keys |
|---|---|
| Brand / focus / activity | `accent`, `accentShimmer`, `activity`, `activityShimmer`, `suggestion`, `remember` |
| Panels and borders | `permission`, `permissionShimmer`, `promptBorder`, `promptBorderShimmer`, `bashBorder`, `planMode`, `ide`, `background` |
| Body text | `text`, `inverseText`, `inactive`, `inactiveShimmer`, `subtle` |
| Tool names and status dots | `toolNameMutate`, `toolNameExec`, `toolDotExec`, `toolDotRead`, `toolDotWrite`, `toolDotWeb`, `toolDotTask` |
| Tool card surfaces | `toolCardBackground`, `toolCardBackgroundDim` |
| Status | `autoAccept`, `success`, `error`, `warning`, `warningShimmer`, `merged` |
| Diff | `diffAdded`, `diffRemoved`, `diffAddedDimmed`, `diffRemovedDimmed`, `diffAddedWord`, `diffRemovedWord` |
| Diff syntax highlighting | `syntaxKeyword`, `syntaxString`, `syntaxComment`, `syntaxNumber`, `syntaxFunction`, `syntaxType`, `syntaxVariable`, `syntaxOperator`, `syntaxPunctuation`, `syntaxConstant` |
| Badges and accents | `mascotBody`, `inputBackground`, `professionalBlue`, `chromeYellow` |
| Messages and input | `userMessageBackground`, `userMessageBackgroundHover`, `messageActionsBackground`, `selectionBg`, `bashMessageBackgroundColor`, `memoryBackgroundColor`, `rate_limit_fill`, `rate_limit_empty`, `fastMode`, `fastModeShimmer`, `userPromptLabel` |
| Subagent messages | `subagentBullet`, `subagentDescription`, `subagentModel`, `subagentElapsed`, `subagentToolName`, `subagentStatusRunning`, `subagentStatusCompleted`, `subagentStatusFailed` |

### Color formats

Accepted: `#rgb`, `#rrggbb`, `#rrggbbaa`, `rgb(r,g,b)`, `ansi256(n)`, and 16-color names such as
`ansi:black` and `ansi:redBright`.

**Colors must be concrete values.** CSS variables, gradients and arbitrary CSS color names are rejected.

### Validation and failure behavior

- Unknown theme key → skip that key with a warning, keep the rest.
- Invalid color value → skip that value with a warning.
- Invalid `base`, malformed JSON, or a non-object `colors` → **skip the whole file**.
- A theme referenced by the environment or a preference file but missing → warn and continue with
  background detection.
- One bad theme never blocks TUI startup or other themes.
- Theme names are user input: the loader verifies that the resolved path stays inside
  `~/.dsh-tui/themes/`. Preserve that containment check when changing the implementation.
- Validate the subsystem with `node --import tsx/esm scripts/verify-themes.mjs`.

## Plugin (runtime) themes

An npm plugin registers a runtime theme through the **`tuiThemes` service** instead of writing to
`~/.dsh-tui/themes/`:

```ts
import type { Context } from '@deepseek-ai/cordis'

export function apply(ctx: Context): void {
  ctx.get('tuiThemes', false)?.register({
    name: 'my-plugin:night',
    base: 'dark',
    colors: { accent: '#88AAFF' },
  }, ctx)
}
```

- Use a lowercase safe id such as `plugin-id:theme-id`.
- **`auto`, the built-in names and `status` are reserved** and cannot be plugin theme names.
- Registrations disappear with the plugin activation; the returned disposer can remove one early.
- Plugin themes appear in the `/theme` picker and completion and persist through the existing
  `~/.dsh-tui/theme.json`.
- Priority: **built-ins > static JSON > same-name plugin theme.**
- On an older profile without `tuiThemes` the plugin degrades silently and static themes are unaffected.
- Full key list, legacy-key mapping and registration contract: [Plugin Admission and Development Guide](https://github.com/T-Auto/dsh-ecosystem-spec/blob/main/docs/plugin-admission-and-development.md).

## Design guidance

- Use color keys rather than changing only `text` and `background`. Check at least body, inactive,
  focus, selection, success, warning, error and diff colors.
- Test light themes in a real light terminal and dark themes in a dark one.
- Check 16-color, 256-color and truecolor fallback behavior.
- Verify narrow layouts, tool diffs, questionnaires, multiline input and selection contrast.
- **Theme files must contain display metadata and colors only — never credentials or other user data.**

## Divergences in the docs

- The color-key **sets** are identical in both languages; only the grouping in the bullets differs.
- The sample `displayName` differs (`"Sakura"` in English vs `"樱花粉"` in the Chinese original).
