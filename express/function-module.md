# Function module rules

Conventions for the stateless logic of an Express app or library: deciding the actions of an item, finding a
free name, checking the shape of a block.

## Rules

- **Plain exported functions, never a class**, when the logic needs no dependency. Logic that needs a store or
  another service is a method of a service ([service.md](service.md)), even when it looks like a helper.
- **The file is named after what its functions do, without a technical suffix**: `template-item-actions.ts`,
  `available-name.ts`. No `.helper.ts` nor `.util.ts`; only the artefacts keep theirs (`.module.ts`, `.router.ts`,
  `.service.ts`, `.store.ts`, `.middleware.ts`, `.model.ts`).
- **It lives in a `functions/` folder at the level of what uses it**: `<resource>/functions/` in an app,
  `<module>/functions/` in a library, `src/shared/functions/` when several resources use it and
  `src/lib/internal/<topic>/` when several modules of the library do. `models/` holds contracts only, `services/`
  only services and `stores/` only stores.
- **Types and constants it needs go in a model file**, never in the function module.
- **Logic about the shared model belongs to the model library**: a function the front needs too, such as
  reconciling the variables of a block or telling a legacy reference, is written there once and never copied into
  the back.
- **The shape of the module follows [model-library/function-module.md](../model-library/function-module.md)**:
  grouped by the question it answers, constants at the top, exported functions first and in alphabetical order, a
  new value returned instead of a mutated argument, plain data taken where it can and one spec per module.
- **A function may throw an error of the library** when its input breaks a rule (`assertMatchesBlockShape`), as
  long as it reads nothing but its arguments.
- **No comments in the code**.

## Example

```ts
export function findAvailableName(name: string, takenNames: string[]): string {
    const taken = new Set(takenNames);
    let candidate = name;

    for (let index = 2; taken.has(candidate); index += 1) {
        candidate = `${name} ${index}`;
    }

    return candidate;
}
```
