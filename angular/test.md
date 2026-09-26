# Test rules

Conventions to follow when writing tests.

## Rules

- **A test states a behaviour, not a detail.** What is asserted is what the consumer configures and what the consumer observes: rendered text, invoked callbacks, emitted values, resulting state.
- **Never assert on presentation**: CSS class names, node counts, element order, inline styles or the shape of the markup. They change with every redesign and prove nothing about the component working.
- **Query by role, `aria-*` or visible text**, never by CSS class. A query by class ties the suite to the stylesheet.
- **Locate only what the test acts on or reads**: the button it clicks, the input it types into, the text it checks. A container is never located to check that it exists; what it contains is checked instead.
- **No `data-testid`.** An element a test needs to reach is one a user reaches too, so when it has no role or accessible name, the template gets the missing `role` or `aria-label`, not a test hook.
- **An interaction is a real event on the element** (`click()`, a `KeyboardEvent`), never a call to the handler. The click proves the template wires the element to the behaviour; complex logic behind it belongs in a model or a service with its own spec.
- **Attributes are asserted only when they are the contract**: `aria-selected`, `aria-current`, `disabled`. Not the ones that merely happen to be there.
- **A variant or state that only changes a class has no test.** If the consumer cannot observe it through text, an ARIA attribute or a callback, either give the template the ARIA attribute that names the state (`aria-expanded`, `aria-current`) and assert that, or leave it untested.
- **One `it()` per behaviour**, not per assertion. Several `expect` in the same `it()` are right when they describe the same behaviour.
- **The name of an `it()` says what happens**, in the third person and without `should`: `renders the items from the config`, `calls onItemClick with the clicked id`.
- **Setup lives in `beforeEach` and in a `build<Thing>()` helper** that returns a config with sensible defaults and takes overrides. An `it()` starts from a ready fixture and never repeats the wiring.
- **Generic helpers are shared, not rewritten.** Rendering (`renderComponent`, `settle`) and generic queries (`queryAll`, `textsOf`, `buttonByName`, `queryButton`) come from the repo's `testing/` module, outside `src/` so they never ship. A spec keeps locally only what belongs to its component: its `build<Thing>()` and the queries that depend on its structure.
- **No per-spec browser mocks** for what `setup-jest.ts` already provides, such as `ResizeObserver`.
- **Private methods and internal state are not tested**: they are reached through the behaviour that uses them.
- **One spec per unit that has behaviour.** A component that only renders its config has one rendering test, not twenty.
- **A fixed bug gets the test that would have caught it**, written as the behaviour that was wrong.
- **An interaction is followed by the same pair**: `ngModel` and friends write the view asynchronously, so
  asserting on the DOM right after a click or an input event reads the state before the change.
- **A render is `detectChanges()` then `await whenStable()`**: with zone-based change detection the first
  pass does not run on its own, and the await covers whatever the component resolves asynchronously.
- **No comments in the test**: the name of the `it()` explains it or the test is badly named.

## Example

```ts
describe('TabsComponent', () => {
  let fixture: ComponentFixture<TabsComponent>;

  function buildConfig(overrides: Partial<TabsConfigParameters> = {}): TabsConfig {
    return new TabsConfig({
      tabs: [
        new Tab({ key: 'general', label: 'General' }),
        new Tab({ key: 'advanced', label: 'Advanced' })
      ],
      ...overrides
    });
  }

  async function render(config: TabsConfig = buildConfig()): Promise<void> {
    fixture = TestBed.createComponent(TabsComponent);
    fixture.componentRef.setInput('config', config);
    fixture.detectChanges();
    await fixture.whenStable();
  }

  function tabs(): HTMLElement[] {
    return Array.from(fixture.nativeElement.querySelectorAll('[role="tab"]'));
  }

  function tab(name: string): HTMLElement {
    const found = tabs().find(element => element.textContent?.trim() === name);

    if (!found) {
      throw new Error(`No tab named ${name}`);
    }

    return found;
  }

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TabsComponent, TranslateModule.forRoot()]
    }).compileComponents();
  });

  it('renders one tab per entry in the config', async () => {
    await render();

    expect(tabs().map(element => element.textContent?.trim())).toEqual(['General', 'Advanced']);
  });

  it('selects the first tab until another one is clicked', async () => {
    await render();

    expect(tab('General').getAttribute('aria-selected')).toBe('true');

    tab('Advanced').click();
    fixture.detectChanges();
    await fixture.whenStable();

    expect(tab('Advanced').getAttribute('aria-selected')).toBe('true');
    expect(tab('General').getAttribute('aria-selected')).toBe('false');
  });

  it('does not activate a disabled tab', async () => {
    const onTabChange = jest.fn();
    await render(
      buildConfig({ tabs: [new Tab({ key: 'general', label: 'General', isDisabled: true })], onTabChange })
    );

    tab('General').click();
    fixture.detectChanges();
    await fixture.whenStable();

    expect(onTabChange).not.toHaveBeenCalled();
  });
});
```
