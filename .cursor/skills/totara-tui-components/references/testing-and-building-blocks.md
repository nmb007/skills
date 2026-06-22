# Testing and core building blocks

Sources:
- [Testing front-end code](https://totara.atlassian.net/wiki/spaces/DEV/pages/250282026)
- [Feature: Uniform](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187191)
- [Weka editor](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184860)
- [Quick-start guide: Creating your first component](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184960)

---

## Jest component tests

Totara uses **Jest** with **Testing Library** (via `tui_test_utils/vtl`).

```bash
tnpm run tui-test
tnpm run tui-test-watch
```

### File placement

Adjacent to the component, e.g.:

```
src/components/ping/PingStatus.vue
src/components/ping/tests/PingStatus.spec.js
```

(or `tests/` folder next to the `.vue` file — match the bundle you are copying from)

### Principles

- **Black-box / semantic queries** — prefer `getByRole('button', { name: /reload/i })` over CSS classes or implementation details.
- Import Testing Library from `tui_test_utils/vtl`, not ad-hoc `vue-test-utils` selectors alone.
- Set sensible `data()` defaults in the component so tests do not flash `undefined` (e.g. `ping: null` before Apollo resolves).

Avoid duplicating Behat coverage in Jest — unit tests for component logic and a11y-friendly DOM; Behat for full browser flows when required.

---

## Minimal spec skeleton

```js
import { render, screen } from 'tui_test_utils/vtl';
import PingStatus from '../PingStatus.vue';

describe('PingStatus', () => {
  it('shows status text', () => {
    render(PingStatus, {
      props: { ping: { status: 'ok', timestamp: 1 } },
    });
    expect(screen.getByText(/ok/i)).toBeTruthy();
  });
});
```

Match patterns from the same Tui bundle before inventing new test utilities.

---

## Feature: Uniform (forms)

Use when you need **declarative form state, validation, and composable fields** — not for one-off static markup.

Raw inputs: `tui/components/form/*` — styled but no shared validation/wiring.

Uniform adds **Reform** under the hood:

```vue
<Uniform>
  <FormRow label="Title" required>
    <FormText name="title" />
  </FormRow>
</Uniform>
```

```js
import { Uniform, FormRow, FormText } from 'tui/components/uniform';
```

- Wrapped inputs use `createUniformInputWrapper()` for custom `v-model` components.
- Data flows via provide/inject from root `Uniform` — no manual field wiring.
- Field errors render automatically; `FormScope` for grouped validation.

See [Feature: Uniform](https://totara.atlassian.net/wiki/spaces/DEV/pages/121187191) for validators, `FormRadioGroup`, etc.

---

## Weka editor

**Weka** is a content editor plugin (JSON / `FORMAT_JSON_EDITOR`), not a typical page component.

- Built from **extensions** (links, lists, embeds, …).
- Custom extensions need Tui pieces + `json_editor` node definitions in PHP.
- Example plugin: [weka-marquee-plugin-example](https://github.com/totara/weka-marquee-plugin-example)

Use Weka docs when adding editor extensions — not when scaffolding a standard admin/report page.

---

## Samples and exploration

- Browse **`client/component/tui`** for core patterns (buttons, forms, tables).
- Totara quickstart references **Totara Samples** and `/totara/tui/index.php` for trying components in isolation.
- On a feature branch, copy patterns from the closest existing bundle before inventing structure.
