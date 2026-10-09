# Commit rules

Rules every commit follows, in every repo.

## Authorisation

- **Never commit without express authorisation.** The work is left ready and permission is asked for, even when the commit meets every rule below. The same goes for the `push`.
- **Reading is free**: listing commits, checking the state of a repo or opening a file needs no permission.

## Branches

- **Never commit on `main`.** `develop` is the base for everything else.
- **`feature/<name>` and `fix/<name>`** branch off `develop` and merge back into it; **`release/x.y.z`** merges into `main` and `develop`; **`hotfix/<name>`** branches off `main`.
- **No pull requests**: the merge is done directly.
- **A change that spans the front and the service of a product lands front first.** The feature branch of the front is pushed, so Jenkins publishes its `<version>-feature-<name>.<build>.<sha>` snapshot, and the feature branch of the service pins it, so the service is tried with the front it will serve. Before the service goes to `develop`, the front merges into `develop`, its remote branch is deleted and the service pins that `develop` snapshot; only then does the service merge. `develop` of the service never serves a front that exists only on a feature branch.

## Message

- **Conventional Commits**: `<type>(<scope>): <description>`.
- **Allowed types**: `feat`, `fix`, `docs`, `refactor`, `chore`, `test`, `style`, `perf`, `ci`.
- **A single line**, describing the change in general terms. No body, no long description.
- **No deep detail**: what was touched point by point does not belong in the message.
- **Never a `Co-Authored-By`**, nor any other trace of the code having been written by an assistant.

## Content

- **Group in one commit** what affects the same module or belongs together; split what does not.
- **Lint and tests green** before committing.
- **No dependency points to a local folder** (`link:`, `file:`, `workspace:`) in a commit. A link is for trying an unpublished change (`install:local`); before the commit, the consumer goes back to the published version (`install:remote`), or waits for the library change to be published and pins that snapshot. `check-dependencies` fails `lint`, the pre-commit hook and Jenkins when one reaches a commit.
- **No commented-out code and no comments in the code**: whatever needs explaining goes in the module's README.
- **The detail goes in the repo's `docs/change-log.md`**, never in a `CHANGELOG.md` at the root: one line per change, two at most, grouped by module.
- **A bug born and fixed within the same piece of work is not mentioned** in the change log; only what changes for whoever uses the code.
