# Prism — Customization Extension

Prism is a modular web-customization extension for Firefox and Zen. It ships
with **no modules** — instead, it pulls module `.json` files from this GitHub
repo (or any GitHub repo you configure) and runs them locally. Once a module
is downloaded, it's yours: edit it, delete it, or update it from the repo.

This README documents the **module file format** so anyone can contribute a
module by dropping a `.json` file into the [`modules/`](./modules) folder.

---

## Table of contents

- [How Prism works](#how-prism-works)
- [Repository layout](#repository-layout)
- [Module file format](#module-file-format)
- [Setting types](#setting-types)
- [CSS conventions](#css-conventions)
- [Authoring a module, step by step](#authoring-a-module-step-by-step)
- [Built-in content scripts](#built-in-content-scripts)
- [Repo sync behavior](#repo-sync-behavior)
- [Local vs. repo modules](#local-vs-repo-modules)
- [Examples](#examples)

---

## How Prism works

1. The extension reads a list of `.json` files from `https://github.com/<owner>/<repo>/tree/<branch>/<path>` (configured in **Dashboard → Settings**; default is this repo's [`/modules`](./modules) folder).
2. It downloads each file and validates that it's a proper module (has `id`, `name`, `matches`, `settings`, `styles.css`).
3. Each downloaded module is stored as a **local copy** in `browser.storage.local`. From that point on it's a local module — your copy, your rules.
4. When you visit a site whose URL matches a module's `matches` patterns, Prism's engine injects the module's CSS at `document_start`, exposing every setting as a CSS variable and a set of HTML classes.
5. Built-in content scripts (currently for `duckduckgo.com` and `www.youtube.com`) add JavaScript features for those specific sites when the matching module is installed and enabled.

**Prism ships with zero modules.** You sync from this repo to get the
defaults, or build your own from scratch in the Dashboard editor.

---

## Repository layout

```
customization-extension/
├── README.md              ← this file (Prism ignores it during sync — only .json files load)
├── modules/               ← drop your .json module files here
│   ├── duckduckgo.json
│   ├── youtube.json
│   └── wikipedia-tweaks.json
└── (any other files)      ← ignored by sync
```

Prism's sync only looks at files in the configured path (`/modules` by
default) and only loads files whose name ends in `.json`. Everything else
(README, scripts, images, dotfiles) is silently skipped — you don't need to
worry about it polluting the module list.

---

## Module file format

A module is a single JSON file. The minimum viable shape:

```json
{
  "id": "my-module",
  "name": "My Module",
  "version": "1.0.0",
  "author": "your-handle",
  "description": "One-line description shown in the dashboard.",
  "emoji": "✨",
  "accent": "#e7b458",
  "matches": ["*://*.example.com/*"],
  "settings": [
    {
      "id": "bgColor",
      "label": "Background",
      "type": "color",
      "default": "#1c1c1e",
      "section": "Theme"
    },
    {
      "id": "hideHeader",
      "label": "Hide header",
      "type": "toggle",
      "default": false,
      "section": "Layout"
    }
  ],
  "styles": {
    "css": "html.prism-mod-my-module body { background: var(--prism-bgColor) !important; }\nhtml.prism-on-hideHeader header { display: none !important; }"
  }
}
```

### Required fields

| Field      | Type     | Description |
|------------|----------|-------------|
| `id`       | string   | Unique identifier (lowercase, hyphenated). Used in CSS classes (`prism-mod-<id>`) and storage keys. |
| `name`     | string   | Human-readable name shown in the dashboard and popup. |
| `matches`  | string[] | Non-empty array of [match patterns](#match-patterns). The module activates on any URL that matches at least one. |
| `settings` | array    | List of setting definitions. May be empty `[]` for a CSS-only module with no controls. |
| `styles.css` | string | The CSS to inject when the module is active. Uses the conventions below. |

### Optional fields

| Field         | Type   | Default | Description |
|---------------|--------|---------|-------------|
| `version`     | string | `"1.0.0"` | Semver-ish version. Shown in the dashboard. |
| `author`      | string | `""`    | Shown in the dashboard. |
| `description` | string | `""`    | Longer description shown in the dashboard. |
| `emoji`       | string | `"◆"`   | Single emoji or character shown as the module icon. |
| `accent`      | string | —       | Hex color, informational only (not currently used by the engine). |
| `builtIn`     | bool   | `false` | Should always be `false` for repo modules. Reserved for hypothetical future bundled modules. |

### Match patterns

Glob-style patterns. `*` matches any sequence of characters. Examples:

| Pattern | Matches |
|---------|---------|
| `*://*.wikipedia.org/*` | any Wikipedia language subdomain, any path |
| `*://www.youtube.com/*` | only `www.youtube.com` |
| `*://duckduckgo.com/*` | duckduckgo.com, any path |
| `*://*.github.com/*` | github.com and any subdomain |
| `*://*/*` | every website (use sparingly) |

Multiple patterns can be combined; the module activates if **any** of them
match the current URL.

---

## Setting types

Each entry in `settings` becomes a control in the Dashboard and (if
`"quick": true`) in the popup. Every setting automatically exposes itself to
your CSS as a variable and/or HTML class (see [CSS conventions](#css-conventions)).

### `toggle`

```json
{ "id": "hideHeader", "label": "Hide header", "type": "toggle", "default": false, "section": "Layout" }
```

- Control: switch
- CSS class on `<html>`: `prism-on-hideHeader` when true, `prism-off-hideHeader` when false
- No CSS variable (booleans aren't useful as values)

### `select`

```json
{
  "id": "themeMode",
  "label": "Theme",
  "type": "select",
  "options": [
    { "value": "dark", "label": "Dark" },
    { "value": "light", "label": "Light" }
  ],
  "default": "dark",
  "section": "Appearance"
}
```

- Control: dropdown
- CSS class on `<html>`: `prism-themeMode-dark` or `prism-themeMode-light`
- CSS variable: `--prism-themeMode: dark` (the raw value, useful in `content:` etc.)

### `color`

```json
{ "id": "bgColor", "label": "Background", "type": "color", "default": "#1c1c1e", "section": "Theme" }
```

- Control: native color picker
- CSS variable: `--prism-bgColor: #1c1c1e`

### `text`

```json
{ "id": "logoText", "label": "Logo text", "type": "text", "default": "Prism", "placeholder": "Your brand", "section": "Logo" }
```

- Control: text input
- CSS variable: `--prism-logoText: Prism`

### `number`

```json
{ "id": "resultCount", "label": "Results per page", "type": "number", "default": 20, "section": "Behavior" }
```

- Control: number input
- CSS variable: `--prism-resultCount: 20`

### `slider`

```json
{ "id": "contentWidth", "label": "Content width", "type": "slider", "min": 600, "max": 1400, "step": 50, "unit": "px", "default": 900, "section": "Reading" }
```

- Control: range slider with value display
- CSS variable: `--prism-contentWidth: 900`
- `unit` is appended to the value display only (not to the variable)

### `logo-grid`

```json
{
  "id": "secretLogo",
  "label": "Secret logo",
  "type": "logo-grid",
  "options": [
    { "value": "none", "label": "None", "emoji": "∅" },
    { "value": "frieren", "label": "Frieren", "emoji": "🧝‍♀️" },
    { "value": "halloween", "label": "Halloween", "emoji": "🎃" }
  ],
  "default": "none",
  "section": "Logos"
}
```

- Control: grid of emoji tiles (used for the DuckDuckGo secret-logo picker)
- CSS class on `<html>`: `prism-secretLogo-frieren` (etc.)
- CSS variable: `--prism-secretLogo: frieren`
- `options[].emoji` is shown as the tile icon; `options[].label` is shown below
- Optional `options[].url` is informational (the engine doesn't use it)

### Common optional setting fields

| Field        | Type   | Description |
|--------------|--------|-------------|
| `section`    | string | Group label in the dashboard. Settings with the same `section` are grouped under a heading. |
| `quick`      | bool   | If `true`, the setting also appears in the popup quick-tweaks panel for the active site. |
| `depends`    | string | ID of another setting (must be a `toggle`). This setting is greyed-out/disabled unless that toggle is on. |
| `placeholder`| string | For `text` settings: placeholder shown in the input. |

---

## CSS conventions

This is the heart of the engine. When a module is active on a page, Prism
adds to `<html>`:

1. **A module marker class:** `prism-mod-<id>` (e.g. `prism-mod-my-module`).
2. **Per-toggle classes:** `prism-on-<key>` or `prism-off-<key>` for every `toggle` setting.
3. **Per-select classes:** `prism-<key>-<value>` for every `select` and `logo-grid` setting (value is sanitized to lowercase `[a-z0-9-]`).

And on `:root`:

4. **CSS variables:** `--prism-<key>: <value>` for every `color`, `text`, `number`, `slider`, `select`, and `logo-grid` setting.

So your CSS can do:

```css
/* Scope everything under the module marker */
html.prism-mod-my-module body {
  background: var(--prism-bgColor) !important;
  font-size: calc(var(--prism-fontScale, 100) * 1%) !important;
}

/* Toggle-gated blocks */
html.prism-mod-my-module.prism-on-hideHeader header {
  display: none !important;
}

/* Per-value styling for selects */
html.prism-mod-my-module.prism-themeMode-dark body { background: #000 !important; }
html.prism-mod-my-module.prism-themeMode-light body { background: #fff !important; }
```

### Template substitution (optional)

For values that need to appear as raw text (e.g. an image URL inside
`content: url(...)`), Prism also substitutes `{{key}}` in your CSS with the
setting's raw value:

```css
html.prism-mod-my-module .site-logo {
  content: url("{{customLogoUrl}}");
}
```

When `customLogoUrl` is `https://example.com/logo.png`, the injected CSS
becomes `content: url("https://example.com/logo.png");`. Empty values become
empty strings.

### Sanitization

Setting values used in class names are sanitized to lowercase
`[a-z0-9_-]`. Non-matching characters become `-`. So a `select` value of
`"My Cool Theme"` becomes the class `prism-themeMode-my-cool-theme`.

---

## Authoring a module, step by step

1. **Pick a target site.** Open it, F12 the elements you want to change, note their selectors.
2. **Create `<your-module-name>.json`** in the [`modules/`](./modules) folder. Start with the minimum viable shape above.
3. **Define your settings.** Each one should map to a discrete visual or behavioral change.
4. **Write your CSS** under the `prism-mod-<id>` scope, using the conventions above. Test by setting reasonable defaults.
5. **Commit and push.** In Prism's Dashboard → Repository, click **Sync now**. Your module will appear in the list.
6. **Install it** (one click). It now runs locally — you can edit it in the Dashboard editor or tweak its values in the module page.
7. **Iterate.** Push updates to the repo; users click **Update from repo** on each module to pull new versions (their setting values are preserved).

### Tips

- **`!important` is your friend.** Site styles often use high specificity; `!important` on Prism overrides is expected and fine.
- **Don't fight JavaScript-rendered DOM.** If a site renders late, scope your CSS to stable ancestor selectors. For JS-driven features (logo replacement, SPA navigation), you need a built-in content script — see below.
- **Test with multiple values.** Toggle each setting on and off, try each select value, drag each slider to extremes.
- **Keep CSS readable.** Group rules by section with comments; future-you (and other contributors) will thank you.

---

## Built-in content scripts

Pure CSS can't do everything (e.g. replacing a YouTube logo that re-renders
on SPA navigation, or triggering a DuckDuckGo secret-logo search). For those
cases, Prism ships built-in content scripts that activate **only when the
matching module is installed and enabled**:

| Module ID | Site | Built-in JS features |
|-----------|------|----------------------|
| `duckduckgo` | `*://duckduckgo.com/*` | Native settings sync via DDG's `ad` cookie (theme, colors, font, behavior); secret-logo trigger (opens a background search tab so DDG sets its own logo cookie); custom logo image URL replacement |
| `youtube` | `*://www.youtube.com/*` | Logo replacement (text/image/preset/hidden/random) with SPA navigation handling; Konami code party mode; toast helper |

These scripts look up the module definition from local storage, so they pick
up your settings automatically. If the matching module isn't installed,
the scripts no-op.

**To add a built-in content script for a new site**, the extension itself
needs to be modified (add a new content script entry to `manifest.json` and
a new file in `content/`). This is by design — Firefox doesn't allow remote
code execution, so JS features can only ship inside the extension package.

---

## Repo sync behavior

- Sync lists files in the configured path via the GitHub Contents API
  (`https://api.github.com/repos/<owner>/<repo>/contents/<path>?ref=<branch>`).
- Only files ending in `.json` are considered. README.md, scripts, images,
  dotfiles — all ignored.
- Each candidate file is fetched from its `download_url`
  (`https://raw.githubusercontent.com/...`) and validated as a module.
- Invalid files are reported as "skipped" with their error; sync continues
  with the rest.
- **GitHub rate-limits unauthenticated API requests to 60/hour per IP.** For
  a single user this is plenty. If you sync often, consider proxying through
  a CDN or using a Personal Access Token (not currently supported by Prism).
- **If the repo is unreachable**, sync fails gracefully with an error
  message. Your local modules keep working unchanged. Nothing is deleted on
  failure.

---

## Local vs. repo modules

Every module in Prism's local storage is just a "local module". Some have
metadata noting where they came from:

| Field          | Meaning |
|----------------|---------|
| `_sourceUrl`   | The raw GitHub URL the module was downloaded from (or `null` for modules created in the editor). |
| `_sourceName`  | The filename in the repo (e.g. `duckduckgo.json`). |
| `_sourceSha`   | The git blob SHA from the GitHub API (used for change detection). |
| `_lastSynced`  | Timestamp of the last successful download. |
| `_localEdited` | `true` if the user has edited the local copy in the editor. |

### Precedence

- **Local always wins.** When you sync, modules that already exist locally
  are **not** overwritten. They show up as "already local" in the sync
  results, with an **Update from repo** button if you explicitly want to
  pull the repo version (which discards your local edits — Prism asks for
  confirmation).
- This means: fork a module, edit it, push to your own repo, reconfigure
  Prism to point at your fork — your edits stay yours.

### Editing a local module

In the Dashboard → Module page → **Edit module definition**. This opens the
visual editor pre-loaded with the module's data. Save to update the local
copy. Your setting values (the user-facing values, not the schema) are
preserved across edits.

### Deleting a local module

Dashboard → Module page → **Delete module**. Removes the module definition
and all its setting values. If it came from the repo, you can re-install it
on the next sync.

### Updating from the repo

Dashboard → Module page → **Update from repo** (only shown for modules with
a `_sourceUrl`). Fetches the latest version from GitHub and overwrites the
local module definition. **Your setting values and enabled state are
preserved** — only the schema and CSS get refreshed. Prism asks for
confirmation because any local edits to the schema/CSS are lost.

---

## Examples

The [`modules/`](./modules) folder contains three reference modules:

| File | What it demonstrates |
|------|----------------------|
| [`duckduckgo.json`](./modules/duckduckgo.json) | A module paired with a built-in content script. Native settings sync (ad cookie), 804 verified secret-logo triggers, layout tweaks, custom colors. |
| [`youtube.json`](./modules/youtube.json) | Another built-in-script module. Logo replacement (text/image/preset/hidden/random), accent color, easter eggs (rainbow progress, CRT, vaporwave, Konami code). |
| [`wikipedia-tweaks.json`](./modules/wikipedia-tweaks.json) | A pure CSS+settings module — no JavaScript. Use this as a template for your own modules for sites Prism doesn't have a built-in script for. |

### A minimal CSS-only module

```json
{
  "id": "github-compact",
  "name": "GitHub Compact",
  "version": "1.0.0",
  "author": "you",
  "description": "Tightens GitHub's layout: denser lists and a custom accent.",
  "emoji": "🐙",
  "matches": ["*://github.com/*"],
  "settings": [
    { "id": "dense", "label": "Dense lists", "type": "toggle", "default": true, "section": "Layout" },
    { "id": "accentColor", "label": "Accent", "type": "color", "default": "#6e40c9", "section": "Theme" }
  ],
  "styles": {
    "css": "html.prism-on-dense .Box-row { padding: 6px 12px !important; }\nhtml.prism-mod-github-compact a { color: var(--prism-accentColor); }\nhtml.prism-mod-github-compact .btn-primary { background-color: var(--prism-accentColor) !important; border-color: var(--prism-accentColor) !important; }"
  }
}
```

Drop that in `modules/github-compact.json`, push, sync, install — done.

---

## License

This repository's contents (the README and any module `.json` files) are
provided as-is for use with the Prism extension. The Prism extension itself
is separate software. See the extension's own license for its terms.

*Refract the web, your way.*
