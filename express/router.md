# Router rules

Conventions for the routes of an Express app or library, on Express 5.

## Rules

- **A router is built by a function**: `create<Resource>Router(dependencies)` returns a `Router` with its routes
  and takes the services it calls in one object. Nothing is created at import time. A router too long for one file
  moves a group of its routes to a file of its own (`base-entity-trash.router.ts`), as
  `register<Thing>Routes(router, dependencies)`: it adds them to the router it is given, and the router calls it
  where their order among its other routes needs them.
- **A handler only wires**: it reads the request, calls one method of a service and writes what it returns. No
  store, no SQL, no loop, no business condition and no body shaped by hand; a handler longer than a few lines is a
  service method waiting to be written.
- **Errors are thrown**, `throw new BeyNotFoundError(...)`, in the handler or anywhere below it. Express 5 hands a
  rejected promise to the error handler, so there is no `try`/`catch`, no `next(error)` and no
  `response.status(4xx).json(...)` of its own: [error.md](error.md) shapes every failure.
- **Every route declares who may call it** with a guard before its handler, from the guards built once and handed
  to the router as dependencies; every guard authenticates too. Where one permission decides, the guard is
  `requirePermission(<Resource>Permission.<Action>)`, from the access guards the app builds with
  `beyCreateAccessGuards(authentication)`, and the permission is the one named after the action (`ConvertToBlock`
  for `/convert-to-block`); a route open to any signed-in user uses `requireAuth()` from the same guards and says
  so. The only routes open without a token are those of the library: the authentication ones and `/health` and
  `/ready` of base-app.
- **Where access is not one permission, the module has its own guard**: `require<Who>()`, a middleware of its own,
  for a role (`requireSuperadmin()`) or for a choice between permissions (`requireGroupReader()`: the managers, or
  the members who may read their own groups). A route the caller must not learn of answers the `404` of the module
  instead of a `403`, as the routes of the superadmin answer `organizations.not-found` to anyone else.
- **The body is parsed, never cast**: `beyParseBody(request, SCHEMA)` validates it against a schema of the library
  and returns it typed, or throws the validation error. Neither `request.body as X` nor
  `request.body['x'] as string` appears.
- **A service validates the body when the check needs data**: when the schema depends on the row, or the answer for
  a missing row or a forbidden action must come before the one for a wrong body, the route hands the service
  `beyReadBody(request)`, which takes anything that is not an object as `{}`, and the service validates it with
  `beyValidate(body, schema)` (`{ isPartial: true }` for an edit) once it has read the row.
- **The lists of the entity routes are parsed by `requireIds` and `requireEntries`** of base-entity, from the body
  `readBody` reads: `{ ids }` is a non-empty array of strings, and `{ items }` a non-empty array of `{ id, type }`
  whose `type` is `item` or `category` (every entry is an `item`, and `type` is not read, where the entity has no
  folders). A missing or malformed list throws the `RequiredFieldError` of `ids` or `items`, and `requireTargetId`
  does the same for a `targetId` that is neither a string nor `null`.
- **The actions a row offers are checked by the service that reads it**: on top of the permission guard of the
  route, a service that works on rows it reads throws `BeyActionUnavailableError` for one whose actions, after
  `filterItemActions`, lack the one asked for, as every route of the library entities does. A route on one item
  whose service does not read the row checks it before its handler, with the guard `beyCreateItemActionGuard`
  builds.
- **Path parameters come typed from the path** (`request.params.id` on `'/:id'`), never cast nor read by index.
  Express infers them from the path only into a middleware generic in its own call, so every guard a route takes
  is one: `<P>(request: Request<P>, response, next) => unknown`, returned by a factory called in the route
  (`requireSuperadmin()`), or a handler typed by the parameters it reads (`RequestHandler<IdParameters>`). A
  factory generic in `P` with a default, or a handler typed `RequestHandler<Record<string, string>>`, turns them
  into `request.params['id']`, where a misspelt name still compiles.
- **A model of the model library is hydrated at the entrance**: the sections, items and variables of a body go
  through `hydrateDocumentSection`, `hydrateDocumentItem` or `hydrateDocumentVariable` before a service sees them.
- **The response body is typed** by an interface in `models/` (`TemplateStatusResponse`), the one the service
  returns. Status codes: `201` with the created resource, `200` with a body, `204` without one; never `200` with an
  empty body.
- **A file goes out as its bytes**, never as base64 inside JSON: the service returns a `Buffer` and the route sends it
  with its `Content-Type`. A file of a user never reaches a shared cache: `Cache-Control: no-store` when the route
  builds it for the request (a PDF), or `private, no-cache` with its checksum as `ETag` when it is a stored file the
  front reads again and again and a user may replace, answering `304` to a matching `If-None-Match`. The headers
  are set once nothing is left to await, so a failure still reaches the error handler as JSON.
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
