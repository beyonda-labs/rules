# Function module rules

Conventions for the stateless logic of an Angular app or library: resolving a label key, formatting a size,
searching a tree, reordering the items of a document.

## Rules

- **Plain exported functions, never a class**, when the logic holds no state and injects nothing. Logic that needs
  `inject()` belongs in a service, and so do the form and table builders of a page, which are `-form` and `-table`
  services (see [service.md](service.md)).
- **The file is named after what its functions do, without a technical suffix**: `tree-node-search.ts`,
  `file-size.ts`, `form-rule-resolution.ts`. No `.util.ts` nor `.helper.ts`; only the Angular artefacts keep
  theirs (`.component.ts`, `.service.ts`, `.model.ts`…).
- **It lives in a `functions/` folder at the level of what uses it**: `<module>/functions/` in a library module or
  a page, `<component>/functions/` when a single component of a module uses it, `shared/functions/` when several
  pages of an app use it, and `internal/<topic>/` when several modules of a library do. `models/` holds contracts
  only and `services/` only injectables. The only function modules outside `functions/` are the config builders a
  screen starts from, such as `<page>-page-config.ts`, next to its component.
- **What another file imports is a contract and lives in a model file**: a type, an enum or a constant that holds
  data (a table, a pattern, a limit) goes in `*.model.ts` once a second file reads it, a spec included. A constant
  whose values are functions, such as a dispatch table, is logic: it stays in the function module and is exported
  only when another file calls it.
- **What only the module reads stays at its top, unexported**: its constants in `SCREAMING_SNAKE_CASE` and the
  types only its functions use.
- **A function module holds no state**: no `let` at the top of the file and no cache. The one exception is the
  counter behind unique DOM ids, written once per repo (`internal/dom-ids/` in a library, `shared/functions/` in an
  app) and used by every component that needs an id, never copied into a module or a component of its own.
- **Functions are grouped by the question they answer**. A module that answers two questions is split.
- **Exported functions first, alphabetical, then the internal ones**, alphabetical too. `perfectionist/sort-modules`
  enforces it and fixes it on save.
- **A function returns a new value and never mutates its argument.**
- **A function takes plain data where it can** (`<Name>Parameters`, a `Pick<>` of the fields it reads) rather than
  a class instance.
- **One spec per module**, testing what the functions return, following [test.md](test.md) wherever it is not
  about the DOM.
- **No comments in the code**: whatever needs explaining goes in the module's README.

## Example

```ts
import { TreeNode } from './tree.model';

const ROOT_LEVEL = 0;

export function collectExpandableKeys<TData>(nodes: TreeNode<TData>[]): string[] {
    return nodes.flatMap(node =>
        node.children.length > 0 ? [node.key, ...collectExpandableKeys(node.children)] : []
    );
}

export function findNodeLevel<TData>(nodes: TreeNode<TData>[], key: string): number | undefined {
    return findLevel(nodes, key, ROOT_LEVEL);
}

function findLevel<TData>(nodes: TreeNode<TData>[], key: string, level: number): number | undefined {
    for (const node of nodes) {
        const found = node.key === key ? level : findLevel(node.children, key, level + 1);

        if (found !== undefined) {
            return found;
        }
    }

    return undefined;
}
```
