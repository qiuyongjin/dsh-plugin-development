# Testing and quality policy

The testing tiers and the rules that keep a green suite meaningful, inlined
from the repository's `docs/testing.md`, `packages/AGENTS.md`, and
`examples/AGENTS.md`. Commands live in the root `AGENTS.md`.

## Tiers

- **Unit** (`pnpm run test`): vitest over package and example specs under
  their `tests/**` directories plus repository script specs under
  `scripts/**/*.spec.ts`; tests stay with the code area they exercise. Every
  registry gets an HMR-safety test (dispose the contributing fiber, assert
  cleanup). Prefer edge cases, error paths, event ordering, concurrency races,
  and permanent tests for contract regressions.
- **Coverage gate** (`pnpm run test:coverage`): the gating run — per-file 100%
  on `packages/*/*/src`. An uncovered line is often dead code the gate is
  correctly flagging for deletion, not a missing test to bolt on. Line coverage
  is necessary, never sufficient.
- **Real-API e2e** (`pnpm run test:e2e`): with-key tests against live provider
  APIs; each suite self-skips without its key so keyless CI stays green.
- **Snapshot** (`pnpm run test:snapshot`): keyless expected outputs covering
  external behavior — transport contracts, presentation, and persisted logs.
  Re-record with `pnpm run test:snapshot:record` when a model transcript
  changes; `test:snapshot:refresh` when replay input remains valid. Review
  every JSONL and expected-output diff.

## The with-key policy

Do not ration real-API tests: a no-key test proves plumbing; only a with-key
run proves the agent works against a real model. The highest-value tests are
smokes that boot the real example, send one prompt, and check the world — they
catch the "green unit tests, broken product" class that mocks cannot. Every
example ships keyless and with-key smokes.

## Prefer the real implementation over a mock

Mock only the expensive or non-deterministic boundary (LLM adapter, network,
clock); keep everything downstream real. A hand-rolled stand-in proves the
bridge moves bytes, not that the shipping tool behaves as asserted.

## Verify the world, not the self-report

An e2e assertion re-runs the command or re-reads the file externally; a
keyword probe on the agent's own output lets a cheating agent pass. Assert
untouched files are byte-identical. e2e tests own their resources: create the
harness in the test, dispose in `afterEach` (even on failure/retry/timeout);
shared fixtures live in a plain `tests/harness.ts`, never another
`*.e2e.ts` (importing a spec re-registers its `describe` and duplicates real
API calls).

## Test the real entry path

- **Product-visible plugins require a non-unit REAL-composition test.**
  Hand-built `ctx.plugin(...)` suites are insufficient: boot test-only
  `cordis.yml` through the Loader and app/process, mock only external services
  or nondeterministic inputs, and assert model-visible request/log, durable
  state, or user-visible output. Keep opt-ins out of shipped defaults.
- A guard only guards if the regression actually fails it. For a plugin
  without `inject` (bundle/composition plugins), a Loader smoke stays green
  when a default export replaces the required named exports — add an explicit
  `expect('default' in mod).toBe(false)` plus an `unwrapExports` round-trip
  assertion, and prove it: introduce the regression, watch red, revert.
- "Real entry path" means the published artifact: a package `bin` runs built
  `lib/bin.js` under plain `node`, exposing failures tsx masks. Keep the
  built-artifact smokes green and assert a genuinely-missing config exits
  non-zero.

## Test resolution and subprocess launch modes

- Bare workspace imports resolve to `src` through the tsconfig paths, never
  through package `exports` to built `lib/` — stale artifacts there load a
  second copy of module singletons. Built artifacts are consumed only
  explicitly: `lib`-mode subprocesses and the built smokes.
- CI and build-having lanes run every example or Cordis-config subprocess from
  built `lib/` through the shared dual-mode launcher; do not hand-write
  `--import tsx` for these subprocesses. Protocol and OS fixtures that do not
  load Cordis run erasable `.ts` directly with Node. Only a test whose subject
  is source-path resolution may select `src`.

## When a snapshot test is required

Every non-trivial model-, protocol-, or human-visible change adds or updates a
keyless scenario in the same PR through a runnable example's owning snapshot
suite. Package tests, e2e assertions, mock-only fixtures, and PR rationale do
not substitute for the assembled application transcript; extend the snapshot
harness when needed. Fixtures must replay on macOS/Linux — fix fixtures, not
normalizers.

## Disposal and invariants

- Registry contributions prove disposal through the HMR-safety test: dispose
  the fiber and observe removal.
- Every package owns `./invariant` (see
  [package-checklist.md](package-checklist.md#6-every-package-owns-invariant)).
- Runtime invariants assert owned relationships: check authoritative event
  streams or mutable data, not service presence, plugin metadata or effects,
  or fixed pure examples.

## Defensive patterns

Before lifecycle, concurrency, subprocess, or teardown work, apply the
repository's `docs/defensive-patterns.md` classes (summarized):

1. **Report orthogonal outcomes independently** — success, failure, and
   cancellation of one operation are separate signals; do not conflate them.
2. **Honor public contracts on BOTH sides** — callers and callees each enforce
   the interface; a bug on one side must fail the other's assumptions loudly.
3. **Async state is not synchronous state** — check state at the point of use
   after every await; captured booleans go stale.
4. **Dispose must reach quiescence, not just request it** — await outstanding
   work and settle resources before returning from teardown.
5. **Contain callback exceptions in the dispatcher** — async errors from
   callbacks are caught and reported by the component that dispatched them,
   not thrown into unrelated paths.
6. **Never hand untrusted output the ambient environment or predictable
   paths** — quote/inject defensively; avoid argv, env, and well-known path
   injection surfaces.
7. **Unlink link-shaped paths** — a path that walks like a symlink or junction
   is not a plain file; resolve and validate before trusting it.
