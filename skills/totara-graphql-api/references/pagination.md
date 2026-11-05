# Pagination with GraphQL

Sources:
- [Pagination with GraphQL](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187541/Pagination+with+GraphQL)

---

## Recommendation

Use **cursor-based pagination** with Totara's built-in `core_pagination_input` input type and `core_pageable_result` interface. Offset-based pagination is available via the same opaque cursor mechanism but has known issues with real-time data sets.

---

## Cursor vs offset trade-offs

| | Cursor-based | Offset-based |
|---|---|---|
| Handles real-time data | Yes | No (items shift between pages) |
| Jump to specific page | No | Yes |
| Load more / infinite scroll | Ideal | Works but fragile |
| Requires unique sequential column | Yes | No |

---

## Schema pattern

```graphql
# Define the result type implementing the interface
type local_todo_items_result implements core_pageable_result {
  """List of todo items for the current page."""
  items: [local_todo_item!]!
  """Total count of matching items (optional — omit for rapidly changing data)."""
  total: Int
  """Opaque cursor string. Pass as `cursor` on the next request to get the next page.
     Empty string indicates the last page."""
  next_cursor: String!
}

extend type Query {
  local_todo_items(
    cursor: String
    filters: local_todo_items_filter
  ): local_todo_items_result!
}
```

> Implementing `core_pageable_result` also means the framework's generic type resolver can handle `items`, `total`, and `next_cursor` automatically when using ORM data providers.

---

## Query usage (persisted query)

```graphql
query local_todo_items($cursor: String, $filters: local_todo_items_filter) {
  local_todo_items(cursor: $cursor, filters: $filters) {
    items {
      id
      title
      completed_at
    }
    next_cursor
    total
  }
}
```

First request: omit `cursor` (or pass `null`). Server uses defaults for page size.

Subsequent requests: pass the `next_cursor` value returned from the previous response.

End of results: server returns `""` (empty string) as `next_cursor`.

---

## Opaque cursor format

Cursors are **base64-encoded JSON** and must be treated as opaque strings by clients — do not parse or construct them manually.

**Offset-based cursor** (internal encoding):

```json
{ "page": 1, "limit": 20 }
```

**Column-based cursor** (internal encoding, for rapidly changing data):

```json
{ "limit": 20, "columns": { "created_at": 1586403210, "id": 125 } }
```

The `columns` cursor requires a unique, sequential sort column (or combination). Always include a secondary sort on `id` when the primary sort column is not unique.

---

## Resolver implementation

```php
namespace local_todo\webapi\resolver\query;

use core\webapi\execution_context;
use core\webapi\query_resolver;
use core\orm\pagination\offset_cursor_paginator;

class items extends query_resolver {
    public static function resolve(array $args, execution_context $ec): array {
        require_login();

        $cursor = $args['cursor'] ?? null;
        $page_size = 20;

        $query = \local_todo\model\item::repository()
            ->order_by('created_at')
            ->order_by('id'); // secondary sort for uniqueness

        $paginator = new offset_cursor_paginator($query, $cursor, $page_size);

        return [
            'items'       => $paginator->get_items(),
            'total'       => $paginator->get_total(),
            'next_cursor' => $paginator->get_next_cursor() ?? '',
        ];
    }
}
```

---

## Example response

```json
{
  "data": {
    "local_todo_items": {
      "items": [
        { "id": "1", "title": "Task 1", "created_at": 1586403200 },
        { "id": "3", "title": "Task 2", "created_at": 1586403205 }
      ],
      "next_cursor": "eyJsaW1pdCI6MjAsImNvbHVtbnMiOnsiY3JlYXRlZF9hdCI6MTU4NjQwMzIwNSwiaWQiOjN9fQ==",
      "total": 42
    }
  }
}
```

When `next_cursor` is `""`, no more pages exist.

---

## `total` field guidance

- **Include** `total` for relatively static datasets where it helps the UI (e.g. report pagination).
- **Omit or leave nullable** for rapidly changing datasets — the count can be stale by the time the client displays it, and it adds query cost.
- Even when defined as `Int` (non-null) in the schema, returning `null` from the resolver for `total` is allowed if the type is `Int` (nullable). Use `Int!` only when you always return a count.
