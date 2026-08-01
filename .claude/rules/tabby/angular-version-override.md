---
paths:
  - "**/*.component.ts"
  - "**/*.service.ts"
  - "**/*.directive.ts"
  - "**/*.pipe.ts"
  - "**/*.module.ts"
  - "**/*.guard.ts"
  - "**/*.resolver.ts"
---
# Angular — Tabby Version Override

> **This rule overrides `ecc/angular/*.md` wherever the two disagree.** Those files are written for
> Angular 17–21. Tabby is on **`@angular/core: ^15.2.6`**, and several of their recommendations
> describe APIs that do not exist in this codebase. Where this file is silent, the ECC angular
> rules still apply.

## What this project actually is

| | Tabby | ECC angular rules assume |
| --- | --- | --- |
| Angular | 15.2, **JIT** (not Ivy AOT) | 17–21 |
| Build | raw **webpack 5** | Angular CLI (`ng build`, `angular.json`) |
| Components | **14 `@NgModule`s**, 0 standalone | standalone by default |
| DI | constructor parameter properties | `inject()`, empty constructors |
| Templates | **Pug** with `*ngFor` / `*ngIf` | HTML with `@for` / `@if` |
| State | RxJS + services | signals, `resource()`, `linkedSignal` |

## Corrections — do not follow the ECC guidance on these

**Dependency injection.** Use constructor parameter properties. This is not a preference; it is
required by [CLAUDE.md](../../../CLAUDE.md) and matches all 14 modules.

```ts
// CORRECT for this codebase
constructor (
    private config: ConfigService,
    private hostApp: HostAppService,
) { }
```

Do **not** "keep constructors empty" or convert to `inject()`. There are 4 `inject()` calls in the
entire codebase; they are the exception, not the direction of travel.

**No standalone components.** Every component is declared in an `@NgModule`. A new component gets
added to its plugin's module `declarations`, and exported if other plugins consume it.

**No Angular CLI.** `ng build`, `ng lint`, `ng generate`, `ng version`, and `angular.json` do not
exist here. The real commands are in [CLAUDE.md](../../../CLAUDE.md):

```
yarn build     # typings, then all plugin bundles
yarn watch     # dev rebuild loop
yarn start     # TABBY_DEV=1, loads plugins from source
yarn lint      # eslint
```

**Unavailable APIs — do not introduce.** `signal`, `computed`, `linkedSignal`, `resource`,
`effect`, `afterRenderEffect`, `input()`, `output()`, `toSignal` (all v16+); `takeUntilDestroyed`
(v16+); `@for` / `@if` block syntax (v17+); signal forms (v21+); `provideHttpClient` /
functional interceptors as the default (v15 has them, but this app makes almost no HTTP calls —
it is a terminal emulator, not a web client).

For subscription cleanup, follow the existing pattern: `TabsService.create()` wires a `destroyed$`
subject, and components pipe `takeUntil(this.destroyed$)`.

**No `OnPush` sweep.** ECC says default every new component to `OnPush`. Tabby's components are
largely default change detection and interact with xterm.js and PTY streams outside the Angular
zone. Adding `OnPush` to a component whose inputs are mutated in place will silently stop it
rendering. Match the surrounding component's strategy rather than applying a blanket rule.

**Ignore the SSR, routing, and HTTP-interceptor sections entirely.** This is an Electron desktop
app: no router, no server rendering, no route guards, no `TransferState`. The extension model is
multi-provider DI, not routes — see the `tabby-core/src/api/` abstract classes.

## Still correct from the ECC angular rules

The generic advice holds: services own logic and components delegate, RxJS operator selection
(`switchMap` / `mergeMap` / `exhaustMap`), always `catchError`, never mutate `@Input()` objects in
place, and the XSS guidance in `ecc/angular/security.md` — which matters here more than in a
typical web app, because terminal output is untrusted input. See `tabby/templates.md` for the
`DomSanitizer` pattern this codebase uses.
