# TUI Theme Overrides

Sources:
- [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198/Creating+custom+themes)
- [Legacy theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185020)
- [Component structure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187834/Component+structure)
- [Deprecation guidelines](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187267/Deprecation+guidelines) — Theme-based TUI component overrides
- [Tui front-end framework](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188209) — Theme UI overrides

---

## Legacy theme bridge

**Legacy** styles non-TUI Learn UI (Bootstrap 3, AMD, Mustache-era pages). **Ventura/Inspire** add branding on top; **TUI** styles Perform/Engage (and growing Learn areas).

Custom child themes typically inherit **Ventura or Inspire → Legacy → base**.

- Do **not** move Legacy Learn styles into `client/` — client is for Vue-rendered UI.
- Legacy applies global Bootstrap resets; TUI components ship compensating overrides.
- Ventura supplies per-component overrides in client where Legacy globals conflict.

Pre-13 Roots/Basis/Less stack: see Legacy doc for historical context only — do not use for new themes.

---

## Theme TUI bundle layout

```
client/component/theme_mytheme/
├── tui.json
├── src/
│   ├── global_styles/
│   │   └── _variables.scss
│   └── ... overrides mirroring core paths ...
└── build/                    # generated
    └── css_variables.json    # after build, if theme:var comments present
```

**Rebuild** after src changes: `totara-tui-components` skill (`npm run tui-build-dev theme_mytheme`).

---

## Overriding core TUI components

Place override `.vue` files under `theme_{name}` using the same path as the source component in `tui` or plugin bundles.

You may override individual SFC blocks:

| Block | Effect |
|-------|--------|
| `<template>` | Replaces template; add `<script>` if new child components needed |
| `<style lang="scss">` | Adds/overrides styles (BEM) |
| `<script>` | Replaces script unless `extends` used |
| `<lang-strings>` | Merged with parent strings |

**Full replacement:** New `<template>` + `<script>` without extends replaces entire component definition (must supply template + strings; styles may still layer).

**Partial script extend:**

```vue
<script extends>
import Popover from 'tui/components/popover/Popover';
export default {
  components: { Popover },
};
</script>
```

- Use `extends` only to **add** components to `components` map
- Do **not** call private methods on extended components
- Adding `<script>` without `extends` disables template/script inheritance from parent

**Strings:** List all template strings in override `<lang-strings>` even if parent defines them — parent may drop unused strings in updates.

**Maintenance risk:** Overridden core components may deprecate blocks — monitor changelogs and `upgrade.txt` ([Deprecation guidelines](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187267/Deprecation+guidelines)).

**Not supported:** Overriding TUI core blocks from a **non-theme** plugin.

---

## Derived variables (`theme:var`, `theme:derive`)

In `_variables.scss`, precede variables with comments:

```scss
/* theme:var */
/* theme:derive adjust-hex-value-brightness(var(--color-state), -10) */
--color-state-hover: #3c6721;
```

- `theme:var` — include in Theme Appearance processing
- `theme:derive` — JS derivation (replaces SCSS `darken`/`lighten` on CSS variables)

Output: `client/component/theme_{name}/build/css_variables.json` consumed by Theme Appearance Vue pages.

Custom themes can add Theme Appearance SFCs for new/altered admin fields.

Reference derivations from parent theme (e.g. copy `theme:derive` patterns from `theme_ventura`).

---

## Variable rules

- Variables defined in Tui core/plugins; **themes override** values
- CSS variables may chain; updating a base updates dependents
- **Prefer CSS variables** over SCSS for theming
- A Tui component must not depend on variables from another component unless listed in `tui.json` `dependencies`

---

## Style block conventions

From [Component structure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187834/Component+structure):

```scss
.tui-myBlock {
  &__myElement {
    &--myModifier { }
  }
}
```

- Core blocks use `tui-` prefix — **do not** use `tui-` on custom theme-only components
- Prefer new class names when fully replacing a component template

Theme authors may define **custom SFC blocks** via `build.config.js` + webpack rule (advanced; see Component structure).

---

## CSS class naming for new components

If replacing a component entirely (new template + style + script), use a **non-`tui-`** class namespace to avoid collision with inherited styles.

---

## Overriding from plugins

TUI core/plugin overrides from **theme bundles only** — not from arbitrary third-party plugins (by design).
