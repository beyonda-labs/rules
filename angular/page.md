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
- **A standard action is never rewritten to change its message**: to warn about something first (a block in use, a
  file in use), the actions service builds the confirmation of `beyPageStandardAction(key, { confirmation })`,
  and the page keeps the standard request, toast and reload.
- **The types and constants of the page** (the stored row, badge variants, tooltip keys) live in
  `models/<entity>.model.ts`.
- **Specs**: the component spec renders the page with `provideBeyTesting()` and checks the wiring through the DOM
  (rows, actions, forms, requests); the page config and every service have their own spec for what they return.

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
    deleteConfirmation: (templates, confirmation) =>
      this.templateActionsService.deleteConfirmation(templates, confirmation),
    formConfig: this.templateFormService.buildTemplatesFormConfig(),
    loadRow: template => this.templateTableService.loadRow(template),
    onChangeStatus: template => this.templateActionsService.changeStatus(template, this.page),
    onReady: handle => (this.page = handle)
  });

  private page?: BeyPageHandle<Template>;
}

// templates-page-config.ts
export function buildTemplatesPageConfig({
  deleteConfirmation,
  formConfig,
  loadRow,
  onChangeStatus,
  onReady
}: TemplatesPageConfigOptions): BeyPageConfig<TemplateFormValue, Template> {
  return new BeyPageConfig<TemplateFormValue, Template>({
    prefix: PREFIX,
    baseUrl: '/templates',
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
        beyPageStandardAction(BeyPageStandardAction.Delete, { confirmation: deleteConfirmation })
      ]
    }),
    tableConfig: new BeyPageTableConfig({ columns: COLUMNS, loadRow }),
    formConfig,
    onReady
  });
}

// services/template-actions.service.ts
changeStatus(template: Template, page?: BeyPageHandle<Template>): void {
  page?.openForm(this.templateStatusFormService.buildChangeStatusFormConfig(template.status), ({ main }) =>
    main.status ? this.templateStatusHttpService.changeStatus(template.id, main.status) : EMPTY
  );
}
```
