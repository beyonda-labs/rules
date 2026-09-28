# Function module rules

Conventions for the stateless logic of an Angular app or library: resolving a label key, formatting a size,
searching a tree, building the cells of a row from a model.

## Rules

- **Plain exported functions, never a class**, when the logic holds no state and injects nothing. Logic that needs
  `inject()` belongs in a service.
- **The file is named after what its functions do, without a technical suffix**: `tree-node-search.ts`,
  `file-size.ts`, `form-rule-resolution.ts`. No `.util.ts` nor `.helper.ts`; only the Angular artefacts keep
  theirs (`.component.ts`, `.service.ts`, `.model.ts`…).
- **It lives next to what it works on**: in the `models/` folder of the module when it works on its models, in the
  module's folder when it serves the component. What several modules of a library share goes in
  `internal/<topic>/`.
- **Types and constants it needs go in a model file**, never in the function module: contracts and definitions
  stay in `*.model.ts`.
- **Functions are grouped by the question they answer**. A module that answers two questions is split.
- **Constants go at the top of the file in `SCREAMING_SNAKE_CASE`**, and what only the module uses stays unexported.
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

export function collectExpandableKeys<TData>(nodes: TreeNode<TData>[]): string[] {
    return nodes.flatMap(node =>
        node.children.length > 0 ? [node.key, ...collectExpandableKeys(node.children)] : []
    );
}

export function findNodeByKey<TData>(nodes: TreeNode<TData>[], key?: string): TreeNode<TData> | undefined {
    for (const node of nodes) {
        const found = node.key === key ? node : findNodeByKey(node.children, key);

        if (found) {
            return found;
        }
    }

    return undefined;
}
```
