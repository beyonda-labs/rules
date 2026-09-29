# Router rules

Conventions for the routes of an Express app or library, on Express 5.

## Rules

- **A router is built by a function**: `create<Resource>Router(dependencies)` returns a `Router` with its routes
  and takes the services it calls in one object. Nothing is created at import time.
- **A handler only wires**: it reads the request, calls one method of a service and writes what it returns. No
  store, no SQL, no loop, no business condition and no body shaped by hand; a handler longer than a few lines is a
  service method waiting to be written.
- **Errors are thrown**, `throw new BeyNotFoundError(...)`, in the handler or anywhere below it. Express 5 hands a
  rejected promise to the error handler, so there is no `try`/`catch`, no `next(error)` and no
  `response.status(4xx).json(...)` of its own: [error.md](error.md) shapes every failure.
- **Every route declares its permission** with `requirePermission(<Resource>Permission.<Action>)`, one of the access
  guards the app builds once with `beyCreateAccessGuards(authentication)` and hands to every router as a
  dependency; it authenticates too. The permission is the one named after the action (`ConvertToBlock` for
  `/convert-to-block`); a route open to any signed-in user uses `requireAuth()` from the same guards and says so.
- **The body is parsed, never cast**: `beyParseBody(request, SCHEMA)` validates it against a schema of the library
  and returns it typed, or throws the validation error. Neither `request.body as X` nor
  `request.body['x'] as string` appears.
- **Path parameters come typed from the path** (`request.params.id` on `'/:id'`), without `as string`.
- **A model of the model library is hydrated at the entrance**: the sections, items and variables of a body go
  through `hydrateDocumentSection`, `hydrateDocumentItem` or `hydrateDocumentVariable` before a service sees them.
- **The response body is typed** by an interface in `models/` (`TemplateStatusResponse`), the one the service
  returns. Status codes: `201` with the created resource, `200` with a body, `204` without one; never `200` with an
  empty body.
- **A list that `bey-page` shows answers a `BeyPageResponse`**, as the entity routes of the library do; any other
  list is returned whole, read with `findAll`.
- **Routes go from the most specific to the least** (`/blocks` before `/:id`), and a router that extends one of the
  library is mounted before it.
- **No comments in the code**.

## Example

```ts
export function createTemplatesRouter({ requirePermission, templateService }: TemplatesRouterDependencies): Router {
    const router = Router();

    router.get('/blocks', requirePermission(TemplatePermission.ReadDefinition), async (_request, response) => {
        response.json(await templateService.findBlocks());
    });

    router.post('/:id/status', requirePermission(TemplatePermission.ChangeStatus), async (request, response) => {
        const { status } = beyParseBody(request, CHANGE_STATUS_SCHEMA);

        response.json(await templateService.changeStatus(request.params.id, status));
    });

    router.post('/:id/duplicate', requirePermission(TemplatePermission.Duplicate), async (request, response) => {
        const { name } = beyParseBody(request, DUPLICATE_SCHEMA);

        response.status(201).json(await templateService.duplicate(request.params.id, name));
    });

    return router;
}
```
