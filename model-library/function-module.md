# Function module rules

Conventions for the logic of a model library: validation, hydration, copying, references. Stateless code that
takes a model and returns a value.

## Rules

- **Plain exported functions, never a class.** There is no state to hold, so there is nothing to instantiate or
  inject; a class would only add a `new` to every call.
- **A function returns a new value and never mutates its argument.**
- **The file is named after what its functions do**, without a technical suffix: `item-problems.ts`,
  `document-cloning.ts`, `attachment-reference-status.ts`. Only model files carry one, `.model.ts`.
- **Functions are grouped by the question they answer**: everything that reports problems of an item in one
  module, everything that copies in another. A module that answers two questions is split.
- **Constants go at the top of the file in `SCREAMING_SNAKE_CASE`**, and what only the module uses stays
  unexported.
- **Functions go in alphabetical order**, the exported ones as a first group and the internal ones after them,
  as the methods of a class do. Function declarations are hoisted, so the order never changes what runs.
  `perfectionist/sort-modules` enforces it and fixes it on save.
- **A function takes plain data where it can** (`<Name>Parameters`, `Pick<>` of the fields it reads) rather than
  a class instance, so it works on JSON too.
- **One spec per module**, testing what the functions return for a given model, following
  [angular/test.md](../angular/test.md) wherever it is not about the DOM.
- **No comments in the code**: whatever needs explaining goes in the package README.

## Example

```ts
import { AttachmentReferenceParameters } from './attachment-reference.model';

export function isLegacyReference(reference: AttachmentReferenceParameters): boolean {
    return !isResolvedReference(reference) && Boolean(reference.legacyContent);
}

export function isResolvedReference(reference: AttachmentReferenceParameters): boolean {
    return Boolean(reference.attachmentId);
}
```
