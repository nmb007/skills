# Extending, Debugging, and Advanced Topics

Sources:
- [Extending GraphQL APIs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187526/Extending+GraphQL+APIs)
- [GraphQL Record Reference Class](https://totara.atlassian.net/wiki/spaces/DEV/pages/194707501/GraphQL+Record+Reference+Class)
- [Enabling debugging in GraphQL APIs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187522/Enabling+debugging+in+GraphQL+APIs)
- [Interacting with sessions](https://totara.atlassian.net/wiki/spaces/DEV/pages/177111075/Interacting+with+sessions)
- [GraphQL schema file changes in Totara 17](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187586/GraphQL+schema+file+changes+in+Totara+17)
- [Extending API documentation](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187524/Extending+API+documentation)
- [Building GraphQL Documentation](https://totara.atlassian.net/wiki/spaces/DEV/pages/879525916/Building+GraphQL+Documentation)

---

## Extending schema from a plugin

Any plugin can contribute schema by placing `.graphqls` files in its `webapi/` directory.

```graphql
# Always prefix with frankenstyle component name to avoid collisions
type mod_test_mytype {
  id: core_id!
  sub: mod_test_my_sub_type!
}

type mod_test_my_sub_type {
  abc: String!
}

input mod_test_my_input_type {
  id: core_id!
}

enum mod_test_my_enum {
  VALUE1
  VALUE2
}

extend type Query {
  mod_test_my_query(input: mod_test_my_input_type!): mod_test_mytype!
}

extend type Mutation {
  mod_test_my_mutation(name: String!): Int!
}
```

> `extend type Query` / `extend type Mutation` — use `extend`, never redefine the root types.
> `extend type SomeObject` in GraphQL modifies the original type (adds fields), it does not create a subtype. To create a "variant" of an existing type you must duplicate the schema.

---

## Creating new resolvers

Extend the appropriate core base class:

```php
// Query resolver
class my_query extends \core\webapi\query_resolver { ... }

// Mutation resolver
class my_mutation extends \core\webapi\mutation_resolver { ... }

// Type resolver
class my_type implements \core\webapi\type_resolver { ... }

// Union resolver
class my_union extends \core\webapi\union_resolver { ... }
```

---

## Extending an existing resolver (inheritance)

```php
namespace local_myextension\webapi\resolver\type;

class user extends \core\webapi\resolver\type\user {
    public static function resolve(string $field, $user, array $args, execution_context $ec) {
        if ($field === 'my_custom_field') {
            return 'custom_value';
        }
        // Delegate all other fields to core
        return parent::resolve($field, $user, $args, $ec);
    }
}
```

Register the extended resolver in your plugin's schema by referencing the same type name (the framework will use the most-derived class).

---

## Totara 17 schema file structure changes

### Pre-Totara 17

All `.graphqls` schema files lived directly in `webapi/` and were merged into a **single master schema** shared by all endpoints (ajax, dev, mobile).

### Totara 17+

Schema files in a named subdirectory are now included **only for that endpoint**:

```
webapi/schema.graphqls          → all endpoints (unchanged)
webapi/ajax/schema.graphqls     → ajax only
webapi/external/schema.graphqls → external only
webapi/mobile/schema.graphqls   → mobile only
```

The `dev` endpoint automatically includes all endpoint schemas, so it will surface dependency/duplication errors across schemas.

### Organising schema files (guidelines)

- If the schema is designed for **one endpoint only**, put it entirely in that endpoint's subdirectory.
- Put **reusable types, enums, scalars** in `webapi/` (core/global) so multiple endpoints can reference them.
- Keep **queries/mutations and their specific input/result types** in the endpoint subdirectory.
- Move to core only when another endpoint genuinely needs the type — don't prematurely generalise.

### Common issues

**Missing dependency**: A type used in `webapi/mobile/schema.graphqls` is defined only in `webapi/ajax/schema.graphqls`. Fix: move the shared type to `webapi/schema.graphqls`.

**Duplication**: The same name appears in two different endpoint schemas. The `dev` schema will catch this because it merges everything. Fix: deduplicate or move to core.

---

## GraphQL Record Reference Class

Source: *GraphQL Record Reference Class*

The `base_record_reference` abstract class (`server/lib/classes/webapi/reference/base_record_reference.php`) is a utility for identifying a single database record by one of several properties (`id`, `idnumber`, `name`, etc.).

Use it in external API input types where you want to let callers reference records flexibly.

### Key behaviours

- Throws if zero or more than one record matches the selection criteria (no ambiguity)
- `load_for_viewer()` static factory method runs capability checks before returning the instance
- Instance behaves like an entity but is **not** user-aware (no implicit capability checks outside of `load_for_viewer()`)

```php
namespace local_myapi\webapi\reference;

use core\webapi\reference\base_record_reference;

class item_reference extends base_record_reference {
    protected function get_record_type(): string {
        return \local_todo\entity\item::class;
    }

    // Properties that can be used to identify the record
    protected function get_identifier_columns(): array {
        return ['id', 'idnumber'];
    }

    public static function load_for_viewer(array $args): self {
        $ref = new self($args);
        // Perform capability checks here
        if (!has_capability('local/todo:view', \context_system::instance())) {
            throw new \moodle_exception('nopermissions');
        }
        return $ref;
    }
}
```

In the schema, define a corresponding `input` type and document which field combinations uniquely identify the record.

---

## Enabling debugging

### Enable performance data (all endpoints)

```php
// config.php
define('MDL_PERF', true);
// OR
$CFG->perfdebug = 15; // any value > 7
```

Performance data appears in `extensions.performance_data` in the response:

```json
{
  "extensions": {
    "performance_data": {
      "core": {
        "db": { "reads": 13, "writes": 3, "time": 0.08 },
        "realtime": 0.56,
        "memory": { "total": 19839072 }
      }
    }
  }
}
```

### Debug levels — external endpoint

```php
// config.php
$CFG->forced_plugin_settings['totara_api'] = [
    // 0 = none (generic "Internal server error" only)
    // 1 = normal (adds debugMessage with specific error text) — default
    // 2 = developer (adds full stack trace)
    'response_debug' => 2
];
```

### Debug levels — ajax / mobile / dev endpoints

Depends on `$CFG->debug`:

```php
// config.php
$CFG->debug = E_ALL; // enables full stack traces in GraphQL error responses
```

When debug is off, stack traces are stripped from errors.

### Complexity data (external endpoint only)

When performance data is enabled, the external API also returns query complexity:

```json
{
  "extensions": {
    "complexity_data": { "query_complexity": 10 },
    "performance_data": { ... }
  }
}
```

### Enabling introspection

| Endpoint | Introspection |
|---|---|
| dev | Always on (when dev mode enabled) |
| external | Off by default. Enable: Quick-access → Development → API → API settings → `enable_introspection` |
| ajax / mobile | Not available |

When disabled, introspection queries return: `GraphQL introspection isn't enabled.`

---

## Sessions and GraphQL

### Session types by endpoint

| Endpoint | Session usage |
|---|---|
| ajax | Uses sessions for user state; sessions managed throughout request |
| mobile | Uses sessions for backward compatibility |
| external | Stateless — does not use sessions |

### Totara 18 session change (ajax endpoint)

In T18+, sessions for writing are **closed before query resolution** to allow concurrent requests. Writing to a session in a resolver now throws `totara_webapi\exception\reopen_session_for_writing_exception`.

**Fix**: add `core\webapi\middleware\reopen_session_for_writing` as the **first** middleware on any query/mutation resolver that writes to the session. See [middleware.md](middleware.md) for details.

### Disable the optimisation (not recommended)

```php
$CFG->graphqlsessionwritecloseenabled = false; // degrades concurrent request performance
```

---

## API documentation

### Schema comments → introspection

Comments in `.graphqls` files (triple-quoted `"""..."""`) are surfaced via GraphQL introspection and displayed in:
- The in-product API reference docs (external API only)
- GraphQL client auto-complete and inline docs (all endpoints with introspection)

### metadata.json — supplementary documentation

For data that the GraphQL spec cannot express (examples, marking items as undocumented), create `metadata.json` files alongside your schema files:

```
server/{plugin}/webapi/metadata.json              ← applies to all schemas
server/{plugin}/webapi/external/metadata.json     ← external schema only
```

Structure example:

```json
{
  "OBJECT": {
    "Query": {
      "fields": {
        "local_todo_items": {
          "args": {
            "cursor": { "documentation": { "example": "eyJwYWdlIjoxLCJsaW1pdCI6MjB9" } }
          }
        }
      }
    },
    "local_todo_item": {
      "fields": {
        "title": { "documentation": { "example": "Buy groceries" } }
      }
    }
  },
  "INPUT_OBJECT": {
    "local_todo_create_item_input": {
      "inputFields": {
        "title": { "documentation": { "example": "My new task" } }
      }
    }
  }
}
```

Note: `INPUT_OBJECT` uses `inputFields` (not `fields`). Queries and mutations go under `OBJECT.Query.fields` and `OBJECT.Mutation.fields`.

### Building the API reference documentation

Two-step process (separate PHP and Node environments):

```bash
# Step 1: PHP — generates schema and metadata files in dataroot/api/
php server/totara/api/cli/prep_api_docs.php

# Step 2: Node — generates HTML documentation from the schema files
cd extensions/api_docs
tnpm ci
tnpm run build /path/to/dataroot/api
```

Single-environment shortcut:

```bash
cd extensions/api_docs; tnpm ci; php ../../server/totara/api/cli/prep_api_docs.php -q | xargs node build
```

Output: `extensions/api_docs/build/spectaql/out.html` and `out.json`.
Available in-product at: Quick-access → Development → API → API Documentation.

The docs page detects if the live site schema differs from the shipped schema and shows a warning when out of date.
