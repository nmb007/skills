# Entity Relationships

Full reference for all five relationship types, property vs method access, eager loading, bulk actions, and inserting related entities.

---

## Overview of relationship types

| Type | Method | Direction | Returns |
|---|---|---|---|
| `has_one` | `$this->has_one(Entity::class, 'fk')` | This table → related (1:1) | Single entity or null |
| `has_many` | `$this->has_many(Entity::class, 'fk')` | This table → related (1:N) | Collection |
| `has_many_through` | `$this->has_many_through(...)` | This → intermediate → target (N:M style) | Collection |
| `has_one_through` | `$this->has_one_through(...)` | This → intermediate → target (1:1 via join) | Single entity or null |
| `belongs_to` | `$this->belongs_to(Entity::class, 'fk')` | Inverse of has_one / has_many | Single entity or null |

---

## has_one

A one-to-one link. The foreign key is on the **related** table.

```php
/**
 * @property-read comp_type_info_data_param $param
 */
class comp_type_info_data extends entity {
    public const TABLE = 'comp_type_info_data';

    public function param(): has_one {
        // has_one(RelatedEntity::class, foreign_key_in_related_table)
        return $this->has_one(comp_type_info_data_param::class, 'dataid');
    }
}

class comp_type_info_data_param extends entity {
    public const TABLE = 'comp_type_info_data_param';
}
```

### Querying

```php
$data = comp_type_info_data::repository()->order_by('id')->first();

// Via method — executes query every call
$param = $data->param()->one();

// Via property — lazy-loaded and cached
$param = $data->param;
```

### Saving the related entity (auto-injects FK)

```php
$param = new comp_type_info_data_param(['value' => 'something']);
$data->param()->save($param); // 'dataid' is set automatically on $param
```

---

## has_many

A one-to-many link. The foreign key is on the **related** table.

```php
/**
 * @property-read \core\orm\collection $modules
 */
class course extends entity {
    public const TABLE = 'course';

    public function modules(): has_many {
        // has_many(RelatedEntity::class, foreign_key_in_related_table)
        return $this->has_many(course_module::class, 'course');
    }

    // Optional: with extra constraints baked in
    public function active_modules(): has_many {
        return $this->has_many(course_module::class, 'course')
                    ->where('visible', 1);
    }
}

class course_module extends entity {
    public const TABLE = 'course_modules';
}
```

> **Note:** Pluralise `has_many` relation method names — `modules()` not `module()`.

> **Note:** Be careful with `or_where` inside relation definitions — they are appended without grouping and can produce incorrect SQL unless wrapped in a nested condition.

### Querying

```php
$course = course::repository()->order_by('id')->first();

// Via property — cached
$modules = $course->modules;

// Via method — re-queries, allows extra constraints
$modules = $course->modules()
    ->where('name', 'module name')
    ->get();
// SQL: SELECT * FROM {course_modules} WHERE course = ? AND name = ?

// Use count on the method (no need to fetch all)
$count = $course->modules()->count();
```

### Inserting a related entity

```php
$course = course::repository()->first();
$course->modules()->save(new course_module(['module' => 5]));
// 'course' FK is auto-injected into the new course_module
```

---

## has_many_through

Connects two entities via an intermediate table.

```php
class competency extends entity {
    public const TABLE = 'comp';

    public function values(): has_many_through {
        return $this->has_many_through(
            competency_scale_assignment::class, // intermediate entity
            competency_scale_value::class,      // target entity
            'frameworkid',                      // FK in current table → intermediate
            'frameworkid',                      // key in intermediate table
            'scaleid',                          // FK in intermediate table → target
            'scaleid'                           // key in target table
        );
    }
}
```

```php
$competency = competency::repository()->order_by('id')->first();
$values = $competency->values;
// SQL: SELECT * FROM {comp_scale_values} JOIN {comp_scale_assignments}
//      ON scale_values.scaleid = scale_assignments.scaleid
//      WHERE scale_assignments.frameworkid = ?
```

### Limitations of has_many_through

- Saving related models via the relation is **not supported**.
- Selecting specific columns during eager loading is **not supported**.

---

## has_one_through

Same as `has_many_through` but returns a single related entity.

```php
class competency extends entity {
    public const TABLE = 'comp';

    public function scale(): has_one_through {
        return $this->has_one_through(
            competency_scale_assignment::class, // intermediate
            competency_scale::class,            // target
            'frameworkid',                      // FK in current → intermediate
            'frameworkid',                      // key in intermediate
            'scaleid',                          // FK in intermediate → target
            'id'                                // key in target
        );
    }
}
```

### Limitations of has_one_through

Same as `has_many_through`: no saving, no column-specific eager loading.

---

## belongs_to

The inverse of `has_one` / `has_many`. The foreign key is on **this** table.

```php
class course_module extends entity {
    public const TABLE = 'course_modules';

    // Foreign key 'course' is a column on THIS table
    public function a_course(): belongs_to {
        return $this->belongs_to(course::class, 'course');
    }
}
```

> Avoid naming the relation the same as an existing column — if the column is `course`, call the relation `a_course` or `parent_course` to prevent conflicts.

### Saving via belongs_to

You **cannot insert** related models via `belongs_to` (FK is on this entity). You **can update** them:

```php
$module->a_course()->update(['fullname' => 'New course name']);
```

---

## Property vs method access

| Access style | Behaviour | Use when |
|---|---|---|
| `$entity->relation` | Lazy-loaded on first access, result **cached** on entity | Preferred — no repeated queries |
| `$entity->relation()` | Always executes a fresh query | Needed when adding extra constraints or using count/recordset |

```php
// Cached — no extra query on second access
$modules1 = $course->modules;
$modules2 = $course->modules; // same in-memory collection

// Re-querying with extra constraint
$visible = $course->modules()->where('visible', 1)->get();
```

---

## Eager loading (avoiding N+1)

### Problem: N+1 with lazy loading

```php
$courses = course::repository()->get();
foreach ($courses as $course) {
    // This fires one query PER course — N+1 problem
    foreach ($course->modules as $module) { ... }
}
```

### Solution: eager loading with `with()`

```php
$courses = course::repository()
    ->with('modules')         // single extra query for all modules
    ->get();

foreach ($courses as $course) {
    // modules already loaded — no extra queries
    foreach ($course->modules as $module) { ... }
}
// SQL: SELECT * FROM {course}
// SQL: SELECT * FROM {course_modules} WHERE course IN (?,?,?,...)
```

### Multiple relations

```php
$courses = course::repository()
    ->with(['modules', 'category'])
    ->get();

// Or fluent
$courses = course::repository()
    ->with('modules')
    ->with('category')
    ->get();
```

### Always eager-load on these methods

`first()`, `first_or_fail()`, `one()`, `get()`, `paginate()`, `load_more()` — all support `with()`.

> Eager loading on recordsets (`get_lazy()` / `fetch_recordset()`) is **not supported**.

### Select specific columns when eager loading

```php
$courses = course::repository()
    ->with('modules:instance,section,idnumber') // id and FK are auto-prepended
    ->get();
```

### Apply conditions during eager loading

```php
$courses = course::repository()
    ->with(['modules' => function (\core\orm\entity\repository $repo) {
        $repo->where('visibleoncoursepage', true)
             ->order_by('added', 'desc');
    }])
    ->get();

// Combine column selection and conditions
$courses = course::repository()
    ->with(['modules:added,module' => function (\core\orm\entity\repository $repo) {
        $repo->where('visibleoncoursepage', true);
    }])
    ->get();
```

### Always-eager-loaded relations (defined on entity)

```php
class course extends entity {
    public const TABLE = 'course';

    protected $with = ['modules']; // always eager-loaded

    public function modules(): has_many {
        return $this->has_many(course_module::class, 'course');
    }
}
```

---

## Bulk actions on related entities

### Bulk update

```php
$course = course::repository()->first();

$course->modules()->update(['visible' => false]);
// SQL: UPDATE {course_modules} SET visible = 0 WHERE course = ?
```

### Bulk delete

```php
$course->modules()->delete();
// SQL: DELETE FROM {course_modules} WHERE course = ?
```

---

## Naming clashes

Avoid naming a relation the same as an existing entity column. For example, if a `course_module` entity has a `course` column, do not name the `belongs_to` relation `course` — use `a_course` or `parent_course` instead.

---

## Limitations summary

| Feature | has_one | has_many | has_many_through | has_one_through | belongs_to |
|---|---|---|---|---|---|
| Save related via relation | Yes | Yes | No | No | No (update only) |
| Column-specific eager load | Yes | Yes | No | No | Yes |
| Bulk update/delete | Yes | Yes | Yes | Yes | Yes |
| Extra conditions on definition | Yes | Yes | Limited | Limited | Yes |
