# Entity Models

Full reference for the entity model layer: scaffolding, property access, whitelisting, accessor methods, formatter integration, and a complete end-to-end worked example.

---

## Mandatory pairing rule

> **Every entity MUST have a corresponding entity model.**
>
> Entities are anemic persistence objects. They must not be exposed directly to services or resolvers.
> The entity model is the **compulsory domain layer** that enforces business rules and controls what the rest of the application can access. Building one without the other is non-compliant with Totara ORM conventions.

---

## What is an entity model?

Based on Domain Driven Design principles:

- **Entity** = anemic, persistence-only. Use for querying/saving DB rows.
- **Entity Model** = rich model. Always in a valid state. Contains business logic. Exposes a controlled API.

A model holds a reference to its entity internally (`$this->entity`) and delegates persistence through it. It never makes DB calls directly.

---

## Basic scaffold

```php
class competency_model extends \core\orm\entity\model {

    /**
     * Required: return the FQCN of the entity this model wraps.
     */
    public static function get_entity_class(): string {
        return competency::class;
    }
}
```

### Loading a model

```php
// Load by primary key
$model = competency_model::load_by_id(42);

// Load from an entity you already have
$entity = competency::repository()->find_or_fail(42);
$model  = competency_model::load_by_entity($entity);
```

---

## Property access — default behaviour

By default, **all entity attributes are readable** via a magic getter (including defined relationships), and there is **no write access**:

```php
echo $model->fullname;  // reads entity->fullname
echo $model->id;
```

---

## Entity attribute whitelist

Restrict which entity columns are readable on the model using `$entity_attribute_whitelist`. When the whitelist is defined, only listed properties are accessible — accessing anything else throws an exception.

```php
/**
 * @property-read int    $id
 * @property-read string $fullname
 * @property-read string $idnumber
 * @property-read int    $timecreated
 */
class competency_model extends \core\orm\entity\model {

    public static function get_entity_class(): string {
        return competency::class;
    }

    protected $entity_attribute_whitelist = [
        'id',
        'fullname',
        'idnumber',
        'timecreated',
        // 'parent_id', // intentionally excluded
    ];
}

$model = competency_model::load_by_id(42);
echo $model->fullname;  // ok
echo $model->parent_id; // throws exception
```

---

## Accessor whitelist (getter methods as properties)

Expose custom getter methods as magic properties using `$model_accessor_whitelist`. List the method name **without** the `get_` prefix.

Properties in `$model_accessor_whitelist` take precedence over `$entity_attribute_whitelist`.

```php
/**
 * @property-read string   $display_name
 * @property-read DateTime $created_at
 */
class competency_model extends \core\orm\entity\model {

    public static function get_entity_class(): string {
        return competency::class;
    }

    protected $model_accessor_whitelist = [
        'display_name',
        'created_at',
    ];

    public function get_display_name(): string {
        return strtoupper($this->entity->fullname);
    }

    public function get_created_at(): \DateTime {
        return (new \DateTime())->setTimestamp($this->entity->timecreated);
    }
}

$model = competency_model::load_by_id(42);

// Via property
echo $model->display_name;
$dt = $model->created_at;

// Or via method directly
echo $model->get_display_name();
```

---

## Business logic in models

Put all domain rules and transformations in the model, not the entity:

```php
class competency_model extends \core\orm\entity\model {

    public static function get_entity_class(): string {
        return competency::class;
    }

    protected $entity_attribute_whitelist = ['id', 'fullname', 'status', 'frameworkid'];

    /**
     * Factory: create a new competency and persist it.
     */
    public static function create(string $fullname, int $framework_id): self {
        $entity = new competency();
        $entity->fullname    = trim($fullname);
        $entity->frameworkid = $framework_id;
        $entity->status      = 1;
        $entity->save();
        return static::load_by_entity($entity);
    }

    /**
     * Business rule: rename is only allowed when status is active.
     */
    public function rename(string $new_name): void {
        if ($this->entity->status !== 1) {
            throw new \coding_exception('Cannot rename an inactive competency.');
        }
        $this->entity->fullname = trim($new_name);
        $this->entity->save();
    }

    /**
     * Business rule: deactivate.
     */
    public function deactivate(): void {
        $this->entity->status = 0;
        $this->entity->save();
    }
}
```

---

## Formatter integration

Use `entity_model_formatter` to integrate models with Totara's GraphQL formatter system.

```php
use core\webapi\formatter\field\string_field_formatter;
use core\webapi\formatter\field\date_field_formatter;

class competency_model_formatter extends \core\orm\formatter\entity_model_formatter {

    protected function get_map(): array {
        return [
            'id'           => null,                          // no transformation
            'fullname'     => string_field_formatter::class, // format_string()
            'idnumber'     => null,
            'status'       => null,
            'timecreated'  => date_field_formatter::class,   // unix timestamp formatting
            'timemodified' => date_field_formatter::class,
        ];
    }
}
```

Usage in a GraphQL type resolver:

```php
public static function resolve(string $field, $model, array $args, execution_context $ec) {
    $formatter = new competency_model_formatter($model, \context_system::instance());
    return $formatter->format($field, $args['format'] ?? null);
}
```

---

## Complete end-to-end worked example

This example shows the full mandatory pair: entity + entity model + repository + formatter.

### 1. Entity (persistence layer)

```php
// classes/entity/competency.php
namespace local_myplugin\entity;

/**
 * @property int    $id
 * @property string $fullname
 * @property string $idnumber
 * @property int    $frameworkid
 * @property int    $status
 * @property int    $timecreated
 * @property int    $timemodified
 */
class competency extends \core\orm\entity\entity {

    public const TABLE              = 'comp';
    public const CREATED_TIMESTAMP  = 'timecreated';
    public const UPDATED_TIMESTAMP  = 'timemodified';

    /**
     * Relationship: belongs to a framework.
     *
     * @return \core\orm\entity\relations\belongs_to
     */
    public function framework(): \core\orm\entity\relations\belongs_to {
        return $this->belongs_to(competency_framework::class, 'frameworkid');
    }
}
```

### 2. Specific repository

```php
// classes/entity/competency_repository.php
namespace local_myplugin\entity;

class competency_repository extends \core\orm\entity\repository {

    public function active(): self {
        $this->where('status', 1);
        return $this;
    }

    public function for_framework(int $framework_id): self {
        $this->where('frameworkid', $framework_id);
        return $this;
    }
}
```

### 3. Entity model (domain layer — MANDATORY)

```php
// classes/model/competency.php
namespace local_myplugin\model;

use local_myplugin\entity\competency as competency_entity;

/**
 * @property-read int    $id
 * @property-read string $fullname
 * @property-read string $idnumber
 * @property-read int    $status
 * @property-read int    $timecreated
 * @property-read string $status_label
 */
class competency extends \core\orm\entity\model {

    public static function get_entity_class(): string {
        return competency_entity::class;
    }

    protected $entity_attribute_whitelist = [
        'id',
        'fullname',
        'idnumber',
        'frameworkid',
        'status',
        'timecreated',
        'timemodified',
    ];

    protected $model_accessor_whitelist = [
        'status_label',
    ];

    // ---- Factory methods ----

    public static function create(string $fullname, string $idnumber, int $framework_id): self {
        $entity = new competency_entity();
        $entity->fullname    = trim($fullname);
        $entity->idnumber    = strtolower(trim($idnumber));
        $entity->frameworkid = $framework_id;
        $entity->status      = 1;
        $entity->save();
        return static::load_by_entity($entity);
    }

    // ---- Business methods ----

    public function rename(string $new_name): void {
        if (empty(trim($new_name))) {
            throw new \coding_exception('Competency name cannot be empty.');
        }
        $this->entity->fullname = trim($new_name);
        $this->entity->save();
    }

    public function deactivate(): void {
        $this->entity->status = 0;
        $this->entity->save();
    }

    // ---- Accessors ----

    public function get_status_label(): string {
        return $this->entity->status === 1 ? 'Active' : 'Inactive';
    }
}
```

### 4. Formatter

```php
// classes/formatter/competency.php
namespace local_myplugin\formatter;

use core\webapi\formatter\field\string_field_formatter;
use core\webapi\formatter\field\date_field_formatter;

class competency extends \core\orm\formatter\entity_model_formatter {

    protected function get_map(): array {
        return [
            'id'           => null,
            'fullname'     => string_field_formatter::class,
            'idnumber'     => null,
            'status'       => null,
            'status_label' => string_field_formatter::class,
            'timecreated'  => date_field_formatter::class,
            'timemodified' => date_field_formatter::class,
        ];
    }
}
```

### 5. Usage in a service / GraphQL resolver

```php
// In a query resolver
$competency_model = \local_myplugin\model\competency::load_by_id($args['id']);

// In a type resolver
$formatter = new \local_myplugin\formatter\competency($competency_model, \context_system::instance());
return $formatter->format($field, $args['format'] ?? null);

// Bulk load — always use the repository, wrap results in models
$entities = \local_myplugin\entity\competency::repository()
    ->active()
    ->for_framework($framework_id)
    ->get();

$models = $entities->map(function ($entity) {
    return \local_myplugin\model\competency::load_by_entity($entity);
});
```

---

## Summary: where each class lives

| Class | Namespace path | Base class |
|---|---|---|
| Entity | `classes/entity/{name}.php` | `\core\orm\entity\entity` |
| Specific repository | `classes/entity/{name}_repository.php` | `\core\orm\entity\repository` |
| Entity model | `classes/model/{name}.php` | `\core\orm\entity\model` |
| Formatter | `classes/formatter/{name}.php` | `\core\orm\formatter\entity_model_formatter` |
