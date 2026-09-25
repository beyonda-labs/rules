# Rules index

Entry point for the rules of every Beyonda Labs repo. Read the file that matches the task before writing
anything; if a change touches several of them, all of them apply.

## Which file to read

| What you are about to do                                                         | File                                                 |
| -------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Write or change an Angular component — folder layout, config input, template      | [angular/component.md](angular/component.md)         |
| Write or change a model class — the data holders paired with `<Name>Parameters`   | [angular/class-model.md](angular/class-model.md)     |
| Write or change a service — injectable, state, HTTP calls, data transformation    | [angular/service.md](angular/service.md)             |
| Write or change a test                                                            | [angular/test.md](angular/test.md)                   |
| Write CSS — classes, variables, dark mode, anything touching Bootstrap            | [angular/styles.md](angular/styles.md)               |
| Add or change a translation                                                       | [angular/i18n.md](angular/i18n.md)                   |
| Work on a publishable component library — exports, packaging, module READMEs      | [angular/library.md](angular/library.md)             |
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
truth and this file only points at it.

| Rule                                                    | Enforced by                        |
| ------------------------------------------------------- | ---------------------------------- |
| Selector and class prefix                               | ESLint + stylelint                 |
| Import order, quotes, formatting                        | ESLint + Prettier, via lint-staged |
| `OnPush` on every component                             | ESLint                             |
| No literal design value outside the token layer         | stylelint                          |
| `--bs-*` never read                                     | stylelint                          |
| State written as `is-*` / `has-*`                       | stylelint                          |
| No `!important`                                         | stylelint                          |
| `.en.json` and `.es.json` in step, key case and depth   | `check-translations.js`            |
| Minimum coverage                                        | Jest thresholds                    |
| What a `public-api.ts` exports                          | prose — not automatable            |
| What a test asserts and what it does not                | prose                              |
| When a `--bey-<module>-*` variable is worth creating    | prose                              |
