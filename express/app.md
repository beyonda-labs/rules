# App rules

Conventions for the entry point, the composition root and the configuration of an Express app: the back of a
product and the demo of the Express library.

## Rules

- **`src/index.ts` only starts the process**: it reads the environment, opens the database, runs the migrations,
  seeds the reference data and listens. When a step fails the process ends with a non-zero code; it never listens
  on an app that did not finish starting.
- **The environment is read once**, in `src/environment.ts`: `readEnvironment(process.env)` checks every variable,
  applies the defaults and returns a typed `Environment`. `process.env` appears nowhere else, and `.env.example`
  lists every variable it reads.
- **`src/app.ts` only wires**: `createApp(dependencies)` builds what several resources share (the app config, the
  access guards, the stores of the entities they all read), calls the module of every resource and hands them to
  the library's `beyInitBaseApp`, which adds JSON, CORS, the authentication routes, the modules in order and the
  error handler last. No route, handler or query of its own.
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
- **`console` only in `src/index.ts`**: everything else logs through the logger it receives, as the error handler
  of the library does.
- **Development runs `tsx watch --env-file-if-exists=.env src/index.ts`** and production
  `node --env-file-if-exists=.env dist/index.js`. No nodemon, ts-node, dotenv nor start script of its own:
  base-config ships both scripts.
- **No comments in the code**: whatever needs explaining goes in the README.

## Example

```ts
// src/index.ts
async function start(): Promise<void> {
    const environment = readEnvironment(process.env);
    const database = beyOpenDatabase(environment.databasePath);

    await beyRunMigrations(database, {
        backupDirectory: environment.backupDirectory,
        entities: buildEntityConfigs(),
        migrations: MIGRATIONS
    });
    await seedSystemVariables(database);
    createApp({ database, environment }).listen(environment.port);
}

start().catch(error => {
    console.error(error);
    process.exitCode = 1;
});

// src/app.ts
export function createApp({ database, environment }: AppDependencies): Application {
    const appConfig = buildAppConfig(environment);
    const accessGuards = beyCreateAccessGuards(appConfig.authentication);
    const templateStore = beyCreateEntityStore<Template>(database, buildTemplatesEntityConfig());
    const shared = { accessGuards, appConfig, database, templateStore };

    return beyInitBaseApp(appConfig, [
        ['/templates', createTemplatesModule(shared)],
        ['/template-definitions', createTemplateDefinitionsModule(shared)]
    ]);
}
```
