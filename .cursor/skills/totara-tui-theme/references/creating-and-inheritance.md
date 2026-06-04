# Creating Themes and Inheritance

Sources:
- [Creating a theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184804/Creating+a+theme)
- [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198/Creating+custom+themes)
- [Themes and Totara](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184816/Themes+and+Totara)
---

## Server theme plugin (required files)

All themes live under `server/theme/{name}/`:

| File | Purpose |
|------|---------|
| `config.php` | Theme behaviour (`$THEME` object) |
| `version.php` | Plugin version (`$plugin->component = 'theme_{name}'`) |
| `lang/en/theme_{name}.php` | `pluginname`, `choosereadme` strings |

Optional but common for child themes: `scss/`, `styles/`, `templates/`, `layouts/`, `renderer.php`, `classes/`.

---

## Choosing a parent (base theme)

| Parent | Role |
|--------|------|
| **base** | Minimal canvas; font icons only. Most work if not using Bootstrap 3. |
| **legacy** | Non-TUI Learn UI; Bootstrap 3 + component SCSS. Foundation for modern stack. |
| **ventura** | Totara 13–18; Legacy + colour tweaks + Theme Appearance settings. |
| **inspire** | Totara 19+; Legacy + UI improvements (e.g. primary nav). |

**Totara 13–18:** `$THEME->parents = ['ventura', 'legacy', 'base'];`

**Totara 19+:** `$THEME->parents = ['inspire', 'legacy', 'base'];`

Non-core parent themes are allowed (`$THEME->parents` can point at another custom theme).

---

## `config.php` essentials

```php
$THEME->doctype = 'html5';
$THEME->name = 'mytheme';
$THEME->parents = ['ventura', 'legacy', 'base']; // or inspire for 19+
$THEME->rendererfactory = 'theme_overridden_renderer_factory';
$THEME->enable_dock = true;
$THEME->enable_hide = true;
$THEME->minify_css = false; // often false during development
```

Other common keys: `$THEME->sheets`, `$THEME->layouts`, `$THEME->parents_exclude_sheets`, `$THEME->yuicssmodules` (set to `''` to disable legacy YUI CSS).

Full list: `server/lib/outputlib.php`.

---

## `version.php` essentials

```php
$plugin->version   = YYYYMMDDXX;
$plugin->requires  = YYYYMMDDXX; // Totara version
$plugin->component = 'theme_mytheme';
// $plugin->dependencies = ['theme_ventura' => YYYYMMDDXX];
```

---

## TUI child theme (Totara 13+)

Modern branding requires **both** server and client:

| Side | Path |
|------|------|
| Server | `server/theme/mytheme/` |
| Client | `client/component/theme_mytheme/` |

Client bundle needs `tui.json`, `src/global_styles/_variables.scss`, and optionally Theme Appearance Vue SFCs.

Rebuild after client changes: `totara-tui-components` skill (`npm run tui-build-dev theme_mytheme`).

---

## Theme naming rules

- Start with lowercase letters (a–z).
- Underscores: segment before first `_ is alpha; char after `_ is alpha; rest alphanumeric/underscore.
- End with lowercase alphanumeric.

Example valid: `mytheme`, `acme_corp`, `client2_brand`.

---

## Recommended customization order (maintenance)

From [Creating a theme](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184804/Creating+a+theme):

1. Overriding CSS variables (preferred)
2. Overriding SCSS variables
3. Overriding CSS
4. Overriding Mustache Templates
5. Overriding PHP renderers
6. Avoid core hacks — read `upgrade.txt` on upgrades

For TUI-era detail and SFC overrides, see [Creating custom themes](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187198) and [override-decision-tree.md](override-decision-tree.md).

---

## Child theme settings limitation

When using a custom child theme, **saved customizations from the parent theme’s Theme Appearance UI are not applied**. The child inherits **default** parent theme settings only.
