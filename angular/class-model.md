# Model class rules

Conventions to follow when creating or changing a model class.

## Rules

- **One pair per entity**: the class and a `<Name>Parameters` interface with the shape its constructor accepts.
- **A single constructor argument**, destructured and typed with that interface. Never loose parameters. When it
  expects no parameter, it does not default to an empty object.
- **A config whose parameters are all optional defaults to `= {}`**:
  `constructor({ path = '/audit', retentionDays = 365 }: AuditConfigParameters = {})`, so `new AuditConfig()` builds
  the defaults and no caller writes `new AuditConfig({})`.
- **Default values live in the constructor's destructuring**, not in the interface nor in the field
  declaration: `separator = '/'`, `isDisabled = false`.
- **The interface states what is optional**; the class declares as required (no `?`) everything that has a
  default, since after construction it always holds a value. Only what may stay undefined keeps the `?` in the
  class (`icon`, `onItemClick`).
- **Order inside the class**: fields without `?` first, alphabetically, a blank line, and the optional ones
  after it, alphabetically too. Callbacks belong to that same block; there is only one blank line.
- **Order inside the interface**: required first, a blank line, then optional, each group alphabetical.
- **Order of the destructuring alphabetical**, whether a field is optional or carries a default.
- **Constructor assignments go in alphabetical order** too.
- **A boolean flag says which kind of yes it holds**: `is*` or `has*` for a state (`isDisabled`, `hasChildren`),
  `can*` for what the caller may do (`canRelease`) and `include*` for an option that widens what a query returns
  (`includeTrash`). It applies to every field of a model file in every stack: a class, its `Parameters`, an
  options object, a request or response body. A value that happens to be a boolean, such as the default of a
  boolean variable, is not a flag.
- **A boolean handed unchanged to a driver keeps the driver's name**, with no prefix (`rejectUnauthorized` of the
  SSL options of MySQL), so the options read as the driver documents them. One the code reads, or renames on the way
  (`isSecure`, handed to nodemailer as `secure`), takes the prefix of its kind.
- **An event callback is prefixed with `on`** and typed inline: `onItemClick?: (id: number) => void`. It tells the
  consumer that something happened, returns nothing and is optional unless the component cannot work without it.
- **A callback the component calls for a value is named after what it returns**, as a function would be:
  `loadRow`, `toItem`, `getItemKey`, and a predicate with a boolean prefix (`isColumnShown`). It is required when
  the component cannot do without it.
- **What an action model runs when the user triggers it is its `action`** (`action: () => void`): a header or menu
  entry, a notice action, a button, a page action. It is required when running it is all the model is for.
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

    constructor({ hint, isDisabled = false, label, onSelect }: ChipItemParameters) {
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
