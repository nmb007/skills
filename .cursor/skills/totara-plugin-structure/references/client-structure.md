# Client Structure — Detailed Reference

Totara splits GPL server code (`server/`) from proprietary client code (`client/`) because they use different, differently-licensed technologies.

## Bundle root: `client/component/<frankenstyle>/`

The frankenstyle name of the client bundle **must match** the server plugin (e.g. `local_myplugin` ↔ `server/local/myplugin/`).

| Path | Purpose |
|------|---------|
| `tui.json` | Registers and describes the bundle with Tui (required) |
| `src/` | Vue and JS source code |
| `build/` | Compiled output from `tnpm run tui-build` do not edit manually |
| `upgrade.txt` | Append-only log of client-side breaking changes |

---

## `tui.json` fields

```json
{
  "component": "local_myplugin",
  "vendor":    "myvendor",
  "dependencies": ["totara_comment"],
  "entry": "./js/internal/entry.js"
}
```

| Field | Required | Purpose |
|-------|----------|---------|
| `component` | Yes | Frankenstyle name — must match the plugin exactly |
| `vendor` | Yes | Build categorisation string |
| `dependencies` | No | Other Tui bundles that must be loaded alongside this one (static imports) |
| `entry` | No | Apollo cache type policies or bundle bootstrap |

---

## `src/` layout — v1 vs v2

### v1 layout (Totara 18 and earlier)

```
src/
├── pages/        # Page-level Vue SFCs wired via tui_view
├── components/   # Reusable presentational and container components
└── js/           # Additional JS libraries and definitions
```

### v2 layout (Totara 19+) — preferred for all new plugins

```
src/
└── {feature}/             # One folder per feature area
    ├── {Feature}Page.vue  # Page SFC (must end in "Page" for easy identification)
    └── use{Feature}.js    # High-level composable implementing page logic
```

**Use v2 for any new plugin.** The feature-based structure separates logic from UI and improves developer experience. There are currently few examples of it in the codebase since it is relatively new.

---

## Core Tui resources

| Resource | Path |
|----------|------|
| Reusable UI components | `client/component/tui/src/components/` |
| Global CSS variables (typography, colour, spacing) | `client/component/tui/src/global_styles/variables/` |

Import core components in your Vue files:

```js
import Button from 'tui/components/buttons/Button';
```

Use the CSS variables from `tui/src/global_styles/variables/` (via the active theme) to maintain visual consistency and an accessible baseline across Inspire, Ventura, and Legacy themes.

---

## Dynamic cross-bundle imports

For **static** cross-bundle dependencies declare them in `tui.json` `"dependencies"`. For **dynamic/lazy** loading use:

```js
tui.asyncComponent('mod_foo/components/Example')
```