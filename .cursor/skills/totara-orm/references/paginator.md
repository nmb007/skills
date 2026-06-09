# Paginator

Full reference for the Totara ORM paginator: simple mode, full mode, standalone usage, and the complete function reference.

---

## Overview

The paginator wraps a `builder` query and returns a page of results along with pagination metadata. It implements the `Iterator` interface so you can use `foreach` directly on it.

Two modes are available:

| Mode | Method | Counts total? | Output extras |
|---|---|---|---|
| **Full** | `paginate($page, $per_page)` | Yes — runs a COUNT query | `pages`, `next`, `prev`, `total` |
| **Simple** | `load_more($page, $per_page)` | No | Only `page` and `items_per_page` |

Use **simple** for "load more" UI patterns where you do not need to know the total number of pages.
Use **full** for traditional pagination with page numbers.

---

## Full pagination

```php
use \core\orm\query\builder;

$page     = 1;
$per_page = 10;

// Via repository (preferred for ORM entities)
$paginator = competency::repository()
    ->where('status', 1)
    ->order_by('fullname')
    ->paginate($page, $per_page);

// Via builder directly
$paginator = builder::table('comp')
    ->where('status', 1)
    ->paginate($page, $per_page);

// Iterate
foreach ($paginator as $item) {
    echo $item->fullname;
}

// to_array() output
print_r($paginator->to_array());
/*
Array
(
    [items] => Array (
        [0] => Array ( [id] => 1, [fullname] => 'Safety', ... )
        ...
    )
    [page]           => 1       // current page
    [pages]          => 5       // total pages
    [items_per_page] => 10
    [next]           => 2       // null on last page
    [prev]           => null    // null on first page
    [total]          => 48      // total matching records
)
*/
```

Passing `0` as the page number returns **all results** with full count information.

---

## Simple pagination ("load more")

```php
$page     = 1;
$per_page = 10;

// Via repository
$paginator = competency::repository()
    ->where('deleted', 0)
    ->load_more($page, $per_page);

// to_array() output — no count, pages, next, prev, or total
print_r($paginator->to_array());
/*
Array
(
    [items] => Array (
        [0] => Array ( ... )
        ...
    )
    [page]           => 1
    [items_per_page] => 10
)
*/
```

---

## Standalone paginator class

Use the `paginator` class directly when you need to construct it separately from the builder chain.

```php
use \core\orm\query\paginator;

$query = competency::repository()->where('status', 1)->order_by('fullname');

// Full mode (fourth argument false, or omitted)
$paginator = new paginator($query, $page, $per_page);

// Simple mode (fourth argument true)
$paginator = new paginator($query, $page, $per_page, true);
```

---

## With entity repository and filters

```php
$paginator = competency::repository()
    ->set_filters(['status' => 1, 'search' => 'Safety'])
    ->order_by('fullname')
    ->paginate(2, 20);

// Items are entity instances (not plain arrays)
foreach ($paginator as $entity) {
    echo $entity->fullname;
}
```

---

## Paginator method reference

| Method | Mode | Returns | Notes |
|---|---|---|---|
| `get_items()` | Both | Collection | Returns items as a collection |
| `to_array()` | Both | array | Items as arrays/objects + metadata (see above) |
| `get_page()` | Both | int | Current page number |
| `get_items_per_page()` | Both | int | Items per page |
| `get_pages()` | Full only | int | Total number of pages |
| `get_next()` | Full only | int\|null | Next page number; `null` on last page |
| `get_prev()` | Full only | int\|null | Previous page number; `null` on first page |
| `get_total()` | Full only | int | Total number of matching records |
| `count()` | Both | int | Number of items on the current page |

Calling `get_pages()`, `get_next()`, `get_prev()`, or `get_total()` in simple mode throws an exception.

---

## Notes on eager loading with pagination

Eager-loading relationships works with `paginate()` and `load_more()`. Only the current page of entities is loaded; their relations are fetched in a second query scoped to those IDs.

```php
$paginator = course::repository()
    ->with('modules')
    ->with('category')
    ->paginate(1, 10);

foreach ($paginator as $course) {
    foreach ($course->modules as $module) { ... } // no extra queries
}
```

> Related models are **not paginated** — all relations for the current page items are loaded at once.
