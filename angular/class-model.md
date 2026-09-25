# Model class rules

Conventions to follow when creating or changing a model class.

## Rules

- **One pair per entity**: the class and a `<Name>Parameters` interface with the shape its constructor accepts.
- **A single constructor argument**, destructured and typed with that interface. Never loose parameters. When it
  expects no parameter, it does not default to an empty object.
- **Default values live in the constructor's destructuring**, not in the interface nor in the field
  declaration: `separator = '/'`, `translate = true`.
- **The interface states what is optional**; the class declares as required (no `?`) everything that has a
  default, since after construction it always holds a value. Only what may stay undefined keeps the `?` in the
  class (`icon`, `onItemClick`).
- **Order inside the class**: fields without `?` first, alphabetically, a blank line, and the optional ones
  after it, alphabetically too. Callbacks belong to that same block; there is only one blank line.
- **Order inside the interface**: required first, a blank line, then optional, each group alphabetical.
- **Order of the destructuring alphabetical**, whether a field is optional or carries a default.
- **Constructor assignments go in alphabetical order** too.
- **Booleans are prefixed with `is`** (`isDisabled`, `isTranslationKey`).
- **Callbacks are prefixed with `on`**, optional and typed inline: `onItemClick?: (id: number) => void`.
- **A model is immutable once built**: whoever holds it replaces the instance to change it, and nothing
  writes into its fields afterwards. Components rely on this, so a mutated model does not reach the view.
- **No behaviour in a model**: it holds data and callbacks. Anything that computes or fetches belongs in a
  service.
- **No comments in the code**: whatever needs explaining goes in the module's README.

## Example

```ts
export class ChipItem {
  isDisabled: boolean;
  label: string;

  hint?: string;
  onSelect?: () => void;

  constructor({
    hint,
    isDisabled = false,
    label,
    onSelect,
  }: ChipItemParameters) {
    this.hint = hint;
    this.isDisabled = isDisabled;
    this.label = label;
    this.onSelect = onSelect;
  }
}

export interface ChipItemParameters {
  label: string;

  hint?: string;
  isDisabled?: boolean;
  onSelect?: () => void;
}
```
