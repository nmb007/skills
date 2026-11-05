# Endpoint Types

Sources:
- [Available APIs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187291/Available+APIs)
- [Using GraphQL APIs](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187394/Using+GraphQL+APIs)
- [Using the developer GraphQL API](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187396/Using+the+developer+GraphQL+API)
- [Making developer requests via a GraphQL client](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187402/Making+developer+requests+via+a+GraphQL+client)
- [Developing for the External GraphQL API](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187581/Developing+for+the+External+GraphQL+API)
- [Developing for the Mobile GraphQL API](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187520/Developing+for+the+Mobile+GraphQL+API)

---

## Endpoint comparison

| | ajax (internal) | external | mobile | dev |
|---|---|---|---|---|
| Endpoint URL | `/totara/webapi/ajax.php` | `/api/graphql.php` | `/totara/mobile/api.php` | `/totara/webapi/dev_graphql_executor.php` |
| Available since | TXP13+ | TXP17+ | TXP13+ | TXP13+ |
| Query type | Persisted only | Custom only | Persisted only | Custom only |
| Introspection | No | Optional (admin setting) | No | Yes |
| Auth mechanism | Session cookie + CSRF token | OAuth 2.0 access token | API key in request header | Session cookie (no CSRF required) |
| Nosession support | Yes (for `_nosession` operations) | No | No | Yes |
| Primary audience | TUI / JS front-end developers | External system integrators | Totara Mobile app | Developers (local only) |

---

## Request format (all endpoints)

```http
POST /totara/webapi/ajax.php
Content-Type: application/json

# Custom query (external / dev)
{
  "query": "query { totara_webapi_status { status } }",
  "variables": {}
}

# Persisted query (ajax / mobile)
{
  "operationName": "totara_webapi_status",
  "variables": {}
}
```

Variables containing complex input types:

```json
{
  "operationName": "local_todo_create_item",
  "variables": {
    "input": {
      "title": "My new task"
    }
  }
}
```

---

## Response format

```json
{
  "data": {
    "local_todo_items": {
      "items": [{ "id": "1", "title": "Task 1" }]
    }
  }
}
```

On error (200 status code is still returned):

```json
{
  "errors": [
    {
      "message": "Internal server error",
      "extensions": { "category": "internal" },
      "locations": [{ "line": 4, "column": 7 }],
      "path": ["local_todo_items", "items", 0, "id"]
    }
  ]
}
```

HTTP status codes:
- **200**: successful request OR request denied/invalid (check `errors` array)
- **429**: rate-limited (external API only)
- **500**: internal server error

---

## Batched queries

Multiple query endpoints can be combined in a single request:

```graphql
query my_dashboard {
  core_user_profile_info {
    username
    email
  }
  mod_perform_user_activity_progress {
    num_incomplete_activities
  }
}
```

Constraints:
- Variables are shared — no name collisions allowed across queries in one request
- Cannot mix `query` and `mutation` in the same request
- If queries need different execution contexts, use JS batch queries (Apollo batch link) instead

---

## Ajax (internal) endpoint

- Designed for TUI/Vue components making AJAX requests
- **Persisted queries only** — arbitrary queries are rejected
- Auth: session cookie + CSRF token (automatically handled by the TUI `WebAPI.call()` helper)
- Schema is specific to `ajax` endpoint type (files in `webapi/ajax/`)
- Global schema elements (in `webapi/`) are also included

```js
// TUI usage (frontend — see totara-tui-apollo skill for full patterns)
import itemsQuery from 'local_todo/graphql/items';
// Resolves to server/local/todo/webapi/ajax/items.graphql
```

---

## External endpoint

- For **external system integrations** (Totara 17+)
- Arbitrary custom queries — no persisted query files needed
- Auth: OAuth 2.0 access token (see `server/totara/oauth2/`)
- Rate limiting applies (complexity points per minute)
- Introspection available when `enable_introspection` admin setting is on

### Stricter requirements for external schema

1. **Full `"""..."""` doc-comments** on every schema element (types, fields, args, enums, scalars)
2. **Reference input types** instead of raw database IDs — use `base_record_reference` subclasses
3. **Enums** instead of integer/magic-number arguments (e.g. use `core_format` enum)
4. **`metadata.json` examples** for all query/mutation args and result fields
5. **Handle all enum options** documented, or return a clear error for unsupported options
6. **Deprecation policies** are stricter — bar for exceptions is higher than internal APIs

### Disabling OAuth 2.0 (custom auth)

```php
// config.php — WARNING: allows unauthenticated requests
$CFG->forced_plugin_settings['totara_api'] = [
    'disable_oauth2_authentication' => 1
];
// Must implement a separate pre-request watcher to set user and client
```

---

## Mobile endpoint

- Used exclusively by the Totara Mobile app
- **Persisted queries only**, schema files in `webapi/mobile/`
- Auth: API key passed in request header
- Sessions are already closed before query resolution (unaffected by the Totara 18 session change)
- Mobile app prefers camelCase — use aliases in persisted queries to rename snake_case fields:

```graphql
query local_todo_items {
  local_todo_items {
    items {
      itemId: id
      itemTitle: title
    }
  }
}
```

---

## Developer endpoint

- For local development only — **never enable on production**
- Schema = union of all endpoint schemas (ajax + external + mobile)
- No CSRF token required
- Supports introspection natively

### Enable dev API

```php
// config.php (above the final require of setuplib.php)
define('GRAPHQL_DEVELOPMENT_MODE', true);
```

### Raw cURL request

```bash
# Authenticated (replace session cookie value)
curl 'https://YOUR-SITE-URL/totara/webapi/dev_graphql_executor.php' \
  -H 'Cookie: TotaraSession=SESSION-COOKIE-HERE' \
  -H 'content-type: application/json' \
  --data-raw '{"operationName":"totara_webapi_status","variables":{}}'

# Unauthenticated nosession operation
curl 'https://YOUR-SITE-URL/totara/webapi/dev_graphql_executor.php' \
  -H 'content-type: application/json' \
  --data-raw '{"query":"query { totara_webapi_status { status } }","variables":{}}'
```

### GraphQL client setup

**Altair (Chrome extension)** — simplest:
1. Install Altair Chrome extension
2. Log in to Totara in another tab
3. Set URL: `https://YOUR-SITE-URL/totara/webapi/dev_graphql_executor.php`
4. Cookies from the active session are shared automatically

**PHPStorm GraphQL plugin**:
1. Install GraphQL plugin via Preferences → Plugins
2. Create `.graphqlconfig` at the project root:

```json
{
  "extensions": {
    "endpoints": {
      "Default GraphQL Endpoint": {
        "url": "[siteurl]/totara/webapi/dev_graphql_executor.php",
        "headers": {
          "user-agent": "JS GraphQL",
          "Cookie": "TotaraSession=[session-id-from-browser-devtools]"
        }
      }
    }
  }
}
```

3. Open GraphQL tab → add new schema configuration pointing to the folder with `.graphqlconfig`

**Postman**:
1. Create a collection with "No Auth"
2. Add HTTP POST request (not "GraphQL" type) to `http://YOUR-SITE-URL/totara/webapi/dev_graphql_executor.php`
3. Body → GraphQL tab
4. Install Postman Interceptor browser extension and sync cookies from your site's wwwroot
5. Log in to Totara in browser; cookies sync automatically

---

## Field arguments and aliases

```graphql
query test {
  my_alias: totara_webapi_status {
    status
    long: timestamp(format: DATETIMELONG)
    short: timestamp(format: DATETIMESHORT)
  }
}
```

Aliases rename fields in the response and allow requesting the same field multiple times with different arguments.
