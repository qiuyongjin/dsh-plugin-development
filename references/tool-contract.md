# Tool `execute` contract

The contracts a model-facing tool must satisfy, inlined from the repository's
`docs/cookbook/adding-a-tool.md`. The minimal registration shape is template 3
in [plugin-templates.md](plugin-templates.md). `packages/shell/tool-bash` is
the production-grade example (three-package seam with
`packages/shell/shell` + `packages/shell/bash-local`).

## Rules of `execute()`

- **Args are validated for you.** `defineTool` validates model-generated
  `arguments` against the unified `ParameterSchemaSpec` before `execute` runs
  (types, required keys, literal constraints, exact-one unions, nested values),
  so inside `execute` the args match `InferArgs`. Explicit object nodes declare
  `additionalProperties: true | false`; the implicit parameter root stays open.
  You still hand-check constraints the DSL does not express (non-empty
  strings, positive numbers, cross-field rules). Raw JSON-Schema tools
  registered directly own their input validation.
- **Registration borrows your readonly definition.** Do not mutate the schema
  or replace callbacks after registration. To hot-swap a tool, dispose its
  owning effect and register the replacement; mutable state inside the
  callback's closure remains ordinary plugin state.
- **Execution identity is protected.** The registry materializes `arguments`
  as detached lossless JSON in one recursive pass, freezes it before policy
  starts, and assigns an opaque `exec.token`. `callId`, `name`, `arguments`,
  `agent`, `token`, the required caller-owned `signal`, and an optional
  enclosing-transport `parent` token stay immutable through dispatch. Treat
  `args` as readonly input. Only an around-dispatch wrapper receives a mutable
  view, and it may replace and restore `exec.signal` to impose a deadline but
  cannot remove it.
- **Declare and return one canonical JSON value.** `output.schema` uses
  `ValueSchemaSpec` and may have an object, array, scalar, or null root.
  `execute` returns only the inferred value; the registry snapshots it as
  lossless JSON, validates it, freezes it, and passes it to
  `output.render(args, value)`. Do not return content blocks from the body or
  make callers parse prose for ids and fields.
- **Throwing or returning an invalid value means `isError`.** The registry
  catches throws and contains schema, renderer, metadata-projector, and
  lossless-JSON failures before observers run. Throw for infrastructure
  failures. Represent a successful domain outcome in the canonical value even
  when its Native renderer explains a non-ideal state (a non-zero process
  exit).
- **Honor `exec.signal`.** Cancel in-flight work when it fires.
- **Project durable card data with `presentationMeta` (optional).**
  `output.presentationMeta(args, value)` derives replayable JSON from the same
  canonical value; the core persists it on `tool/result` and hands it to
  `presentResult`, so a card needing result-time facts survives replay without
  persisting the canonical value.
- **Use `exec.agent` for async notifications.** `agent.inject({ content,
  source: { kind: 'plugin', plugin: '<name>' } })` appends durable context the
  NEXT model request sees — it is not a wake-up. Guard against disposed agents.

## Execution policy and observation — pick the right gate

Prefer not to build deployment policy into the tool. The extension points
(`packages/core/tools/README.md` defines each one's inputs, order, return
values, and failure behavior):

- `tools/pre-execute` — extensible allow/deny/ask policy (waterfall).
- `ctx.tools.guard()` — a final monotonic deny that later listeners cannot undo.
- `tools/execute` — wrap the actual dispatch lifetime: deadline, retry,
  metrics. Only `exec.signal` is replaceable.
- `tools/post-execute` — replace presentation content or the returned value,
  block the result, or attach model-facing context.
- `tools/result` — observe the immutable normalized outcome.

## How your tool renders in a UI

UI cards are a separate concern from `output.render` (model-facing content),
declared through pure presentation projections and optional `presentCall` /
`presentResult` methods. Design them alongside the canonical value; a tool with
no presentation falls back to a generic card.

Both methods return a **`card`-tagged render intent**:

- `presentCall(args)` → the PENDING card:
  - `{ card: 'generic', title, kind?, rawInput?, content?, locations? }` — the
    default; set `kind` for an icon; `locations` for files the tool touches.
  - `{ card: 'terminal', title, description?, cwd? }` — the call IS a shell
    command (tool-bash).
  - `{ card: 'diff', title, diffs, locations? }` — creates/modifies a file;
    `diffs: [{ path, oldText, newText }]` with `oldText: null` for a new file.
- `presentResult(args, { content, isError, meta? })` — the completed card:
  `generic` (optional title/content), `terminal` (raw output + exit metadata),
  `diff` (applied hunks, often from `presentationMeta`), `search`
  (grouped-by-file matches `shape: 'matches'` or flat paths `shape: 'paths'`,
  plus `truncated`/`total`), `web` (discriminated `kind: 'search' | 'fetch'`
  from `result.meta`).

Hard rules:

- **Purity.** These run on live streaming AND on session-log replay — pure
  functions of `args` (+ the result): no I/O, no session state, no
  clock/random. UI state that a presenter needs belongs in durable
  `presentationMeta`, not in session reads.
- **UI-only formatting stays out of the model result.** Fenced console blocks,
  diffs, relativized paths — none of these belongs in the canonical value or
  Native content merely to serve a UI.
- **`defineTool` soft-validates the display path.** Malformed or older logged
  arguments make the wrapper return `undefined` (generic fallback) rather than
  throw — display must never crash a replay.

## Code Mode reaches your tool for free

In Code Mode every visible registered tool is available as
`await tools.<name>(args)` without extra integration: `ToolArgsMap` /
`ToolOutputMap` derive exact types from the same schemas, calls re-enter the
normal execution pipeline, a successful call resolves to the final canonical
JSON value after policy, and a failed call rejects with the real
`ToolCallError` (only `name`, `toolName`, and `message` are inspectable).
Design `output.schema` as a useful programmatic API.

## Long-running work

Gate `run_in_background` with producer config, then register through
`ctx.jobs.start({ kind, label, owner: exec.agent, run })`. The producer
supplies synchronous `cancel`, non-rejecting `done`, and optional bounded
`readOutput`. A successful background branch returns a typed canonical handle
such as `{ kind: 'background', jobId }`; its Native renderer may keep human
prose, but Code Mode must never parse that prose to recover the id. Once
`ctx.jobs.start()` publishes the id, use a task-owned cancellation signal
rather than `exec.signal`: later outer-call cancellation stops waiting but does
not kill published work (`job_kill`, owner disposal, and service teardown own
that lifetime).
