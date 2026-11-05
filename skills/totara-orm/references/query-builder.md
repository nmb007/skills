# Query Builder

Advanced reference for `\core\orm\query\builder`. Organized by intent — jump to the section you need. For the most common methods, see the cheatsheet table in `SKILL.md`.

```php
use \core\orm\query\builder;
```

The builder lives under `\core\orm\query`. All examples assume the `use` statement above. Named parameters are always used for values (SQL injection protection built-in). Raw methods bypass this — see the **Raw methods** section.

---

## Section index

1. [Starting a query](#1-starting-a-query)
2. [Retrieving results](#2-retrieving-results)
3. [Selects](#3-selects)
4. [Joins](#4-joins)
5. [Where clauses](#5-where-clauses)
6. [Ordering](#6-ordering)
7. [Grouping and HAVING](#7-grouping-and-having)
8. [Limit and offset](#8-limit-and-offset)
9. [Result mapping](#9-result-mapping)
10. [Raw methods](#10-raw-methods)
11. [Insert, update, delete](#11-insert-update-delete)
12. [Unions](#12-unions)

---

## 1. Starting a query

### From a table

```php
builder::table('user')          // alias defaults to table name
    ->...;

(new builder())
    ->from('user', 'u')         // explicit alias
    ->...;
```

### From a subquery

```php
$sub = builder::table('user')->where('status', 1);

$users = builder::table($sub)
    ->as('sq')                  // alias is required for subquery sources
    ->where('name', 'like', 'John')
    ->first();
// SQL: SELECT "sq".* FROM (SELECT * FROM {user} WHERE status = ?) sq WHERE "sq".name LIKE ? LIMIT 1
```

---

## 2. Retrieving results

### Single row methods

| Method | Returns | Throws if missing | Notes |
|---|---|---|---|
| `find($id)` | stdClass / null | No | Ignores conditions; always fetches all columns |
| `find_or_fail($id)` | stdClass | Yes | Same as find but throws exception |
| `first()` | stdClass / null | No | Requires `order_by()`; cannot use with `limit()` |
| `first_or_fail()` | stdClass | Yes | Same as first but throws |
| `one()` | stdClass / null | Optional (`$strict=true`) | Throws if multiple rows match; cannot use with `offset()`/`limit()` |

```php
$user = builder::table('user')->find(1);
$user = builder::table('user')->find_or_fail(1);
$user = builder::table('user')->where('status', 1)->order_by('name')->first();
$user = builder::table('user')->where('idnumber', 'u001')->one();
$user = builder::table('user')->where('idnumber', 'u001')->one(true); // throws if not found
```

### Multiple row methods

| Method | Returns | Notes |
|---|---|---|
| `fetch()` | Array of stdClass | Default — use when you don't need collection methods |
| `get()` | Collection | Preferred for ORM work; supports collection helper methods |
| `fetch_recordset()` / `get_lazy()` | `lazy_collection` (recordset) | Memory-efficient; forward-only iteration; no count |
| `fetch_counted()` | `['count' => int, 'items' => array]` | Single call returns items + total count |
| `paginate($page, $per_page)` | Paginator (full) | page / pages / next / prev / total |
| `load_more($page, $per_page)` | Paginator (simple) | No count; "load more" style |

```php
$users = builder::table('user')->fetch();
$users = builder::table('user')->get();               // returns Collection

// Lazy — use for large tables
foreach (builder::table('user')->fetch_recordset() as $user) { ... }

// Paginate
$paginator = builder::table('user')->where('deleted', 0)->paginate(1, 20);
foreach ($paginator as $user) { ... }
```

### Scalar / aggregate methods

| Method | Returns |
|---|---|
| `count()` | int — number of matching rows |
| `exists()` | bool |
| `does_not_exist()` | bool |
| `value('column')` | mixed — value of the column from the first matching row |

```php
$count  = builder::table('user')->where('status', 1)->count();
$exists = builder::table('user')->where('email', 'bob@example.com')->exists();
$name   = builder::table('user')->where('id', 5)->value('firstname');
```

---

## 3. Selects

### Field selection

`select()` **replaces** the select list. `add_select()` **appends** to it.

```php
builder::table('user')->select(['id', 'firstname'])->fetch();
builder::table('user')->select('id')->add_select('firstname as name')->fetch();
```

Field names are validated against `[a-zA-Z0-9_.*]` plus aliases/aggregates. Invalid names throw an exception.

### Aggregate functions

```php
builder::table('orders')
    ->select([
        'min(quantity) as min_qty',
        'max(quantity) as max_qty',
    ])
    ->add_select('avg(quantity) as avg_qty')
    ->add_select('count(id) as total')
    ->first();
```

Supported aggregates: `MIN`, `MAX`, `AVG`, `SUM`, `COUNT`.

### Subquery in select

```php
use core\orm\query\subquery;

builder::table('user')
    ->select(['id', 'firstname', new subquery(function (builder $sub) {
        $sub->from('orders')
            ->select('max(quantity)')
            ->where('user_id', 'user.id');
    })->as('max_order_quantity')])
    ->get();
```

### Raw select

When you need SQL that cannot be expressed as a simple field name, use raw methods. **Values must be parameterised manually.**

```php
builder::table('user')
    ->select_raw('id, firstname, :role as role', ['role' => 'admin'])
    ->first();

// Combining raw and non-raw
builder::table('user')
    ->select('id')
    ->add_select('firstname')
    ->add_select_raw(':label as label', ['label' => 'active'])
    ->first();

// Using the sql class for positional params
use core\dml\sql;
builder::table('user')
    ->select(new sql('id, firstname, ? as role', ['admin']))
    ->first();
```

---

## 4. Joins

### Basic joins

```php
// INNER JOIN (= is optional)
builder::table('orders')
    ->join('users', 'user_id', '=', 'id')
    ->select('*')
    ->add_select('users.name as user_name')
    ->get();

// LEFT / RIGHT / FULL / CROSS
builder::table('orders')->left_join('users', 'user_id', 'id')->get();
builder::table('orders')->right_join('users', 'user_id', 'id')->get();
builder::table('orders')->full_join('users', 'user_id', 'id')->get();
builder::table('orders')->cross_join('users')->get(); // no condition arguments
```

### Table alias in join

```php
use core\orm\query\table;

// Using table class
builder::table('orders')
    ->join((new table('users'))->as('u'), 'user_id', 'id')
    ->select('*')->add_select('u.name as user_name')
    ->get();

// Using array shorthand [tablename, alias]
builder::table('orders')
    ->join(['users', 'u'], 'user_id', 'id')
    ->select('*')->add_select('u.name as user_name')
    ->get();
```

### Multiple table joins

```php
builder::table('orders')
    ->join(['users', 'u'], 'user_id', 'id')
    ->join('goods', 'goods_item_id', 'id')
    ->select(['orders.id as order_id', 'u.name as user_name', 'goods.name as item_name'])
    ->get();
```

### Complex join conditions (closure)

```php
builder::table('orders')
    ->join('users', function (builder $join, builder $main) {
        $join->where_field('orders.user_id', 'users.id')
             ->where('users.status', 1);
    })
    ->get();
```

### Subquery join

```php
use core\orm\query\subquery;

$sub = builder::table('users')->where('status', 1);

builder::table('orders')
    ->join([$sub, 'u'], 'user_id', 'id')  // array shorthand
    ->select('*')->add_select('u.name as user_name')
    ->get();

// Or using subquery class
builder::table('orders')
    ->join((new subquery($sub))->as('u'), 'user_id', 'id')
    ->get();
```

---

## 5. Where clauses

### Basic

```php
// Two-arg form: operator defaults to = (or IN for arrays)
builder::table('user')->where('status', 1)->fetch();
builder::table('user')->where('id', [1, 2, 3])->fetch();    // IN (1,2,3)

// Three-arg form: explicit operator
builder::table('user')->where('votes', '>=', 10)->fetch();
builder::table('user')->where('name', '!=', 'Bob')->fetch();
```

### OR

```php
builder::table('user')
    ->where('name', 'John')
    ->or_where('name', 'Jane')
    ->fetch();
// WHERE name = ? OR name = ?
```

### Nested conditions (grouping)

```php
builder::table('user')
    ->where('status', 1)
    ->where(function (builder $b) {
        $b->where('name', 'John')
          ->or_where('name', 'Jane');
    })
    ->fetch();
// WHERE status = ? AND (name = ? OR name = ?)
```

### where_field — compare two columns

```php
builder::table('user')
    ->where_field('last_login', 'created')        // last_login = created
    ->fetch();

builder::table('user')
    ->where_field('last_login', '>', 'created')   // last_login > created
    ->fetch();
```

OR variant: `or_where_field`.

### where_null / where_not_null

```php
builder::table('user')->where_null('last_login')->fetch();
builder::table('user')->where_not_null('email')->fetch();
```

OR variants: `or_where_null`, `or_where_not_null`.

### where_in / where_not_in

```php
builder::table('user')->where_in('id', [1, 2, 3])->fetch();
builder::table('user')->where_not_in('status', [0, 2])->fetch();
```

OR variants: `or_where_in`, `or_where_not_in`.

### where_like — LIKE statements

| Method | SQL equivalent |
|---|---|
| `where_like($col, $val)` | `col LIKE '%val%'` (case-sensitive) |
| `where_like_starts_with($col, $val)` | `col LIKE 'val%'` |
| `where_like_ends_with($col, $val)` | `col LIKE '%val'` |
| `where_ilike` / `where_ilike_starts_with` / `where_ilike_ends_with` | Case-insensitive variants |
| `where('col', '!like', $val)` / `!ilike` etc. | NOT LIKE variants |

```php
builder::table('user')->where_like('name', 'Bob')->fetch();        // LIKE '%Bob%'
builder::table('user')->where_ilike('name', 'bob')->fetch();       // case-insensitive
builder::table('user')->where('name', '!like', 'Bob')->fetch();    // NOT LIKE '%Bob%'
```

For full control (no auto-wrapping with `%`):

```php
// Escape the value manually first
builder::table('user')
    ->where_like_raw('name', 'John%Doe')  // no auto-wrapping
    ->fetch();
// Warning: SQL injection risk — always escape values with $DB->sql_like_escape()
```

### where_exists

```php
$sub = builder::table('orders')
    ->select('id')
    ->where('user_id', 'users.id'); // comparing columns — not parameterised

builder::table('users')
    ->where_exists($sub)
    ->fetch();

// Or closure form
builder::table('users')
    ->where_exists(function (builder $b) {
        $b->select('id')
          ->from('orders')
          ->where_field('orders.user_id', 'users.id');
    })
    ->fetch();
```

OR variants: `or_where_exists`, `where_not_exists`, `or_where_not_exists`.

### Raw where clauses

```php
// Named params only (builder standard)
builder::table('user')
    ->where_raw('name = :name AND status = :status', ['name' => 'John', 'status' => 1])
    ->fetch();

// Or using sql class for positional params
use core\dml\sql;
builder::table('user')
    ->where(new sql('name = ? AND status = ?', ['John', 1]))
    ->fetch();
```

**Raw methods bypass SQL injection protection — always parameterise values.**

OR variant: `or_where_raw`.

---

## 6. Ordering

```php
// Single column
builder::table('user')->order_by('name')->fetch();           // ASC (default)
builder::table('user')->order_by('name', 'desc')->fetch();

// Multiple columns — each call appends
builder::table('user')
    ->order_by('status', 'asc')
    ->order_by('name', 'asc')
    ->fetch();

// Raw order
builder::table('user')
    ->order_by_raw('FIELD(status, 1, 0, 2)')
    ->fetch();
```

---

## 7. Grouping and HAVING

```php
builder::table('orders')
    ->select(['user_id', 'sum(quantity) as total'])
    ->group_by('user_id')
    ->having('total', '>', 100)
    ->fetch();

// Multiple group by columns
builder::table('orders')
    ->group_by('user_id')
    ->group_by('goods_item_id')
    ->fetch();

// Raw having
builder::table('orders')
    ->group_by('user_id')
    ->having_raw('sum(quantity) > :min', ['min' => 100])
    ->fetch();
```

---

## 8. Limit and offset

```php
builder::table('user')->limit(10)->fetch();
builder::table('user')->limit(10)->offset(20)->fetch();

// Cannot combine limit() with first() — use order_by()->limit()->fetch() instead
```

---

## 9. Result mapping

### Map to a class

```php
// Results as instances of a specific class
$users = builder::table('user')
    ->map_to(user_entity::class)
    ->get();

// Results as plain arrays instead of stdClass
$users = builder::table('user')
    ->results_as_arrays()
    ->fetch();

// Results as stdClass (default)
$users = builder::table('user')
    ->results_as_objects()
    ->fetch();
```

When `map_to()` is used with `fetch_recordset()`, each item is mapped as it is iterated (no full load into memory).

---

## 10. Raw methods

All raw methods embed strings directly into the SQL statement. **You must protect against SQL injection manually** — always use named or positional parameters and escape LIKE wildcards with `$DB->sql_like_escape()`.

| Method | Purpose |
|---|---|
| `select_raw($sql, $params)` | Replace select list with raw SQL |
| `add_select_raw($sql, $params)` | Append to select list with raw SQL |
| `where_raw($sql, $params)` | Add raw WHERE condition (AND) |
| `or_where_raw($sql, $params)` | Add raw WHERE condition (OR) |
| `having_raw($sql, $params)` | Add raw HAVING condition |
| `order_by_raw($sql)` | Add raw ORDER BY expression |
| `where_like_raw($col, $val)` | LIKE with no auto-% wrapping |
| `or_where_like_raw($col, $val)` | OR LIKE with no auto-% wrapping |

Using the `sql` class directly in any clause:

```php
use core\dml\sql;

builder::table('user')
    ->select(new sql('id, firstname, ? as role', ['admin']))
    ->where(new sql('status = ? OR status = ?', [1, 2]))
    ->fetch();
```

---

## 11. Insert, update, delete

The builder can execute DML operations directly without loading entities. Use this for bulk operations or when ORM entity overhead is unnecessary.

### Insert

```php
builder::table('comp')
    ->insert([
        'fullname'    => 'New Competency',
        'frameworkid' => 5,
        'timecreated' => time(),
    ]);

// Insert and return the new record ID
$id = builder::table('comp')
    ->insert_return_id([
        'fullname'    => 'New Competency',
        'frameworkid' => 5,
    ]);
```

### Update

```php
builder::table('comp')
    ->where('status', 0)
    ->update(['status' => 1, 'timemodified' => time()]);
```

### Delete

```php
builder::table('comp')
    ->where('frameworkid', 99)
    ->delete();
```

---

## 12. Unions

Combine two queries with UNION:

```php
$query1 = builder::table('comp')->select('id')->where('status', 1);
$query2 = builder::table('comp_framework')->select('id')->where('visible', 1);

$results = $query1->union($query2)->fetch();

// UNION ALL (include duplicates)
$results = $query1->union_all($query2)->fetch();
```

---

## Builder vs entity repository — when to use which

| Use case | Recommended tool |
|---|---|
| Fetch entities with ORM relationships | `entity::repository()` |
| Complex joins, aggregates, subqueries | `builder::table()` |
| Bulk insert/update/delete without loading entities | `builder::table()` |
| Need `map_to(MyEntity::class)` on custom SQL | `builder::table()->map_to(entity::class)` |
| Filters + pagination on an entity | `entity::repository()->set_filters()->paginate()` |
