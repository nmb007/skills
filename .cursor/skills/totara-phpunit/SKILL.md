---
name: totara-phpunit
description: >-
  PHPUnit test setup, execution commands, and test authoring conventions for
  Totara. Use when writing PHPUnit test cases, creating _test.php files,
  running phpunit.php commands, setting up parallel test runs, filtering tests,
  naming test classes with frankenstyle prefix, tagging tests with PHPUnit
  groups, or asking about how to add or run tests in Totara.
---

# Totara PHPUnit

## Environment

In case of local development environment, all these commands should be run inside the php container.

---

## Setup & Teardown

```bash
# First time setup for single thread run, this will install the database tables, prepare data root
php test/phpunit/phpunit.php init

# When you want to remove all phpunit database tables and to remove its dataroot
php test/phpunit/phpunit.php util --drop
```

---

## Running Tests

```bash
# Run all tests (in a single thread)
php test/phpunit/phpunit.php run

# Run a test suite
php test/phpunit/phpunit.php run --testsuite="core_testsuite"

# Run a single test case
php test/phpunit/phpunit.php run core_user_userdata_username_testcase user/tests/userdata_username_test.php

# Run a single test, To run a single test within a test case using the filter argument.
php test/phpunit/phpunit.php run --filter test_export user/tests/userdata_username_test.php

# Run tests by PHPUnit group (see "Test groups" below)
php test/phpunit/phpunit.php run --group local_todo
```

---

## Parallel Running

```bash
# To initialize your site for a parallel test run, initialises PHPUnit for multiple threads
php test/phpunit/phpunit.php parallel_init --processes=2

# Run tests in parallel threads
php test/phpunit/phpunit.php parallel_run --processes=2
```

---

## How to add more tests?

1. Create `tests/` directory in your add-on
2. Add test file, for example `local/mytest/tests/my_test.php` file with `local_my_testcase` class that extends `\core_phpunit\testcase`
3. Add some test_*() methods
4. If a tearDown() method is added, confirm it calls parent::tearDown() at the end
5. Execute `php test/phpunit/phpunit.php init` to get the plugin tests included in main phpunit.xml configuration file
6. Execute your new test case `php test/phpunit/phpunit.php run local/mytest/tests/my_test.php`

---

## Adding new tests — Naming conventions

The test case class name must start with a component prefix (also known as frankenstyle) and must end with the _test suffix. Namespaces are not allowed.

The test case file name is derived from the class name; it omits the component prefix and uses the _test.php suffix.

For example, the `server/totara/cohort/tests/enrol_test.php` file contains class:

```php
class totara_cohort_enrol_test extends \core_phpunit\testcase {
    public function test_role_updates(): void {
        // test goes here
    }
}
```

---

## Test groups

Tag test classes (or methods) with a PHPUnit group so they can be run selectively via `--group`. Use the component frankenstyle name (e.g. `local_todo`, `totara_competency`).

**Do not use docblock annotations** such as `@group` — PHPUnit 11 deprecates metadata in doc-comments and PHPUnit 12 will remove support. Totara runs PHPUnit 11; docblock `@group` triggers `PHPUnit Deprecations` and `OK, but there were issues!` even when all tests pass.

**Use PHP attributes instead:**

```php
use PHPUnit\Framework\Attributes\Group;

#[Group('local_todo')]
class local_todo_webapi_todo_items_test extends \core_phpunit\testcase {
    public function test_create_todo_item(): void {
        // test goes here
    }
}
```

To inspect PHPUnit deprecation details (hidden by default in Totara's `phpunit.xml`):

```bash
php test/phpunit/phpunit.php run --testsuite="local_todo_testsuite" --display-phpunit-deprecations
```

Note: many existing Totara core tests still use `@group` in docblocks. When writing **new** tests, always use `#[Group(...)]`.
