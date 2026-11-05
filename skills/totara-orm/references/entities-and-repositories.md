# Entities and Repositories

Full reference for entity definition, CRUD operations, advanced entity features, DML interoperability, and custom repositories.

---

## Entity definition

Every database table has a corresponding entity class. Extend `\core\orm\entity\entity` and set the `TABLE` constant.

```php
/**
 * @property int    $id
 * @property string $fullname
 * @property string $idnumber
 * @property int    $status
 * @property int    $timecreated
 * @property int    $timemodified
 */
class competency extends \core\orm\entity\entity {

    public const TABLE = 'comp';

    public const CREATED_TIMESTAMP  = 'timecreated';
    public const UPDATED_TIMESTAMP  = 'timemodified';
}
```

- `TABLE` = DB table name without the `mdl_` prefix.
- PHPDoc `@property` annotations are optional but strongly recommended for IDE autocompletion, refactoring, and find-usages.
- Columns are handled as **attributes** internally — no need to declare them as PHP properties.
- Accessing a non-existent attribute emits a debugging message.
- Setting a non-existent attribute throws an exception.

---

## Retrieving entities

### Load a single entity by ID

```php
// Throws exception if record does not exist
$competency = new competency(123);

// Returns null if not found
$competency = competency::repository()->find(123);

// Throws exception if not found
$competency = competency::repository()->find_or_fail(123);
```

### Access attributes

```php
echo $competency->fullname;
$data = $competency->to_array(); // returns all column values as array
```

### Refresh from database

```php
$competency = new competency(123);
$competency->fullname = 'new name';
$competency->refresh();         // reloads from DB, discards local changes
echo $competency->fullname;     // original value
var_dump($competency->changed()); // false
```

---

## Inserting entities

```php
$framework = new competency_framework();
$framework->fullname  = 'My Framework';
$framework->idnumber  = 'fw1';
$framework->save();

echo $framework->id; // auto-set after save
```

`save()` is intelligent — it detects whether to INSERT or UPDATE based on whether `id` is set. You can also call `create()` or `update()` explicitly.

---

## Updating entities

```php
$competency = new competency(123);
$competency->fullname = 'New name';

var_dump($competency->changed()); // true
$competency->save(); // only dirty (changed) attributes are written to DB
```

The entity tracks "dirty attributes" internally. Unchanged attributes are never written to the database.

---

## Deleting entities

```php
$competency = new competency(123);
$competency->delete();

var_dump($competency->deleted()); // true
echo $competency->id;             // null after delete
```

---

## Timestamp handling

Define `CREATED_TIMESTAMP` and/or `UPDATED_TIMESTAMP` constants. The entity sets them automatically on `save()`.

```php
class user extends \core\orm\entity\entity {
    public const TABLE              = 'user';
    public const CREATED_TIMESTAMP  = 'timecreated';
    public const UPDATED_TIMESTAMP  = 'timemodified';
}

$user = new user();
$user->firstname = 'Bob';
$user->save();
echo $user->timecreated; // auto-set on insert
echo $user->timemodified; // null on insert

$user->firstname = 'Robert';
$user->save();
echo $user->timemodified; // auto-set on update

// Skip timestamp update for this single call
$user->do_not_update_timestamps()->save();
```

---

## Accessors

Accessors modify or derive attribute values on read. Define a method matching `get_{attribute}_attribute()`.

```php
/**
 * @property int    $status
 * @property-read string $status_name
 */
class competency extends \core\orm\entity\entity {

    public const TABLE = 'comp';

    protected function get_status_name_attribute(): string {
        return $this->status === 1 ? 'active' : 'disabled';
    }
}

$competency = new competency(123);
echo $competency->status;       // 0
echo $competency->status_name;  // 'disabled'
```

If the underlying column exists it is passed as the argument to the accessor. Derived attributes (computed from other fields) receive no argument.

---

## Mutators

Mutators transform a value before it is stored in the internal attributes array. Define a method matching `set_{attribute}_attribute($value)`.

```php
protected function set_idnumber_attribute($value): void {
    $this->attributes['idnumber'] = strtolower($value);
}
```

Always write to `$this->attributes[...]` inside the mutator — direct `$this->property = ...` inside a mutator causes infinite recursion.

---

## Extra attributes

By default `to_array()` returns only DB column values. To include computed/accessor attributes in `to_array()`, list them in `$extra_attributes`.

```php
/**
 * @property-read string $status_name
 */
class competency extends \core\orm\entity\entity {

    public const TABLE = 'comp';

    protected $extra_attributes = ['status_name'];

    protected function get_status_name_attribute(): string {
        return $this->status === 1 ? 'active' : 'disabled';
    }
}

print_r(competency::repository()->find(1)->to_array());
// ['id'=>1, 'status'=>1, 'status_name'=>'active', ...]
```

For dynamic extra attributes, use `add_extra_attribute()`:

```php
public function with_status_name(): self {
    $this->add_extra_attribute('status_name');
    return $this;
}

$competency->with_status_name()->to_array();
```

---

## DML interoperability

You can wrap plain DML records inside entities to benefit from ORM features without rewriting existing queries.

```php
// Single record from DML
$record = $DB->get_record('comp', ['id' => 123]);
$competency = new competency($record);
echo $competency->fullname;

// Collection from arbitrary SQL
$sql = "SELECT c.* FROM {comp} c JOIN {comp_framework} cf ON c.frameworkid = cf.id WHERE cf.visible = 1";
$records = $DB->get_records_sql($sql, []);

// from_records() validates attributes and returns a collection of entities
$competencies = competency::repository()->from_records($records);
```

`from_records()` validates that the incoming fields match entity attributes (by default). Pass `false` as the second argument to skip validation.

---

## Entity repository — basic usage

```php
// Factory method (preferred — returns specific repository if one is defined)
$repo = competency::repository();

// Direct instantiation
$repo = new \core\orm\entity\repository(competency::class);

// Get all as collection
$competencies = competency::repository()->get();

// Filtered
$competencies = competency::repository()
    ->where('status', 1)
    ->order_by('fullname')
    ->get();

// Iterate with foreach (collection implements Iterator)
foreach ($competencies as $competency) {
    echo $competency->fullname;
}

// Paginate
$paginator = competency::repository()->paginate(2, 20);
print_r($paginator->to_array());

// Lazy collection (memory-efficient for large result sets)
$records = competency::repository()->get_lazy();
foreach ($records as $record) {
    print_r($record->to_array());
}
```

### Repository save / delete helpers

```php
$user = new user(['firstname' => 'Bob', 'lastname' => 'Jones']);

user::repository()->create_entity($user);
user::repository()->save_entity($user);    // handles create or update
user::repository()->delete_entity($user);
user::repository()->refresh_entity($user);
```

---

## Specific repository

When you need custom reusable query methods for an entity, create a repository class in the **same namespace** named `{entity_classname}_repository`.

```php
/**
 * @method static competency_repository repository()
 */
class competency extends \core\orm\entity\entity {
    public const TABLE = 'comp';
}

class competency_repository extends \core\orm\entity\repository {

    /**
     * Scope to active competencies.
     */
    public function active(): self {
        $this->where('status', 1);
        return $this;
    }

    /**
     * Filter by framework.
     */
    public function for_framework(int $framework_id): self {
        $this->where('frameworkid', $framework_id);
        return $this;
    }

    /**
     * Filter by name or idnumber (example nested OR condition).
     */
    public function filter_by_name(string $name): self {
        $this->builder->where(function (\core\orm\query\builder $builder) use ($name) {
            $builder->where('fullname', $name)
                    ->or_where('idnumber', $name);
        });
        return $this;
    }
}

// Usage
$competencies = competency::repository()
    ->active()
    ->for_framework(5)
    ->filter_by_name('Safety')
    ->get();
```

### Custom repository class name

If you cannot follow the naming convention, override `repository_class_name()` on the entity:

```php
class competency extends \core\orm\entity\entity {
    public const TABLE = 'comp';

    public static function repository_class_name(): string {
        return my_custom_competency_repository::class; // must extend \core\orm\entity\repository
    }
}
```
