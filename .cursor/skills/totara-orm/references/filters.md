# Filters

Full reference for entity repository filter objects: custom filters, params, default filters, on-demand filters, and the built-in generic filters.

---

## Overview

Filters are reusable, composable query conditions that can be applied to any entity repository. Instead of writing `filter_by_*` methods directly on a repository (which couples the filter to one entity), you create a `filter` class that can be shared across multiple repositories.

A filter receives a `builder` instance and calls builder methods inside `apply()`.

```php
class \core\orm\entity\filter\filter {
    protected $builder; // set automatically before apply() is called
    protected $value;   // the filter value, set via set_value() or constructor
    protected $params;  // optional params array, set via constructor

    abstract public function apply(): void;
}
```

---

## Custom filter class

Extend `\core\orm\entity\filter\filter` and implement `apply()`.

```php
use core\orm\entity\filter\filter;

class path_filter extends filter {

    public function apply(): void {
        if ($this->value > 0) {
            $value = $this->value;
            $this->builder->where_like('path', "/{$value}/");
        }
    }
}
```

### Filter with constructor params

Params let you configure a filter for different columns or modes without subclassing.

```php
class column_value_filter extends filter {

    public function apply(): void {
        $column         = $this->params[0];
        $case_sensitive = $this->params[1] ?? false;

        if ($case_sensitive) {
            $this->builder->where($column, $this->value);
        } else {
            $this->builder->where_like($column, $this->value);
        }
    }
}

// Instantiate with params
$filter = new column_value_filter('fullname', true);
$filter->set_value('Safety Competency');
```

Or pass params through the constructor:

```php
// Constructor signature: __construct(mixed $params_or_value = null)
// When passing an array it becomes $this->params; when passing a scalar it becomes $this->value
$filter = new column_value_filter(['fullname', false]);
$filter->set_value('Safety');
```

---

## Default filters (always available on a repository)

Register filters in `get_default_filters()` on a specific repository. Each key becomes the filter name used in `set_filters()`.

```php
class competency_repository extends \core\orm\entity\repository {

    protected function get_default_filters(): array {
        return [
            'path'   => new path_filter(),
            'status' => new status_filter(),
        ];
    }
}
```

Apply by key name:

```php
// Single filter
$competencies = competency::repository()
    ->set_filter('path', 23)
    ->get();

// Multiple filters at once
$competencies = competency::repository()
    ->set_filters([
        'path'   => 23,
        'status' => 1,
    ])
    ->get();
```

---

## On-demand filters (no registration required)

You can pass a filter object directly to `set_filter()` or `set_filters()` without pre-registering it.

```php
$path_filter = new path_filter();
$path_filter->set_value(23);

// Single filter object
$competencies = competency::repository()
    ->set_filter($path_filter)
    ->get();

// Multiple filter objects
$status_filter = new status_filter();
$status_filter->set_value(1);

$competencies = competency::repository()
    ->set_filters([$path_filter, $status_filter])
    ->get();
```

---

## Generic built-in filters

Totara provides ready-to-use generic filters under `\core\orm\entity\filter\`. Use them directly or extend them.

### `equal` — exact match

```php
use core\orm\entity\filter\equal;

// Single column: firstname = 'Max'
$filter = new equal('firstname');
$filter->set_value('Max');

// Multiple columns (OR): firstname = 'Max' OR middlename = 'Max'
$filter = new equal(['firstname', 'middlename']);
$filter->set_value('Max');
```

### `like` — partial match

```php
use core\orm\entity\filter\like;

// Default: firstname LIKE '%Max%'
$filter = new like('firstname');
$filter->set_value('Max');

// Custom placeholder: firstname LIKE 'Max%'
$filter = new like('firstname', '[text]%');
$filter->set_value('Max');

// Multiple columns (OR): firstname LIKE '%Max%' OR middlename LIKE '%Max%'
$filter = new like(['firstname', 'middlename']);
$filter->set_value('Max');
```

### `in` — value in list

```php
use core\orm\entity\filter\in;

// id IN ('John', 'Max', 'Bob')
$filter = new in('firstname');
$filter->set_value(['John', 'Max', 'Bob']);
```

---

## Combining filters and repository conditions

Filters are applied in addition to any `where()` conditions already on the repository:

```php
$competencies = competency::repository()
    ->where('frameworkid', 5)          // direct condition
    ->set_filters([
        'status' => 1,                  // via registered filter
        'path'   => 23,
    ])
    ->order_by('fullname')
    ->get();
```

---

## Full example: filters in a repository used via service

```php
// classes/entity/competency_repository.php
class competency_repository extends \core\orm\entity\repository {

    protected function get_default_filters(): array {
        return [
            'status'      => new status_filter(),
            'search'      => new like(['fullname', 'idnumber']),
            'framework'   => new equal('frameworkid'),
        ];
    }
}

// In a service method:
$filters = [
    'status'    => $input['status'] ?? null,
    'search'    => $input['search'] ?? null,
    'framework' => $input['framework_id'] ?? null,
];

$competencies = competency::repository()
    ->set_filters(array_filter($filters, fn($v) => $v !== null))
    ->order_by('fullname')
    ->paginate($input['page'] ?? 1, 20);
```
