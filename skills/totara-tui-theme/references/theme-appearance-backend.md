# Theme Appearance Back-end

Source: [Back-end Infrastructure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188020/Back-end+Infrastructure)

Related: [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198/Creating+custom+themes) (front-end Theme Appearance pages, `theme:var`)

---

## Purpose

Totara 13+ provides **tenant-aware** theme settings for Site Administrators and Tenant Domain Managers. The back-end is **dynamic**: TUI front-end defines categories/properties; PHP persists JSON in `config_plugins`.

Use this reference when:

- Extending which options appear in Theme Appearance UI
- Understanding how admin colour picks reach CSS
- Implementing tenant-specific branding overrides
- Wiring custom theme settings to GraphQL/TUI pages

Simple child themes that only ship static `_variables.scss` may not need this layer.

---

## Front-end entry points (TUI)

| Component | Role |
|-----------|------|
| Theme tenants list | Select site vs tenant context |
| Theme settings page | Main integration with settings API |

Settings UI is built from categories/tabs (e.g. Brand, Colours) with typed properties (text, colour value, boolean toggles, dependencies).

Example colour properties: `color-state`, `color-primary`, `nav-bg-color`, `nav-text-color` — often gated by `formcolours_field_useoverrides`.

Full property tables: Confluence page [Back-end Infrastructure](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188020/Back-end+Infrastructure).

---

## Settings model

- **Categories** contain **properties** (key/value pairs)
- Structure serialized as **JSON** in `config_plugins`
- **Class:** `core\theme\settings` (main settings API)
- Designed so front-end components dictate shape without rigid PHP schema per field

### Retrieval priority

Settings resolve in order (site vs tenant overrides) — see Confluence for tenant vs site precedence.

### Tenant customization

Default list of which settings tenants may override is defined in the main theme settings class. Hooks allow restricting or extending tenant-updatable categories/properties (`server/lib/classes/hook/...`).

---

## How settings affect the UI

1. Admin saves Theme Appearance form  
2. Back-end stores JSON in `config_plugins`  
3. System emits **CSS variable overrides** (second-to-last in cascade)  
4. Optional **Custom CSS** box remains last  

Front-end theme pages use GraphQL to load/save settings — extending API may require `totara-graphql-api` skill.

---

## Theme files (customizable assets)

The back-end supports customization of specific **theme files** (list on Confluence — logos, favicons, etc.). Custom themes register which files appear in admin UI.

Pair file settings with `theme:var` variables when colours must stay consistent across Legacy and TUI.

---

## Linking code theme to admin UI

1. Define variables in `client/component/theme_{name}/src/global_styles/_variables.scss` with `/* theme:var */` and optional `/* theme:derive */`  
2. Build theme bundle → `build/css_variables.json`  
3. Implement or extend Theme Appearance Vue SFCs for extra fields (see Creating custom themes)  
4. Ensure server-side Legacy variables align for non-TUI pages  

---

## Child theme caveat

Child themes do **not** inherit a parent’s **saved** Theme Appearance customizations — only default parent settings. Plan branding defaults in the child theme’s own variables/files.

---

## When not to use this layer

| Scenario | Approach |
|----------|----------|
| Fixed brand colours in code | CSS variables in `_variables.scss` only |
| One-off CSS tweak | [style-overrides.md](style-overrides.md) |
| Markup change | [markup-and-php-overrides.md](markup-and-php-overrides.md) |
