---
name: totara-graphql-api
description: PHP backend development of GraphQL APIs in Totara: schema files (.graphqls),
query/mutation/type/union resolvers, persisted queries (.graphql), middleware,
pagination, formatters, and endpoint-type differences (ajax, external, mobile, dev).
Use when writing or debugging GraphQL schema files, PHP resolver classes under
webapi/resolver/, persisted .graphql queries, GraphQL middleware, or when asked about
Totara GraphQL API structure, endpoint types, or best practices.
---

# Totara GraphQL API (Backend)

## Scope

Covers **PHP backend** development of GraphQL APIs in Totara plugins and core.

---

## Endpoint types at a glance

| Endpoint | Path | Query type | Auth | Introspection |
|---|---|---|---|---|
| ajax (internal) | `/totara/webapi/ajax.php` | Persisted only | Session cookie + CSRF | No |
| external | `/api/graphql.php` | Custom only | OAuth 2.0 | Optional (admin setting) |
| mobile | `/totara/mobile/api.php` | Persisted only | API key header | No |
| dev | `/totara/webapi/dev_graphql_executor.php` | Custom only | Session cookie (no CSRF) | Yes |

- **ajax** = "internal" in docs. Used by TUI/JS front end.
- **dev** = includes schema from _all_ endpoint types; never use on production.
- **external** = Totara 17+; stricter docs, deprecation, and `reference input types` required.
- **mobile** = Totara Mobile app only. Sessions closed before query resolution.

For full comparison see [references/endpoint-types.md](references/endpoint-types.md).

---

## Naming convention → file path

Every GraphQL object name must be prefixed with the frankenstyle component name (e.g. `local_todo`, `mod_perform`, `totara_job`) to avoid collisions.

| GraphQL kind | Name pattern | PHP class path | PHP classname |
|---|---|---|---|
| query | `{component}_{queryname}` | `classes/webapi/resolver/query/{queryname}.php` | `{component}\webapi\resolver\query\{queryname}` |
| mutation | `{component}_{mutationname}` | `classes/webapi/resolver/mutation/{mutationname}.php` | `{component}\webapi\resolver\mutation\{mutationname}` |
| type / input | `{component}_{typename}` | `classes/webapi/resolver/type/{typename}.php` | `{component}\webapi\resolver\type\{typename}` |
| union | `{component}_{unionname}` | `classes/webapi/resolver/union/{unionname}.php` | `{component}\webapi\resolver\union\{unionname}` |
| persisted query | `{component}_{queryname}` | `webapi/{endpoint}/{queryname}.graphql` | N/A |
| schema (endpoint-specific) | — | `webapi/{endpoint}/schema.graphqls` | N/A |
| schema (all endpoints) | — | `webapi/schema.graphqls` | N/A |

> `{component_path}` = resolved by `core_component::get_component_directory()`, e.g. `server/local/todo/`.

---

## Schema file locations (Totara 17+)

```
server/{plugin}/webapi/schema.graphqls          ← included in ALL endpoint schemas
server/{plugin}/webapi/ajax/schema.graphqls     ← ajax only
server/{plugin}/webapi/external/schema.graphqls ← external only
server/{plugin}/webapi/mobile/schema.graphqls   ← mobile only
```

Schema files in the root `webapi/` → available to every endpoint.
Schema files in a subdirectory → available only to that endpoint.

See [references/schema-and-resolvers.md](references/schema-and-resolvers.md) for full schema and resolver details.

---

## Quick-start: new internal (ajax) query

```
- [ ] 1. Create webapi/ajax/schema.graphqls — define type + result type + extend type Query
- [ ] 2. Create webapi/ajax/{queryname}.graphql — persisted query selecting required fields
- [ ] 3. Create classes/webapi/resolver/query/{queryname}.php — check permissions, return data
- [ ] 4. Create classes/webapi/resolver/type/{typename}.php — dispatch fields, apply formatter
- [ ] 5. Purge Totara caches to pick up schema changes (dev only)
- [ ] 6. Test via dev endpoint or TUI component
```

For a mutation, replace step 2's `query` keyword with `mutation`, create `resolver/mutation/` instead of `query/`, and add `extend type Mutation` in the schema.

---

## Quick-start: new external query

Same steps as above but:
- Place schema in `webapi/external/schema.graphqls`
- Use reference input types (`base_record_reference`) instead of raw database IDs
- Use enums instead of integer/magic-number arguments
- Add full `"""..."""` doc-comments on every schema element
- Add `metadata.json` with examples for API reference docs

---

## Hard rules

1. **Always prefix with frankenstyle name** — `local_todo_item`, never `item`.
2. **Always use input types** for mutations — never a long list of scalar arguments.
3. **Always wrap query results in a result type** — `local_todo_items_result { items: [...] }`, not a bare array.
4. **Do NOT use inline `gql` strings** — queries/mutations must live in `.graphql` persisted files (ajax/mobile) or `.graphqls` schema files.
5. **Check permissions in the query/mutation resolver** — `require_login()` and capability checks belong there, not in type resolvers.
6. **Use middleware for cross-cutting concerns** — authentication, capability checks, record loading. See [references/middleware.md](references/middleware.md).
7. **Use cursor-based pagination** via `core_pagination_input` + `core_pageable_result`. See [references/pagination.md](references/pagination.md).
8. **External API**: use `reference input types`, full schema comments, `metadata.json` examples, and enums over integers.

---

## Schema basics

```graphql
# Define a type
type local_todo_item {
  id: core_id!
  title: String!
  completed_at: core_date
}

# Wrap results — always
type local_todo_items_result {
  items: [local_todo_item!]!
}

# Extend Query (or Mutation) — never redefine
extend type Query {
  local_todo_items: local_todo_items_result!
}

extend type Mutation {
  local_todo_create_item(
    input: local_todo_create_item_input!
  ): local_todo_create_item_result!
}

input local_todo_create_item_input {
  title: String!
}
```

---

## Resolver skeleton

```php
// classes/webapi/resolver/query/items.php
namespace local_todo\webapi\resolver\query;

use core\webapi\execution_context;
use core\webapi\query_resolver;

class items extends query_resolver {
    public static function resolve(array $args, execution_context $ec): array {
        require_login();
        // fetch + return raw data; type resolver handles field formatting
        return ['items' => local_todo\model\item::get_all()];
    }
}
```

```php
// classes/webapi/resolver/type/item.php
namespace local_todo\webapi\resolver\type;

use core\webapi\execution_context;
use core\webapi\type_resolver;
use local_todo\formatter\item as item_formatter;

class item implements type_resolver {
    public static function resolve(string $field, $item, array $args, execution_context $ec) {
        $formatter = new item_formatter($item, \context_system::instance());
        return $formatter->format($field, $args['format'] ?? null);
    }
}
```

---

## Common scalars and types

| Name | Purpose |
|---|---|
| `core_id` | Integer database IDs |
| `core_date` | Unix timestamp |
| `core_format` | Output format enum (`HTML`, `PLAIN`, `MARKDOWN`, `JSON_EDITOR`) |
| `param_*` | Validated input scalars (`param_alphaext`, `param_component`, etc.) |
| `core_pagination_input` | Cursor pagination input |
| `core_pageable_result` | Interface for paginated results (`total`, `next_cursor`) |

---

## Reference files

| File | When to read |
|---|---|
| [references/schema-and-resolvers.md](references/schema-and-resolvers.md) | Full schema/resolver/persisted-query details, error handling, ORM data providers |
| [references/endpoint-types.md](references/endpoint-types.md) | Endpoint comparison, request/response format, auth, batching, dev API setup |
| [references/middleware.md](references/middleware.md) | Before/after middleware, groups, global middleware, hooks |
| [references/pagination.md](references/pagination.md) | Cursor vs offset, opaque cursors, `next_cursor` sentinel |
| [references/best-practices.md](references/best-practices.md) | Input/result types, filtering, formatters, performance (N+1), naming |
| [references/extending-and-advanced.md](references/extending-and-advanced.md) | Extending schema/resolvers, Record Reference Class, debugging, sessions, Totara 17 schema split, API docs build |
