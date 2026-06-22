# Build and development environment

Sources:
- [Setting up your development environment](https://totara.atlassian.net/wiki/spaces/DEV/pages/121184498)
- [The build process](https://totara.atlassian.net/wiki/spaces/DEV/pages/121188213)
- [Configuring Totara for development](https://totara.atlassian.net/wiki/spaces/DEV/pages/121185652)

---

## config.php — Tui development mode

Add to `config.php` (independent of legacy theme/AMD settings):

```php
$CFG->forced_plugin_settings['totara_tui'] = [
    'cache_js' => false,
    'cache_scss' => false,
    'development_mode' => true,
];
```

Same settings in UI: `/admin/settings.php?section=totara_tui_settings`.

Never leave development mode enabled on production — slow loads and non-minified assets.

---

## Initial setup

```bash
tnpm ci
tnpm run tui-build    # production + development builds (both committed at release)
```

For day-to-day component work, development builds + watch are enough:

```bash
tnpm run tui-build-dev
tnpm run tui-build-watch
```

Check `package.json` `engines.node` for minimum Node version (LTS recommended).

---

## Production vs development builds

| Build | Served when | Contents |
|-------|-------------|----------|
| **Production** | Default (production mode) | Minified, no debug extras |
| **Development** | `development_mode` true | Source maps, warnings, easier debugging |

Prior to release, **both** are generated and committed. While developing, only dev builds are required unless testing production-like behaviour.

Each Tui bundle gets **four variants**: production/development × legacy/modern browser targets.

---

## Watch mode

`tnpm run tui-build-watch` stays running and rebuilds **only the bundle you changed** on save — much faster than full `tui-build-dev`.

**After `tnpm run tui-init`** for a new bundle, **restart** watch so it picks up the new component.

Single-bundle build:

```bash
tnpm run tui-build-dev local_myplugin
```

---

## Docker / mutagen

```bash
tnpm run tui-build-watch
```

Otherwise dev builds may not load correctly after branch switches.

---

## How the build works (essentials)

- Source lives in `client/component/{name}/src/` (and related paths).
- Webpack outputs to `client/component/{name}/build/` — **Totara serves build files, not src**.
- One webpack build **per Tui bundle**; only the `tui` core bundle may create the shared vendor chunk.
- Cross-bundle imports use a **global module registry** at runtime (not webpack resolving across builds).
- Vue SFCs processed via `vue-loader` (template, script, style, lang-strings blocks).

Custom webpack: add `build.config.js` (or `src/build.config.js` on Totara 13–18). Example: `client/component/samples/build.config.js`.

---

## IDE setup

**VS Code**

- Vue - Official (syntax + SFC support)
- Optional: ESLint, Stylelint, Prettier, EditorConfig

**PhpStorm / WebStorm**

- Vue plugin; enable custom `lang-strings` block support (Totara 13–18)
- Optional file watcher for format-on-save

**Browser extensions (helpful)**

- Vue.js devtools
- Apollo Client devtools (when using GraphQL)

---

## Linting

Tui uses ESLint (JavaScript) and Stylelint (SCSS/CSS). Run project lint scripts from repo root when fixing CI failures — align with existing config rather than inventing rules.

---

## Legacy assets (not Tui)

Learn/Legacy theme CSS and AMD still have separate build steps — see configuring Totara for development. Do not confuse with `tnpm run tui-*` commands.
