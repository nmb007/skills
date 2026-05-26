---
name: totara-tui-apollo
description: Frontend-only Vue Apollo patterns for Totara TUI components. Use when writing or debugging TUI .vue files that fetch or mutate via GraphQL, apollo queries/mutations, GraphQL imports, Apollo cache/type policies, refetchQueries, or @vue/apollo-composable in Totara.
---

# Vue Apollo for Totara TUI

## Scope

Applies to **Totara TUI frontend code only** — Vue components, Apollo client usage, and cache policies in `client/component/`. Does **not** cover PHP resolvers, `webapi/ajax/` schema authoring, or page controllers. For backend GraphQL, use companion skills and [Totara GraphQL docs](https://totara.atlassian.net/wiki/spaces/DEV/pages/840663073).

Totara uses **Vue Apollo + Apollo Client** for server state via GraphQL — not Pinia, not ad-hoc fetch.

Official references:
- Totara: [Tui front-end framework](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188209), [State management](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187041)
- Vue Apollo: [v4 docs](https://v4.apollo.vuejs.org/) | [vuejs/apollo](https://github.com/vuejs/apollo)

## Hard rules

1. **Do NOT use React Apollo patterns** — no `useQuery` from `@apollo/client/react`, no `ApolloProvider`, no JSX examples from apollographql/skills `apollo-client`.
2. **Do NOT create a new ApolloClient in components** — Totara/TUI bootstraps the client. Components consume it via Vue Apollo.
3. **Import persisted GraphQL documents via Tui paths**, not relative `.graphql` file paths:

   ```js
   import statusQuery from 'totara_webapi/graphql/status_nosession';
   // pattern: '{owner}/graphql/{query_name}'
   // owner = frankenstyle Tui component or plugin name (e.g. local_myplugin)
   ```

4. **Match existing API style in the file** — most Totara TUI code uses **Options API + `apollo:` property** (`@vue/apollo-option`). Use **Composition API** (`@vue/apollo-composable`) only in new code or when the surrounding file already uses `setup()`.

## Options API (default in Totara)

```vue
<script>
import statusQuery from 'totara_webapi/graphql/status_nosession';

export default {
  data() {
    return {
      ping: null, // optional default; helps tests and avoids undefined flashes
    };
  },

  apollo: {
    ping: {
      query: statusQuery,
      variables() {
        return { filter: this.filterValue };
      },
      update: data => data.totara_webapi_status,
      // skip() { return !this.ready; }  // conditional fetch
    },
  },

  methods: {
    reload() {
      this.$apollo.queries.ping.refetch();
    },

    async save() {
      await this.$apollo.mutate({
        mutation: updateMutation,
        variables: { input: this.form },
        refetchQueries: [
          { query: listQuery, variables: { id: this.id } },
        ],
      });
    },
  },
};
</script>
```

Template rules:
- Prefer `v-if="ping"` (or mapped data) over `v-if="$apollo.loading"` — loading toggles during **refetch** and causes flicker/unmount in Totara.
- Show errors with `$apollo.queries.<name>.error` or mutation catch blocks.

## Composition API (when file uses setup)

```vue
<script>
import { computed } from 'vue';
import { useQuery, useMutation } from '@vue/apollo-composable';
import listQuery from 'local_myplugin/graphql/items';

export default {
  setup() {
    const { result, loading, error, refetch } = useQuery(listQuery);

    const items = computed(() => result.value?.local_todo_items ?? []);

    return { items, loading, error, refetch };
  },
};
</script>
```

For conditional queries, pass reactive options (don't copy React `skip` blindly):

```js
useQuery(document, variables, () => ({ enabled: !!variables.value.id }));
```

Use `useLazyQuery` when the query must not run on mount.

## Mutations & cache updates

Totara's recommended post-mutation refresh:

1. **`refetchQueries`** — preferred for keeping UI in sync.
2. **`readQuery` / `writeQuery`** — when you know the exact cache shape.
3. Avoid optimistic UI unless the surrounding codebase already does it.

After mutations, related components sharing the same cached query update automatically.

## Apollo cache & Totara GraphQL lint

Totara expects Apollo-normalizable types to include **`id` + `__typename`** so cache entries merge correctly.

If a nested object is not a standalone entity (e.g. metadata blobs):
- Add a **type policy** in the Tui component entry:

  ```js
  // client/component/my_component/js/internal/entry.js
  import { addTypePolicies } from 'tui/apollo_client';

  addTypePolicies({
    totara_todos_todo: {
      fields: {
        metadata: { merge: true },
      },
    },
  });
  ```

- Register in `tui.json`: `"entry": "./js/internal/entry.js"`
- If the backend schema author marked a field with `@cache(merge: true)`, add a matching `merge: true` type policy here. See [Developing for the AJAX GraphQL API](https://totara.atlassian.net/wiki/display/DEV/Developing+for+the+AJAX+GraphQL+API).

## Frontend contract (assumes operation exists)

When wiring a component to an existing persisted operation:

- **Import:** `import op from '{frankenstyle}/graphql/{operation_name}'`
- **Response mapping:** root field in `update:` / `computed` must match the operation's response shape (e.g. `data => data.local_todo_items.items`)
- **Variables:** must match the persisted operation's declared arguments
- **Operation missing?** Use `graphql-operations` + Totara backend docs — not this skill

## Common mistakes

| Wrong | Right |
|--------|--------|
| `import gql from 'graphql-tag'` + inline query in component | Import persisted query from `{owner}/graphql/...` |
| React `useQuery` from `@apollo/client` | `@vue/apollo-composable` or `apollo:` option |
| `v-if="$apollo.loading"` on main content | `v-if="dataField"` so refetch doesn't unmount |
| Manual `fetch('/graphql')` | Vue Apollo query/mutation |
| New `ApolloClient({ uri: ... })` in component | Use TUI-provided client |

## When stuck

1. Find a similar component in the same Tui bundle and copy its `apollo` / `$apollo.mutate` pattern.
2. Confirm the import path resolves (`{owner}/graphql/{operation_name}`) and the response root field matches your `update:` / `computed` mapping.
3. Check browser console for Apollo cache merge warnings → add type policy in `entry.js` or add `__typename`/`id` to query selections.
4. If data looks stale after a backend schema change, purge Totara caches in dev.

## Additional resources

For full copy-paste examples (Ping.vue, mutation + refetchQueries, type policies), see [references/totara-patterns.md](references/totara-patterns.md).

## Companion skills

Install alongside this skill:

```bash
# Backend GraphQL operation/schema authoring (use instead of this skill for PHP/webapi work)
npx skills add apollographql/skills --skill graphql-operations --skill graphql-schema -a cursor -y

# Vue 3 patterns for TUI components (not Apollo-specific)
npx skills add vuejs-ai/skills --skill vue-best-practices -a cursor -y
```

Skip `apollo-client` from apollographql/skills — it targets React, not `@vue/apollo-composable`.

## Usage tip

For reliability, prefix prompts with:

```text
Use totara-tui-apollo skill: add a mutation to update item title and refetch the list query.
```
