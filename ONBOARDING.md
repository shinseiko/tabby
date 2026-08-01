# Onboarding Guide: Tabby

> Generated 2026-07-31 for the `dev-roku` fork. Companion to [CLAUDE.md](CLAUDE.md),
> which holds the enforceable rules; this file holds the explanation.

## Overview

Tabby is a cross-platform terminal emulator (SSH, serial, telnet, local shells) built as an
Electron app with an Angular 15 frontend. This checkout is a fork (`origin`) of `Eugeny/tabby`
(`upstream`), currently on branch `dev-roku`.

## Tech Stack

| Layer | Technology | Version |
| --- | --- | --- |
| Language | TypeScript | 4.9 (`strictNullChecks`, `noImplicitAny: false`) |
| UI framework | Angular | 15.2 (JIT mode, not Ivy AOT) |
| Templates | Pug | 3 (`.component.pug`) |
| Styles | Sass | `.component.scss` |
| Desktop shell | Electron | 38 |
| Bundler | webpack | 5 (no Angular CLI) |
| UI kit | ng-bootstrap + Bootstrap 5 | 14.1 / 5.3-alpha |
| Packaging | electron-builder | 26 |
| CI | GitHub Actions | `.github/workflows/build.yml` |
| Tests | none upstream | see [Testing](#testing) |

## Architecture

Tabby is a **plugin host, and its built-in features are plugins.** Three cooperating layers:

1. **Electron main process** — `app/lib/`, entry `app/lib/index.ts`, bundled by
   `app/webpack.config.main.mjs` to `app/dist/main.js`. Owns windows, PTY spawning, config file
   I/O, keytar, the updater.
2. **Renderer bootstrap** — `app/src/`, entry `app/src/entry.ts`. Deliberately minimal: pre-loads
   Angular/rxjs into a module cache, discovers plugins, builds the root NgModule from them.
3. **Plugins** — the 13 `tabby-*` directories. All real functionality lives here.

```text
Electron main (app/lib)  ──IPC──  Renderer bootstrap (app/src)
                                          │ loads NgModules
                                          ▼
   tabby-core ◄── everything depends on this (services + api/ extension points)
        ▲
        ├── tabby-terminal ── tabby-ssh / tabby-serial / tabby-telnet / tabby-local
        ├── tabby-settings
        ├── tabby-electron | tabby-web   (platform backends, mutually exclusive)
        └── tabby-plugin-manager, tabby-linkifier, tabby-community-color-schemes
```

### The bundling rule that governs everything

`webpack.plugin.config.mjs` marks `@angular/*`, `rxjs`, and `tabby-*` as **externals for every
plugin bundle**. Plugins do not ship Angular. The Electron renderer pre-loads one copy and injects
it through a patched `require` (`app/src/plugins.ts`).

Consequences:

- Every plugin declares Angular under `peerDependencies`, never `dependencies`.
- Bundling Angular into a plugin produces two Angular runtimes and breaks dependency injection.
- Plugin bundles are UMD, unminified, emitted to `<plugin>/dist/index.js`.

## Key Entry Points

- **Main process**: `app/lib/index.ts` → `app/lib/app.ts`
- **Renderer**: `app/src/entry.ts` → `app/src/app.module.ts`
- **Plugin discovery**: `app/src/plugins.ts` — only packages with a `tabby-plugin` /
  `tabby-builtin-plugin` keyword load
- **Extension points**: `tabby-core/src/api/` — the abstract classes you subclass to extend Tabby
- **Plugin registry**: `scripts/vars.mjs` (`builtinPlugins`) — add new built-in plugins here
- **Shared webpack config**: `webpack.plugin.config.mjs` — each plugin's config is a 10-line call in

## Directory Map

| Directory | Purpose |
| --- | --- |
| `app/` | Electron shell — main process (`lib/`) + renderer bootstrap (`src/`). Own `package.json`/`node_modules`; runtime deps ship from here |
| `tabby-core/` | Base UI, tab management, config, hotkeys, profiles, vault, theming. The dependency root |
| `tabby-terminal/` | xterm.js integration, terminal tab base classes, color schemes |
| `tabby-ssh/`, `tabby-serial/`, `tabby-telnet/`, `tabby-local/` | Connection-type plugins; each provides a `ProfileProvider` + tab component |
| `tabby-electron/` / `tabby-web/` | Platform backends implementing `HostAppService`, `PlatformService`, etc. |
| `tabby-settings/` | Settings tab UI |
| `tabby-plugin-manager/` | Installs third-party plugins from npm |
| `scripts/` | Build/package/i18n maintenance scripts (all `.mjs`, ESM) |
| `locale/` | Crowdin-managed `.po` translations |
| `patches/`, `app/patches/` | `patch-package` patches applied on `postinstall` |
| `web/`, `tabby-web-demo/` | Browser build — **currently disabled** in `webpack.config.mjs` |

## Lifecycle: opening a new tab

The closest analogue to a "request lifecycle" here. Tracing "user opens an SSH connection":

1. **Profile selected** — `ProfilesService.openNewTabForProfile()`
   (`tabby-core/src/services/profiles.service.ts`)
2. **Provider resolved** — `providerForProfile()` matches `profile.type` against multi-injected
   `ProfileProvider` instances (`'ssh'` → `SSHProfilesService`)
3. **Config merged** — `getConfigProxyForProfile()` layers profile defaults → group → user config
   into a `ConfigProxy`
4. **Params built** — the provider returns `NewTabParameters { type: SSHTabComponent, inputs: {…} }`
5. **Component instantiated** — `TabsService.create()`
   (`tabby-core/src/services/tabs.service.ts`) resolves a component factory, assigns inputs,
   wires `destroyed$`
6. **Attached** — `AppService.openNewTab()` adds it to the tab list; `appRoot.component` renders it
7. **Session started** — the tab's `initializeSession()` calls into `PlatformService` → IPC →
   main process spawns the PTY / opens the socket

The pattern to internalize: **`type` string → multi-injected provider → `NewTabParameters`.**
That indirection is how a plugin adds a connection type without core knowing about it.

## Conventions

**File naming** — camelCase, dot-suffixed by role: `foo.component.ts` / `.pug` / `.scss`,
`foo.service.ts`, `api/fooProvider.ts`. Directories: `src/{api,components,services,directives}/`.

**Extension pattern** — extend an abstract class from `tabby-core`, register with
`{ provide: XProvider, useClass: MyX, multi: true }`. Never import another plugin's internals;
import from its `api.ts`.

**Code style** (`.eslintrc.yml`, `@typescript-eslint/all` with ~60 rules relaxed):

- **No semicolons**, 4-space indent, single quotes, trailing commas on multiline
- Space before method parens: `create <T> (params: X): T`
- `no-public` member accessibility — write `foo()`, not `public foo()`
- Constructor parameter properties required over manual assignment
- 1TBS braces, `curly` always, `eqeqeq` (smart)

**Angular** — `strictTemplates: true`, `strictInjectionParameters: true`,
`@Injectable({ providedIn: 'root' })` for services.

**Git** — Conventional Commits, loosely applied: `fix(ssh): …`, `feat(core): …`, `feat: …`, plus
bare messages (`bump deps`, `lint & fixes`) for maintenance. PRs squash-merged with `(#NNNN)`.

## Testing

Upstream has no tests. This fork uses **scoped TDD**: required for fork-authored plugins, not
required for upstream-owned files. See the Testing section of [CLAUDE.md](CLAUDE.md) for the
enforceable rule and its rationale.

No test runner is installed yet — the first fork-authored plugin needs one bootstrapped.

## Common Tasks

| Task | Command |
| --- | --- |
| Install | `yarn` (root), and `cd app && yarn` |
| Full build | `yarn build` (typings → all plugin bundles) |
| Watch/rebuild | `yarn watch` |
| Run dev | `yarn start` (sets `TABBY_DEV=1`, loads plugins from source) |
| Lint | `yarn lint` |
| Package installer | `node scripts/prepackage-plugins.mjs` then `node scripts/build-windows.mjs` |
| Extract i18n | `yarn i18n:extract` |

Node 22 (CI), Yarn 1. Normal dev loop is `yarn watch` and `yarn start` in two terminals.

## Where to Look

| I want to… | Look at… |
| --- | --- |
| Add a new connection type | New `tabby-*` dir + register in `scripts/vars.mjs`; model on `tabby-telnet/` (smallest example) |
| Add a toolbar button / context menu / hotkey | Subclass the provider in `tabby-core/src/api/` |
| Change default settings | `tabby-core/src/configDefaults*.yaml` |
| Touch OS-level behavior | `app/lib/` (main process) or `tabby-electron/` |
| Change terminal rendering | `tabby-terminal/src/frontends/` |
| Add a settings UI pane | `SettingsTabProvider` in `tabby-settings/src/api.ts` |
| Fix a build/bundling issue | `webpack.plugin.config.mjs` — especially `externals` |
