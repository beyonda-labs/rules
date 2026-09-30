# Express library rules

Conventions that apply to a publishable Express library (`api-components`), on top of the rules for routers,
services, stores, migrations, errors and tests.

## Rules

- **Generic, never tied to a product**: nothing of the domain of a product and no dependency on a product
  package. A module only one product uses, such as the PDF generation of document builder, lives in that product.
- **Every module has its own `public-api.ts`** in `src/lib/modules/<module>/`, and the root `src/public-api.ts`
  only re-exports those files. A consumer never imports a deep path. Inside the library a module imports the file
  of another one directly, under its own unprefixed name, never through a `public-api.ts`; what several modules
  share and no consumer needs goes to `src/lib/internal/<topic>/`.
- **The modules form no import cycle**: each one depends only on the ones below it (`http-errors`, `validation`,
  `persistence`, `user-store`, `authentication`, `base-entity`, `attachments`, `base-app`). A module takes the part
  of a config it reads as a structural context (`AuthenticationContext`), never the app config of the module that
  composes it.
- **Everything public is renamed with the library prefix** on the way out, as in the component library: types and
  classes start with `Bey` (`BeyAppConfig`, `BeyNotFoundError`), functions with `bey` (`beyCreateAuthModule`,
  `beyRequirePermission`). Without exception, so a consumer never has to alias.
- **A module is built by a factory**: `<module>.module.ts` exports the function that builds its router from one
  object of dependencies (`{ appConfig, store, ...hooks }`). Next to it sit `<module>.router.ts`, `models/`,
  `services/`, `stores/` (with `stores/adapters/` when there is more than one engine), `middleware/`,
  `functions/`, `migrations/` and `docs/`. A middleware is `<name>.middleware.ts`, a factory that returns a
  `RequestHandler`.
- **Config classes are models**: they follow [angular/class-model.md](../angular/class-model.md), with their
  `<Name>Parameters` interface exported as `export type`, in `models/<name>.model.ts`. There is no `.config.ts`
  suffix, and a function that builds a config is a function module.
- **The library never reads `process.env`**: it receives values (`jwtSecret`), never the name of a variable to read
  (`jwtSecretEnv`). The app reads its environment, as [app.md](app.md) says.
- **Test support is the `<package>/testing` entry point**, with sources under `testing/src/`: the test server, the
  tokens, the in-memory database with its migrations and the test authentication config. It imports the library
  by its package name, nothing in the library imports it, and the specs of the library use it too, so no
  `*.test-helpers.ts` sits next to the code. A fixture several specs of the library share and no consumer needs
  lives in `src/lib/internal/testing/`, which the coverage leaves out.
- **`express` is a peer dependency**, as Angular is in the component library, so the app and the library share
  one copy and one set of types. The other runtime dependencies are few and justified (the SQLite driver, JWT,
  hashing, umzug), and every version is exact.
- **Built as CommonJS and ES modules with types** (`tsup`: `cjs`, `esm`, `dts`), with one entry per public entry
  point (`.`, `./testing`) listed in `exports`, and `"sideEffects": false`.
- **Published to the private registry** through `publishConfig.registry`, with `.npmrc` sending the scope there;
  every push to develop publishes a `<version>-develop.<build>.<sha>` snapshot.
- **Every module has a `docs/<module>-readme.md`** with the shape the component library uses: one paragraph on what
  it does, a table of its config and its dependencies, and a minimal usage example. The change detail goes in
  `docs/change-log.md`.
- **A breaking change to an exported name is a major**, and the old name is not kept as an alias.

## Example

```ts
// src/lib/modules/http-errors/public-api.ts
export { createErrorHandler as beyCreateErrorHandler } from './middleware/error-handler.middleware';
export { AppError as BeyAppError } from './models/app-error.model';
export type { AppErrorOptions as BeyAppErrorOptions } from './models/app-error.model';
```

```json
{
    "name": "@beyonda-labs/express-components",
    "publishConfig": { "registry": "https://verdaccio.home.arpa/" },
    "exports": {
        ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs", "require": "./dist/index.js" },
        "./testing": { "types": "./dist/testing.d.ts", "import": "./dist/testing.mjs", "require": "./dist/testing.js" }
    },
    "sideEffects": false,
    "peerDependencies": { "express": "5.2.1" }
}
```
