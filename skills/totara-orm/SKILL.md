---
name: totara-orm
description: >-
  Totara ORM: defining entities, creating entity models (mandatory pairing),
  entity repositories, entity relationships, filters, query builder, and
  paginator. Use when writing or querying database records via ORM, creating
  entity classes, entity model classes, custom repositories, relationship
  definitions, filter objects, or builder queries in Totara PHP code.
---

# Totara ORM

## Scope

Covers all PHP ORM usage in Totara: entities, entity models, repositories, relationships, filters, query builder (`\core\orm\query\builder`), and paginator. Always uses `\core\orm` namespace.

---

## MANDATORY: Entity + Entity Model pairing

> **Rule #1 — Every entity MUST have a corresponding entity model.**
>
> - **Entity** = persistence layer only (ActiveRecord). Maps 1:1 to a DB table row. No business logic.
> - **Entity Model** = domain layer (rich model). Wraps the entity, enforces business rules, exposes properties safely.
>
> Creating an entity without its paired entity model is **incomplete and non-compliant** with Totara ORM conventions. The entity model is the compulsory interface between the persistence layer and the rest of the application (services, GraphQL resolvers, etc.). Always create both.

### Entity vs Entity Model at a glance

| Dimension | Entity | Entity Model |
|---|---|---|
| Base class | `\core\orm\entity\entity` | `\core\orm\entity\model` |
| Purpose | Persistence — maps to a DB table row | Domain logic — always valid state |
| Business logic | None (accessors/mutators for shaping only) | Yes — invariants, transformations, service methods |
| Write access | Direct property assignment | Via factory/update methods you define |
| Property exposure | All columns + `$extra_attributes` | Whitelist via `$entity_attribute_whitelist` / `$model_accessor_whitelist` |
| When to use | ORM queries, repos, relationships, raw data loading | GraphQL resolvers, service layer, enforcing rules |
| Relationship | Entity is the persistence layer **inside** the model (`$this->entity`) | Model wraps entity and delegates persistence to it |

---

## Defining an entity

```php
/**
 * @property int    $id
 * @property string $fullname
 * @property string $idnumber
 * @property int    $timecreated
 * @property int    $timemodified
 */
class competency extends \core\orm\entity\entity {

    public const TABLE = 'comp';

    // Optional: automatic timestamp management
    public const CREATED_TIMESTAMP  = 'timecreated';
    public const UPDATED_TIMESTAMP  = 'timemodified';
}
```

- `TABLE` constant = DB table name (without prefix).
- PHPDoc `@property` annotations are optional but strongly recommended for IDE autocompletion.
- Timestamps: define `CREATED_TIMESTAMP` / `UPDATED_TIMESTAMP` constants and they are managed automatically on `save()`.

---

## Entity CRUD

```php
// Load by ID — throws exception if not found
$competency = new competency(123);

// Load by ID — returns null if not found
$competency = competency::repository()->find(123);

// Create
$competency = new competency();
$competency->fullname = 'OOP Design';
$competency->save();
echo $competency->id; // auto-set after save

// Update — only dirty (changed) attributes are written
$competency = new competency(123);
$competency->fullname = 'New name';
$competency->save();

// Delete
$competency = new competency(123);
$competency->delete();

// Refresh from DB
$competency->refresh();

// Convert to array
$competency->to_array();
```

---

## Defining an entity model (mandatory counterpart)

```php
/**
 * @property-read int    $id
 * @property-read string $fullname
 */
class competency_model extends \core\orm\entity\model {

    // Required: point to the entity class
    public static function get_entity_class(): string {
        return competency::class;
    }

    // Restrict which entity attributes are readable (recommended)
    protected $entity_attribute_whitelist = ['id', 'fullname', 'idnumber'];

    // Expose getter methods as properties
    protected $model_accessor_whitelist = ['display_name'];

    public function get_display_name(): string {
        return strtoupper($this->entity->fullname);
    }
}

// Load by ID
$model = competency_model::load_by_id(42);

// Load from an existing entity
$entity = competency::repository()->find_or_fail(42);
$model = competency_model::load_by_entity($entity);

// Access properties (read-only)
echo $model->fullname;
echo $model->display_name;
```

See [references/entity-models.md](references/entity-models.md) for full model details, formatter integration, and end-to-end worked example.

---

## Entity repository

```php
// Factory method (preferred)
$repo = competency::repository();

// Or instantiate directly
$repo = new \core\orm\entity\repository(competency::class);

// Fetch all
$competencies = competency::repository()->get();

// With conditions
$competencies = competency::repository()
    ->where('status', 1)
    ->order_by('fullname')
    ->get();

// Single record (throws if multiple rows match)
$competency = competency::repository()
    ->where('idnumber', 'fw1')
    ->one();

// Paginate (returns paginator)
$paginator = competency::repository()->paginate(1, 20);

// Lazy collection (memory-efficient for large sets)
$records = competency::repository()->get_lazy();
foreach ($records as $record) { ... }

// From existing DML records
$records = $DB->get_records_sql($sql, $params);
$competencies = competency::repository()->from_records($records);
```

### Specific repository

Create `{entity}_repository` in the same namespace as the entity:

```php
class competency_repository extends \core\orm\entity\repository {
    public function active(): self {
        $this->where('status', 1);
        return $this;
    }
}

// usage
$competencies = competency::repository()->active()->get();
```

See [references/entities-and-repositories.md](references/entities-and-repositories.md) for full details.

---

## Entity relationships summary

| Type | Method | Description |
|---|---|---|
| `has_one` | `$this->has_one(EntityClass::class, 'foreign_key')` | One-to-one: this table has a related row |
| `has_many` | `$this->has_many(EntityClass::class, 'foreign_key')` | One-to-many: this table has many related rows |
| `has_many_through` | `$this->has_many_through(Intermediate::class, Target::class, ...)` | Many through an intermediate table |
| `has_one_through` | `$this->has_one_through(Intermediate::class, Target::class, ...)` | One through an intermediate table |
| `belongs_to` | `$this->belongs_to(EntityClass::class, 'foreign_key')` | Inverse of has_one / has_many |

```php
// Access via property (cached, preferred)
$modules = $course->modules;

// Access via method (re-queries, allows extra constraints)
$modules = $course->modules()->where('visible', 1)->get();

// Eager load to avoid N+1
$courses = course::repository()->with('modules')->get();
$courses = course::repository()->with(['modules', 'category'])->get();
```

See [references/relationships.md](references/relationships.md) for all relationship types and eager loading details.

---

## Filters

```php
// Define a reusable filter
class status_filter extends \core\orm\entity\filter\filter {
    public function apply() {
        if ($this->value !== null) {
            $this->builder->where('status', $this->value);
        }
    }
}

// Register as default filter on a repository
class competency_repository extends \core\orm\entity\repository {
    protected function get_default_filters(): array {
        return ['status' => new status_filter()];
    }
}

// Use on the repository
$competencies = competency::repository()
    ->set_filters(['status' => 1])
    ->get();

// Generic built-in filters
use \core\orm\entity\filter\equal;
use \core\orm\entity\filter\like;
use \core\orm\entity\filter\in;

competency::repository()->set_filter(new equal('idnumber', 'fw1'))->get();
competency::repository()->set_filter((new like('fullname'))->set_value('OOP'))->get();
competency::repository()->set_filter((new in('id'))->set_value([1,2,3]))->get();
```

See [references/filters.md](references/filters.md) for full details.

---

## Query builder cheatsheet

```php
use \core\orm\query\builder;
```

| Task | Method |
|---|---|
| Start from a table | `builder::table('user')` |
| Select columns | `->select(['id', 'name'])` / `->add_select('email')` |
| WHERE equal | `->where('status', 1)` |
| WHERE operator | `->where('votes', '>=', 10)` |
| WHERE OR | `->or_where('name', 'Jane')` |
| WHERE IN | `->where_in('id', [1, 2, 3])` / `->where('id', [1,2,3])` |
| WHERE NULL | `->where_null('deleted')` / `->where_not_null('email')` |
| WHERE LIKE | `->where_like('name', 'Bob')` |
| WHERE nested | `->where(function(builder $b) { $b->where(...)->or_where(...); })` |
| Inner join | `->join('other_table', 'foreign_key', 'id')` |
| Left join | `->left_join('other_table', 'foreign_key', 'id')` |
| Order | `->order_by('name', 'asc')` |
| Group by | `->group_by('status')` |
| Limit / offset | `->limit(10)->offset(20)` |
| Get collection | `->get()` |
| Get array | `->fetch()` |
| Single row | `->first()` / `->first_or_fail()` |
| One row (strict) | `->one()` |
| Find by ID | `->find(123)` / `->find_or_fail(123)` |
| Count | `->count()` |
| Exists | `->exists()` / `->does_not_exist()` |
| Single value | `->value('column')` |
| Paginate | `->paginate($page, $per_page)` / `->load_more($page, $per_page)` |
| Lazy recordset | `->fetch_recordset()` |
| Map results | `->map_to(MyClass::class)` / `->results_as_arrays()` |
| Subquery source | `builder::table($sub_builder)->as('alias')` |
| Raw where | `->where_raw('name = :name', ['name' => 'Bob'])` |
| Raw select | `->select_raw('id, :val as extra', ['val' => 1])` |

See [references/query-builder.md](references/query-builder.md) for advanced usage (subquery joins, grouping, HAVING, unions, insert/update/delete, raw methods).

---

## Paginator

```php
// Full pagination (page/pages/next/prev/total)
$paginator = competency::repository()->paginate(1, 20);
$paginator->to_array();
// ['items'=>[...], 'page'=>1, 'pages'=>5, 'items_per_page'=>20, 'next'=>2, 'prev'=>null, 'total'=>98]

// Simple pagination (no count, "load more" style)
$paginator = builder::table('comp')->load_more(1, 20);
// ['items'=>[...], 'page'=>1, 'items_per_page'=>20]

foreach ($paginator as $item) { ... }
```

See [references/paginator.md](references/paginator.md) for standalone usage and full method reference.

---

## Quick-start checklist

### New table + ORM from scratch

```
- [ ] 1. Define entity class (extend entity, set TABLE constant, PHPDoc @property annotations)
- [ ] 2. Define entity model class (extend model, implement get_entity_class(), set whitelists) — MANDATORY
- [ ] 3. Create specific repository if custom query methods are needed (optional)
- [ ] 4. Define relationships on the entity if related tables exist
- [ ] 5. Create filter classes for reusable query conditions (optional)
- [ ] 6. Use entity model in services / GraphQL resolvers — never expose the raw entity
```

### Querying existing data

```
- [ ] 1. Use entity::repository() for ORM-aware queries returning entity instances
- [ ] 2. Use builder::table() for complex SQL (joins, aggregates, subqueries) returning plain objects/arrays
- [ ] 3. Eager-load relationships with ->with('relation') to avoid N+1 queries
- [ ] 4. Use ->paginate() or ->load_more() for paginated results
- [ ] 5. Wrap results in entity model before returning to service/resolver layer
```

---

## Hard rules

1. **Always create an entity model alongside every entity** — the entity handles persistence only; the model is the mandatory domain layer.
2. **Never put business logic in entities** — use accessors/mutators for attribute shaping only; all rules and transformations belong in the model.
3. **Never expose raw entities to service or resolver layers** — always wrap in an entity model first.
4. **Always use `::repository()` instead of direct `$DB` calls** when ORM entities exist for a table.
5. **Always eager-load related entities** (`->with(...)`) when iterating over a collection that accesses relations — never lazy-load inside a loop.
6. **Raw SQL in `where_raw` / `select_raw` requires manual SQL injection protection** — always use named parameters.
7. **Do not name a relationship the same as an existing entity column** — this causes silent attribute/relation conflicts.
8. **`has_many_through` and `has_one_through` do not support saving related models or column-specific eager loading** — use `has_many` / `has_one` with a direct FK when you need those features.

---

## Reference files

| File | When to read |
|---|---|
| [references/entities-and-repositories.md](references/entities-and-repositories.md) | Full entity definition, accessors, mutators, extra_attributes, timestamps, DML interop, specific repository, `repository_class_name` |
| [references/entity-models.md](references/entity-models.md) | Model scaffold, property/accessor whitelists, formatter integration, complete entity+model+formatter worked example |
| [references/relationships.md](references/relationships.md) | All 5 relationship types with examples, eager loading, column selection, closure conditions, bulk update/delete, inserting related entities |
| [references/filters.md](references/filters.md) | Filter class with `apply()`, params, default vs on-demand filters, generic filter examples |
| [references/query-builder.md](references/query-builder.md) | Advanced builder usage: subquery joins, grouping, HAVING, raw methods, unions, insert/update/delete via builder |
| [references/paginator.md](references/paginator.md) | Simple vs full mode, standalone usage, complete function reference |
