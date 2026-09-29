# Store rules

Conventions for the code that reads and writes the database in an Express app or library. Only SQLite exists
today; these rules keep the interface of a store free of the engine.

## Rules

- **SQL lives only in stores**: `<thing>.store.ts` in `stores/`, one per table or per group of tables written
  together. A library store that supports several engines keeps one adapter per engine in `stores/adapters/`.
- **A store is a factory that receives the connection**: `create<Thing>Store(database)` prepares its statements
  and returns an object typed by a `<Thing>Store` interface in `models/`. It never opens a connection, never keeps
  one in a module variable and never ignores the one it is given.
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
- **A write that touches several rows or tables is one transaction** (`database.transaction`).
- **Values are always bound** with `?` placeholders; a list is bound with one placeholder per value, never joined
  into the SQL.
- **Only migrations change the schema**, as [migration.md](migration.md) describes: no `CREATE`, `ALTER` nor
  `PRAGMA table_info` when a store is created. The exception is an entity the library manages, whose table
  `ensureTable` completes with the columns its config declares.
- **No comments in the code**.

## Example

```ts
export function createTemplateDefinitionStore(database: BeyDatabase): TemplateDefinitionStore {
    const selectById = database.prepare<[string], TemplateDefinitionRow>(
        'SELECT id, content, template_id FROM template_definitions WHERE id = ?'
    );
    const upsert = database.prepare<[string, string, string]>(`
        INSERT INTO template_definitions (id, content, template_id) VALUES (?, ?, ?)
        ON CONFLICT (id) DO UPDATE SET content = excluded.content
    `);

    return {
        async findById(id) {
            const row = selectById.get(id);

            return row ? toTemplateDefinition(row) : undefined;
        },

        async save({ id, sections, templateId }) {
            upsert.run(id, JSON.stringify({ sections }), templateId);
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
