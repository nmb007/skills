# SFC blocks: HTML, CSS, JavaScript, i18n, SVG, accessibility

Sources:
- [HTML](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184966)
- [CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188217)
- [JavaScript](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188221)
- [String internationalisation](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184974)
- [SVG images](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184970)
- [SVG icons](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185112)
- [Accessibility](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185963)

---

## HTML — semantic markup

- Use the **most appropriate element** (material honesty): links for navigation, `button type="button"` for actions, native inputs for form controls.
- Labels: `label for="id"` linked to inputs; required fields use visible asterisk + screen-reader text.
- Groups: `role="radiogroup"` + `aria-labelledby` for radio/checkbox sets.
- ARIA on custom widgets where no native element exists.
- Avoid multiple root nodes where the linter/runtime expects a single root; use one wrapper if needed.

---

## CSS — SCSS + BEM

- Styles live in `<style lang="scss">` within the SFC (or shared SCSS imported from `js/`).
- **BEM** naming: block `tui-{name}`, elements `__element`, modifiers `--modifier`.
- Each Vue component is treated as a **block**; `tui-` prefix for core; plugins use their own block names.
- **CSS variables** for themeable values; SCSS variables for compile-time-only values.
- No Shadow DOM / scoped styles in core — global namespace; BEM reduces collisions. Legacy + Bootstrap still affect Learn pages — components may include defensive overrides.

Theme-wide variable overrides are **`totara-tui-theme`** territory, not bundle authoring.

---

## JavaScript

Tui uses ESLint with project rules; avoid patterns that fight the bundler or browser support targets.

### Static imports (default)

```js
import Button from 'tui/components/buttons/Button';
import MyWidget from 'local_myplugin/components/MyWidget';
```

### Async module

```js
const lib = await tui.import('local_myplugin/js/my_lib');
const fn = tui.defaultExport(lib); // or lib.default for ES modules
```

### Async Vue component (preferred for dynamic UI)

```js
// Returns a component object immediately; shows loader until ready
this.modalContent = tui.asyncComponent('mod_foo/components/Example');
```

Also loads lang strings for that component automatically.

```js
const Comp = await tui.loadComponent('mod_foo/components/Example');
```

### Async style

Prefer **async/await** and Promises over callback chains.

Cross-bundle async/static usage still requires **tui.json dependencies** and PHP plugin dependencies where applicable.

---

## Internationalisation

In SFC templates and script:

```vue
<template>
  <span>{{ $str('reload', 'local_myplugin') }}</span>
</template>
```

```js
this.$str('key', 'component', param);
this.$tryStr('key', 'component'); // null if missing
this.$hasStr('key', 'component');
```

Totara 13–18: declare keys in `<lang-strings>` block. Imported child components must be listed in `components: {}` so their strings preload — or load via async component / `tui.loadRequirements()`.

Programmatic load:

```js
import { langString, loadLangStrings } from 'tui/i18n';
```

Do not use strings from other components without declaring plugin/component dependencies.

---

## SVG images

- Prefer **dedicated `.vue` files** wrapping one SVG so themes can replace them.
- Set `viewBox` for scaling; style with classes + CSS variables.
- Optimise with [SVGOMG](https://jakearchibald.github.io/svgomg/); remove IDs; use `currentColor` where icons should inherit text colour.

---

## SVG icons

Icons: SVG file under `icons/internal/obj/` (e.g. `bootstrap-icons/`, `totara/`), wrapped in `client/component/.../src/components/icons/`.

```js
import icon from './internal/obj/bootstrap-icons/plus.svg';
import { createIconComponent } from 'tui/lib/svg_icon';

export default createIconComponent(icon);
```

Name wrappers by **semantic** meaning (`Delete.js`), not appearance (`TrashBin.js`).

`fill` and `stroke` should be `currentColor` for theme colouring.

---

## Accessibility (component author)

- Reuse core components (Uniform, Modal, Tabs, etc.) — they include keyboard and ARIA patterns.
- Uniform + FormRow auto-link labels; use `InputSet` / `FieldGroup` for multiple inputs in one row.
- Test with keyboard; screen readers (NVDA, VoiceOver) for critical flows.
- Query tests by **role + accessible name** (see testing reference).

Full product ACR docs are separate — do not duplicate WCAG tables here.
