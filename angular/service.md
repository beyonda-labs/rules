# Service rules

Conventions to follow when creating or changing a service.

## Rules

- **One service per responsibility**, named `<Thing>Service` in `<thing>.service.ts`, inside a `services/` folder next to what it serves.
- **`@Injectable({ providedIn: 'root' })` by default**. A bare `@Injectable()` only when the service must hold per-instance state and is listed in a component's `providers`.
- **Dependencies with `inject()`**, in `private readonly` fields at the top of the class. Never through constructor parameters; a service with no state has no constructor at all.
- **Order inside the class**: injected dependencies, private state and attributes, public state and attributes, the constructor when there is one, methods. One blank line between blocks and alphabetical order inside each of them. `eslint.class-order` enforces it and fixes it on save.
- **State lives in signals**: the writable one is private and prefixed with `_`, and what the outside reads is its `asReadonly()` or a `computed`. Nothing outside the service writes state.
- **Methods go in alphabetical order**, the public ones as a first group and the private ones after them.
- **Whatever does not need `this` is a plain function** outside the class, and constants go at the top of the file in `SCREAMING_SNAKE_CASE`.
- **A service that transforms data returns a new value** and never mutates its argument.
- **No callbacks as properties**: what a component needs to trigger is a specific method on the service.
- **An error reaches the user in the error modal, never in a toast**, whether a service or a component reports it. The library's HTTP service opens that modal with the server's reason by default; `handleError` is only for showing a different modal, and a toast only confirms what went right (`successToast`).
- **No comments in the code**: whatever needs explaining goes in the module's README.

## Example

```ts
const MAX_TERMS = 5;
const STORAGE_KEY = "recent-searches";

function isBlank(term: string): boolean {
  return term.trim().length === 0;
}

@Injectable({
  providedIn: "root",
})
export class RecentSearchService {
  private readonly storageService = inject(StorageService);

  private readonly _terms = signal<string[]>(
    this.storageService.get<string[]>(STORAGE_KEY) ?? [],
  );

  readonly hasTerms = computed(() => this._terms().length > 0);
  readonly terms = this._terms.asReadonly();

  add(term: string): void {
    if (isBlank(term)) {
      return;
    }

    this._terms.update((terms) =>
      [term, ...terms.filter((current) => current !== term)].slice(
        0,
        MAX_TERMS,
      ),
    );
    this.persist();
  }

  clear(): void {
    this._terms.set([]);
    this.persist();
  }

  private persist(): void {
    this.storageService.set(STORAGE_KEY, this._terms());
  }
}
```
