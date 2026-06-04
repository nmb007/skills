# Plugin Development Workflow

Recommended scaffolding order when building a new Totara plugin from scratch.

## Before writing code

1. **Choose the name** — the plugin type determines the prefix; the second part must be unique across the site. There cannot be two `auth_saml` plugins on one Totara instance.
2. **Plan the schema** — identify the database entities, the actions that operate on them, and the access controls needed to keep data secure.

---

## Scaffolding checklist

Do **server foundation first**, then **client**, then **cross-cutting**. Within server, follow data → access → HTTP → API. See integration order below when a client step depends on a specific server artifact.

### Server (`server/<type>/<plugin_name>/`)

```
- [ ] Create server/<type>/<plugin_name>/ directory
- [ ] version.php — register the plugin
- [ ] db/install.xml — database schema (XMLDB editor)
- [ ] db/access.php — capability definitions
- [ ] classes/entity/ — one class per DB table
- [ ] classes/model/ — static create(), get_*/set_* methods, saves via entity
- [ ] classes/model/ interactor — enabled, configured, user can create, user can act on object
- [ ] lang/en/<frankenstyle>.php — localisation strings
- [ ] settings.php — admin settings (if required)
- [ ] PHP entry points + matching controllers/:
        index.php  (list)   add.php  (create)
        view.php   (detail)  edit.php (update)
- [ ] webapi/schema.graphqls — types and operations
- [ ] classes/webapi/ — resolvers, formatters, middleware
- [ ] tests/ — unit tests (*_test.php); Behat in tests/behat/
```

### Client (`client/component/<frankenstyle>/`)

```
- [ ] Create client/component/<frankenstyle>/ directory
- [ ] tui.json — register the bundle (component name matches frankenstyle)
- [ ] src/ — v2 feature folders (Totara 19+); Page SFCs end in Page
- [ ] One page SFC per PHP entry point; consumes props from the controller
- [ ] npm run tui-init / tui-build-watch as needed — see totara-tui-components skill
- [ ] Persisted .graphql queries + Apollo in .vue — see totara-tui-apollo skill
```

### Cross-cutting (after both roots exist)

```
- [ ] Wire each controller tui_view path to the matching page SFC import path
- [ ] Confirm GraphQL operations used in Vue match schema + persisted queries
- [ ] upgrade.txt on server and client when introducing breaking API changes
```

### Integration order (server ↔ client)

| Step | Why order matters |
|------|-------------------|
| `install.xml` → entities → models → interactor | Data layer before access checks and API |
| `schema.graphqls` → `classes/webapi/` resolvers | Schema before resolver implementation |
| Controllers + entry points → page Vue SFCs | `tui_view` props define what each page receives |
| Resolvers before Apollo in `.vue` | Client calls operations the server must already expose |

---

## Key access control convention

Access control logic lives in the **model / interactor class**, but is *checked* in the **webapi resolvers**.

This allows scheduled tasks, CLI scripts, and other non-user processes to call model methods without being blocked by user-level access control.

---

## Subsystem integrations (optional but recommended)

Consider whether the plugin should implement:

| Subsystem | Where |
|-----------|-------|
| User data management | `classes/userdata/` + `db/` registration |
| Multitenancy | Interactor checks + entity-level tenant isolation |
| Centralised notifications | `classes/watcher/` + `db/hooks.php` |
| Report Builder | `rb_sources/` |
| Fine-grained capabilities | `db/access.php` |
| Backup / restore | `backup/` |

---
