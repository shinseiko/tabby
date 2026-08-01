---
paths:
  - "**/*.mjs"
---
# Tabby Build Scripts

> Fork-local rule. Extends `common/coding-style.md`. Covers `scripts/*.mjs` and the
> `webpack.*.config.mjs` family — 34 tracked files that no ECC language rule globs
> (`typescript/coding-style.md` matches `**/*.js`, which does not match `.mjs`).

## ESM only — no CommonJS

Every `.mjs` file is a native ES module. `require`, `module.exports`, and bare `__dirname` /
`__filename` are all unavailable and will throw at runtime.

Derive the directory the way `scripts/vars.mjs` does:

```js
import * as url from 'url'
const __dirname = url.fileURLToPath(new URL('.', import.meta.url))
```

Read JSON with `fs.readFileSync` + `JSON.parse`. Do not use import assertions.

## Shared state lives in vars.mjs

`scripts/vars.mjs` is the single source of truth for build metadata. Import from it rather than
recomputing:

- `builtinPlugins` — the 13 plugin directory names
- `allPackages` — `builtinPlugins` plus the root and `app`
- `version` — derived from `git describe --tags`, with nightly handling
- `packagesWithDocs`, `electronVersion`

Registering a new built-in plugin means adding it to `builtinPlugins` in `vars.mjs`. Missing this
step is the single most common cause of "my plugin builds but never loads."

## Webpack config invariant

`webpack.plugin.config.mjs` marks `@angular/*`, `rxjs`, and `tabby-*` as **externals** for every
plugin bundle. Do not add any of these to a plugin's bundle. Bundling Angular produces a second
Angular runtime and breaks dependency injection at load time, with errors that surface far from
the cause.

Each plugin's own `webpack.config.mjs` should stay a thin call into the shared config.

## Conventions

- Node 22, Yarn 1. Scripts are run directly (`node scripts/foo.mjs`) or via `yarn` aliases.
- Use the existing toolchain — `shelljs` for filesystem and shell work, `npmlog` for output.
  Do not introduce a second logger or a shell wrapper.
- `sh.exec(..., { fatal: true })` for any step whose failure must abort the build. A non-fatal
  `exec` that fails silently will produce a broken package with a green exit code.
- Prefer `log.info('<stage>', detail)` over `console.log` so build output stays greppable by stage.

## Error handling

Build scripts run unattended in CI (`.github/workflows/build.yml`). A script that swallows an
error produces a corrupt artifact rather than a failed job. Let failures propagate, and make any
`catch` block either re-throw or exit non-zero.
