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
- **No literal design values outside the token layer**: colours, spacing, radii, font sizes, weights and line
  heights, letter spacing, shadows, opacities, z-indexes and durations always read a `--bey-*` token. If there is no
  token for it, the token is created first. A colour mixed from tokens with `color-mix()` is a colour too: when two
  modules mix the same one (the soft accent of a dialog, a translucent page), the mix is a token.
- **Three literals stay in the module**: a dimension of its own layout (the columns of a grid, the box of an icon,
  the max height of a panel) and the ratio of a `color-mix()` no other module makes, since a token would serve that
  module alone, and a Bootstrap breakpoint in a media query, which cannot read a variable, written as Bootstrap
  writes it (`max-width: 767.98px`). Each one is written once; a dimension several modules need (the height of a
  control) is a token.
- **A `--bey-<module>-*` variable on `:host` is a customisation point**, created only when the module wants to offer
  it. Its value is a token, or one of the module's own literals above. When there is nothing to offer, the class
  reads the token directly.
- **A module variable first declared on an element, not on `:host`, is private**: it lets a state or a variant
  swap one value (`--bey-tabs-tab-fg`) instead of repeating every declaration that reads it, and a consumer is not
  meant to set it.
- **Dark mode lives in the token layer.** A module declares `:host-context(body.dark)` only for what cannot be expressed as a token swap, such as `filter: invert(1)` on an image.
- **Bootstrap first**: layout, spacing and typography utilities come from Bootstrap. A custom class exists only when Bootstrap does not cover the case or the style is semantic to the module.
- **A class name is `bey-<module>-<part>[-<subpart>]`** and always sits in the static `class` attribute.
- **State is `is-*` or `has-*`**, always applied through a binding, never written in the static `class`.
- **A variant is `bey-<module>--<variant>`**: a fixed modality chosen by the config, not a state that changes while the component lives.
- **No `!important`.** If a rule does not win, the selector or the layer is wrong.
- **`:host` declares its own `display`**, since an unstyled host is inline by default: in the component's `.css`, or
  in the shared file it imports when it has none, as a style-guide demo that reads only the shared demo styles does.
  A component with no stylesheet at all keeps an inline host, which is right only when nothing sizes or spaces it
  (a routed page that wraps the app layout) or when its parent sets its display or lays it out as a grid or flex
  item.
- **What several children of a module share goes in one file next to them**, imported through `styleUrls`. It is never duplicated and never imported from another module.
- **Rules follow the template.** `:host` and its custom properties open the file; then one block per element in the order the template renders them; the states (`is-*`, `has-*`) and variants (`bey-<module>--*`) of an element come right after its base rule; media queries and `:host-context` close the file. Rules are never sorted alphabetically: the cascade depends on their order, and reading them next to the template does too.
- **Inside a block, custom properties come first and declarations are alphabetical**, with a shorthand before its longhands. Stylelint enforces and fixes it (`order/order`, `order/properties-alphabetical-order`), so nobody has to remember a grouping.
- **No comments in a module's CSS**: whatever needs explaining goes in its README. Only the token layer may label its groups with a comment that mirrors the token catalogue, and a `stylelint-disable` stays only with its reason when a rule cannot be met.

## Example

```css
:host {
  --bey-tabs-fg: var(--bey-text-muted);
  --bey-tabs-fg-active: var(--bey-text-primary);
  --bey-tabs-line: var(--bey-border-subtle);
  --bey-tabs-tab-min-width: 6rem;

  display: block;
}

.bey-tabs {
  border-bottom: var(--bey-border-width) solid var(--bey-tabs-line);
}

.bey-tabs-tab {
  --bey-tabs-tab-fg: var(--bey-tabs-fg);

  color: var(--bey-tabs-tab-fg);
  font-size: var(--bey-font-size-sm);
  min-width: var(--bey-tabs-tab-min-width);
  padding: var(--bey-space-sm) var(--bey-space-md);
}

.bey-tabs-tab.is-active {
  --bey-tabs-tab-fg: var(--bey-tabs-fg-active);
}

.bey-tabs-tab.is-disabled {
  opacity: var(--bey-opacity-disabled);
}

.bey-tabs--segmented .bey-tabs-tab {
  border-radius: var(--bey-radius-pill);
}

.bey-tabs-tab-count {
  border: var(--bey-border-width) solid var(--bey-tabs-tab-fg);
  border-radius: var(--bey-radius-pill);
  padding: 0 var(--bey-space-xs);
}

@media (max-width: 575.98px) {
  .bey-tabs-tab {
    min-width: 0;
  }
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
      @if (tab.count) {
        <span class="bey-tabs-tab-count">{{ tab.count }}</span>
      }
    </button>
  }
</div>
```
