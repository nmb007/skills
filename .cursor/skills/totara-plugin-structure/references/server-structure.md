# Server Structure — Detailed Reference

## Root-level PHP files

| File | Required | Purpose |
|------|----------|---------|
| `version.php` | Yes | Declares plugin version (bump when DB/lang changes needed), minimum required Totara version, and frankenstyle `$plugin->component`. Used by install/upgrade verification. |
| `index.php` | Recommended | Default entry point; prevents directory listing on misconfigured servers. Nearly all plugins provide one. |
| `settings.php` | If needed | Adds admin settings to the site admin tree. Executed when the admin tree is built. |
| `lib.php` | If needed | Functions used by core or other components to interact with this plugin. Frequently included from outside the plugin's scope. Prefer autoloaded classes for new code. |
| `locallib.php` | If needed | Internal-only functions; never included outside the plugin. Legacy approach — prefer autoloaded classes. |
| `externallib.php` | Legacy | Web service definitions and handlers. Prefer GraphQL (`webapi/`) for new code. |
| `renderer.php` | If needed | Output renderer; factory-driven and theme-overridable. Links plugin logic to the Mustache template system. |
| `styles.css` | If needed | CSS the plugin adds to every theme. Source may be compiled from `less/` or `sass/`. Theme-specific variants: `styles_{theme_frankenstyle}.css`. |
| `upgrade.txt` | Recommended | Append-only log of deprecations, API changes, and dev notes. Essential reference for anyone maintaining customisations. |
| `*_form.php` | Legacy | Form definitions (old pre-autoloading standard). Prefer namespaced autoloaded form classes. |

---

## `db/` — system definition files

| File | Purpose |
|------|---------|
| `install.xml` | Database schema. |
| `install.php` | Code executed once during plugin installation. |
| `upgrade.php` | Upgrade steps keyed to version numbers from `version.php`. |
| `upgradelib.php` | Extracts upgrade operations into functions for testability and reuse. |
| `access.php` | Capability definitions consumed by `core_role`. |
| `caches.php` | Cache store definitions consumed by `core_cache`. |
| `events.php` | Event observer registrations — lets this plugin react to fired events. |
| `hooks.php` | Hook watcher registrations — lets this plugin modify core behaviour at runtime. |
| `renamedclasses.php` | Maps old class names to new ones; auto-creates aliases and emits deprecation notices. Required when renaming classes per the Deprecation guidelines. |
| `services.php` | Legacy web service definitions. Do not use for new code |
| `subplugins.php` | Declares any sub-plugin types this plugin introduces. |
| `tag.php` | Tag collections and areas introduced by this plugin. |

---

## `classes/` — PHP autoloaded classes

All classes are autoloaded under the `\type_plugin_name` namespace. File path mirrors the namespace:  
`classes/model/foo.php` → `\type_plugin_name\model\foo`.

### Well-known subnamespaces

| Namespace | Purpose |
|-----------|---------|
| `controllers/` | HTTP request handlers; create an HTML page view in response to the browser. Each controller typically pairs with a PHP entry point (`index.php`, `view.php`, etc.) and a `tui_view`. |
| `data_provider/` | Filterable, sortable, paginated collections of entities for display in lists or selectors. |
| `entity/` | Classes that map directly to database tables. Usually thin wrappers; wrapped by model classes for business logic. |
| `event/` | System event observer classes. |
| `exception/` | Locally-defined exception classes. |
| `local/` | "Private" classes not meant to be extended or depended on outside the plugin. |
| `model/` | Business logic classes. Contains three sub-roles: |
| — model class | Uses entities for persistent storage. Has a static `create()` and dynamic `set_*()` / `get_*()` methods. |
| — interactor class | Implements access control checks (is plugin enabled, is it configured, can user create/act on an object). |
| — helper class | Tailored, testable implementation of integration logic. |
| `task/` | Scheduled task and adhoc task classes. |
| `testing/` | Classes used only by automated tests, notably the data generator class. |
| `webapi/` | GraphQL resolver, formatter, and middleware classes.|

---

## Other server directories

| Directory | Purpose |
|-----------|---------|
| `webapi/` | GraphQL schema files (`.graphqls`) and persisted query files (`.graphql`) for the web front end. See `totara-graphql-api` skill. |
| `lang/` | Localisation strings. Only `lang/en/` ships in the main distribution; other languages are distributed separately. All user-facing strings must come through the translation API. |
| `rb_sources/` | Report Builder source classes and embedded report definitions. |
| `tests/` | Unit test files (`*_test.php`, class name `{frankenstyle}_*_testcase`). Behat acceptance tests live in `tests/behat/*.feature`. |
| `pix/` | Plugin images; accessible through the mediation layer and overridable by themes. |
| `templates/` | Mustache templates; overridable by themes. |
| `backup/` | Backup and restore code for items within the course context. Used by the Backup API. |
