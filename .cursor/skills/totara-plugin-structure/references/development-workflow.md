# Plugin Development Workflow

Recommended scaffolding order when building a new Totara plugin from scratch.

## Before writing code

1. **Choose the name** — the plugin type determines the prefix; the second part must be unique across the site. There cannot be two `auth_saml` plugins on one Totara instance.
2. **Plan the schema** — identify the database entities, the actions that operate on them, and the access controls needed to keep data secure.

---

## Scaffolding checklist

```
- [ ] 1.  Create containing directories (server/<type>/<name>/ and client/component/<frankenstyle>/)
- [ ] 2.  version.php — register the plugin
- [ ] 3.  db/install.xml — database schema
- [ ] 4.  webapi/schema.graphqls — GraphQL type and operation definitions
- [ ] 5.  tui.json — register the client bundle
- [ ] 6.  entity classes (classes/entity/) — one per DB table
- [ ] 7.  model classes (classes/model/) — static create(), dynamic get_*/set_* methods, saves via entity
- [ ] 8.  interactor class (classes/model/) — checks: enabled, configured, user can create, user can act on object
- [ ] 9.  PHP entry points + controller classes:
          index.php  + controllers/index_controller.php  (list objects)
          add.php    + controllers/add_controller.php    (create)
          view.php   + controllers/view_controller.php   (view details)
          edit.php   + controllers/edit_controller.php   (update)
- [ ] 10. GraphQL resolvers, formatters, middleware (classes/webapi/)
- [ ] 11. Page Vue SFCs in client — one per entry point, consumes props from the controller
- [ ] 12. db/access.php — capability definitions
- [ ] 13. lang/en/<frankenstyle>.php — localisation strings
- [ ] 14. settings.php — admin settings (if required)
- [ ] 15. Unit tests alongside each class; Behat features in tests/behat/
```

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
