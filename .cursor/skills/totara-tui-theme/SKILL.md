---
name: totara-tui-theme
description: >-
  Totara child/custom themes: create theme plugins, set $THEME->parents (Ventura,
  Inspire, Legacy), override CSS variables, SCSS, TUI SFC blocks, Mustache, renderers,
  page layouts, Theme Appearance settings, and HTML email templates. Use when creating
  or customizing themes, theme inheritance, branding, or theme_* client components.
---

# Totara TUI Theme

## Scope

Covers **all theme customization levels** in Totara: server theme plugin, TUI `theme_*` bundle, admin Theme Appearance, and email HTML.

Does **not** cover:

- **TUI bundle authoring** (scaffold, `tui.json`, page SFCs) → `totara-tui-components`
- **GraphQL in theme settings API** (extending backend) → `totara-graphql-api`
- **Vue Apollo in theme pages** → `totara-tui-apollo`

---

## Version-specific parent themes

| Totara version | Recommended `$THEME->parents` |
|----------------|-------------------------------|
| **13–18** | `['ventura', 'legacy', 'base']` |
| **19+** | `['inspire', 'legacy', 'base']` |

`base` is injected if missing (font icons). Do **not** use Roots/Basis for new themes (deprecated/removed).

See [Themes and Totara](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184816/Themes+and+Totara).

---

## Quick-start: new child theme

```
- [ ] 1. Server: theme/{name}/config.php, version.php, lang/en/theme_{name}.php
- [ ] 2. Set $THEME->parents per version table above; rendererfactory = theme_overridden_renderer_factory
- [ ] 3. Client: client/component/theme_{name}/ with tui.json + src/global_styles/_variables.scss
- [ ] 4. Mirror colour/spacing variables in server scss/theme/variables.scss (Legacy) where needed
- [ ] 5. Rebuild theme TUI bundle: totara-tui-components skill (tui-build-dev theme_{name})
- [ ] 6. Legacy SCSS changes: npx grunt css from server/ (see style-overrides reference)
- [ ] 7. Purge theme cache / enable themedesignermode only when testing CSS (not documented here)
```

Naming: lowercase alpha start; underscores only with rules in [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198).

Example plugin: Bitbucket **example** theme — [FAQs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185118/FAQs).

---

## Override decision tree (prefer top)

Use the **highest row that solves the requirement** to minimize upgrade pain:

| Priority | Mechanism | When |
|----------|-----------|------|
| 1 | **CSS variables** | Colours, fonts, gaps — server + `theme_*` `_variables.scss` |
| 2 | **Theme Appearance UI** | Admin/tenant branding via `theme:var` / `theme:derive` |
| 3 | **SCSS** | `scss/theme/variables.scss`, `styles.scss`, `parents_exclude_sheets` |
| 4 | **Plain CSS** | Small deltas via `$THEME->sheets` |
| 5 | **Custom CSS box** | Site admin overrides — last in cascade |
| 6 | **TUI SFC blocks** | Override template/style/script/lang per component |
| 7 | **Mustache** | Layout/HTML not achievable with CSS |
| 8 | **Page layouts** | `$THEME->layouts` + `layouts/*.php` |
| 9 | **PHP renderers** | Output HTML beyond templates/CSS |
| 10 | **Email Mustache** | `templates/core/email_html.mustache` — inline CSS only |

Details: [references/override-decision-tree.md](references/override-decision-tree.md).

---

## Hard rules

1. **Prefer CSS variables** over SCSS and selector overrides.
2. **Mirror variables** in both `server/theme/{name}/` SCSS/CSS and `client/component/theme_{name}/src/global_styles/_variables.scss` when both stacks must match.
3. **Child themes** inherit **default** parent theme settings only — not saved parent Theme Appearance customizations.
4. **Do not** use `tui-` class prefix on new custom components (collision risk).
5. **SFC script `extends`** — only to add components to `components`; never call private methods on parent.
6. **Email templates** — inline styles only; not general UI CSS.
7. **Deprecated parents** — Roots/Basis; use Ventura/Inspire + Legacy instead.
8. **After `client/component/theme_*` src changes** — rebuild via `totara-tui-components` (not this skill’s build docs).

---

## Key paths

| Artifact | Path |
|----------|------|
| Theme config | `server/theme/{name}/config.php` |
| Legacy SCSS entry | `server/theme/{name}/scss/totara.scss` → `scss/theme/variables.scss`, `styles.scss` |
| Legacy CSS variable catalogue | `server/theme/legacy/scss/totara/css_variables/` |
| TUI theme bundle | `client/component/theme_{name}/` |
| Global TUI variables (core) | `client/component/tui/src/global_styles/_variables.scss` |
| Theme Appearance JSON | `client/component/theme_{name}/build/css_variables.json` (after build) |
| Mustache override | `server/theme/{name}/templates/{component}/{template}.mustache` |
| Email wrapper | `server/theme/{name}/templates/core/email_html.mustache` |

---

## CSS load order (TUI themes)

1. Legacy CSS  
2. SCSS variables (if any)  
3. Tui core CSS variables + CSS  
4. CSS variables and CSS from each Tui component  
5. CSS variables and CSS from each Tui theme.  
6. Theme Appearance UI — CSS variable overrides  
7. Theme customisations - user-defined custom CSS (last)

---

## Official documentation index

### Getting started

| Topic | Link |
|-------|------|
| Creating a theme (server plugin, config, override order) | [Creating a theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184804/Creating+a+theme) |
| Creating custom themes (TUI child theme, SFC overrides) | [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198/Creating+custom+themes) |
| Base / Legacy / Ventura / Inspire | [Themes and Totara](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184816/Themes+and+Totara) |

### Style overrides

| Topic | Link |
|-------|------|
| SCSS, sheets, CSS variables, Grunt | [Overwriting CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184812/Overwriting+CSS) |
| BEM, SCSS vs LESS, variables, isolation | [CSS](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188217/CSS) |
| Theme UI overrides, cascade | [Tui front-end framework](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188209) |

### Markup and PHP

| Topic | Link |
|-------|------|
| Theme Mustache overrides | [Overriding Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184808/Overriding+Mustache+templates) |
| Mustache (general) | [Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184794) |
| PHP renderers | [Overriding renderers](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184810) |
| Page layouts | [Page layouts](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184814/Page+layouts) |
| HTML email | [Customising HTML email](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184806/Customising+HTML+email) |

### TUI theme customization

| Topic | Link |
|-------|------|
| Legacy theme bridge | [Legacy theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185020)
| Theme override deprecation risk | [Deprecation guidelines](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187267/Deprecation+guidelines) |

### Theme Appearance (admin)

| Topic | Link |
|-------|------|
| Settings storage, tenants, theme files | [Back-end Infrastructure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188020/Back-end+Infrastructure) |

---

## Reference files

| File | When to read |
|------|----------------|
| [references/creating-and-inheritance.md](references/creating-and-inheritance.md) | New theme plugin, `config.php`, parents, required files |
| [references/override-decision-tree.md](references/override-decision-tree.md) | Choosing override mechanism |
| [references/style-overrides.md](references/style-overrides.md) | CSS variables, SCSS, sheets, cascade, Grunt |
| [references/markup-and-php-overrides.md](references/markup-and-php-overrides.md) | Mustache, renderers, layouts, email |
| [references/tui-theme-overrides.md](references/tui-theme-overrides.md) | SFC block overrides, `theme:derive`, Legacy |
| [references/theme-appearance-backend.md](references/theme-appearance-backend.md) | Admin Theme Appearance, tenant settings |

---

## When stuck

1. Confirm Totara version and correct parent chain (Ventura vs Inspire).
2. Compare with `theme_inspire`
3. Branding only? Start with CSS variables in both server and `theme_*` `_variables.scss`.
4. TUI pages wrong but Learn OK? Check Legacy + TUI variable parity.
5. Override not applying? Check cascade order and whether Custom CSS box overrides.
6. After TUI theme src edits — rebuild `theme_{name}` bundle (`totara-tui-components`).
