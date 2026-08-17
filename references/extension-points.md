# Extension points and naming reference

Reference tables for the dsh plugin developer. This file is a working index;
the authoritative sources are the repo-root files named in plain text
(`docs/architecture.md`, `docs/glossary.md`) — consult them when this index
and the code disagree.

## Where new behavior goes

From `docs/architecture.md` § Where new behavior goes. New behavior attaches
to a documented extension point; changing the agent loop itself updates that
map in the same PR.

| Goal | Mechanism |
|---|---|
| Add a model provider | register its adapter on `ctx.llm` |
| Add a model-facing capability | register on `ctx.tools`; its schema joins prompt assembly |
| Give one session a different capability set | compose an agent preset; a service row there needs an `isolate` realm |
| Add shell execution | register a `ctx.shell` backend; the local one spawns through `ctx.subprocess` |
| Add persistent terminal execution | register a `ctx.terminals` backend plus `dsh-tool-terminal` |
| Add a human command | register on `ctx.commands`; it dispatches without a model turn |
| Add background work | register on `ctx.jobs`; `job_*` tools collect or stop it |
| Add filesystem access or policy | register a `ctx.fs` provider or listen to `fs/*` events |
| Confine spawned processes | use a `ctx.sandbox` backend; consumers wrap argv before spawning |
| Intercept a request, tool, or turn | use its `agent/*` or `tools/*` event; `agent/turn-stopping` stops a turn |
| Add model-facing context | call `agent.inject()`; it lands in the next admitted request |
| Add UI or editor integration | drive `ctx.agents` and render from `session/event` |
| Add a Web Client Chat node | register a `ConversationNodeDefinition` + keyed renderer |
| Add durable session state | extend `SessionEventMap`; render and replay from the log |
| Generate session titles | register the sole `ctx.sessionTitle` provider |
| Manage a same-session objective | use `ctx.goals`; continue through `agent/*` |
| Fork a live session | `ctx.sessions.fork(source, boundary?, childSessionId?)` |
| Scope a registration to one agent | use that agent's `agent.ctx` |

`docs/cookbook/extension-cookbook.md` maps product features to these
mechanisms with minimal code shapes — consult it for a feature you recognize.

## Event domains

- **Session events** are durable facts appended to the session log and broadcast
  through `session/event`. Use one when the fact must survive a reload, fork, or
  replay. Extend `SessionEventMap` (declaration merging on
  `@deepseek-ai/dsh-session/types`); a member is required-on-read by default —
  builds that do not know its type refuse the log unless the envelope carries
  `ignorable: true`.
- **Agent events** (`agent/*`) carry a live `Agent` and are declared in
  `packages/core/agent/src/runtime-types.ts`: `agent/created`, `agent/disposed`,
  `agent/status`, `agent/error`, `agent/inbox/inserted`, `agent/inbox/claimed`,
  `agent/inbox/discarded`, `agent/session-start` (all `emit`),
  `agent/pre-step`, `agent/request`, `agent/request-error` (all `waterfall`),
  and `agent/turn-stopping` (`serial`). Use one to observe or intercept work in
  flight.
- **Capability events** attach policy and adapters to a seam (`fs/*`,
  `tools/*`, `telemetry/*`) without importing the loop.

`docs/event-producer-consumer.md` lists every event's producers and consumers.
Event JSDoc needs an `@mode` tag naming the dispatch mode.

## Dispatch modes

From `docs/cordis-primer.md` § Dispatch modes.

| Mode | Awaited? | Dispatch Order | Has Return Value? |
|---|---|---|---|
| `emit` | No | listeners observe in registration order | No |
| `waterfall` | No | listeners observe in registration order | Yes |
| `parallel` | Yes | all listeners observe the event in parallel | No |
| `serial` | Yes | listeners observe in registration order | Yes |

`ctx.waterfall` is around-middleware: a listener receives `(...args, next)`;
call `next()` to delegate the possibly wrapped result to the next listener,
return without it to short-circuit. For single-decision events,
short-circuiting is the design (a policy listener owns the decision); a
listener that only annotates or observes must delegate. Use `prepend: true`
only when the listener must run before ordinary registrations.

Known waterfalls: `agent/pre-step`, `agent/request`, `agent/request-error`,
`llm/stream`, and the three `tools/*` gates (`tools/pre-execute`,
`tools/execute`, `tools/post-execute`). `agent/turn-stopping` is serial and
has no `next()`.

## Capability seams

A seam is a swappable capability with three roles (see `docs/architecture.md`
§ Capability seams and `docs/capability-seams.md`):

1. **Service Definition** — declares the interface (typically a `Service`
   subclass owning one `ctx.<key>`).
2. **Service Provider** — implements it behind the interface.
3. **Consumer** — uses it, commonly a model-facing tool.

One role alone is not a seam; adding a capability means designing all three.
Split roles into separate packages only when they evolve independently
(`docs/cookbook/adding-a-package.md` § Decide the package topology); a
single-purpose plugin stays one package. A capability's registration is
usually a **registry** pattern: the Definition owns the registry, providers
register into it, consumers read the merged view (`ctx.skills`,
`ctx.subagents`, `ctx.tools` are the shipped examples).

## Naming rules

### Package names and groups

- Every package is `@deepseek-ai/dsh-<name>`; vendored packages are rescoped.
- Choose an existing group in `packages/` when one matches the role — the
  authoritative group list, roles, and release expectations live in the group
  table of `packages/README.md` (groups include `core`, `llm`, `shell`,
  `subprocess`, `terminal`, `fs`, `lsp`, `skill`, `web`, `compaction`,
  `context`, `subagent`, `bundle`, `workflow`, `todo`, `plan`, `preset`,
  `guard`, `hooks`, `session`, `identity`, `settings`, `credentials`, `acp`,
  `interaction`, `boot`, `sdk`, `examples`, `util`, `client`, and more). A new
  group is a pure container: no `package.json`, no source files.
- Name the stable current responsibility, not the first implementation or a
  possible future expansion. `local` only when same-host execution is part of
  the contract.

### `ctx` keys

- **Singular** key for one engine, runtime, policy, controller, resolver,
  store, or current configuration (`ctx.planMode`).
- **Plural** key for a registry or a service owning multiple named members
  (`ctx.tools`, `ctx.subagents`, `ctx.terminals`).
- The class role and key number must agree. Do not reuse one key for
  incompatible host and client declarations (declaration merging sees both
  faces). Add a role suffix when the natural plural already belongs to another
  face.

### Class role suffixes

Full table in `docs/cookbook/adding-a-package.md` § Name the role that
exists. Frequent picks:

| Word | Use it when | Do not use it when |
|---|---|---|
| `Service` | It owns a cohesive domain service that no sharper role states honestly | The name exists only because the class extends Cordis `Service` |
| `Registry` | It owns a dynamic set of named registrations, including lookup, precedence, lifetime, and disposal | Its main contract is dispatch, execution, cancellation, policy, or orchestration |
| `Provider` | It supplies one implementation of a capability definition | It is the capability definition, provider registry, or consumer runtime |
| `Backend` | It implements replaceable lower-level persistence, transport, or execution | It is a user-facing service or one returned live-resource reference |
| `Controller` | It accepts commands or user intent and changes one existing domain state | It executes arbitrary work or owns a provider fleet |
| `Handle` | It refers to one live resource and controls or observes that resource | It creates and manages the complete resource pool |

## Model-visible means logged

Anything that reaches a model request must be reconstructable from the session
log; a new model-visible input requires a new session event. A runtime
invariant asserts it. Prompt sections and tool schemas are assembled from what
plugins registered; tool results reach the model through `output.render` and
`tools/result`.

## Client-side extension points (Web app)

- **UI slots** — `ctx.slots` (`@deepseek-ai/dsh-client-ui-slots`): plugin A
  declares a slot key by merging into `SlotMap` and declaring `children`
  (single / list / chain); plugin B registers a component into that key.
  Occupying the built-in `root` slot is reserved for `ui-layout`; use
  `shell.overlay` for additive surfaces.
- **Chat business nodes** — register a `ConversationNodeDefinition` and a
  `conversation.chat.node` keyed renderer (`docs/cookbook/adding-a-conversation-node.md`
  is the guide).
- **Client plugin contract** — `packages/client/AGENTS.md`: client plugins
  declare `dsh.client` in package.json, export `./client`, and use the shared
  tsdown preset.

## Where the API lives

- `ctx.tools` — tool registry, guarded execution pipeline, `guard()` /
  `restrict()`; `packages/core/tools/README.md` defines each extension point
- `ctx.systemPrompt` — prompt-section assembly; `section()` with ordering
- `ctx.agents` / `ctx.agentLoop` — agent lifecycle and events
- `ctx.sessions` — durable session log and fork (`ctx.sessionProjections`
  owns projection units)
- `ctx.llm` — model adapters and streaming
- `ctx.skills`, `ctx.subagents`, `ctx.shell`, `ctx.terminals`, `ctx.fs`,
  `ctx.sandbox`, `ctx.jobs`, `ctx.commands`, `ctx.goals`, `ctx.compaction`,
  `ctx.sessionTitle` — capability seams
- Generated catalogs: `docs/config-catalog.md` (every plugin config field),
  `docs/tool-catalog.md`, `docs/event-producer-consumer.md`
