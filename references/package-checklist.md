# Package checklist

The file-by-file requirements for a new `@deepseek-ai/dsh-<name>` workspace
package, inlined from the repository's `docs/cookbook/adding-a-package.md` and
`packages/AGENTS.md`. This checklist is validated against shipped packages
(agent-loop, tools, bash) as templates; when it drifts from the source docs,
the docs win.

## 1. Layout

```
packages/<group>/<pkg>/
  package.json     # copy from packages/core/tools, adjust name/description/deps
  tsconfig.json    # extends ../../../tsconfig.base.json, rootDir src,
                   # outDir lib/types, references: ../../../vendor/cosmokit,
                   # ../../../vendor/cordis (+ ../../../vendor/schemastery if
                   # you use Config, + ../../<group>/<dep> for each dsh dep)
  src/index.ts     # service default export or plugin (name/inject/Config/apply)
  README.md        # service API, config, events, extension points, design notes
```

Choose an existing group (`packages/README.md` owns the authoritative group
table); a new group is a pure container: no `package.json`, no source files,
and packages sit exactly one level below it.

## 2. package.json invariants

Enforced by `pnpm run constraints` / `scripts/check-workspace-constraints.ts`:

- `private: true`
- `version` matching the root `package.json`
- `type: module`
- `main: "lib/index.js"`, `types: "lib/types/index.d.ts"`
- `exports["."].types: "./lib/types/index.d.ts"`, `exports["."].default: "./lib/index.js"`
- `@deepseek-ai/cordis` in BOTH `peerDependencies` and `devDependencies` (same range)
- every dsh peer dependency mirrored in `devDependencies`
- `@deepseek-ai/schemastery` in `dependencies` (it is a runtime validator)
- `files` contains exactly `lib/index.js`, `lib/invariant.js`,
  `lib/types/**/*.d.ts`, and package-specific runtime artifacts recognized by
  the gate; never `src`, declaration maps, JS maps, or stale root declarations
- CLI app packages with a `bin` include `lib/bin.js` immediately after
  `lib/index.js` in `files`

In-package relative imports use explicit `.ts` specifiers in source (`export *
from './types.ts'`); the compiler rewrites them to `.js` in emitted JS and
leaves `.ts` in declarations, which NodeNext/Node16 consumers resolve to the
sibling `.d.ts` files.

## 3. Root config registration

| File | Change |
|---|---|
| `tsconfig.host.json` (Host package) or `tsconfig.client.json` (Client package) | add `{ "path": "./packages/<group>/<pkg>" }` to `references` — an ordinary package belongs to exactly ONE aggregate, never both; `api/remotes` uses a repository-specific split, new packages must not copy it |
| `tsconfig.base.json` | only for a new group: add a `./packages/<group>/*/src` candidate to the `@deepseek-ai/dsh-*` wildcard |
| `knip.json` | only if the package has entrypoints repository discovery does not cover |

A `packages/client/*` package additionally extends `tsconfig.base.client.json`
instead of `tsconfig.base.json`, declares `dsh.client` in package.json, exports
`./client`, and calls the shared tsdown preset (`packages/client/tsdown.client.ts`).

Covered automatically by globs or package-manifest discovery — no edits:
root `package.json` workspaces, `scripts/publint-all.ts`, `tsdown.config.ts`,
`.oxlintrc.json`, `scripts/check-workspace-constraints.ts`.

## 4. Plugin export forms

From `packages/AGENTS.md`:

- **Service packages default-export their service class.**
- **Function plugins named-export `name` / `inject` / `Config` / `apply` and
  have no default export.** Mixing the forms makes the Loader discard the
  function plugin's namespace (postmortem
  `docs/postmortem/0001-acp-default-export-drops-inject.md`).
- Optional services use `ctx.get(name)`; reserve `ctx.<name>` for declared
  injections (the property proxy is topology-sensitive while strict `ctx.get`
  reads the global service store).

## 5. Package README

- Package-specific service API, config, events, extension points, and design
  notes come first.
- End with the canonical Model Experience section: one H3 per direct,
  conditional, capped, lifetime, or auxiliary model-context entry, with the
  ordered H4 fields `Request context and condition` / `What the model sees` /
  `Token effect` / `KV Cache effect`; a package with no context effect uses the
  audited `None, as ` / `Indirectly, through ` sentence. A tool-schema entry
  links its anchored section in the generated `docs/tool-catalog.md`.
- `## Known Limitations and Deferred Work` records durable consumer gaps and
  non-obvious maintainer constraints; ordinary cleanup stays in source TODOs
  or an Agent Note. Packages with none use a justified allowlist entry in
  `scripts/verify-package-readme-limitations.ts`.

## 6. Every package owns `./invariant`

Register the manifest name and check an owned event-stream or mutable-data
relationship at the point where the package can observe it; or give empty
installers a package-specific `No runtime invariant:` reason. Service or method
presence, plugin metadata or effects, and fixed pure examples belong in type,
load, or unit tests — not in the invariant.

## 7. Verify

```sh
pnpm install        # registers the workspace
pnpm run doc-sync
pnpm run constraints && pnpm run typecheck && pnpm run lint
pnpm run build && pnpm run hygiene
```

Behavior-specific checks follow the testing policy
([testing-policy.md](testing-policy.md)).
