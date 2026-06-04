# Style Overrides

Sources:
- [Overwriting CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184812/Overwriting+CSS)
- [CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188217/CSS)
- [Tui front-end framework](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188209) — Theme UI overrides, cascade

---

## CSS variables (preferred)

Many Totara aspects expose **CSS custom properties** (colours, font sizes, line heights, gap base).

**Rule:** If a variable exists, override the variable — do not target elements with high-specificity selectors.

**Dual placement** (common mistake to skip one side):

| Stack | Location |
|-------|----------|
| Legacy / server | `server/theme/{name}/scss/theme/variables.scss` |
| TUI | `client/component/theme_{name}/src/global_styles/_variables.scss` |

**Reference list:** `server/theme/legacy/scss/totara/css_variables/`

**SCSS functions that fail on CSS variables:** `darken()`, `lighten()`, `mix()`, `button-variant()` — use `theme:derive` instead (see [tui-theme-overrides.md](tui-theme-overrides.md)).

---

## CSS load order (TUI themes)

1. Legacy CSS  
2. SCSS variables (if present)  
3. Tui core — CSS variables + CSS  
4. Each Tui component  
5. Each Tui theme component (`theme_*`)  
6. Theme Appearance UI output (CSS variable overrides)  
7. **Custom CSS** admin input (last; selectors and variables)

---

## Theme Appearance runtime overrides

Admins change branding in UI → system writes CSS variable overrides (near end of cascade).

**Custom CSS box** is the only layer after Theme Appearance output — use for final per-site tweaks.

Prior Totara versions used PHP string replacement + runtime SCSS parsing; TUI themes use CSS variables instead.

---

## Plain CSS sheets

Minor changes without SCSS compile:

1. Add `server/theme/{name}/styles/mystyles.css`
2. Register in `config.php`: `$THEME->sheets = ['mystyles'];`
3. Add `mystyles-rtl.css` if RTL support required

Changes need cache purge unless themedesignermode is on (not covered in this skill).

---

## SCSS child theme pattern

Totara does **not** use Moodle’s “include all parent CSS sheets” inheritance for SCSS themes.

Instead:

1. Set `$THEME->parents_exclude_sheets` appropriately (exclude parent sheets you replace)
2. In `scss/totara.scss`, import parent `theme/variables` and `theme/styles`, then child files
3. Create child `scss/theme/variables.scss` and `scss/theme/styles.scss`
4. **Order:** all `variables` imports first, then all `styles` imports

Major customization: copy parent `totara.scss` and trim imports.

**LESS vs SCSS:** SCSS is imperative — variables must be defined **before** use (unlike LESS). This is why the server-side `totara.scss` entry point separates variable imports from style imports — loading variables after styles would have no effect on the already-evaluated styles.

---

## TUI SCSS variable load order (two-phase, implicit)

In TUI there is no explicit entry point like `totara.scss` — the load order is handled implicitly:

**Phase 1 — Variables only (in this order):**
All SCSS variables defined in `_variables.scss` from TUI core → parent themes → your theme.

**Phase 2 — Remaining code (in this order):**
All remaining code from `_variables.scss` + `static.scss` from parent themes → your theme.

This means your variable overrides in `_variables.scss` are evaluated before any styles run, so all inherited styles pick up your values correctly. No manual ordering is required — just place overrides in `_variables.scss`.

---

## Compiling Legacy SCSS

From `server/` after `npm ci` or `npm install`:

```bash
npx grunt css
```

Faster when run from the theme directory: `cd server/theme/mytheme && npx grunt css` (paths relative to Totara root may vary by setup).

Re-run after **upgrades** — base theme SCSS may change.

---

## TUI theme bundle rebuild

After editing `client/component/theme_{name}/src/**`:

run `npm run tui-build-dev theme_mytheme` or `tui-build-watch`.

This skill does not document webpack/npm build internals.

---

## BEM and SCSS in TUI components

- Components use **BEM** (`tui-` prefix on core blocks only)
- SCSS in SFC `<style>` blocks; not Vue scoped styles (breaks theme inheritance)
- Global TUI variables: `client/component/tui/src/global_styles/_variables.scss`
- Static global CSS: `client/component/tui/src/global_styles/static.scss`
- Import in static SCSS via frankenstyle paths, e.g. `@import 'theme_mytheme/oldstyles/totara/nav';`

**Style isolation:** No Shadow DOM / scoped styles — global namespace; BEM reduces collisions. TUI components include Legacy/Bootstrap resets as needed.

---

## Sizing (Totara 19+)

Root font size follows browser default (~16px). Use `rem-px(32)` helper where available.

Totara 18 and earlier: root 10px; simple `rem` math.

**Units:** Prefer `rem` over `px` wherever possible (border widths are a reasonable exception). `rem` scales with the user's root font size, improving accessibility.

**Spacing:** Use the `gap()` helper rather than raw pixel values. One `gap()` unit equals 4px.

```scss
gap(1)  // 4px
gap(2)  // 8px
gap(4)  // 16px
```

---

## IE11 / Totara 16 and earlier

CSS variables resolved at build time for IE11; root-level variables only in those versions.

---

## When to read other references

| Need | File |
|------|------|
| SFC style block overrides | [tui-theme-overrides.md](tui-theme-overrides.md) |
| Admin settings backend | [theme-appearance-backend.md](theme-appearance-backend.md) |
| Mustache/layout | [markup-and-php-overrides.md](markup-and-php-overrides.md) |
