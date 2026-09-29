# Rules index

Entry point for the rules of every Beyonda Labs repo. Read the file that matches the task before writing
anything; if a change touches several of them, all of them apply.

## Which file to read

| What you are about to do                                                         | File                                                 |
| -------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Write or change an Angular component — folder layout, config input, template      | [angular/component.md](angular/component.md)         |
| Write or change a model class — the data holders paired with `<Name>Parameters`   | [angular/class-model.md](angular/class-model.md)     |
| Write or change a service — injectable, state, HTTP calls, data transformation    | [angular/service.md](angular/service.md)             |
| Write stateless logic in an Angular repo — a label key, a size, a tree search     | [angular/function-module.md](angular/function-module.md) |
| Build a page of an app on `bey-page` — folder, page config, services by kind     | [angular/page.md](angular/page.md)                   |
| Write or change a test in an Angular repo                                         | [angular/test.md](angular/test.md)                   |
| Write CSS — classes, variables, dark mode, anything touching Bootstrap            | [angular/styles.md](angular/styles.md)               |
| Add or change a translation                                                       | [angular/i18n.md](angular/i18n.md)                   |
| Work on a publishable component library — exports, packaging, module READMEs      | [angular/library.md](angular/library.md)             |
| Work on a model library — a shared domain package, no framework, front and back   | [model-library/package.md](model-library/package.md) |
| Write or change a class of a model library                                        | [model-library/model.md](model-library/model.md)     |
| Write logic in a model library — validation, hydration, copies                    | [model-library/function-module.md](model-library/function-module.md) |
| Start or wire an Express app — `index.ts`, `app.ts`, environment, configuration   | [express/app.md](express/app.md)                     |
| Add or change a resource of an Express app — folder, module, entity config        | [express/resource.md](express/resource.md)           |
| Write or change a route of an Express repo                                        | [express/router.md](express/router.md)               |
| Write or change a service of an Express repo                                      | [express/service.md](express/service.md)             |
| Read or write the database — a store, a query, a transaction                      | [express/store.md](express/store.md)                 |
| Change the schema or the stored data of a database                                | [express/migration.md](express/migration.md)         |
| Throw an error in an Express repo, or change what the front receives for one      | [express/error.md](express/error.md)                 |
| Write stateless logic in an Express repo                                          | [express/function-module.md](express/function-module.md) |
| Write or change a test in an Express repo                                         | [express/test.md](express/test.md)                   |
| Work on the publishable Express library — exports, packaging, testing entry point | [express/library.md](express/library.md)             |
| Commit or push anything, or write a change-log entry                              | [commits.md](commits.md)                             |

## How to use them

- **The rules describe what the repos should do.** Where the code contradicts a rule, say so instead of copying
  the code; where a rule is wrong, change the rule instead of working around it.
- **A new kind of artefact gets its own file** under the folder of its stack, with the same shape: `## Rules` as
  a bullet list and `## Example` with the shortest code that shows every rule at once.
- **A new stack gets its own folder** next to `angular/`, and a row in the table above.
- **Nothing gets committed just because it follows the rules**: authorisation comes first, and that is in
  [commits.md](commits.md).

## Language

- **Everything written into a repo is in English**: code, identifiers, Markdown, commit messages and change-log
  entries. That includes README files, the docs of a module and any rule written here.
- **Conversation is not a repo artefact.** Talking through a change in Spanish is fine; what lands in a file is
  not.

## What each repo keeps for itself

These rules hold everything that is true in more than one repo. A repo only documents what exists because of
what it is: the token catalogue of a library, the domain of a product. If a document would read the same in
another repo, it belongs here instead.

## Which rule is enforced by what

Most of these rules are not kept by reading them. Where a tool can hold the rule, the tool is the source of
truth and this file only points at it. The tools are configured once in
[base-config](https://github.com/beyonda-labs/base-config) (`@beyonda-labs/base-config`) and every repo takes
them from there through its `beyonda.config.json`.

| Rule                                                    | Enforced by                        |
| ------------------------------------------------------- | ---------------------------------- |
| Selector and class prefix                               | ESLint + stylelint                 |
| Import order, quotes, formatting                        | ESLint + Prettier, via lint-staged |
| `OnPush` on every component                             | ESLint                             |
| Signal inputs, outputs and queries; `inject()`           | ESLint                             |
| Member order of components, directives and services     | ESLint (`eslint.class-order`)      |
| Declaration order; contract-only model files            | ESLint (`eslint.sort-declarations`, `eslint.model-files`) |
| No literal design value outside the token layer         | stylelint                          |
| `--bs-*` never read                                     | stylelint                          |
| State written as `is-*` / `has-*`                       | stylelint                          |
| No `!important`                                         | stylelint                          |
| `.en.json` and `.es.json` in step, key case and depth   | `bey-check-translations`           |
| Minimum coverage                                        | Jest thresholds                    |
| Exact versions, no local link in a commit               | `bey-check-dependencies`           |
| What a `public-api.ts` exports                          | prose — not automatable            |
| What a test asserts and what it does not                | prose                              |
| When a `--bey-<module>-*` variable is worth creating    | prose                              |
