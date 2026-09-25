# Style rules

Conventions to follow when writing CSS.

## Layers

Four layers, each one only allowed to read from the one above it:

```
Bootstrap            base: reset, grid, utilities, components
  ↑ configured by
--bey-* tokens       the single source of truth for every design value
  ↑ read by
--bey-<module>-*     optional, only where a module offers a customisation point
  ↑ read by
.bey-<module>-*      the module's own classes
```

## Rules

- **`--bs-*` variables are only ever assigned, never read.** A `--bs-*` value always comes from a `--bey-*` token, never from a literal and never from another `--bs-*`.
- **No literal design values outside the token layer**: no colours, sizes, radii, font sizes, shadows or durations. If there is no token for it, the token is created first.
- **A `--bey-<module>-*` variable is created only when the module wants to offer that point for customisation.** Its value is always a token. When there is nothing to offer, the class reads the token directly.
- **Dark mode lives in the token layer.** A module declares `:host-context(body.dark)` only for what cannot be expressed as a token swap, such as `filter: invert(1)` on an image.
- **Bootstrap first**: layout, spacing and typography utilities come from Bootstrap. A custom class exists only when Bootstrap does not cover the case or the style is semantic to the module.
- **A class name is `bey-<module>-<part>[-<subpart>]`** and always sits in the static `class` attribute.
- **State is `is-*` or `has-*`**, always applied through a binding, never written in the static `class`.
- **A variant is `bey-<module>--<variant>`**: a fixed modality chosen by the config, not a state that changes while the component lives.
- **No `!important`.** If a rule does not win, the selector or the layer is wrong.
- **`:host` declares its own `display`**, since an unstyled host is inline by default.
- **What several children of a module share goes in one file next to them**, imported through `styleUrls`. It is never duplicated and never imported from another module.

## Example

```css
:host {
  --bey-tabs-fg: var(--bey-text-muted);
  --bey-tabs-fg-active: var(--bey-text-primary);
  --bey-tabs-line: var(--bey-border-subtle);

  display: block;
}

.bey-tabs {
  border-bottom: var(--bey-border-width) solid var(--bey-tabs-line);
}

.bey-tabs-tab {
  padding: var(--bey-space-2) var(--bey-space-3);
  color: var(--bey-tabs-fg);
  font-size: var(--bey-font-size-sm);
}

.bey-tabs-tab.is-active {
  color: var(--bey-tabs-fg-active);
}

.bey-tabs-tab.is-disabled {
  opacity: var(--bey-opacity-disabled);
}

.bey-tabs--segmented .bey-tabs-tab {
  border-radius: var(--bey-radius-pill);
}
```

```html
<div class="bey-tabs" [class.bey-tabs--segmented]="isSegmented()">
  @for (tab of tabs(); track tab.key) {
    <button
      class="bey-tabs-tab"
      role="tab"
      type="button"
      [class.is-active]="isActive(tab)"
      [class.is-disabled]="tab.isDisabled"
      [attr.aria-selected]="isActive(tab)"
    >
      {{ tab.label | translate }}
    </button>
  }
</div>
```
