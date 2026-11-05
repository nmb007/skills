# GraphQL Middleware

Sources:
- [Implementing GraphQL Middleware](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187532/Implementing+GraphQL+Middleware)
- [Extending GraphQL APIs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187526/Extending+GraphQL+APIs)

---

## What middleware is for

Middleware wraps resolvers to apply cross-cutting concerns: authentication, capability checks, record pre-loading, rate-limiting. Each middleware class can be tested independently and reused across multiple resolvers or the entire site.

```
Request → [Middleware A] → [Middleware B] → Resolver → [Middleware B after] → [Middleware A after] → Response
```

---

## Defining a middleware class

```
# File location
server/{plugin}/classes/webapi/middleware/{classname}.php

# Namespace
{component}\webapi\middleware\{classname}
```

Middleware must implement `core\webapi\middleware`:

```php
namespace local_todo\webapi\middleware;

use Closure;
use core\webapi\middleware;
use core\webapi\resolver\payload;
use core\webapi\resolver\result;

class require_valid_item implements middleware {
    public function handle(payload $payload, Closure $next): result {
        $item_id = $payload->get_variable('item_id');
        $item = \local_todo\model\item::repository()->find($item_id);
        if (!$item) {
            throw new \moodle_exception('invaliditem');
        }
        // Pass item via variables so the resolver can access it without re-querying
        $payload->set_variable('item', $item);
        return $next($payload);
    }
}
```

---

## Before vs after middleware

**Before middleware** (most common): call `$next($payload)` at the end after your logic.

**After middleware**: call `$next($payload)` first, then act on the result.

```php
// After middleware — inspect or modify the result
public function handle(payload $payload, Closure $next): result {
    $result = $next($payload);
    // post-process $result here
    return $result;
}
```

---

## Registering middleware on a resolver

Override `get_middleware()` in your query/mutation resolver:

```php
namespace local_todo\webapi\resolver\query;

use core\webapi\query_resolver;
use core\webapi\execution_context;
use local_todo\webapi\middleware\require_valid_item;
use totara_webapi\middleware\require_login;

class item extends query_resolver {
    public static function get_middleware(): array {
        return [
            require_login::class,
            require_valid_item::class,
        ];
    }

    public static function resolve(array $args, execution_context $ec): array {
        $item = $ec->get_variable('item'); // pre-loaded by middleware
        return ['item' => $item];
    }
}
```

Middleware is called in order: first `require_login`, then `require_valid_item`, then `resolve`.

---

## Middleware groups

Group related middleware for reuse:

```php
namespace local_todo\webapi\middleware;

use core\webapi\middleware_group;

class item_access_group implements middleware_group {
    public function get_middleware(): array {
        return [
            \totara_webapi\middleware\require_login::class,
            require_valid_item::class,
            check_item_capability::class,
        ];
    }
}
```

Register the group the same way as a single middleware class in `get_middleware()`. Groups are resolved recursively, so groups can nest other groups.

---

## Global middleware (Totara 17+)

Global middleware applies to **all resolvers** of a specific endpoint type, including type resolvers. It is declared in the endpoint type class (`server/totara/webapi/classes/endpoint_type/`).

### Global middleware hook

Any plugin can add or modify global middleware via the `totara_webapi\hook\api_hook`:

```php
namespace local_example\watcher;

use totara_webapi\hook\api_hook;

class graphql_middleware_watcher {
    public static function watch(api_hook $api_hook): void {
        // Available properties:
        $endpoint_type = $api_hook->endpoint_type; // e.g. 'totara_webapi\endpoint_type\external'
        $component     = $api_hook->component;     // e.g. 'mod_perform'
        $resolver      = $api_hook->resolver;      // e.g. 'mod_perform\webapi\resolver\query\activities'

        // Append custom middleware
        $api_hook->middleware = array_merge(
            $api_hook->middleware,
            [\local_example\webapi\middleware\custom_auth::class]
        );
    }
}
```

Register the watcher in your plugin's `db/hooks.php`. See Totara hooks documentation for details.

> **Note**: Global middleware is called for query, mutation, AND type resolvers. Because type resolvers run once per field per item, global middleware can execute many times per request. Use a request-scoped cache or an early-exit check to avoid performance issues.

---

## Pre- and post-request hooks (Totara 17+)

For logic that must run at the very start or end of a GraphQL request (not around individual resolvers), use the pre- and post-request hooks. These have access to the `request` object and the `execution_context`.

Totara uses the pre-request hook to implement OAuth 2.0 authentication for the external API. See `server/totara/api/classes/watcher/handle_request_pre_watcher.php` for a reference implementation.

---

## Common built-in middleware

| Class | Purpose |
|---|---|
| `totara_webapi\middleware\require_login` | Calls `require_login()` — use in most query/mutation resolvers |
| `core\webapi\middleware\reopen_session_for_writing` | Required in Totara 18+ when the resolver or its middleware writes to the session. Must be placed first in the middleware list. |

---

## Session writing in middleware (Totara 18+)

In Totara 18, sessions are closed for writing before query resolution to support concurrent requests. If any middleware or resolver writes to the session, add `reopen_session_for_writing` as the **first** middleware:

```php
public static function get_middleware(): array {
    return [
        \core\webapi\middleware\reopen_session_for_writing::class, // must be first
        \totara_webapi\middleware\require_login::class,
        // ...
    ];
}
```

Alternatively, disable the optimisation globally (not recommended):

```php
// config.php
$CFG->graphqlsessionwritecloseenabled = false;
```
