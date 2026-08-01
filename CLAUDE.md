# Tabby — Project Instructions

## Development Environment

- OS: Windows 10.0.26200
- Shell: Git Bash
- Path format: Windows (use forward slashes in Git Bash)
- File system: Case-insensitive
- Line endings: CRLF (configure Git autocrlf)

## Project

Tabby — cross-platform terminal emulator. Electron 38 + Angular 15 (JIT) + Pug + Sass,
bundled by webpack 5 (no Angular CLI). TypeScript 4.9, `strictNullChecks` on,
`noImplicitAny` off. Fork of `Eugeny/tabby`; `upstream` remote is the source of truth.

## Architecture

Tabby is a **plugin host, and its built-in features are plugins**. Three layers:

- `app/lib/` — Electron main process (windows, PTY, config I/O, keytar). Entry `lib/index.ts`.
- `app/src/` — renderer bootstrap. Pre-loads Angular/rxjs, discovers plugins, builds root NgModule.
- `tabby-*/` — the 13 built-in plugins, listed in `scripts/vars.mjs`. All real functionality.

`tabby-core` is the dependency root. Extend Tabby by subclassing an abstract class from
`tabby-core/src/api/` and registering it as a **multi-provider**:

```ts
{ provide: ToolbarButtonProvider, useClass: MyProvider, multi: true }
```

A plugin's default export must be an `NgModule`. Cross-plugin imports go through the other
plugin's `api.ts` only — never its internals.

**Critical bundling rule:** `@angular/*`, `rxjs`, and `tabby-*` are webpack `externals` for every
plugin (`webpack.plugin.config.mjs`). Angular is loaded once by the renderer and injected via a
patched `require`. Always declare Angular under `peerDependencies`, never `dependencies` —
bundling it produces two Angular runtimes and breaks DI.

## Code Style

Enforced by `.eslintrc.yml` (`@typescript-eslint/all` with many rules relaxed). Run `yarn lint`.

- **No semicolons.** 4-space indent, single quotes, trailing commas on multiline.
- Space before method parens: `create <T> (params: X): T`
- `no-public` accessibility — write `foo()`, not `public foo()`.
- Prefer constructor parameter properties over manual assignment.
- 1TBS braces, `curly` always, `eqeqeq` smart, `no-var`, no unused vars (`_`-prefix to ignore).
- `@Injectable({ providedIn: 'root' })` for services. `strictTemplates` is on.

## File Naming

camelCase with role suffixes, grouped by role not by feature:

```text
tabby-foo/src/
  api.ts                     # public exports — the ONLY cross-plugin surface
  index.ts                   # NgModule default export
  api/barProvider.ts         # abstract extension points
  components/baz.component.{ts,pug,scss}
  services/qux.service.ts
```

## Build & Run

Node 22, Yarn 1. Install with `yarn` at the **root only**. Its `postinstall` runs
`patch-package`, then `scripts/install-deps.mjs` — which installs `app/`, `web/`, and every
plugin — then `scripts/build-native.mjs`. Running `cd app && yarn` by hand is redundant.

- Build: `yarn build` (typings, then all plugin bundles)
- Dev loop: `yarn watch` in one terminal, `yarn start` in another (`TABBY_DEV=1`, loads plugins from source)
- Lint: `yarn lint`
- Package: `node scripts/prepackage-plugins.mjs` then `node scripts/build-{windows,linux,macos}.mjs`

Adding a built-in plugin requires registering it in `scripts/vars.mjs` (`builtinPlugins`).

## Testing — scoped TDD

Upstream Tabby has **no test suite**: zero spec files, no runner, no coverage tooling. CI
(`.github/workflows/build.yml`) runs lint plus platform builds only. That is upstream's choice and
this fork does not try to change it retroactively.

**The boundary is authorship, not location.**

| Code | Policy |
| --- | --- |
| Fork-authored plugins (`tabby-*` dirs that do not exist upstream) | **TDD required.** Test first, 80% coverage target. |
| Any upstream file — `app/`, `tabby-core/`, other `tabby-*`, `scripts/` | **No test requirement.** Do not add tests here. |

Rationale for the exemption: upstream code is instantiated through `TabsService.create()` via
`ComponentFactoryResolver`, depends on multi-injected providers, and talks to a live PTY over
Electron IPC — testing it means TestBed plus a mocked `PlatformService`, which is infrastructure
work, not a `describe` block. Worse, tests added to upstream-owned directories are pure merge
surface: upstream edits those files and will never maintain the tests. Keeping the test tree inside
fork-owned directories keeps `git merge upstream/master` clean.

**No test runner is installed yet.** The first fork-authored plugin needs one bootstrapped
(Jest or Vitest + Angular TestBed, configured to collect only from fork-owned dirs). Treat that as
its own task — do not scaffold it as a side effect of unrelated work.

**Verification honesty applies everywhere, including exempt code.** Never claim a change is
verified without stating how. Default for exempt code is `yarn lint` plus a manual `yarn start`
exercising the affected path — say explicitly which of the two was actually run, and say so plainly
if neither was.

## Conventions

- Commits: Conventional Commits, loosely — `fix(ssh): ...`, `feat(core): ...`, `feat: ...`.
  Bare messages (`bump deps`) are acceptable for maintenance. PRs squash-merge with `(#NNNN)`.
- Branch: work happens on `dev-roku`; `master` tracks upstream.
- Default settings live in `tabby-core/src/configDefaults*.yaml` (per-platform variants).
- `patches/` and `app/patches/` are `patch-package` patches applied on `postinstall` — edit the
  patch, not `node_modules`.
- The `web/` browser build is currently disabled in `webpack.config.mjs`; ignore unless re-enabling.
