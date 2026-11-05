# PHP integration and MVC

Sources:
- [Integration with Totara Core](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184986)
- [Model-view-controller (MVC) pattern](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184478) — covers **both** Mustache views and Tui views; for Tui work, use only the `tui_view` parts below, not `totara_mvc\view` examples from that page.
- [Quick-start guide: Creating your first component](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184960)

---

## How Tui connects to Totara Core

- Tui lives in `/client` (not web-accessible directly).
- Plugin **`totara_tui`** mediates JS/CSS via `totara/tui/javascript.php` and `totara/tui/styles.php`.
- Core registers intent to render a component → small HTML fragment on the page → Tui bootstraps and replaces it with the Vue tree.
- **Data channel:** GraphQL (ajax persisted queries) and props embedded from PHP — not legacy AJAX endpoints.

Build requires `graphql_locations.json` at repo root so Tui knows where schema files live.

---

## MVC — render a Tui page

Controllers extend `totara_mvc\controller` (from `totara_mvc` plugin).

```php
// server/local/myplugin/classes/controllers/index.php
namespace local_myplugin\controllers;

class index extends \totara_mvc\controller {

    public function action(): \totara_mvc\viewable {
        require_login();
        // capability checks, load models, etc.

        $additional_props = [
            'someId' => 42,
        ];

        // Render Vue's page component from its respective TUI component
        return \totara_mvc\tui_view::create(
            'local_myplugin/pages/Ping',
            $additional_props
        );
    }
}
```

Path format: `{frankenstyle}/pages/{ComponentName}` — matches import path without `.vue` (PascalCase component name).

### PHP entry script

```php
// server/local/myplugin/index.php
require(__DIR__ . '/../../config.php');

$controller = new \local_myplugin\controllers\index();
echo $controller->process();
```

Layout is set early in the request lifecycle — follow existing controllers in the plugin for layout conventions.

---

## Props from PHP to Vue

PHP passes JSON on the placeholder element; Tui decodes it into **root component props**. Use `props` in the page SFC to receive them.

---

## Tui component loading and dependencies

When a page uses a Tui component:

1. Framework records the required Tui component.
2. Reads that Tui component's `dependencies.json` (generated at build).
3. Loads dependent Tui component's JS/CSS in order before initializing the page component.

**tui.json** `"dependencies"` must list other Tui components you import statically.

Also declare the corresponding **PHP plugin dependency** in your plugin's `version.php` / dependencies so install order is correct.

---

## GraphQL (server side)

Tui front end does not call arbitrary PHP scripts. New server operations:

1. Implement ajax schema + resolvers in `server/{plugin}/` — **`totara-graphql-api`**
2. Wire `.vue` with persisted query imports — **`totara-tui-apollo`**

Test operations via [developer GraphQL API](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187396) before relying on ajax persisted queries.

---

## Mustache `view` — not for Tui components

The [MVC wiki](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184478) shows `return new \totara_mvc\view('my_plugin/show')` — that is **Mustache template rendering**, not Tui. Do not use this pattern when wiring a Vue page component.

**Not relevant to Tui (Mustache only):**

```php
return new \totara_mvc\view('my_plugin/show');
// or: new \totara_mvc\view('my_plugin/my_first_template', $template_data);
```

**Correct for a Tui page:**

```php
return \totara_mvc\tui_view::create('my_plugin/pages/MyPageComponent', $additional_props);
```

Controller setup from the wiki (`setup_context()`, `set_url()`, capabilities, layout) still applies to Tui controllers; only the **returned view type** differs.

---

## Checklist: PHP + Tui page

```
- [ ] Page SFC exists at import path used in tui_view::create()
- [ ] Controller returns tui_view (not raw echo HTML)
- [ ] Plugin entry script calls $controller->process()
- [ ] tui.json component name matches frankenstyle
- [ ] Dev build run / watch active
- [ ] totara_tui development_mode enabled for debugging
```
