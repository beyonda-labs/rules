# Page rules

Conventions for a page of an app built on the library's `bey-page`: a list of entities with its header actions,
table, forms, categories and trash.

## Rules

- **One folder per page** under `src/app/pages/<page>/`: the component, `<page>-page-config.ts`, `models/` for its
  contracts, `services/` for its injectables, `functions/` for its stateless logic and `assets/` for its
  translations. A page reached from another one lives inside it, in
  `pages/<parent>/pages/<page>/`.
- **The component only wires**: it injects its services, builds the app layout `config` and its page config
  through `build<Page>PageConfig(options)`, keeps the `BeyPageHandle` that `onReady` delivers, and hands every
  callback to a service. No private method, no request, no form and no cell of its own.
- **`<page>-page-config.ts` is a plain function** that takes everything the page needs as options (form configs,
  row loaders, categories config, confirmations, action callbacks) and returns the `BeyPageConfig`. Standard
  actions come from `beyPageStandardAction` and `beyPageAddAction`; a custom action is a `BeyPageAction` whose
  `handler` calls one of the options. The constants the config needs, such as the options of a search field, sit
  at the top of the file.
- **Services by kind**, as [service.md](service.md) names them: `<entity>-http.service.ts` sends the requests,
  `<entity>-form.service.ts` builds the forms, `<entity>-table.service.ts` builds the rows (`loadRow`,
  `loadCategoryRow`) and `<entity>-actions.service.ts` runs what the custom actions do. A service that holds state
  for the page, such as the choices a form offers or a file waiting to be uploaded, keeps a plain name.
- **A custom action that edits through a form** opens it with `handle.openForm(config, submit)`: the form service
  builds the config without an `onSubmit`, and the actions service, which receives the handle as a parameter,
  passes the request. The page closes the modal and reloads once the request answers.
- **A standard action is never rewritten to change its message**: to warn about something first (a featured
  product, a published one), the actions service builds the confirmation of
  `beyPageStandardAction(key, { confirmation })`, and the page keeps the standard request, toast and reload.
- **A row other rows use warns through its usages, never through a confirmation of its own**: the page config
  declares `usagesConfig: new BeyPageUsagesConfig({ suffixes })`, kept in `models/<entity>.model.ts` with the base
  URL of the page, and the page asks the backend before `delete` and `delete-trash-item`. A form or a cell that
  names the users of a row takes them from `BeyPageUsagesService` (`listUsers`, `describeRowUsers`), never from a
  request or a description of its own.
- **The columns of the table** are declared in the page config. A column showing a field the backend stores says
  `isSortable`, with `sortField` when that field is not its key; one built from data the backend adds to each row
  (a count, the names of its users) or from a list (tags) never does, since the list cannot be sorted by it. The
  column that names the row says `isHideable: false`, and a column worth having but not at first sight says
  `isVisible: false`. The table gets a `storageKey` named as the page, so the columns the user hides are
  remembered, and the widths are shares that fit a 1440 px window without cutting a badge.
- **The types and constants of the page** (the stored row, badge variants, tooltip keys) live in
  `models/<entity>.model.ts`.
- **Specs**: the component spec renders the page with `provideBeyTesting()` and checks the wiring through the DOM
  (rows, actions, forms, requests); the page config and every service have their own spec for what they return.
- **A workspace page that is not a `bey-page`** (an editor) keeps the same split. Its state lives in
  `<page>.service.ts`, a bare `@Injectable()` listed in the component's `providers` so every open page has its own:
  signals written only through its methods, and what derives from them as `computed`. What the user does lives in
  `<page>-actions.service.ts`, listed there too, working on that state, the `-http` and `-form` services and the
  modals. The component exposes the state to its template and hands every event to the actions service; its
  layout and the options it offers are built in `<page>-config.ts`, and its panels are child components with inputs
  and outputs.

## Example

```text
pages/templates/
├── assets/templates.en.json
├── models/template.model.ts
├── services/
│   ├── template-actions.service.ts
│   ├── template-form.service.ts
│   ├── template-status-form.service.ts
│   ├── template-status-http.service.ts
│   └── template-table.service.ts
├── templates-page-config.ts
└── templates.component.ts
```

```ts
// templates.component.ts
export class TemplatesComponent {
  private readonly templateActionsService = inject(TemplateActionsService);
  private readonly templateFormService = inject(TemplateFormService);
  private readonly templateTableService = inject(TemplateTableService);

  readonly templatesPageConfig = buildTemplatesPageConfig({
    formConfig: this.templateFormService.buildTemplatesFormConfig(),
    loadRow: template => this.templateTableService.loadRow(template),
    onChangeStatus: template => this.templateActionsService.changeStatus(template, this.page),
    onReady: handle => (this.page = handle)
  });

  private page?: BeyPageHandle<Template>;
}

// templates-page-config.ts
export function buildTemplatesPageConfig({
  formConfig,
  loadRow,
  onChangeStatus,
  onReady
}: TemplatesPageConfigOptions): BeyPageConfig<TemplateFormValue, Template> {
  return new BeyPageConfig<TemplateFormValue, Template>({
    prefix: PREFIX,
    baseUrl: TEMPLATES_BASE_URL,
    usagesConfig: TEMPLATE_USAGES_CONFIG,
    headerConfig: new BeyPageHeaderConfig({
      title: `${PREFIX}.title`,
      actions: [
        beyPageAddAction(),
        beyPageStandardAction(BeyPageStandardAction.Edit),
        new BeyPageAction({
          key: 'change-status',
          scope: BeyPageActionScope.Single,
          zone: BeyPageActionZone.Menu,
          handler: ([template]) => onChangeStatus(template)
        }),
        beyPageStandardAction(BeyPageStandardAction.Delete)
      ]
    }),
    tableConfig: new BeyPageTableConfig({ columns: COLUMNS, loadRow }),
    formConfig,
    onReady
  });
}

// models/template.model.ts
export const TEMPLATES_BASE_URL = '/templates';
export const TEMPLATE_USAGES_CONFIG = new BeyPageUsagesConfig({
  suffixes: { [TemplateBlockType.ContentBlock]: 'document-builder.templates.usages.block' }
});

// services/template-actions.service.ts
changeStatus(template: Template, page?: BeyPageHandle<Template>): void {
  page?.openForm(this.templateStatusFormService.buildChangeStatusFormConfig(template.status), ({ main }) =>
    main.status ? this.templateStatusHttpService.changeStatus(template.id, main.status) : EMPTY
  );
}
```
