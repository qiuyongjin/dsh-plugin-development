# Plugin templates

Verified shapes distilled from shipped packages. Adjust names and imports to
the real surface of the services you inject; the reference implementations at
the bottom show the production-grade versions. All templates assume ESM with
`.ts` specifiers in local relative imports.

## 1. Function plugin (single-purpose behavior)

The majority of plugins. Named exports only — **no default export**: the Loader
discards a function plugin's namespace when a default export is present (the
rule's postmortem: `docs/postmortem/0001-acp-default-export-drops-inject.md`).
Modeled on `packages/llm/llm-retry/src/index.ts`.

```ts
import type { Context } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'

export const name = 'my-plugin'
export const inject = ['tools']

/** Validated deployment-owned configuration. */
export interface Config {
  /** Timeout for one operation, in milliseconds. */
  timeoutMs: number
  /** Optional label shown in diagnostics. */
  label?: string
}

/** Runtime schema for {@link Config}. */
export const Config: z<Config> = z.object({
  timeoutMs: z.number().step(1).min(1).default(30000),
  label: z.string(),
})

export function apply(ctx: Context, config: Config): void {
  // validate config fail-loud at load; then register everything as effects:
  // ctx.on(...), ctx.effect(...), ctx.tools.register(...) all return disposers.
}
```

Config with no fields uses `export type Config = Readonly<Record<string, never>>`
and `export const Config = z.object({}) as unknown as z<Config>` (llm-retry),
plus a `validateConfig` that throws on unknown keys.

## 2. Service subclass (owns a `ctx` key, provides a capability)

Modeled on `packages/plan/plan-mode/src/index.ts`. The class is the Service
Definition; default-export it. Package-level requirements (package.json,
tsconfig, invariant) live in [package-checklist.md](package-checklist.md).

```ts
import { Context, Service } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    myThing: MyThingService
  }
}

/**
 * Owns ... (one paragraph stating the contract, model visibility, and
 * durability). Extension points: ...
 */
export class MyThingService extends Service {
  static inject = ['tools', 'systemPrompt']

  constructor(ctx: Context, config: Config = {}) {
    super(ctx, 'myThing')          // singular key: one engine/controller
    // listeners and registrations go here; all dispose with the fiber
  }
}

export default MyThingService
```

For a registry (plural key: `ctx.myThings`), the same shape with a
`register(entry): () => void` API whose returned disposer unregisters.

## 3. Model-facing tool

The minimal registration; the full `execute` contract and render-intent rules
are in [tool-contract.md](tool-contract.md). The tool registers on
`ctx.tools`; its schema joins prompt assembly automatically.

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // what the model sees
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // optional by default
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args is TYPED from the schema; exec.signal is the operational field
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

## 4. Event listener (interception)

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision } from '@deepseek-ai/dsh-tools'

export const name = 'permission-gate'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()   // waterfall listeners MUST delegate with next()
  })
}
```

Waterfall rule: return without `next()` short-circuits the chain — the right
shape for a policy listener that owns the decision, wrong for an observer.
`agent/turn-stopping` is serial: no `next()`. Event domains and dispatch modes:
[extension-points.md](extension-points.md#event-domains).

## 5. Durable session event

Anything model-visible must be reconstructable from the session log. Extend
`SessionEventMap` and append through the session:

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    /**
     * What changed, durably. @mode emit — log-only, whole-value replace.
     * @param active Whether ...
     */
    'my/mode': { active: boolean }
  }
}

// inside a listener or service method, when the fact is committed:
session.append('my/mode', { active: true })
```

Event JSDoc needs `@mode` and payload `@param`. Scoped keys absent from
payloads need `@dshScopeScan unsupported`. Only structural format changes bump
`SESSION_FORMAT_VERSION` in `dsh-session`.

## 6. Prompt section

```ts
ctx.systemPrompt.section({
  name: 'my:policy',
  order: 50,
  text: (context) => {
    if (context.agent === undefined) return ''
    return activeNow(context) ? sectionText : ''
  },
})
```

Sections are model-visible: the text they produce must be logged or derivable
from the log.

## 7. Optional children and optional services

A registration that needs a service that may not be composed uses
`ctx.inject(...)` (plan-mode's projection and command children) or `ctx.get(...)`
for a read inside a call path:

```ts
// child activates only when the service is composed
ctx.inject(['sessionProjections'], (childCtx) => {
  childCtx.sessionProjections.register({ key: 'my', ... })
})

// inside a tool execute / handler
const interaction = ctx.get('userQuestions')
if (interaction === undefined) throw new Error('no user-questions channel')
```

Use `ctx.get(name)` for optional reads; reserve `ctx.<name>` for declared
injections (the property proxy is topology-sensitive).

## 8. cordis.yml row and patch overlay

A row mounts one plugin. `!!js` expressions interpolate at mount (`config`
against the plugin context, `disabled` against the loader context); other
metadata stays literal — use overlays when the environment selects plugins.

```yaml
- id: my-plugin
  name: '@deepseek-ai/dsh-my-plugin'
  config:
    timeoutMs: 60000
    cwd: !!js process.cwd()
```

A patch targets a row by id and replaces its **whole** config (restate kept
fields), or inserts new rows:

```yaml
- id: webserver
  config:
    port: 3081

- insert:
    - id: my-plugin
      name: '@deepseek-ai/dsh-my-plugin'
```

Layer order (lowest to highest): each bundle in the profile's listed order, the
profile's `cordis.patch.yml`, the home-level `cordis.patch.yml`, then any
`--patch` overlay. Inspect the tree your machine actually boots with
`dsh --profile web --dump-config`.

## 9. Distribution: bundle and profile

- **Bundle** — an npm package whose manifest declares its patch layer:
  `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` and whose
  `cordis.patch.yml` contains the rows it inserts. A profile lists bundles in
  `dsh.profile.bundles` order.
- **Profile** — `$DSH_HOME/profiles/<name>/` (default `~/.dsh/profiles`) holds
  a `package.json` whose `dependencies` are the out-of-tree plugins, the
  `dsh.profile` manifest with ordered `bundles`, and the user's own
  `cordis.patch.yml`. `web` and `headless` auto-initialize; other names fail
  loud until created through the `dsh plugin` path.
- Bare plugin specifiers (`@deepseek-ai/dsh-*`, npm packages) resolve through
  the Loader from the config directory; relative specifiers resolve against the
  config directory.

## Reference implementations

| Pattern | Package |
|---|---|
| Function plugin + waterfall recovery + durable retry events | `packages/llm/llm-retry` |
| Service subclass + session events + prompt section + optional children + tool | `packages/plan/plan-mode` |
| Tool trio (Definition / local provider / tool Consumer) | `packages/shell/tool-bash` (with `packages/shell/shell`, `packages/shell/bash-local`) |
| Provider registry (Definition merges provider catalogs) | `packages/skill/skill` + `packages/skill/skill-filesystem` |
| Config-rich adapter | `packages/llm/llm-deepseek` |
| Keyless snapshot leaf (mounts a plugin tree and drives it) | `examples/headless-agent` |
| Patch overlay over the web profile | `examples/web-cordis/cordis.yml` |
