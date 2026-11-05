# Override Decision Tree

Synthesized from:
- [Creating a theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184804/Creating+a+theme)
- [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198/Creating+custom+themes)
- [Overwriting CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184812/Overwriting+CSS)

---

## Rule

Pick the **lowest-numbered** approach that fully meets the requirement. Escalate only when the current level cannot express the change.

---

## 1. CSS variables (preferred)

**Use for:** Brand colours, typography scale, spacing bases, state colours (`--color-primary`, `--color-state`, hover/focus variants).

**Where:**

- Server: `server/theme/{name}/scss/theme/variables.scss` (or `.css` with custom properties)
- TUI: `client/component/theme_{name}/src/global_styles/_variables.scss`
- Catalogue: `server/theme/legacy/scss/totara/css_variables/`

**Why first:** Works across Legacy and TUI; chains of CSS variables compose; no SCSS functions on runtime values.

**Admin UI:** Add `/* theme:var */` and `/* theme:derive ... */` comments above variables — see [tui-theme-overrides.md](tui-theme-overrides.md) and [theme-appearance-backend.md](theme-appearance-backend.md).

---

## 2. Theme Appearance UI (runtime, admin)

**Use for:** Site/tenant branding without code deploy; derived hover/focus colours from base picks.

**Mechanism:** Settings saved dynamically; CSS variable overrides emitted **second-to-last** in cascade.

**Code prerequisite:** `theme:var` / `theme:derive` in theme `_variables.scss` → `build/css_variables.json`.

Not a substitute for shipping default variables in the theme plugin.

---

## 3. SCSS (compile-time)

**Use for:** Structural style changes, component SCSS imports, Bootstrap/Legacy integration.

**Pattern:**

- `$THEME->parents_exclude_sheets` — disable Moodle sheet inheritance
- `scss/totara.scss` imports parent then child `theme/variables` then `theme/styles`
- All **variables** imports before all **styles** imports

**Compile:** `tnpm ci` in `server/`, then `tgrunt css` (from `server/` or theme directory).

See [style-overrides.md](style-overrides.md).

---

## 4. Plain CSS (`$THEME->sheets`)

**Use for:** Small, isolated tweaks.

**Path:** `server/theme/{name}/styles/{sheet}.css` — register in `config.php` `$THEME->sheets` (include `-rtl` variant if needed).

No SCSS compile; purge cache or themedesignermode to see changes.

---

## 5. Custom CSS box (admin)

**Use for:** Per-site overrides by administrators.

**Cascade:** Last — overrides all prior CSS including Theme Appearance variable output. Accepts CSS variables and selector rules.

---

## 6. TUI Single File Component blocks

**Use for:** Changing Vue UI (template, styles, strings, script) without forking entire plugins.

**Where:** Override files under `client/component/theme_{name}/` mirroring core component paths.

**Risk:** Upstream component deprecations — see [Deprecation guidelines](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187267/Deprecation+guidelines).

Details: [tui-theme-overrides.md](tui-theme-overrides.md).

---

## 7. Mustache templates

**Use when:** Layout or markup cannot be fixed with CSS alone.

**Path:** `server/theme/{name}/templates/{frankenstyle}/{template}.mustache`

Example: override `mod_foo/bar.mustache` → `theme/mytheme/templates/mod_foo/bar.mustache`.

See [markup-and-php-overrides.md](markup-and-php-overrides.md).

---

## 8. Page layouts

**Use when:** Page shell structure (regions, wrapper HTML) must change.

**Config:** `$THEME->layouts` in `config.php` → PHP files in `theme/{name}/layouts/`.

See [Page layouts](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184814/Page+layouts).

---

## 9. PHP renderers

**Use when:** HTML generation logic must change beyond templates.

**Pattern:** `theme_{child}_core_renderer extends theme_{parent}_core_renderer` (or `core_renderer` if parent has no extension).

Locations: `theme/{name}/renderer.php`, `theme/{name}/classes/`.

See [Overriding renderers](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184810).

---

## 10. HTML email Mustache

**Use for:** Email wrapper/branding only — separate from page UI.

**Path:** `server/theme/{name}/templates/core/email_html.mustache`

**Constraint:** Inline CSS only (email clients).

See [Customising HTML email](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184806/Customising+HTML+email).

---

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Heavy selector overrides when a CSS variable exists | Variable override |
| Copying entire parent `totara.scss` without need | Import parent variables/styles only |
| `tui-` prefixed classes on custom components | Own prefix |
| Core file edits | Theme/plugin overrides |
| Roots/Basis parents on new projects | Ventura/Inspire + Legacy |
