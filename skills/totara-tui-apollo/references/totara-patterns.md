# Totara Vue Apollo reference patterns

Frontend-only copy-paste examples for Totara TUI + Vue Apollo. Read when implementing queries, mutations, or cache policies in `.vue` files — not on every task.

Sources:
- [Quick-start guide: Creating your first component](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184960)
- [Developing for the AJAX GraphQL API](https://totara.atlassian.net/wiki/display/DEV/Developing+for+the+AJAX+GraphQL+API) (cache/type policies)

---

## 1. Ping.vue — query + refetch (Options API)

Full page component from the Totara first-component quickstart. Uses persisted query import, `apollo:` block, and `$apollo.queries.*.refetch()`.

**Import pattern:**

```js
import statusQuery from 'totara_webapi/graphql/status_nosession';
// {frankenstyle_owner}/graphql/{persisted_query_filename}
```

**Page component:**

```vue
<template>
  <div>
    <PingStatus v-if="ping" :ping="ping" />
    <Button :text="$str('reload', 'local_quickstart')" @click="reload" />
  </div>
</template>

<script>
import Button from 'tui/components/buttons/Button';
import PingStatus from 'local_quickstart/components/ping/PingStatus';
import statusQuery from 'totara_webapi/graphql/status_nosession';

export default {
  components: {
    Button,
    PingStatus,
  },

  data() {
    return {
      // default value until query loads; optional but helps unit tests
      ping: null,
    };
  },

  apollo: {
    ping: {
      query: statusQuery,
      update: data => data.totara_webapi_status,
    },
  },

  methods: {
    reload() {
      this.$apollo.queries.ping.refetch();
    },
  },
};
</script>
```

**With query variables (filters):**

```vue
<script>
import Button from 'tui/components/buttons/Button';
import PingStatus from 'local_quickstart/components/ping/PingStatus';
import statusQuery from 'totara_webapi/graphql/status_nosession';

export default {
  components: {
    Button,
    PingStatus,
  },

  data() {
    return {
      ping: null,
      filterValue: '123',
    };
  },

  apollo: {
    ping: {
      query: statusQuery,
      variables() {
        return {
          filter: this.filterValue,
        };
      },
      update: data => data.totara_webapi_status,
    },
  },

  methods: {
    reload() {
      this.$apollo.queries.ping.refetch();
    },
  },
};
</script>
```

**Presentational child (PingStatus.vue):**

```vue
<template>
  <div class="tui-pingStatus">
    <div>{{ $str('status', 'core') }}</div>
    <div>{{ ping.status }}</div>
    <div>{{ $str('time', 'core') }}</div>
    <div>{{ ping.timestamp }}</div>
  </div>
</template>

<script>
export default {
  props: {
    ping: Object,
  },
};
</script>

<style lang="scss">
.tui-pingStatus {
  max-width: 50rem;
  background: var(--color-neutral-2);
  margin-top: var(--gap-2);
  padding: var(--gap-2);
}
</style>
```

**Key UX rule:** `v-if="ping"` on main content — not `v-if="$apollo.loading"`. Refetch toggles loading and would unmount content.

---

## 2. Mutation + refetchQueries (local_todo example)

Frontend wiring for list + update. Assumes persisted operations already exist at `local_todo/graphql/items` and `local_todo/graphql/update_item`.

**Imports:**

```js
import listQuery from 'local_todo/graphql/items';
import updateItemMutation from 'local_todo/graphql/update_item';
```

**Expected variables shape** (must match the persisted mutation):

```js
{
  item_reference: { id },
  input: { title },
}
```

### Options API — list + update with refetchQueries

```vue
<template>
  <div>
    <ul v-if="items.length">
      <li v-for="item in items" :key="item.id">
        <input
          :value="item.title"
          @change="updateTitle(item.id, $event.target.value)"
        />
      </li>
    </ul>
    <div v-if="error">{{ error.message }}</div>
  </div>
</template>

<script>
import listQuery from 'local_todo/graphql/items';
import updateItemMutation from 'local_todo/graphql/update_item';

export default {
  data() {
    return {
      items: [],
      error: null,
    };
  },

  apollo: {
    items: {
      query: listQuery,
      update: data => data.local_todo_items.items,
    },
  },

  methods: {
    async updateTitle(id, title) {
      try {
        await this.$apollo.mutate({
          mutation: updateItemMutation,
          variables: {
            item_reference: { id },
            input: { title },
          },
          refetchQueries: [
            { query: listQuery },
          ],
        });
      } catch (e) {
        this.error = e;
      }
    },
  },
};
</script>
```

**Why refetchQueries:** Totara recommends refetching related queries after mutations so all components sharing that cache entry stay in sync. Prefer this over manual cache surgery unless you know the exact cache shape.

### Composition API equivalent

```vue
<script>
import { computed, ref } from 'vue';
import { useQuery, useMutation } from '@vue/apollo-composable';
import listQuery from 'local_todo/graphql/items';
import updateItemMutation from 'local_todo/graphql/update_item';

export default {
  setup() {
    const { result, error: queryError } = useQuery(listQuery);
    const items = computed(() => result.value?.local_todo_items?.items ?? []);

    const { mutate, error: mutationError } = useMutation(updateItemMutation);
    const error = computed(() => queryError.value ?? mutationError.value);

    async function updateTitle(id, title) {
      await mutate(
        {
          item_reference: { id },
          input: { title },
        },
        {
          refetchQueries: [{ query: listQuery }],
        },
      );
    }

    return { items, error, updateTitle };
  },
};
</script>
```

---

## 3. Apollo type policy in Tui entry (Totara 19+)

When a nested field is not a standalone entity (e.g. metadata blobs), Apollo 3+ overwrites the whole field on refetch unless you configure merge behavior.

### Fix in query selections — normalize with id + __typename (preferred)

Include `__typename` and `id` on nested objects wherever they represent real entities:

```graphql
query {
  todos: totara_todos_todos {
    __typename
    id
    owner {
      __typename
      id
      name
      email
    }
  }
}
```

### Fix in Tui component entry — type policy for non-entity fields

When a field groups extra data (not an independent entity), add a type policy in the Tui component:

**entry.js** — `client/component/totara_todos/js/internal/entry.js`:

```js
import { addTypePolicies } from 'tui/apollo_client';

addTypePolicies({
  totara_todos_todo: {
    fields: {
      metadata: { merge: true },
    },
  },
});
```

**Register in tui.json:**

```json
{
  "component": "totara_todos",
  "vendor": "totara",
  "entry": "./js/internal/entry.js"
}
```

If the backend schema author marked the field with `@cache(merge: true)`, the type policy above must match. Coordinate with them — see [Developing for the AJAX GraphQL API](https://totara.atlassian.net/wiki/display/DEV/Developing+for+the+AJAX+GraphQL+API).

**Rule of thumb:**
- Use `__typename` + `id` in query selections where the nested object is a real entity.
- Use `merge: true` in `entry.js` for grouped data that is not an independent entity.

Browser console warnings about cache merge usually mean one of these fixes is needed.
