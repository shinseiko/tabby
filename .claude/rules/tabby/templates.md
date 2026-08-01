---
paths:
  - "**/*.pug"
---
# Tabby Pug Templates

> Fork-local rule. Extends `common/coding-style.md`. Covers the 62 `.component.pug` files.
> No ECC rule set globs `*.pug` — `angular/coding-style.md` assumes `*.component.html`.

## These are Angular templates written in Pug

Angular's own syntax passes through Pug untouched, so structural directives, bindings, and
interpolation all work as normal — they are just written in Pug's indentation-based attribute
syntax:

```pug
a.list-group-item.list-group-item-action(
    *ngFor='let command of commands; trackBy: buttonsTrackBy',
    (click)='command.run()',
)
    span {{command.label}}
```

`strictTemplates` is on. Type errors in a template are build errors.

- Multi-attribute elements go one attribute per line with trailing commas, as above.
- Prefer Pug class/id shorthand (`.list-group.mb-4`) over `div(class='...')`.
- Bootstrap 5 utility classes are the styling default; reach for a `.scss` file only when a
  utility composition will not do.

## trackBy on every *ngFor

30 template files use `*ngFor`. Terminal UI re-renders frequently — an `*ngFor` without `trackBy`
destroys and rebuilds every DOM node on each change-detection pass over a mutated list. Supply a
`trackBy` function from the component.

## i18n — the extraction pipeline dictates the syntax

`scripts/i18n-extract.mjs` cannot read Pug. It compiles every plugin's templates to temporary HTML
with `yarn pug`, then runs `gettext-extractor` over the result. Two selectors do all the work:

```js
HtmlExtractors.elementContent('translate, [translate=""]', options)
HtmlExtractors.elementAttribute('[translate*=" "]', 'translate', options)
```

**Element-content form** — a bare `translate` attribute, string as the element body:

```pug
span(translate) Report a problem
```

**Attribute form** — for strings carrying interpolation placeholders:

```pug
.form-control-static(translate='Version: {version}', [translateParams]='{version: version}')
```

Use `translatecontext` to disambiguate a string that means different things in different places.

> **Trap:** the attribute-form selector is `[translate*=" "]` — it matches only when the value
> **contains a space**. `translate='Version'` is silently dropped from `locale/app.pot`; it will
> never appear in Crowdin and will never be translated. A single-word string must use the
> element-content form instead.

> **Trap:** the TypeScript extractor globs `./tabby-*/src/**/*.ts` only. Translatable strings in
> `app/` are never extracted.

Run `yarn i18n:extract` and confirm your string landed in `locale/app.pot` before assuming it is
covered.

## innerHTML must go through DomSanitizer

Terminal output, profile names, and plugin-supplied icons are untrusted input. Exactly one
template currently binds `[innerHTML]`, and it routes through a sanitizer:

```pug
.d-flex.align-self-center([innerHTML]='sanitizeIcon(command.icon)')
```

```ts
sanitizeIcon (icon?: string): SafeHtml {
    return this.domSanitizer.bypassSecurityTrustHtml(icon ?? '')
}
```

Never bind raw user or plugin data to `[innerHTML]`. If a new binding is genuinely required, add a
sanitizing helper on the component rather than calling `bypassSecurityTrust*` inline in the
template, and satisfy yourself that the source is trusted — `bypassSecurityTrust*` disables
Angular's escaping entirely.
