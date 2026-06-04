# Markup and PHP Overrides

Sources:
- [Overriding Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184808/Overriding+Mustache+templates)
- [Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184794)
- [Overriding renderers](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184810)
- [Page layouts](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184814/Page+layouts)
- [Customising HTML email](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184806/Customising+HTML+email)

---

## Mustache template overrides

**When:** Layout or structure cannot be achieved with CSS alone.

**Placement:**

| Template source | Theme override path |
|-----------------|---------------------|
| Theme-local calls | `theme/{name}/templates/{template}.mustache` (base dir) |
| Plugin `mod/foo/templates/bar.mustache` | `theme/{name}/templates/mod_foo/bar.mustache` |

Use **frankenstyle** directory names (`mod_foo`, not `mod/foo`).

**JavaScript:** Theme inheritance does not apply to AMD the same way as CSS. Template-specific JS often uses Mustache `{{#js}}` helper; new JS should be AMD modules via `require()`.

**Upgrades:** Watch `upgrade.txt` in each component for context variable and template changes.

General Mustache usage: [Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184794).

---

## PHP renderer overrides

**When:** HTML output logic exceeds Mustache + CSS capability.

**Inheritance chain:**

- Child `theme_b_*_renderer` extends parent `theme_a_*_renderer`
- If parent theme lacks renderer for component, extend `core_renderer` (or core component renderer)

**Locations:**

| Core location | Theme override |
|---------------|----------------|
| `renderer.php` in component | `theme/{name}/renderer.php` |
| `classes/output/...` | `theme/{name}/classes/...` |

Require parent class with `require_once` before extending.

**Required config:** `$THEME->rendererfactory = 'theme_overridden_renderer_factory';`

**Upgrades:** Check component `upgrade.txt` for renderer changes.

---

## Page layouts

**When:** Page shell (regions, doctype, surrounding HTML) must change.

**Config** (`config.php`):

```php
$THEME->layouts = [
    'standard' => [
        'file' => 'default.php',
        'regions' => ['side-pre', 'side-post'],
        'defaultregion' => 'side-pre',
        'options' => [], // optional, passed as $PAGE->layout_options
    ],
];
```

**Files:** `theme/{name}/layouts/{file}.php`

Layout PHP must call `$OUTPUT->main_content()` and standard header/footer hooks.

Core layout names include `base`, `standard`, `course`, `incourse`, etc. Undefined layouts fall back to `base`.

---

## HTML email customization

**Separate channel** from page UI — email clients need inline CSS.

**Override file:**

```
server/theme/{name}/templates/core/email_html.mustache
```

**Strip wrapper** (content only):

```mustache
{{{body}}}
```

**Full custom wrapper:** Add HTML structure with **inline** `style` attributes on elements.

**Placeholders:**

| Placeholder | Description |
|-------------|-------------|
| `sitefullname` | Site full name |
| `siteshortname` | Site short name |
| `sitewwwroot` | Site URL |
| `subject` | Email subject |
| `to` | Recipient email |
| `toname` | Recipient full name |
| `from` | Sender email |
| `fromname` | Sender display name |
| `body` | Message body (often triple-stash `{{{body}}}`) |

Reference implementation: `theme/basis/templates/core/email_html.mustache` (historical; use as structural reference).

Relates to Mustache overrides — see [Overriding Mustache templates](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184808/Overriding+Mustache+templates).

---

## Escalation order (markup stack)

1. CSS / CSS variables  
2. Mustache (structure)  
3. Page layout (shell)  
4. Renderer (generation logic)  
5. Email Mustache (transactional only)

Full tree: [override-decision-tree.md](override-decision-tree.md).
