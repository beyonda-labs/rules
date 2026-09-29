# Component rules

Conventions to follow when creating or changing a component.

## Rules

- **One folder per module**, named after it, holding `<name>.component.ts`, `.html`, `.css` and `.spec.ts`, plus `models/`, `services/`, `components/` for its children, `assets/` for its translations and `docs/` for its README. Its demo lives in the style-guide entry point, under `style-guide/src/<module>/`, since a secondary entry point can only compile sources inside its own folder.
- **Standalone components**, with `imports` listing only what the template uses, `selector` prefixed with the repo default prefix, and `templateUrl`/`styleUrls` always in separate files. Never an inline template.
- **`ChangeDetectionStrategy.OnPush` always.** There is no component that opts out.
- **A child component that only makes sense inside the module lives in its `components/`** and is not exported.
- **The component is driven by a config model**, taken as `config = input.required<Config>()`. Anything the consumer must react to is a callback on that config, not an output.
- **The config is immutable**: to change it the consumer replaces the instance. A component never writes into the config it receives, and never relies on the consumer mutating it in place.
- **Outputs are for the module's own children**: `output()` to let a child tell its parent inside the module.
- **A component wrapping an event source is the exception**: where the thing being wrapped emits a stream of
  its own — a viewer, an editor, a map — a curated set of `output()` is the honest API, and folding those
  events into config callbacks only hides where they come from.
- **A helper that is not part of the public surface may take plain inputs**: an internal wrapper whose whole
  API is one or two primitives does not get a config class. The config model is there to keep a published
  API stable, not to add ceremony to a layout helper.
- **Derived state is a `computed()`**, local state a `signal()`. Never a setter input with a `_`-prefixed field, never an injected `ChangeDetectorRef`.
- **A method instead of a `computed()` only when the value reads something that is not a signal** — form control validity, a DOM measurement — since a `computed()` would not recompute.
- **Dependencies are injected with `inject()`**, never through constructor parameters.
- **Subscriptions are cleaned up with `takeUntilDestroyed()`**, in the field initialiser or the constructor. No `destroy$` subject, no `ngOnDestroy` written just to unsubscribe.
- **Order inside the class**: injected dependencies, inputs, outputs, public signals and computed, private state, constructor, lifecycle hooks in the order Angular runs them, methods. One blank line between blocks. `eslint.class-order` and `sort-lifecycle-methods` enforce it; the first fixes it on save.
- **Methods go in alphabetical order**, the public ones as a first group and the private ones after them.
- **The template holds no logic**: it reads signals and computed, calls methods for the rest, and guards its content with `@if` when the data may not be there yet.
- **Nothing is computed twice in the template**: what the class can resolve, it resolves.
- **Attributes follow a fixed order**: structural directive, `#ref`, `id`, `class`, other static attributes, inputs (`[x]`, `[attr.*]`, `[class.*]`), two-way bindings, outputs; alphabetical within each group. Prettier enforces it with `prettier-plugin-organize-attributes`, so it is never ordered by hand.
- **Labels and placeholders are translation keys**, resolved with the `translate` pipe and defaulted from the config prefix when the consumer does not give one.
- **Styles live in the component's own `.css`**, and what several children share goes in one file next to them, imported through `styleUrls`.
- **No comments in the code**: whatever needs explaining goes in the module's README.

## Example

```ts
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [ReactiveFormsModule, TranslateModule],
  selector: 'bey-form-text-field',
  standalone: true,
  styleUrls: ['../field-control.styles.css'],
  templateUrl: './field-text.component.html'
})
export class FormTextFieldComponent {
  private readonly formService = inject(FormService);

  readonly field = input.required<FormTextField>();

  readonly control = computed(() => this.formService.getFieldControl(this.field()));
  readonly hint = signal('');
  readonly placeholder = computed(() => this.field().placeholder ?? `${this.field().key}.placeholder`);

  constructor() {
    toObservable(this.control)
      .pipe(
        switchMap(control => control?.valueChanges ?? EMPTY),
        takeUntilDestroyed()
      )
      .subscribe(value => this.hint.set(value ? '' : `${this.field().key}.hint`));
  }

  isInvalid(): boolean {
    const control = this.control();

    return (control?.invalid && control?.touched) ?? false;
  }
}
```

```html
@if (control(); as control) {
  <input
    class="form-control form-control-sm"
    type="text"
    [id]="field().key"
    [formControl]="control"
    placeholder="{{ placeholder() | translate }}"
  />

  @if (hint()) {
    <small class="form-text">{{ hint() | translate }}</small>
  }
}
```
