# Model library rules

Conventions for a package that holds the shared model of a product, consumed by its front and its back alike
(`document-builder-models`).

## Rules

- **Plain TypeScript, no framework.** Nothing from Angular, Express or any other runtime is imported, so the
  same code runs in the browser and in Node.
- **No I/O and no i18n of its own.** The package never reads a file, calls the network or holds a translation:
  a message it produces is a translation key the consuming app resolves.
- **One entry point.** Everything public is exported from `src/public-api.ts` and re-exported by `src/index.ts`;
  a consumer never imports a deep path.
- **Only what a consumer needs is exported**: the model classes with their `Parameters` interfaces (as
  `export type`), enums, constants and the functions that work on the model.
- **No library prefix on public names.** The package name is the namespace, and the names are specific to the
  domain (`DocumentItem`, `TemplateStatus`). A consumer that needs its own type with the same name renames it on
  import; a concept defined in two packages is a duplication to remove, not a name to prefix.
- **Built as CommonJS and ES modules with types** (`tsup`: `cjs`, `esm`, `dts`), listed in `exports`, with
  `"sideEffects": false` so consumers tree-shake what they do not use.
- **Runtime dependencies are few and justified.** Every one of them ends up in every consumer.
- **Every dependency has an exact version**, never a range nor a local `link:`/`file:` reference, and
  `check-dependencies` fails `lint` when one does.
- **Published to the private registry** through `publishConfig.registry`, with `.npmrc` sending the scope there.
- **The gates are the same as in any library**: `lint` (ESLint, `typecheck`, `check-dependencies`), `format`,
  `test:ci` with coverage thresholds set just under the current numbers, and `verify` running them all. Pure
  logic that front and back both depend on is the code that most deserves the thresholds.
- **Model files are left out of the coverage** (`!**/*.model.ts`): they hold no logic, and counting their
  constructor assignments would only inflate the numbers and hide untested functions. The thresholds measure the
  function modules.
- **The change detail goes in `docs/change-log.md`**, one line per change a consumer notices.

## Example

```json
{
    "name": "@beyonda-labs/document-builder-models",
    "publishConfig": { "registry": "https://verdaccio.home.arpa/" },
    "main": "dist/index.js",
    "module": "dist/index.mjs",
    "types": "dist/index.d.ts",
    "exports": {
        ".": { "types": "./dist/index.d.ts", "import": "./dist/index.mjs", "require": "./dist/index.js" }
    },
    "files": ["dist"],
    "sideEffects": false,
    "dependencies": { "uuid": "11.1.0" }
}
```
