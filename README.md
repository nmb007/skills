# ai-skills

Custom agent skills for project-specific workflows. Each skill is a directory with a `SKILL.md` and optional supporting files. The `skills/` layout is the location the [skills CLI](https://skills.sh/) discovers.

## Structure

```text
skills/
└── {skill-name}/
    ├── SKILL.md              # required
    └── references/           # optional
        └── ...
```

## Usage

Install the skills in this repo:

```bash
npx skills add nmb007/skills
```

Install one skill:

```bash
npx skills add nmb007/skills --skill totara-orm
```

The CLI copies the chosen skills into your agent's skills directory (for Cursor, that is `.agents/skills/` in the project, or the user skills directory with `-g`).

To invoke explicitly in chat:

```text
Use {skill-name} skill: <your task>
```

## Skills

| Skill | Description |
|-------|-------------|
| totara-plugin-structure | Totara plugin layout — server/client roots, frankenstyle, db/, classes/, v1/v2 src/, scaffolding workflow |
| totara-tui-components | TUI bundles and Vue SFCs — scaffold, tui.json, build, MVC, structure, testing |
| totara-tui-theme | Totara child/custom themes — inheritance, CSS variables, SCSS, TUI overrides, Mustache, renderers, Theme Appearance |
| totara-tui-apollo | Totara TUI frontend with Apollo — Vue Apollo queries, mutations, cache, and GraphQL imports |
| totara-orm | Totara ORM PHP — entities, mandatory entity models, repositories, relationships, filters, query builder, paginator |
| totara-graphql-api | Totara GraphQL PHP backend — schema, resolvers, persisted queries, middleware, endpoints |
| totara-phpunit | Totara PHPUnit — setup, running tests (single/parallel/suite/filter), adding test cases, naming conventions |

## Adding a skill

1. Create `skills/{skill-name}/SKILL.md` with YAML frontmatter (`name`, `description`). The `name` must match the directory name.
2. Add optional reference files alongside (keep `SKILL.md` under ~500 lines).
3. Update the table above with a one-line summary and link.

Skill names: lowercase, hyphens only, max 64 characters.
