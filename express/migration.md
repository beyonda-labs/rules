# Migration rules

Conventions for changing the schema and the stored data of a database, in an Express app or library.

## Rules

- **Every change to a database is a migration**: a numbered step applied once per database, whether it creates a
  table, adds a column, fills one or rewrites stored documents. There is no one-off script with an `--apply` flag.
- **The library runs them**, with umzug behind
  `beyRunMigrations(database, { backupDirectory, entities, logger, migrations })`: it records what was applied in
  `schema_migrations`, applies the pending ones in order and awaits each, and the app does not start when one fails.
  Neither an app nor a module of the library calls umzug itself.
- **A migration is atomic on SQLite only**: there each one runs in its own transaction, so a failing one leaves the
  database as the previous one left it and the next start retries it. MySQL commits every change of the schema on
  its own, so a migration runs without a transaction: one that fails halfway keeps the changes it made and is not
  recorded, the next start may fail at its first change, and the way back is a restore of the backup the install
  takes before an upgrade.
- **A copy of the database comes first, on SQLite**: when a migration is pending, the runner copies the database
  file into the backup directory before applying it, so no migration runs on a SQLite database without that copy. A
  MySQL database is never copied: the runner logs that the backup is skipped, and the install backs the server up
  with its own tools. There is no `down`: a wrong migration is undone by a new one or by that backup.
- **Order at start-up**: the copy, the migrations of the library, the entity tables their configs declare
  (`beyEnsureEntityTables` creates or completes them), then the migrations of the app, which may read those tables.
  Seeding comes after all of them, each seed awaited.
- **Two changes of the schema are not migrations**, and `beyRunMigrations` makes both: the `schema_migrations`
  table it keeps its record in, and the tables of the entities the library manages, which `beyEnsureEntityTables`
  creates or completes from their configs (columns, indexes, the owner of the rows from before ownership, the content
  table of the attachments). Nothing else creates or alters a table, or reads what it holds, outside a migration.
- **A migration is written once for both engines**: `up(database)` may be async, changes the schema through
  `beySchema(database)` with a `BeyColumnType` per column, one statement per change, and reads and writes rows on
  `database.db`, with `beySql(database)` for what the engines write their own way, as [store.md](store.md) says. A
  string that is unique or indexed takes an indexable type, as [store.md](store.md) says too. Only what neither
  covers branches on `database.type`.
- **Every index is created with `ensureIndex`**: `beySchema(database).ensureIndex(table, name, columns, { isUnique })`
  creates it only when the table lacks it and, on MySQL, checks first that every column is indexable. A migration
  never builds one with the `createIndex` builder of Kysely. Moving an applied migration to `ensureIndex` is a rewrite
  that keeps its effect, so the schema snapshot spec proves it, as the rule below says.
- **An app keeps its migrations in `src/migrations/`**, one file per migration named `<number>-<change>.ts`
  (`002-template-definition-owner-index.ts`) exporting the function that applies it, and `migrations.ts` listing
  them in order as `{ name, up }`. The numbering is one sequence for the whole app, because a migration of one
  resource may need the table of another. A module of the library keeps its own in `migrations/`, named
  `<module>-<number>-<change>`, and they run before those of the app.
- **A SQLite database that already has data starts from a baseline**: the first migration of the app creates the
  current schema with `ifNotExists()`, so the database that exists goes through it unchanged and records it, and a
  new one ends up the same. A MySQL install always starts on an empty schema, which `beyOpenDatabase` checks, and
  goes through every migration from the first. Every later migration assumes the state the earlier ones left.
- **A repair of data only an older SQLite database can hold returns early on MySQL**, which never held it; a repair
  of data any engine may hold runs on both.
- **An applied migration never changes what it does**: once it has run on a database, a change of its effect is a new
  migration. Rewriting one while its effect stays the same on every engine where it ran (making it portable, adding
  what only an engine where it never ran needs) is allowed, and a schema snapshot spec proves it: a fresh database
  built by the new code equals the one the old code built, and a database that already ran it runs nothing again.
- **A migration that changes stored documents** hydrates them with the model library, changes them through its
  functions (`isLegacyReference`, `toImageReference`) and writes them back; it never picks the JSON apart by hand.
- **Reference data that follows the code is seeded, not migrated**: system variables and roles are upserted after
  the migrations on every start, so a new one appears without a migration. An app seeds through its stores, from a
  method of a service (`seedSystemVariables`); a seeding function of the library (`beySeedAccounts`) takes the
  database, as a store does, is awaited, and keeps its SQL where [store.md](store.md) says.
- **Every migration has a spec, on every engine**: `describe.each(beyTestEngines())`, a database in the state
  before it (`beyOpenTestDatabase({ engine })` and the migrations before it, or
  `beyCreateTestDatabase({ engine, ... })`), the migration, and what the data and the schema (`beyColumnsOf`,
  `beyIndexesOf`) look like after it, as [test.md](test.md) says.
- **No comments in the code**.

## Example

```ts
// src/migrations/001-baseline.ts
export async function createBaseline(database: BeyDatabase): Promise<void> {
    const schema = beySchema(database);

    await schema
        .withColumns(schema.createTable('template_definitions').ifNotExists(), [
            { name: 'id', type: BeyColumnType.Id, isPrimaryKey: true },
            { name: 'content', type: BeyColumnType.Document, isNotNull: true },
            { name: 'template_id', type: BeyColumnType.Id, isNotNull: true }
        ])
        .execute();
}

// src/migrations/002-template-definition-owner-index.ts
export async function indexTemplateDefinitionOwner(database: BeyDatabase): Promise<void> {
    await beySchema(database).ensureIndex('template_definitions', 'template_definitions_template_id', ['template_id']);
}

// src/migrations/003-orphan-template-definitions.ts
export async function deleteOrphanTemplateDefinitions(database: BeyDatabase): Promise<void> {
    if (database.type === BeyPersistenceType.MySQL) {
        return;
    }

    await database.db
        .withTables<TemplateDefinitionTables>()
        .deleteFrom('template_definitions')
        .where('template_id', '=', '')
        .execute();
}

// src/migrations/migrations.ts
export const MIGRATIONS: BeyMigration[] = [
    { name: '001-baseline', up: createBaseline },
    { name: '002-template-definition-owner-index', up: indexTemplateDefinitionOwner },
    { name: '003-orphan-template-definitions', up: deleteOrphanTemplateDefinitions }
];
```
