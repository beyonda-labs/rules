# Error rules

Conventions for the failures an Express app or library reports, and for the body the front receives.

## Rules

- **An expected failure is an error of the library, thrown**: `BeyBadRequestError`, `BeyUnauthorizedError`,
  `BeyForbiddenError`, `BeyNotFoundError`, `BeyConflictError` and the field validation errors, all of them
  `BeyAppError`. A plain `Error` means a bug, and it is answered with a `500`.
- **The error handler writes every error body**: `{ errorCode, messageKey, messageParameters, details, timestamp }`,
  leaving out the fields without a value. Nothing else answers a `4xx` or a `5xx`.
- **`errorCode` names the kind of failure** and comes from the class (`bad-request`, `not-found`,
  `invalid-field-range`). A kind of failure several products can meet gets its class in the library; a reason that
  belongs to one domain is a `messageKey` on an existing class.
- **`messageKey` says why, in the words of the domain**: `<domain>.<reason>` in kebab case
  (`template.invalid-transition`). It is required on `BeyBadRequestError` and `BeyConflictError`, whose generic text
  tells the user nothing; the classes whose code and parameters already say it all (`not-found` with its
  `resource`, the field errors) may go without.
- **The front translates it**: every `messageKey` the back throws exists in the `en.json` and `es.json` of the
  front under `angular-components.http.error.`, and changing one changes both repos together.
- **The values its text needs travel in `messageParameters`** (`{ usageCount: 3 }`), never inside the key.
- **Structured data the front shows goes in `details`**: the problems of a document that does not validate, the
  items that block a delete.
- **The first argument of an error is for the log**, in English and with the ids that help debugging
  (`` `Template ${id} cannot move to ${status}` ``). It never reaches the user.
- **`BeyNotFoundError` names the resource by its path** (`templates`, `template-definitions`), the same string
  everywhere.
- **An unexpected error is logged with its stack** and answered as `internal-server-error`, with nothing of its
  message in the body.
- **No comments in the code**.

## Example

```ts
throw new BeyConflictError(`Block ${id} is used by ${templateIds.length} templates`, {
    messageKey: 'template.block-in-use',
    messageParameters: { usageCount: templateIds.length },
    details: { templateIds }
});
```

```json
{
    "errorCode": "conflict",
    "messageKey": "template.block-in-use",
    "messageParameters": { "usageCount": 2 },
    "details": { "templateIds": ["3f2a…", "9c41…"] },
    "timestamp": "2026-09-29T18:00:00.000Z"
}
```
