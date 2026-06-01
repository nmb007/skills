# Component structure and rendering

Sources:
- [Component structure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187834)
- [Tui component file structure](https://totara.atlassian.net/wiki/spaces/DEV/pages/854687876)
- [Component rendering](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187035)
- [State management](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187041)

---

## Separation of concerns

Legacy Totara split HTML (Mustache), CSS (LESS), and JS (AMD) by technology. Tui splits by **Vue SFC** — each file can contain template, script, style, and lang strings together.

---

## Single-file component blocks

Standard Vue blocks plus Totara additions:

```vue
<template>
  <!-- markup -->
</template>

<script>
import Button from 'tui/components/buttons/Button';

export default {
  components: { Button },
  props: { /* from PHP or parent */ },
  data() { return { /* local state */ }; },
  methods: { /* ... */ },
};
</script>

<style lang="scss">
/* BEM-scoped styles */
</style>

<lang-strings>
{
  "local_myplugin": ["reload", "title"]
}
</lang-strings>
```

Totara 13–18: strings used in the component must be declared in `<lang-strings>` (preloaded before mount). Newer versions may use centralized i18n — match surrounding code in the bundle.

Custom webpack blocks are possible via `build.config.js` (e.g. extra loaders).

---

## v1 structure (Totara 13+)

```
client/component/local_myplugin/
  tui.json
  src/
    pages/           ← PHP entry SFCs
    components/      ← reusable UI (group by feature)
    js/              ← non-Vue modules, constants, helpers
```

Example:

```
src/pages/Ping.vue
src/components/ping/PingStatus.vue
src/js/constants.js
```

---

## v2 structure (Totara 19+)

`tui.json` must sit **directly under the bundle root** (not inside `src/`).

```
client/component/totara_catalog/
  tui.json
  src/
    explore/               ← feature
      ExplorePage.vue      ← page ends in Page
      use_filtered_results.js
      components/
        FilterBar.vue
    filtering/             ← feature
      ...
```

Goals:

- Feature folders group pages, composables, and subcomponents
- Easier Vue 3 composables; logic separated from presentational SFCs
- Import `totara_catalog/explore/ExplorePage` → `src/explore/ExplorePage.vue`

**Single-feature plugins** (e.g. perform element, block): omit feature subfolder — place contents directly under `src/`.

v1 and v2 may **coexist** in one bundle during migration; new work should use v2.

---

## Atomic design (informal)

| Level | Role | Example |
|-------|------|---------|
| Low | One job | `Button.vue`, `Loader.vue` |
| Mid | Aggregates low-level | `DataTable.vue` |
| Page | Layout + entry from PHP | `ExplorePage.vue` |

Core Tui organises by function: `client/component/tui/src/components/form/`. Plugins often group by feature for clarity.

---

## Renderless components

Manage state/behaviour only; expose a **single scoped slot** for markup. Example: `client/component/tui/src/components/form/Upload.vue`.

```vue
<template>
  <slot :upload="upload" />
</template>
```

---

## Rendering pipeline

1. PHP embeds a minimal placeholder (component name + JSON props).
2. On page load, Tui finds placeholders and mounts the root SFC.
3. Child components are usually **static imports** in `<script>` and used in `<template>`.
4. UI updates via Vue reactivity when `data()` or props change.

### Cross-bundle usage

**Known dependency** — add to `tui.json`:

```json
"dependencies": ["totara_comment"]
```

**Unknown or lazy dependency** — use async component:

```js
this.dynamicComponent = tui.asyncComponent('mod_foo/components/Example');
```

```vue
<component :is="dynamicComponent" />
```

Shows a loading indicator until the bundle loads. See [sfc-blocks.md](sfc-blocks.md) for `tui.import` / `tui.loadComponent`.

### Slots

Pass markup into children (e.g. modal title/body slots) without prop drilling for HTML structure.

---

## State management

| Kind | Mechanism |
|------|-----------|
| Parent → child | `props` |
| Child → parent | `$emit` / events |
| Local UI | `data()` / composables |
| Server data | GraphQL + Vue Apollo → **`totara-tui-apollo`** skill |
| User prefs (limited) | `LocalStorage` via Tui helpers |

Major stateful core widgets: **Uniform** (forms), **Weka** (editor) — see [testing-and-building-blocks.md](testing-and-building-blocks.md).

---

## Reactivity tips

Vue may not detect:

- New properties added to existing objects
- Direct index assignment to arrays (`arr[i] = x`)

Use `Vue.set(object, key, value)` or make objects reactive with `Vue.observable()` when driving render from non-reactive data.

Only changes to observed data (from `data()`, props, computed) trigger re-renders.
