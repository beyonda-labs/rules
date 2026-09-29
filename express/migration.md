# Migration rules

Conventions for changing the schema and the stored data of a database, in an Express app or library.

## Rules

- **Every change to a database is a migration**: a numbered step applied once per database, whether it creates a
  table, adds a column, fills one or rewrites stored documents. There is no one-off script with an `--apply` flag.
- **The library runs them**, with umzug behind
  `beyRunMigrations(database, { backupDirectory, entities, migrations })`: it records what was applied in
  `schema_migrations`, applies the pending ones in order, each in its own transaction, and the app does not start
  when one fails. Neither an app nor a module of the library calls umzug itself.
- **A copy of the database comes first**: when a migration is pending, the runner copies the database into the
  backup directory before applying it, so no migration runs on a database without that copy. There is no `down`:
  a wrong migration is undone by a new one or by that copy.
- **Order at start-up**: the copy, the migrations of the library, the entity tables their configs declare
  (`ensureTable` creates or completes them), then the migrations of the app, which may read those tables. Seeding
  comes after all of them.
- **An app keeps its migrations in `src/migrations/`**, one file per migration named `<number>-<change>.ts`
  (`002-template-definition-owner-index.ts`) exporting the function that applies it, and `migrations.ts` listing
  them in order as `{ name, up }`. The numbering is one sequence for the whole app, because a migration of one
  resource may need the table of another. A module of the library keeps its own in `migrations/`, named
  `<module>-<number>-<change>`, and they run before those of the app.
- **An app that already has data starts from a baseline**: its first migration creates the current schema with
  `IF NOT EXISTS`, so the database that exists goes through it unchanged and records it, and a new one ends up the
  same. Every later migration assumes the state the earlier ones left.
- **An applied migration is never edited**: once it has run on a database, a change is a new migration.
- **A migration that changes stored documents** hydrates them with the model library, changes them through its
  functions (`isLegacyReference`, `toImageReference`) and writes them back; it never picks the JSON apart by hand.
- **Reference data that follows the code is seeded, not migrated**: system variables and roles are upserted after
  the migrations on every start, so a new one appears without a migration.
- **Every migration has a spec**: an in-memory database in the state before it, the migration, and what the data
  looks like after it.
- **No comments in the code**.

## Example

```ts
// src/migrations/001-baseline.ts
export function createBaseline(database: BeyDatabase): void {
    database.exec(`
        CREATE TABLE IF NOT EXISTS template_definitions (
            id TEXT PRIMARY KEY,
            content TEXT NOT NULL,
            template_id TEXT NOT NULL DEFAULT ''
        )
    `);
}

// src/migrations/002-template-definition-owner-index.ts
export function indexTemplateDefinitionOwner(database: BeyDatabase): void {
    database.exec('CREATE INDEX template_definitions_template_id ON template_definitions (template_id)');
}

// src/migrations/migrations.ts
export const MIGRATIONS: BeyMigration[] = [
    { name: '001-baseline', up: createBaseline },
    { name: '002-template-definition-owner-index', up: indexTemplateDefinitionOwner }
];
```
