# GraphQL Best Practices

Sources:
- [GraphQL development best practices](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187551/GraphQL+development+best+practices)
- [GraphQL documentation authoring best practices](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187530/GraphQL+documentation+authoring+best+practices)
- [GraphQL formatters](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187562/GraphQL+formatters)
- [GraphQL performance](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187564/GraphQL+performance)

---

## Input types

**Always** use a dedicated input type for mutations — never a long list of scalar arguments.

```graphql
# Bad
extend type Mutation {
  core_widget_create_widget(
    name: String!
    description: String!
    type: param_alphaext!
  ): core_widget_widget_result!
}

# Good
input core_widget_widget_input {
  name: String!
  description: String!
  type: param_alphaext!
}

extend type Mutation {
  core_widget_create_widget(
    widget: core_widget_widget_input!
  ): core_widget_widget_result!
}
```

Design input types for **reuse**: prefer adding an optional argument to an existing mutation over duplicating input type fields in a new type.

When create and update operations require different nullability (e.g. all fields required on create, all optional on update), separate input types are acceptable.

---

## Built-in reference types

Reuse existing Totara reference inputs (e.g. `core_user_user_reference`) before defining your own — core already provides resolvers and handling for them.

---

## Result types

**Always** wrap query and mutation results in a dedicated result type. Never return a bare array or the entity type directly.

```graphql
# Bad
extend type Query {
  core_widget_my_widgets: [core_widget_widget!]!
}

# Good — allows adding metadata (pagination, status) later without breaking the API
type core_widget_my_widgets_result {
  items: [core_widget_widget!]!
}

extend type Query {
  core_widget_my_widgets: core_widget_my_widgets_result!
}
```

---

## Filtering

Use an input type with optional fields for each filter dimension:

```graphql
enum mod_perform_subject_instance_about_filter {
  SELF
  OTHERS
}

input mod_perform_subject_instance_filters {
  about: [mod_perform_subject_instance_about_filter!]
  activity_type: [core_id!]
  overdue: param_boolean
}

extend type Query {
  mod_perform_my_subject_instances(
    filters: mod_perform_subject_instance_filters
    pagination: core_pagination_input
  ): mod_perform_subject_sections_page!
}
```

---

## Naming persisted queries vs query endpoints

```graphql
# Persisted query name → maps to file: webapi/ajax/{name_without_component_prefix}.graphql
# Query endpoint name → maps to resolver class
query mod_example_my_dashboard {         # persisted query name (can differ from endpoint)
  mod_example_query_endpoint_name {     # must match the resolver class
    some_field
  }
}
```

- When one endpoint, one persisted query: use the **same name** for both.
- When multiple persisted queries use the same endpoint (e.g. a summary block and a full-page view), give the persisted queries distinct names that describe their purpose.
- Mobile app: use aliases to return camelCase keys.

---

## Multiple endpoints in a single persisted query

You can combine multiple query endpoints in one persisted query for efficiency:

```graphql
query core_user_dashboard {
  core_user_profile_info {
    username
    email
  }
  mod_perform_user_activity_progress {
    num_incomplete_activities
  }
}
```

Limitation: all endpoints share one execution context. Avoid combining endpoints that each set the execution context — this causes a "Context can only be set once per execution" error. Use JS batch queries (Apollo batch link) as a workaround.

---

## Large queries vs multiple small queries

| Approach | When to use |
|---|---|
| Single high-level query | Page is non-functional until all data arrives; avoids visible reflow |
| Multiple component-level queries | Component isolation is more important; use pagination to limit data |

Test with realistic data volumes. Use Xstate when complex loading orchestration is needed.

---

## Resolvers

Keep query/mutation resolvers thin: orchestrate in the resolver, put business logic in testable service classes.

---

## Formatters

Use the `formatter` class instead of `if/else` switch chains in type resolvers.

```php
// classes/formatter/item.php
namespace local_todo\formatter;

use core\webapi\formatter\formatter;
use core\webapi\formatter\field\string_field_formatter;
use core\webapi\formatter\field\date_field_formatter;

class item extends formatter {
    protected function get_map(): array {
        return [
            'id'           => null,                          // no formatting — return as-is
            'title'        => string_field_formatter::class, // runs format_string()
            'completed_at' => date_field_formatter::class,   // formats unix timestamp
        ];
    }
}
```

Usage in the type resolver:

```php
public static function resolve(string $field, $item, array $args, execution_context $ec) {
    $formatter = new \local_todo\formatter\item($item, \context_system::instance());
    return $formatter->format($field, $args['format'] ?? null);
}
```

### Built-in field formatters

| Class | Behaviour |
|---|---|
| `string_field_formatter` | `format_string()` — strips HTML, handles multilang |
| `text_field_formatter` | `format_text()` — full rich-text rendering |
| `date_field_formatter` | Formats Unix timestamps in various formats |

### Custom formatter for non-standard objects

Override `get_field()` and `has_field()` to work with model objects that are not `stdClass`/arrays:

```php
protected function get_field(string $field) {
    return $this->object->get($field); // custom getter
}

protected function has_field(string $field): bool {
    return $this->object->has($field);
}
```

---

## Performance — the N+1 problem

### Option 1: Eager-loading with ORM relationships

Preload related data in the query/mutation resolver using ORM relationships:

```php
$items = item::repository()
    ->with('user')       // preloads user relationship
    ->with('category')
    ->get();
return ['items' => $items];
```

Limitation: only works if query/mutation always needs those relationships regardless of which fields were requested.

### Option 2: Deferred loading (entity buffer)

Use the `Deferred` class from the graphql-php library and Totara's `buffer::defer()`:

```php
use GraphQL\Deferred;
use totara_webapi\entity\buffer;

public static function resolve(string $field, $entity, array $args, execution_context $ec) {
    switch ($field) {
        case 'user':
            // Defers DB load until all items are buffered, then one query for all
            return new Deferred(buffer::defer($entity, 'user'));
        case 'category':
            return new Deferred(buffer::defer($entity, 'category'));
    }
    $formatter = new \local_todo\formatter\item($entity, \context_system::instance());
    return $formatter->format($field, $args['format'] ?? null);
}
```

The buffer accumulates all entity references throughout the request, then triggers a single query per entity class when promises are resolved. Works for ORM entities and their relationships.

---

## Schema documentation authoring

### Technical: how to write comments

Use triple-quoted strings (GraphQL SDL description syntax):

```graphql
""" Single-line comment """
scalar foo

"""
Multi-line comment.
Markdown is supported for basic formatting.
Use https://YOUR-SITE-URL as a placeholder — it is replaced with the real site URL at build time.
"""
type local_todo_item {
  """ Internal database id of the todo item. """
  id: core_id!
  """ User-provided title. Max 255 characters and must be unique per user. """
  title: String!
}
```

Document **every** schema element: types, fields, field arguments, input types, input fields, enums, enum values, scalars, interfaces, unions, and queries/mutations.

### Style rules

- Write for a third-party developer integrating with Totara — not the Totara admin.
- Use in-product terms: "audience" not "cohort".
- Full stops at the end of sentences.
- Present tense: "Returns the user's timezone" not "Will return".
- Capitalise acronyms: `SCORM`, `HTML`.
- Use `id` not `ID`.

### Common description patterns

| Scenario | Template |
|---|---|
| `id` field | `Internal database id of the {entity}.` |
| `idnumber` field | `Unique reference used to represent the {entity} across multiple systems.` |
| Rich-text `description` field | `Rich-text description of the {entity}.` |
| `descriptionformat` field | `Format of the description field, as stored in the database.` |
| Admin setting dependency | `{Description}. Uses the admin setting '{setting_name}' if configured.` |
| Input with example | Include example in description: `Language of the user, for example 'en_us'.` |

### Avoid superfluous comments

```graphql
# Bad — mirrors the field name, adds no value
""" widget id """
id: core_id!

# Good — explains uniqueness, type, and purpose
""" Internal database id of the widget. """
id: core_id!
```

### Reference types: what to document

- Describe which combination of fields uniquely identifies the record
- Describe any filtering applied (e.g. excludes deleted users, tenant-scoped)

### Deprecating fields

Use the built-in GraphQL `@deprecated` directive with a reason string. See the Totara deprecation policies in the external API docs for when deprecation is required.
