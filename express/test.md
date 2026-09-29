# Test rules

Conventions for the specs of an Express app or library, on top of what [angular/test.md](../angular/test.md) says
about any test.

## Rules

- **The general rules hold**: a test states a behaviour, one `it()` per behaviour, names in the third person and
  without `should`, setup in `beforeEach` and in `build<Thing>()` helpers, a fixed bug gets the test that would have
  caught it, no comments. `describe` names the unit (`templates router`, `createTemplateService`) and, inside a
  router spec, the route (`POST /:id/status`), in English.
- **What is asserted is what a client or the database observes**: the status code, the response body and what a
  later request or a store read returns. Never the SQL, the calls between the functions of the repo or the private
  state of a factory.
- **A router is tested over HTTP on the real app**: `beyStartTestServer(createApp(...))` listens on a free port and
  sends each request through `fetch` with a token for the given roles. Its spec covers the wiring (permission,
  parsing, status and body); the cases of the logic belong to the spec of the service.
- **Stores are never faked**: a spec that needs data uses the real stores on an in-memory database from
  `beyCreateTestDatabase({ entities: buildEntityConfigs(), migrations: MIGRATIONS })`, built the way production
  builds its own. Nothing is written to disk, so nothing needs cleaning.
- **A service is tested through its factory**, with real stores on that database and fakes only for what leaves
  the process or depends on time: a mail sender, an OAuth provider, the clock.
- **No `jest.mock` of the modules of the repo**: a factory takes its dependencies, so the spec passes the one it
  wants.
- **Generic helpers come from `<package>/testing`**: the test server, the tokens, the test database and the test
  app config. A spec keeps locally only its `build<Thing>()` fixtures, and a fixture several specs of an app share
  lives in `src/testing/`. A helper the library lacks is added there, never rewritten in a spec.
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
