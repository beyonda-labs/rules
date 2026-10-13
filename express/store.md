# Store rules

Conventions for the code that reads and writes the database in an Express app or library. Only SQLite exists
today; these rules keep the interface of a store free of the engine.

## Rules

- **SQL lives only in stores and migrations**: `<thing>.store.ts` in `stores/`, one per table or per group of
  tables written together. Only the plumbing of the connection in the library (a transaction, the health check,
  the runner of the migrations) speaks SQL elsewhere.
- **A store of the library keeps the engine out of its file**: `<thing>.store.ts` picks the adapter and the SQL sits
  in `stores/adapters/sqlite-<thing>.adapter.ts`, so a second engine is one more adapter. A part an adapter splits
  out stays next to it, named after the engine and what it builds (`sqlite-group-search.ts`), and its row types go
  in `models/sqlite-<thing>.model.ts`. The SQL helpers the adapters of several modules share (`placeholders`,
  `escapeLike`) live in `src/lib/internal/sqlite/`, as [library.md](library.md) places what modules share, and a
  seeding function of the library (`beySeedAccounts`) sits with the adapters of its module.
- **A store is a factory that receives the connection**: `create<Thing>Store(database)` returns an object typed by a
  `<Thing>Store` interface in `models/`. It never opens a connection, never keeps one in a module variable and
  never ignores the one it is given.
- **A statement is prepared where it runs**: in the method, or by a function the factory declares once and the
  methods call (`() => database.prepare(...)`), since many depend on their arguments (an `IN` list, a visibility
  condition). Creating a store runs no SQL.
- **One connection per database**, opened by `beyOpenDatabase` in `src/index.ts` (WAL journal, foreign keys on)
  and passed to every store, those of the library included. Only the library imports the driver; an app types the
  connection as `BeyDatabase`.
- **Its methods return promises**, even on SQLite, so the interface does not change with the engine.
- **It speaks in models, not rows**: the row type of a table (`TemplateDefinitionRow`, with the column names as
  they are) is turned into the model inside the store, both ways, and nothing outside it sees a column name. A JSON
  column is hydrated with the functions of the model library on its way out.
- **An entity store is typed**: `beyCreateEntityStore<Template>(...)` returns a `BeyEntityStore<Template>`, so no
  caller reads `row['name'] as string`.
- **A read names what it returns**: `findById`, `findByIds`, `findAll`, `exists`, `count<Thing>`. A read that needs
  every row is `findAll`, never a search with the maximum page size.
- **A write a user makes takes the write options last** (`update(id, changes, options)`) and stamps `updatedAt`
  and `updatedBy` from them in the same statement, never from a bare time it is handed, as
  [service.md](service.md) says.
- **A write that touches several rows or tables is one transaction** (`database.transaction`) inside the store.
  When a service writes through several stores, it wraps them in `store.transaction(work)`, as
  [service.md](service.md) says, and the transaction of each store joins it.
- **Values are always bound** with `?` placeholders; a list is bound with one placeholder per value, never joined
  into the SQL.
- **A store never changes the schema**: neither its creation nor its methods issue `CREATE`, `ALTER` or
  `PRAGMA table_info`. Only migrations do, with the two exceptions [migration.md](migration.md) names.
- **No comments in the code**.

## Example

```ts
export function createTemplateDefinitionStore(database: BeyDatabase): TemplateDefinitionStore {
    const selectByIds = (ids: string[]) =>
        database.prepare<string[], TemplateDefinitionRow>(
            `SELECT id, content, template_id FROM template_definitions WHERE id IN (${ids.map(() => '?').join(', ')})`
        );

    return {
        async findByIds(ids) {
            return ids.length === 0 ? [] : selectByIds(ids).all(...ids).map(toTemplateDefinition);
        },

        async save({ id, sections, templateId }) {
            database
                .prepare<[string, string, string]>(`
                    INSERT INTO template_definitions (id, content, template_id) VALUES (?, ?, ?)
                    ON CONFLICT (id) DO UPDATE SET content = excluded.content
                `)
                .run(id, JSON.stringify({ sections }), templateId);
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
