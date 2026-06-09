# ai-skills

Custom [Cursor Agent Skills](https://cursor.com/docs) for project-specific workflows. Each skill is a directory with a `SKILL.md` and optional supporting files.

## Structure

```text
.cursor/skills/
└── {skill-name}/
    ├── SKILL.md              # required
    └── references/           # optional
        └── ...
```

## Usage

**Project scope** — copy or symlink the skills folder into your repo:

```bash
cp -r .cursor/skills /path/to/your-project/.cursor/
```

**Global scope** — copy individual skills to your user directory:

```bash
cp -r .cursor/skills/{skill-name} ~/.cursor/skills/
```

Cursor loads skills from `.cursor/skills/` (project) and `~/.cursor/skills/` (global).

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

## Adding a skill

1. Create `.cursor/skills/{skill-name}/SKILL.md` with YAML frontmatter (`name`, `description`).
2. Add optional reference files alongside (keep `SKILL.md` under ~500 lines).
3. Update the table above with a one-line summary and link.

Skill names: lowercase, hyphens only, max 64 characters.
