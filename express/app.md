# App rules

Conventions for the entry point, the composition root and the configuration of an Express app: the back of a
product and the demo of the Express library.

## Rules

- **`src/index.ts` only starts the process**: it reads the environment, creates the logger, opens the database, runs
  the migrations, seeds the reference data and listens. When a step fails the process ends with a non-zero code; it never listens
  on an app that did not finish starting.
- **The environment is read once**, in `src/environment.ts`: `readEnvironment(variables = process.env)` checks every
  variable, applies the defaults and returns a typed `Environment`; a spec passes its own variables. `process.env`
  appears nowhere else, and `.env.example` lists every variable it reads.
- **`src/app.ts` only wires**: `createApp(dependencies)` builds what several resources share (the app config, the
  access guards, the user store, the stores of the entities they all read), calls the module of every resource and
  hands them to the library's `beyInitBaseApp({ appConfig, routes, userStore, logger })`, which adds the request log,
  CORS for `appConfig.corsOrigin`, the rate limits, JSON up to `appConfig.bodyLimit`, the authentication routes, the
  modules in order and the error handler last. No route, handler or query of its own.
- **One logger for the whole process**: `src/index.ts` builds it with `beyCreateLogger(buildLoggerConfig(environment))`
  and hands it to `createApp`, so the start, every request and every unhandled error end up in the same place. Where
  it writes (files or the standard output), its format and its level come from the environment; a spec takes
  `beyCreateTestLogger()`.
- **The limits are declared in `buildAppConfig`**: a general `rateLimit`, an `authenticationRateLimit` that counts only
  the failed attempts, and `trustProxy` from the environment. A route that is expensive to answer, such as drawing a
  PDF, gets a limiter of its own from `beyCreateRateLimiter`, created once in `createApp` and passed to its module.
- **Configuration is built by functions, never at import time**: `src/app-config.ts` exports
  `buildAppConfig(environment)` and `buildEntityConfigs()`, and an entity the library manages has
  `<resource>-entity-config.ts` exporting `build<Resource>EntityConfig()` next to the module of its resource, as a
  page keeps its page config next to its component. A module-level `new BeyAppConfig(...)` depends on the order of
  the imports.
- **An entity config declares the entity, not its behaviour**: fields, table, categories, unique and immutable
  fields. The hooks of its router (`filterItemActions`, `onDeleted`, `decorateResults`) are passed by the module of
  the resource when it builds the router, from its services and its function modules.
- **Permissions are an enum per resource**, in its model file: singular PascalCase name, PascalCase members and
  the stored string as value (`TemplatePermission.ChangeStatus = 'templates.change-status'`). The roles that hold
  them are declared in `buildAppConfig`.
- **Named exports only**: no `export default`, so every import says what it takes.
- **Node built-ins are imported with `node:`** (`node:fs`, `node:crypto`), in code and specs alike.
- **`console` only in `src/index.ts`**, for a failure before the logger exists: everything else logs through the logger
  it receives, as the error handler of the library does.
- **Development runs `tsx watch --env-file-if-exists=.env src/index.ts`** and production
  `node --env-file-if-exists=.env dist/index.js`. No nodemon, ts-node, dotenv nor start script of its own:
  base-config ships both scripts.
- **No comments in the code**: whatever needs explaining goes in the README.

## Example

```ts
// src/index.ts
async function start(): Promise<void> {
    const environment = readEnvironment();
    const logger = beyCreateLogger(buildLoggerConfig(environment));
    const database = beyOpenDatabase(environment.databasePath);

    await beyRunMigrations(database, {
        backupDirectory: environment.backupDirectory,
        entities: buildEntityConfigs(),
        migrations: MIGRATIONS
    });
    beySeedAccounts(database, buildAppConfig(environment).authentication.persistence, message => logger.info(message));
    await seedSystemVariables(database);
    createApp({ database, environment, logger }).listen(environment.port);
}

start().catch(error => {
    console.error(error);
    process.exitCode = 1;
});

// src/app.ts
export function createApp({ database, environment, logger }: AppDependencies): Application {
    const appConfig = buildAppConfig(environment);
    const accessGuards = beyCreateAccessGuards(appConfig.authentication);
    const templateStore = beyCreateEntityStore<Template>(database, buildTemplatesEntityConfig());
    const userStore = beyCreateUserStore(database);
    const shared = { accessGuards, appConfig, database, templateStore, userStore };

    return beyInitBaseApp({
        appConfig,
        logger,
        routes: [
            ['/templates', createTemplatesModule(shared)],
            ['/template-definitions', createTemplateDefinitionsModule(shared)]
        ],
        userStore
    });
}
```
