# Test rules

Conventions for the specs of an Express app or library, on top of what [angular/test.md](../angular/test.md) says
about any test.

## Rules

- **The general rules hold**: a test states a behaviour, one `it()` per behaviour, names in the third person and
  without `should`, setup in `beforeEach` and in `build<Thing>()` helpers, a fixed bug gets the test that would have
  caught it, no comments. `describe` names the unit (`templates router`, `createTemplateService`), in English. A
  spec too long for one file is split by topic into `<unit>.<topic>.spec.ts`, whose `describe` names both
  (`users router — content`).
- **Inside a router spec, a `describe` names a route** (`POST /:id/status`) **or a context several routes share**
  (`access`, `the superadmin`, `with a password policy`).
- **What is asserted is what a client or the database observes**: the status code, the response body and what a
  later request or a store read returns. Never the SQL a store runs, the calls between the functions of the repo or
  the private state of a factory. A spec reads a table itself only where no store can show the state: a change of
  the schema (a migration, `beyEnsureEntityTables` on an older database) or a row the stores hide by design, such as
  a definition left without its template. The driver specs below are the one place a spec asserts SQL.
- **A router is tested over HTTP on the real app**: `beyStartTestServer(createApp(...))` listens on a free port and
  sends each request through `fetch` with a token for the given roles; the library mounts its module on an app as
  a consumer does. Its spec covers what a client observes: who may call each route, what it parses, every status
  and `messageKey` it answers, the body it returns, and a flow that only shows across routes (an invitation
  accepted, then a sign-in). The variations of the logic behind one answer (each sort, filter or case of a rule)
  belong to the spec of the service; where no service holds that logic, as in the entity routes of the library and
  the hooks a resource hands them, the router spec covers them.
- **A spec that runs the app under several configurations has one `start(overrides)` helper**: it starts the app
  with the defaults and the changes of the case. `beforeEach` calls it, an `it()` that needs another configuration
  calls it again (or `restart(overrides)`, which first closes the server running), and `afterEach` closes the last
  one.
- **Stores are never faked**: a spec that needs data writes it through the real stores, on the test database of
  `beyCreateTestDatabase({ entities: buildEntityConfigs(), migrations: MIGRATIONS })`, built the way production
  builds its own: in memory on SQLite, a schema of its own on the MySQL server of the tests. A row no store writes
  the way the spec needs it (a fixed id, a user outside every organization) goes through a seeding helper of the
  library (`beySeedEntityRows`, `<package>/testing`), never through SQL in a spec or a fixture. The one exception is a
  state the stores refuse by design, such as a definition whose template is gone, which a spec proving that the
  stores ignore it writes in SQL, as it reads such a row.
- **A store is made to fail by closing its database**: `await database.close()` before the call, so every store on it
  rejects as it would on a lost connection. Never a partial fake of a store, nor a spy that replaces one of its
  methods.
- **Nothing else touches the disk**, unless the behaviour under test is a file (a backup, a log, the front a server
  serves): the spec then works in a directory it creates under `os.tmpdir()` (`fs.mkdtempSync`) and removes when
  it is done, a file database included.
- **A spec that exercises SQL runs on every engine**: the spec of a store, a migration, the schema or a transaction
  is `describe.each(beyTestEngines())('<unit> (%s)', engine => ...)` and opens its database with `{ engine }`
  (`beyCreateTestDatabase`, or `beyOpenTestDatabase` for one no migration has run on). Without `BEY_TEST_MYSQL_URL`
  the list holds SQLite alone, so the specs of routers and services, which run on SQLite, need nothing more.
- **A spec that opens a test database resets it in its `afterEach`**: `beyResetTestDatabase()` closes every database
  `beyCreateTestDatabase` and `beyOpenTestDatabase` opened and drops the schema of MySQL, so neither a connection
  nor a schema outlives the case, on any engine.
- **A spec stays on SQLite alone when SQLite is what it tests**: the copy of the database file the runner takes
  before a migration, or a database built by the statements earlier versions ran on SQLite (the schema snapshot of
  `src/lib/internal/testing/`), which no MySQL install ever held. It opens its SQLite database itself and closes it
  in `afterEach`.
- **The driver specs read the SQL of MySQL without a server**: the specs of what turns the code into the SQL of one
  engine (the dialect helpers, the schema of a table, the runner of the migrations, the MySQL branch of an adapter
  or a migration) also run on `openCompiledMysqlDatabase()` of `src/lib/internal/testing/`, which compiles every
  statement for MySQL and answers what the spec gives it, and assert the statements it compiled. The spec of the
  connection replaces the driver with `jest.mock('mysql2')`. So the pipeline, on SQLite, still covers what only
  MySQL is sent.
- **The MySQL battery runs by hand**, never in Jenkins nor in a script of `package.json`, so the pipeline stays on
  SQLite: a throwaway `mysql:8.4` server started with `docker run` and its data on a `tmpfs`, then the specs with
  `BEY_TEST_MYSQL_URL` naming it, `jest -t "(mysql)"` for the MySQL half alone and `--testTimeout=30000`, since the
  first migrations of a spec file on MySQL take longer than the 5 seconds of Jest. The testing readme of the library
  holds both commands; `docker rm -f` removes the server when it is done.
- **A statement the driver refuses is asserted with `rejects.toHaveProperty('message')`**, never
  `rejects.toThrow()`: the error of a driver is not always an `Error` of the realm of the spec file, and `toThrow`
  does not recognise it then.
- **A service is tested through its factory**, with real stores on that database and fakes only for what leaves
  the process or depends on time: a mail sender, an OAuth provider, the clock.
- **No `jest.mock` of the modules of the repo**: a factory takes its dependencies, so the spec passes the one it
  wants.
- **Generic helpers come from `<package>/testing`**: the test server, the tokens, the test database and the test
  authentication config. A spec keeps locally only its `build<Thing>()` fixtures, and a fixture several specs
  share lives in `src/testing/` in an app and in `src/lib/internal/testing/` in the library
  ([library.md](library.md)). A generic helper the library lacks is added to `<package>/testing`, never rewritten
  in a spec.
- **Every migration, store, service, function module and router has its spec**; `index.ts` and `app.ts` are
  covered by the router specs that start the app.
- **Coverage thresholds** sit just under the current numbers, as in every repo.

## Example

```ts
describe('templates router', () => {
    let server: BeyTestServer;

    beforeEach(async () => {
        server = await beyStartTestServer(createApp(await buildTestAppDependencies()));
    });

    afterEach(() => server.close());

    describe('POST /:id/status', () => {
        it('rejects a status the current one cannot move to', async () => {
            const { body: template } = await server.request<Template>('POST', '/templates', {
                body: { name: 'Offer' },
                roles: ['editor']
            });
            const response = await server.request('POST', `/templates/${template.id}/status`, {
                body: { status: 'draft' },
                roles: ['editor']
            });

            expect(response.status).toBe(400);
            expect(response.body).toMatchObject({ messageKey: 'template.invalid-transition' });
        });

        it('forbids a role without the change-status permission', async () => {
            const response = await server.request('POST', '/templates/any/status', {
                body: { status: 'published' },
                roles: ['viewer']
            });

            expect(response.status).toBe(403);
        });
    });
});
```
