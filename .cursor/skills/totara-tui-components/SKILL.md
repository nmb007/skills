---
name: totara-tui-components
description: >-
  Totara TUI component bundles: scaffold client/component/{frankenstyle}, tui.json,
  Vue SFCs (pages, presentational, renderless), v1/v2 file layout, tui-init,
  tui-build-watch, imports from tui/components, tui.asyncComponent, MVC tui_view,
  and Jest component tests. Use when creating or modifying TUI .vue files, Tui bundles,
  or wiring pages via totara_mvc. Not for theme overrides (totara-tui-theme),
  Vue Apollo (totara-tui-apollo), or PHP GraphQL (totara-graphql-api).
---

# Totara TUI Components

## Scope

Covers **TUI bundle authoring** in `client/component/{frankenstyle}/` — Vue single-file components, `tui.json`, build workflow, and minimal **PHP glue** (`totara_mvc` + `tui_view`) to render a page.

Does **not** cover:

- **Theme branding / overrides** → `totara-tui-theme` skill
- **GraphQL in `.vue`** → `totara-tui-apollo` skill
- **PHP GraphQL schema/resolvers** → `totara-graphql-api` skill
- **Legacy AMD / Mustache / modal registry** — pre-TUI; do not use for new UI

Official hubs: [Tui front-end framework](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188209) | [Quick-start: Creating your first component](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184960)

---

## Mental model

| Term | Meaning |
|------|---------|
| **Tui component (bundle)** | High-level unit under `client/component/{frankenstyle}/` with `tui.json` (e.g. `local_myplugin`, `mod_perform`) |
| **Vue SFC** | Low-level `.vue` file inside a bundle (e.g. `PingPage.vue`, `PingStatus.vue`) |

Totara splits **GPL server code** (`server/`) from **proprietary client code** (`client/`). Tui lives in `client/` and talks to core via designated channels (GraphQL, MVC), not direct core coupling. See [Totara 13 code reorganisation](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188190).

Complex SPAs may use XState (`tui_xstate`) — see [Architectural overview](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188053) only when building state-machine-driven flows.

---

## Directory layout

| Artifact | Path |
|----------|------|
| Bundle root | `client/component/{frankenstyle}/` |
| Manifest | `tui.json` (directly under bundle root — required for v2) |
| v1 pages | `src/pages/` |
| v1 UI | `src/components/` |
| v1 JS | `src/js/` |
| v2 feature (Totara 19+) | `src/{feature}/` — page SFCs should end in `Page` |
| Custom webpack | `src/build.config.js` (Totara 13–18) or `build.config.js` at bundle root |
| Apollo type policies | `js/internal/entry.js` + `"entry"` in `tui.json` → see `totara-tui-apollo` |

Import map example: `totara_catalog/explore/ExplorePage` → `totara_catalog/src/explore/ExplorePage.vue`.

---

## Quick-start: new Tui bundle + page

```
- [ ] 1. config.php totara_tui dev settings + npm ci (see build reference)
- [ ] 2. npm run tui-init {frankenstyle} {vendor}
- [ ] 3. npm run tui-build-watch (restart watch after adding a new bundle)
- [ ] 4. Add page SFC as defined in v2 structure in the relevant feature folder
- [ ] 5. Add presentational / container child components as needed
- [ ] 6. PHP: controller + tui_view::create('{frankenstyle}/explore/ExplorePage', $props)
- [ ] 7. Add tui.json "dependencies" if statically importing another bundle
- [ ] 8. If server data needed → totara-graphql-api + totara-tui-apollo (no inline fetch/gql)
- [ ] 9. Add Jest spec next to component (see testing reference)
```

For end-to-end GraphQL + Apollo wiring, use totara-graphql-api and the totara-tui-apollo skill.

---

## Hard rules

1. **Frankenstyle bundle name matches the plugin** — `local_myplugin` ↔ `server/local/myplugin`.
2. **PascalCase for `.vue` filenames** — e.g. `PingPage.vue`, not `ping-page.vue`.
3. **Import core UI from `tui/components/...`** — own bundle from `{frankenstyle}/components/...` or `{frankenstyle}/pages/...`.
4. **Static cross-bundle deps** — `"dependencies": ["totara_comment"]` in `tui.json`; **dynamic** loads → `tui.asyncComponent('mod_foo/components/Example')`.
5. **Rebuild after src changes** unless `tui-build-watch` is running.
6. **No theme override workflows here** — CSS variable / theme `.vue` overrides → `totara-tui-theme` skill.
7. **No inline `gql` or raw `fetch('/graphql')`** for Totara data — use persisted imports + Vue Apollo (`totara-tui-apollo`) skill.
8. **When using Apollo**, prefer `v-if="dataField"` over `v-if="$apollo.loading"` on main content (refetch flicker).
9. **GraphQL is the server channel for Tui** — not legacy AJAX scripts (see php-integration reference).
10. **New features in Totara 19+** — should ALWAYS use v2 feature-based layout under `src/{feature}/`.

---

## Import path conventions

| Import | Resolves to |
|--------|-------------|
| `tui/components/buttons/Button` | Core Tui bundle (`client/component/tui/...`) |
| `local_foo/explore/filtering/FilterBar` | Same bundle's `src/explore/filtering/FilterBar.vue` |
| `local_foo/explore/ExplorePage` | Same bundle's page SFC |
| `totara_webapi/graphql/status_nosession` | Persisted query (Vue Apollo — see `totara-tui-apollo`) skill |

Module resolution is global per loaded bundle — see [component-structure.md](references/component-structure.md).

---

## Component roles

| Role | Purpose | Typical path |
|------|---------|--------------|
| **Page** | PHP entry via `tui_view`; layout + wiring | `src/{feature}/FooPage.vue` |
| **Presentational** | Reusable UI, limited logic | `src/{feature}/filtering/FilterBar.vue` |
| **Container** | Composes children + local state | Same as presentational, larger scope |
| **Renderless** | State/behavior only; scoped slot for markup | e.g. `tui/.../form/Upload.vue` |

See [Component structure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187834).

---

## `tui.json` skeleton

```json
{
  "component": "local_myplugin",
  "vendor": "myvendor",
  "dependencies": ["totara_comment"],
  "entry": "./js/internal/entry.js"
}
```

- `component` — frankenstyle name (required).
- `vendor` — build categorisation (required).
- `dependencies` — other Tui bundles loaded with this one (optional).
- `entry` — Apollo cache policies or bundle bootstrap (optional; see apollo skill).

---

## npm commands

| Command | Purpose |
|---------|---------|
| `npm run tui-init {frankenstyle} {vendor}` | Scaffold new bundle + `tui.json` |
| `npm run tui-build-dev` | Development builds (all or one component) |
| `npm run tui-build-watch` | Rebuild on save (per changed bundle) |
| `npm run tui-build-prod` | Production/minified builds |
| `npm run tui-test` | Run Jest tests |
| `npm run tui-test-watch` | Jest watch mode |

Docker + mutagen (`totara-docker-dev` on macOS): prefix with `t` — e.g. `tnpm run tui-build-watch`.

---

## Related skills

| Need | Skill |
|------|-------|
| New ajax query/mutation + PHP resolvers | `totara-graphql-api` |
| `apollo:` / `@vue/apollo-composable` in `.vue` | `totara-tui-apollo` |
| Custom theme / override core `.vue` in theme | `totara-tui-theme` |

---

## Reference files

| File | When to read |
|------|----------------|
| [references/build-and-dev.md](references/build-and-dev.md) | config.php, dev mode, build/watch, IDE |
| [references/component-structure.md](references/component-structure.md) | v1/v2 layout, SFC blocks, rendering, renderless |
| [references/php-integration.md](references/php-integration.md) | MVC, `tui_view`, `totara_tui`, dependencies |
| [references/sfc-blocks.md](references/sfc-blocks.md) | HTML, CSS/BEM, JS async APIs, i18n, SVG, a11y |
| [references/testing-and-building-blocks.md](references/testing-and-building-blocks.md) | Jest/VTL, Uniform, Weka pointer |

---

## When stuck

1. Confirm `tui-build-watch` is running and development mode is on (`totara_tui` settings).
2. Check `client/component/{name}/build/` updated after save.
3. For PHP page blank/wrong component — verify `tui_view::create()` path matches SFC import path.
4. For cross-bundle import errors — add `dependencies` in `tui.json` and matching PHP plugin dependency.
5. After backend GraphQL schema changes — purge Totara caches in dev; see totara-graphql-api skill.
