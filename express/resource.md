# Resource rules

Conventions for a resource of an Express app: the routes under one path (`/templates`), with the services,
stores, models and functions behind them.

## Rules

- **One folder per resource** under `src/resources/<resource>/`, named as its path: `<resource>.module.ts`,
  `<resource>.router.ts`, `<resource>-entity-config.ts` when the library manages the entity, `models/` for its
  contracts, `services/`, `stores/` and `functions/` for its stateless logic. Specs sit next to what they test.
- **`<resource>.module.ts` builds the resource**: `create<Resource>Module(dependencies)` creates its stores, its
  services and its router from what it receives, and returns the router `app.ts` mounts. A store only this
  resource uses is created here; one that several resources use is created in `app.ts` and passed in.
- **A resource extends an entity of the library** by mounting its own router before the library's on the same
  path: the module returns `Router().use(resourceRouter, beyCreateBaseEntityModule(...))`, so a custom route such
  as `/blocks` is matched before `/:id`. A router of the library is never changed after it is built.
- **A resource other rows point at answers what uses each row** through the `findUsages` hook of the library
  entity: its service answers `findUsers(ids)`, every user named as `{ id, name, resource, kind? }`, and the
  library gives every row its `usageCount` and `usedBy` and answers `GET /usages`. The resource writes no usages
  route and adds no usage field through `decorateResults`.
- **Dependencies travel as one object**, typed by a `<Thing>Dependencies` interface in `models/`: a module, a
  router, a service or a store takes `{ templateService, definitionStore }`, never loose parameters. Adding one
  does not touch the order of the others, and `max-params` is never the reason to merge two of them.
- **Services and stores by kind**, as [service.md](service.md) and [store.md](store.md) describe:
  `<thing>.service.ts` runs what a route does, `<thing>.store.ts` reads and writes its tables.
- **A resource imports another one's models, never its services, stores or functions**: those arrive as
  dependencies, or move to `src/shared/` (`shared/functions/`, `shared/models/`) when several resources use them.
- **The contracts of a resource** (row types, request and response bodies, dependency interfaces, the permission
  enum, the constants) live in `models/<thing>.model.ts`.
- **Migrations are not part of a resource**: they live in `src/migrations/`, as [migration.md](migration.md)
  explains, because their order crosses resources.
- **Specs**: the router spec calls the real app over HTTP, and every service, store and function module has its
  own, as [test.md](test.md) describes.

## Example

```text
src/
├── app.ts
├── app-config.ts
├── environment.ts
├── index.ts
├── migrations/
│   ├── 001-baseline.ts
│   └── migrations.ts
├── resources/
│   └── templates/
│       ├── functions/template-item-actions.ts
│       ├── models/template.model.ts
│       ├── services/template.service.ts
│       ├── stores/template-definition.store.ts
│       ├── templates-entity-config.ts
│       ├── templates.module.ts
│       └── templates.router.ts
└── shared/functions/document-variables.ts
```

```ts
// templates.module.ts
export function createTemplatesModule(dependencies: TemplatesModuleDependencies): Router {
    const { accessGuards, appConfig, database, templateStore, userStore } = dependencies;
    const definitionStore = createTemplateDefinitionStore(database);
    const templateService = createTemplateService({ definitionStore, templateStore });

    return Router().use(
        createTemplatesRouter({ requirePermission: accessGuards.requirePermission, templateService }),
        beyCreateBaseEntityModule({
            authentication: appConfig.authentication,
            entityConfig: buildTemplatesEntityConfig(),
            filterItemActions: filterTemplateItemActions,
            onDeleted: templates => templateService.deleteDefinitions(templates),
            store: templateStore,
            userStore
        })
    );
}
```
