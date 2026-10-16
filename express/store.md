# Store rules

Conventions for the code that reads and writes the database in an Express app or library. SQLite and MySQL 8.4 are
both supported: every query is written once on Kysely, and these rules keep a store, its interface and its SQL free
of the engine.

## Rules

- **SQL lives only in stores and migrations**: `<thing>.store.ts` in `stores/`, one per table or per group of
  tables written together. Only the plumbing of the database in the library (a transaction, the health check, the
  runner of the migrations) builds queries elsewhere.
- **A store is written once for both engines**: its queries go through the Kysely builder, and what each engine
  writes its own way (an upsert, a folded search, the order of text, a lock) goes through `beySql(database)` of
  persistence, as a change of the schema goes through `beySchema(database)` (`dialectSql` and `dialectSchema` inside
  the library). A store never writes SQL that only one engine understands; `database.type` is read only for what one
  engine alone needs, such as the start-up check of the `max_allowed_packet` of MySQL.
- **A store of the library keeps its SQL out of its file**: `<thing>.store.ts` picks the adapter and the queries sit
  in `stores/adapters/`, one adapter that serves both engines. A new adapter is named after what it stores
  (`<thing>.adapter.ts`), and after where it stores it only when a store has more than one
  (`disk-attachment-content.adapter.ts`); a part an adapter splits out stays next to it, named after what it builds
  (`<thing>-search.ts`), and its row types go in `models/<thing>-row.model.ts`. Only what one engine alone runs keeps
  its name (`internal/sqlite/`, with the `bey_fold` function of SQLite). The helpers the adapters of several modules
  share (`escapeLike`, the row counts of an organization) live in `src/lib/internal/<topic>/`, as
  [library.md](library.md) places what modules share, and a seeding function of the library (`beySeedAccounts`) sits
  with the adapters of its module.
- **A store is a factory that receives the database**: `create<Thing>Store(database)` returns an object typed by a
  `<Thing>Store` interface in `models/`. It never opens a database, never keeps one in a module variable and never
  ignores the one it is given.
- **A query is built on `database.db` where it runs**: in the method, or by a function the factory declares once and
  the methods call (`() => database.db.selectFrom(...)`), since `database.db` answers the transaction open around
  the work and many queries depend on their arguments (an `IN` list, a visibility condition). Neither `database.db`
  nor a builder is kept in a variable, and creating a store runs no SQL.
- **One database per app**, opened by `await beyOpenDatabase(persistence, { logger })` in `src/index.ts` and passed to
  every store, those of the library included: one connection on SQLite (WAL journal, foreign keys on), a pool on
  MySQL. Only the library imports the drivers; an app types the handle as `BeyDatabase` and its own tables with a
  function that answers `database.db` as `Kysely<Tables>` (`(): Kysely<Tables> => database.db`), read on every call.
- **Its methods return promises**, as every query of Kysely does.
- **It speaks in models, not rows**: the row type of a table (`TemplateDefinitionRow`, with the column names as
  they are) types its Kysely table and is turned into the model inside the store, both ways, and nothing outside it
  sees a column name. A JSON column is hydrated with the functions of the model library on its way out.
- **An entity store is typed**: `beyCreateEntityStore<Template>(...)` returns a `BeyEntityStore<Template>`, so no
  caller reads `row['name'] as string`.
- **A read names what it returns**: `findById`, `findByIds`, `findAll`, `exists`, `count<Thing>`. A read that needs
  every row is `findAll`, never a search with the maximum page size.
- **A write a user makes takes the write options last** (`update(id, changes, options)`) and stamps `updatedAt`
  and `updatedBy` from them in the same statement, never from a bare time it is handed, as
  [service.md](service.md) says.
- **A write that touches several rows or tables is one transaction** (`beyRunInTransaction(database, work)`) inside
  the store. When a service writes through several stores, it wraps them in `store.transaction(work)`, as
  [service.md](service.md) says, and the transaction of each store joins it. A transaction owns a connection until
  it ends and a transaction of another request waits for it, so the slow work goes before it starts: a hash, a scan,
  the extraction of a file, a network call. A read that decides the write, where two requests at once could both
  pass it, locks its rows with `lockForUpdate`.
- **Values are always bound**: Kysely binds every value the builder receives, and a list goes through
  `where(column, 'in', ids)`, never joined into the SQL. Only text of the code goes through `sql.raw`.
- **No `RETURNING`**, which MySQL lacks: a write that needs what the database filled in selects the row after it,
  inside the same transaction.
- **A write over several rows resolves the ids first**: it selects the ids it may change, then writes
  `WHERE id IN (…)` with them, on both engines. MySQL refuses a write whose subquery reads the table it changes
  (error 1093).
- **An order or a comparison never mixes an id with text people read**: on MySQL an `Id` or `Key` column compares in
  binary and text in `utf8mb4_0900_ai_ci`, so a `COALESCE` or a `CASE` in an `ORDER BY` that falls back from a name to
  an id orders in binary. Such a fallback is resolved in the code, after the query, and text people read is ordered
  through `orderText`.
- **A string that is unique or indexed is an indexable type**: an `Id`, `Key`, `Name` or `BinaryName` of `beySchema`,
  since MySQL indexes no `TEXT` without a length, and a unique string takes 255 characters at most, which the schema
  of its field holds.
- **A store never changes the schema**: neither its creation nor its methods create or alter a table or read what
  it holds (`beySchema(database).columnsOf`). Only migrations do, with the two exceptions
  [migration.md](migration.md) names.
- **No comments in the code**.

## Example

```ts
export function createTemplateDefinitionStore(database: BeyDatabase): TemplateDefinitionStore {
    const tables = (): Kysely<TemplateDefinitionTables> => database.db;
    const selectDefinitions = () =>
        tables().selectFrom('template_definitions').select(['id', 'content', 'template_id']);

    return {
        async findByIds(ids) {
            if (ids.length === 0) {
                return [];
            }

            const rows = await selectDefinitions().where('id', 'in', ids).execute();

            return rows.map(toTemplateDefinition);
        },

        async save({ id, sections, templateId }) {
            const { upsert } = beySql(database);

            await upsert(
                tables().insertInto('template_definitions'),
                ['id'],
                { id, content: JSON.stringify({ sections }), template_id: templateId },
                ['content']
            ).execute();
        }
    };
}

function toTemplateDefinition(row: TemplateDefinitionRow): TemplateDefinition {
    const { sections } = JSON.parse(row.content) as StoredDefinitionContent;

    return {
        id: row.id,
        sections: sections.map(section => hydrateDocumentSection(section)),
        templateId: row.template_id
    };
}
```
