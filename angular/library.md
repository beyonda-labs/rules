# Library rules

Conventions that apply to a publishable component library (`ui-components`), on top of the rules for components,
models, services, styles, translations and tests.

## Rules

- **Every module has its own `public-api.ts`**, and the root `public-api.ts` only re-exports those files. Nothing is imported from a deep path by a consumer.
- **Everything public is renamed with the library prefix** on the way out: `BeyTabsComponent`, `BeyTabsConfig`, `beyAuthGuard`, `provideBeyToast`. Without exception, so a consumer never has to alias.
- **Only what a consumer needs to build a config is exported**: the entry component, its models, its services, its providers and its guards.
- **Never exported**: children under `components/`, anything under `internal/`, and every `docs/` and `style-guide` artefact. A plain function a consumer needs to build the same keys or texts as the library lives in `src/lib/utilities/`, which is public.
- **`<Name>Parameters` interfaces are exported** as `export type`, since a consumer that builds a config from data needs to type it.
- **Modules are declared by layer** in the root `public-api.ts`, in this order and under a comment naming each one: `primitives` (self-contained UI: badge, tabs, breadcrumb), `composites` (coordinate other modules: form, table, app-layout), `product` (tied to one product, stable only by agreement), then `services` (providers, guards and interceptors under `src/lib/services/`) and `utilities` (plain functions under `src/lib/utilities/`).
- **A module is reachable from a style-guide, never the style-guide from the public API.** The demo is a secondary entry point, `<package>/style-guide`, whose sources live under `style-guide/src/` and import the library by its package name, exactly as a consumer does; it is never pulled in by importing a component.
- **Test support is the `<package>/testing` entry point**, with sources under `testing/src/`: the DOM helpers, `provide<Prefix>Testing` and a fake per service a consumer would otherwise mock. It imports the library by its package name, nothing in the library imports it, and its code does not depend on Jest, so fakes record calls in signals and a spec may still spy on them.
- **`sideEffects` lists only the CSS files.** Anything else there stops consumers from tree-shaking the library.
- **Only runtime assets are packaged**: translations, images and the style-guide translations the `<package>/style-guide` entry point loads at runtime. Never `docs/`.
- **Every module has a `docs/<module>-readme.md`** with a fixed shape: one paragraph on what it does, a table of the config's fields, and a minimal usage example. Nothing that the types already say, no changelog, no roadmap.
- **A breaking change to an exported name is a major**, and the old name is not kept as an alias.

## Example

```ts
// src/lib/components/tabs/public-api.ts
export { TabsComponent as BeyTabsComponent } from './tabs.component';
export { Tab as BeyTab, TabsConfig as BeyTabsConfig } from './models/tabs.model';
export type { TabParameters as BeyTabParameters, TabsConfigParameters as BeyTabsConfigParameters } from './models/tabs.model';
```

```ts
// src/public-api.ts
/* primitives */
export * from './lib/components/badge/public-api';
export * from './lib/components/breadcrumb/public-api';
export * from './lib/components/tabs/public-api';

/* composites */
export * from './lib/components/app-layout/public-api';
export * from './lib/components/form/public-api';
export * from './lib/components/table/public-api';

/* product */
export * from './lib/components/page/public-api';
export * from './lib/components/properties-menu/public-api';

/* services */
export * from './lib/services/session/public-api';

/* utilities */
export * from './lib/utilities/public-api';
```

```json
{
    "sideEffects": ["**/*.css"]
}
```
