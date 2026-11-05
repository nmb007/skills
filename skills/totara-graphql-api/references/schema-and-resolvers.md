# Schema and Resolvers

Sources:
- [Implementing GraphQL services](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187514/Implementing+GraphQL+services)
- [Developing GraphQL APIs for Totara](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187517/Developing+GraphQL+APIs+for+Totara)

---

## File types overview

| Extension | Purpose |
|---|---|
| `.graphqls` | Schema definition (SDL) — types, queries, mutations, enums, scalars |
| `.graphql` | Persisted query/mutation — used by ajax and mobile endpoints |

---

## Schema file locations

```
# Included in ALL endpoint schemas (global)
server/{plugin}/webapi/schema.graphqls
server/lib/webapi/schema.graphqls            ← core/global

# Endpoint-specific (Totara 17+)
server/{plugin}/webapi/ajax/schema.graphqls
server/{plugin}/webapi/external/schema.graphqls
server/{plugin}/webapi/mobile/schema.graphqls
```

Multiple `.graphqls` files per directory are fine — all are merged by the schema loader.
Convention: use `schema.graphqls` when there is only one file.

---

## Query definitions

The top-level `Query` type is defined once in `lib/webapi/schema.graphqls`. Every plugin *extends* it:

```graphql
# lib/webapi/schema.graphqls — core definition (do not redefine)
type Query {
  core_template(...): String!
}

# plugin schema — always extend
extend type Query {
  local_todo_items: local_todo_items_result!
}

extend type Mutation {
  local_todo_create_item(input: local_todo_create_item_input!): local_todo_create_item_result!
}
```

---

## Type definitions

```graphql
type local_todo_item {
  id: core_id!
  title: String!
  completed_at: core_date
}

# Always wrap results — never return a bare array or the type itself
type local_todo_items_result {
  items: [local_todo_item!]!
}

# Input types for mutations
input local_todo_create_item_input {
  title: String!
}
```

`!` = non-nullable. `[type!]!` = non-null array of non-null items.

---

## Persisted queries (ajax / mobile only)

```
server/{plugin}/webapi/ajax/{queryname}.graphql
server/{plugin}/webapi/mobile/{queryname}.graphql
```

File name = query name minus the component prefix. Example: operation `local_todo_items` → `webapi/ajax/items.graphql`.

```graphql
# webapi/ajax/items.graphql
query local_todo_items {
  local_todo_items {
    items {
      id
      title
      completed_at
    }
  }
}

# With variables
query local_todo_items($filter: local_todo_items_filter) {
  local_todo_items(filter: $filter) {
    items { id title }
  }
}
```

- External and dev endpoints use **custom (arbitrary) queries** — no persisted query files needed.
- Fragments are supported within a file but cannot be shared across files.

### Extending persisted queries (third-party plugins)

Copy the original `.graphql` to a new file (e.g. `extended_items.graphql`) and add fields. Switch call sites to the new operation name. This avoids upgrade conflicts.

---

## Query resolver

Responsibility: permission checks, data fetching. **Does not format output.**

```
# File location
server/{plugin}/classes/webapi/resolver/query/{queryname}.php

# Classname
{component}\webapi\resolver\query\{queryname}
```

```php
namespace local_todo\webapi\resolver\query;

use core\webapi\execution_context;
use core\webapi\query_resolver;

class items extends query_resolver {
    public static function resolve(array $args, execution_context $ec): array {
        global $USER;
        require_login();
        // capability check here if needed
        $items = \local_todo\model\item::repository()
            ->where('user_id', $USER->id)
            ->order_by('id')
            ->get();
        return ['items' => $items];
    }
}
```

---

## Mutation resolver

```
server/{plugin}/classes/webapi/resolver/mutation/{mutationname}.php
{component}\webapi\resolver\mutation\{mutationname}
```

```php
namespace local_todo\webapi\resolver\mutation;

use core\webapi\execution_context;
use core\webapi\mutation_resolver;

class create_item extends mutation_resolver {
    public static function resolve(array $args, execution_context $ec): array {
        require_login();
        $input = $args['input'];
        $item = \local_todo\model\item::create(['title' => $input['title']]);
        return ['item' => $item];
    }
}
```

---

## Type resolver

Responsibility: given a field name and raw data object, return the formatted value (or raw data for complex sub-types).

```
server/{plugin}/classes/webapi/resolver/type/{typename}.php
{component}\webapi\resolver\type\{typename}
```

```php
namespace local_todo\webapi\resolver\type;

use core\webapi\execution_context;
use core\webapi\type_resolver;

class item implements type_resolver {
    public static function resolve(string $field, $item, array $args, execution_context $ec) {
        // Use a formatter for clean field dispatch (see best-practices.md)
        $formatter = new \local_todo\formatter\item($item, \context_system::instance());
        return $formatter->format($field, $args['format'] ?? null);
    }
}
```

**Rule**: type resolvers for scalar fields do not perform access control — that is the query resolver's responsibility. Access control is required only when the type resolver must load child entities.

### ORM data providers (no type resolver needed)

If your query resolver returns a Totara ORM data provider and the schema type implements `core_pageable_result`, the framework resolves fields automatically:

```graphql
type local_todo_items_result implements core_pageable_result {
  items: [local_todo_item!]!
  total: Int!
  next_cursor: String!
}
```

---

## Union resolver

```
server/{plugin}/classes/webapi/resolver/union/{unionname}.php
{component}\webapi\resolver\union\{unionname}
```

---

## Returning errors

```php
// Generic (hidden in production, shown as "Internal server error")
throw new \moodle_exception('invaliduser');

// Client-aware (message visible to API caller in all modes)
use core\webapi\exception\client_aware_exception;
$e = new create_item_exception('Title cannot be blank.');
throw new client_aware_exception($e, ['category' => 'validation']);
```

Use `client_aware_exception` only for errors that are safe to expose to the API caller (validation, not-found, permissions denial).

---

## Important server-side files

| File | Purpose |
|---|---|
| `server/totara/webapi/classes/schema_file_loader.php` | Locates schema files across the codebase |
| `server/totara/webapi/classes/schema_builder.php` | Builds the combined schema per endpoint |
| `server/totara/webapi/classes/server.php` | Handles a single GraphQL request |
| `server/totara/webapi/classes/default_resolver.php` | Routes field resolution to the correct class |
| `server/totara/webapi/classes/processor.php` | Sets up the graphql-php library for execution |
| `server/lib/classes/webapi/execution_context.php` | Stores contextual data across a request lifetime |
| `server/lib/classes/webapi/query_resolver.php` | Base class for query resolvers |
| `server/lib/classes/webapi/mutation_resolver.php` | Base class for mutation resolvers |
| `server/lib/classes/webapi/type_resolver.php` | Base class for type resolvers (interface) |

---

## Schema generation CLI

```bash
# Generate schema for a specific endpoint type to a file
php ./server/totara/api/cli/generate_external_schema.php -t=ajax -f=totara.graphqls
# -t options: ajax, dev, mobile, external
# Use -f=- to print to stdout

# Download full dev schema via URL (requires GRAPHQL_DEVELOPMENT_MODE = true)
https://YOUR-SITE-URL/totara/webapi/dev_graphql_schema.php
```
