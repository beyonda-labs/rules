# Service rules

Conventions for a service of an Express app or library: the code that does what a route asks, between the router
and the stores.

## Rules

- **A service is a factory**: `create<Thing>Service(dependencies)` returns an object whose methods do the work,
  typed by a `<Thing>Service` interface, and `<Thing>ServiceDependencies` types what it receives. Both interfaces
  are contracts and live in `models/`. No class, no `new`, no `this`.
- **Everything it uses arrives as a dependency**: stores, other services, the logger, the clock when a spec needs
  to fix it. It never opens a connection, reads `process.env` nor imports a store to call it, so a spec builds it
  with whatever it needs.
- **No module-level state**: no `let` at the top of the file and no cache shared between instances. What must be
  remembered goes in a store.
- **One service per responsibility, named after it**: `template.service.ts` for what the template routes do,
  `template-duplication.service.ts` once that grows on its own. A store is not a service; see
  [store.md](store.md).
- **It knows nothing of HTTP**: it takes plain values (an id, a parsed body, the write options the route builds
  with `beyReadWriteOptions(request)`), returns a plain value and throws an error of the library when the request
  cannot be fulfilled. Never a `Request`, a `Response` nor a status code.
- **It stamps what the user changes**: a method that changes a row for the user takes the write options
  (`{ userId }`) and passes them last to the write of the store, which stamps `updatedAt` and `updatedBy` in that
  same write. A change kept in another table marks the row it belongs to with `update(id, {}, options)`, in the
  same transaction. A write of its own bookkeeping, such as linking a copy to its definition, goes without them.
- **Writes through several stores are one transaction**: `store.transaction(work)`, which every store on the same
  connection joins. Inside it the service awaits only those stores: a file, a hash or a network call goes before
  or after it.
- **It never writes text for a user**: what it produces for the screen is a translation key with its parameters,
  resolved by the front. A default the user sees, such as the name of a copy, comes from the front.
- **Many rows are read with one call**, `findByIds` or `findAll`: never a `findById` inside a loop, and never a page
  of the maximum size to mean "all".
- **Whatever does not need the dependencies is a plain function**: of the same file, below the factory, when only
  this service uses it; in `functions/` when another file uses it or it deserves its own spec. The factory stays a
  list of its methods.
- **No comments in the code**.

## Example

```ts
export function createTemplateService(dependencies: TemplateServiceDependencies): TemplateService {
    const { definitionStore, templateStore } = dependencies;

    return {
        async convertToBlock(id, blockType, options) {
            const template = await findTemplate(templateStore, id);

            if (template.blockType !== TemplateBlockType.Template) {
                throw new BeyBadRequestError(`Template ${id} is already a ${template.blockType}`, {
                    messageKey: 'template.already-converted'
                });
            }

            await templateStore.update(id, { blockType }, options);

            return { blockType };
        },

        async deleteDefinitions(templates) {
            await definitionStore.deleteForTemplates(templates.map(({ id }) => id));
        },

        async saveDefinition(id, content, options) {
            const { templateDefinition } = await findTemplate(templateStore, id);

            await templateStore.transaction(async () => {
                await definitionStore.updateContent(templateDefinition, content);
                await templateStore.update(id, {}, options);
            });
        }
    };
}

async function findTemplate(templateStore: BeyEntityStore<Template>, id: string): Promise<Template> {
    const template = await templateStore.findById(id);

    if (!template) {
        throw new BeyNotFoundError('templates');
    }

    return template;
}
```
