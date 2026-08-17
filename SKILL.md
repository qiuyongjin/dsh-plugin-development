---
name: dsh-plugin-development
description: Use when developing, extending, or debugging plugins in the deepseek-harness repo — any new or modified Cordis plugin: model-facing tools (ctx.tools, defineTool), services and ctx.* keys, event listeners, providers and capability seams, prompt sections, commands, bundles, profile layers, or cordis.yml rows. Trigger whenever the task mentions "插件", "plugin", "扩展点", "extension point", "capability", "service", "tool registration", "cordis.yml", "bundle", "profile", "ctx.xxx", or "add a new tool/provider/backend", even if the user does not name the mechanism. Also use it before modifying the agent loop or registering a new session event, so the change attaches to a documented extension point instead.
---

# Developing a DeepSeek Harness plugin

Everything in dsh is a Cordis plugin: the model adapter, the tool registry, the
session log, and the agent loop are all plugins, so there is no privileged core
to patch — you extend the product by mounting a plugin beside the others. This
skill is self-contained: every rule it needs is inside this directory, in
`SKILL.md` and the `references/` files it points to.

> **Path convention.** This skill lives in the user-global `~/.claude/skills`
> directory, outside the repository. Repository paths appear in plain text
> (never as links) relative to the **deepseek-harness repository root** — the
> directory containing `AGENTS.md`, `packages/`, and `docs/` — so resolve them
> against that root while working in the checkout. The only links in this
> skill are internal.

## What this skill contains

- [references/plugin-templates.md](references/plugin-templates.md) — code
  templates for every plugin shape (function plugin, service, tool, event
  listener, session event, prompt section, cordis.yml rows, bundle/profile).
- [references/extension-points.md](references/extension-points.md) — the
  extension-point map, event domains and dispatch modes, capability seams,
  naming rules, client-side extension points.
- [references/package-checklist.md](references/package-checklist.md) — the
  package.json / tsconfig / registration / README / invariant checklist.
- [references/tool-contract.md](references/tool-contract.md) — the tool
  `execute` contract, policy gates, and render-intent rules.
- [references/testing-policy.md](references/testing-policy.md) — test tiers,
  REAL-composition requirement, snapshot duty, defensive patterns.

For details beyond these references, consult the repo-root files named in
plain text throughout (e.g. `docs/architecture.md`, `packages/AGENTS.md`); the
references are inlined from them, and the originals win when they disagree.

## Step 1 — Choose the plugin shape

| Shape | Use when | Form |
|---|---|---|
| Function plugin | Single-purpose behavior: a tool, a policy listener, a provider registration | named exports `name` / `inject` / `Config` / `apply`; **no default export** |
| Service subclass | You own a `ctx.<key>` and provide a capability to other plugins | `class X extends Service`, `static inject`, `super(ctx, key)`; default-export the class |
| Capability seam | A swappable capability (Definition / Provider / Consumer roles) | Definition owns the interface; split roles into packages only when they evolve independently |
| Client plugin | Browser Web Client surface | `packages/client/AGENTS.md` contract; `dsh.client` in package.json, `./client` export |

Templates for every shape: [references/plugin-templates.md](references/plugin-templates.md).
A function plugin with a default export is silently discarded by the Loader —
the rule and its postmortem are in the function-plugin template.

## Step 2 — Name and place it

- Pick the existing `packages/<group>/` group that matches the role (authoritative group table: `packages/README.md`); a new group is a pure container (no files of its own).
- Name the stable current responsibility: an interface package names the capability, an implementation adds its mechanism/vendor; `local` only when same-host execution is part of the contract.
- **Singular `ctx` key** for one engine/controller (`ctx.planMode`); **plural** for a registry (`ctx.tools`). Class role and key number must agree.
- Role suffixes (`Service` / `Registry` / `Provider` / `Backend` / `Controller` / `Handle` / ...): full table in
  [references/extension-points.md](references/extension-points.md#class-role-suffixes) — pick the honest role, never "Service because it extends Service".

## Step 3 — Write the skeleton

Copy the matching template from [references/plugin-templates.md](references/plugin-templates.md), then follow
[references/package-checklist.md](references/package-checklist.md) for the `package.json` invariants (`private: true`,
version matching root, `type: module`, `main`/`types`/`exports`, `@deepseek-ai/cordis` in both peer and dev dependencies at
the same range, `@deepseek-ai/schemastery` in `dependencies`, exact `files` list), the `tsconfig.json` (`extends
../../../tsconfig.base.json`, `rootDir: src`, `outDir: lib/types`, references for vendor/cosmokit, vendor/cordis (+
vendor/schemastery when you use Config), and each dsh dep), and registration in exactly one aggregate (`tsconfig.host.json` or
`tsconfig.client.json` — `api/remotes` is the one exception, do not copy it). Tests live at package level under `tests/`, not
`src/__tests__/`. In-package relative imports use explicit `.ts` specifiers; everything is ESM.

## Step 4 — Register on extension points

The extension-point map and event domains:
[references/extension-points.md](references/extension-points.md#where-new-behavior-goes).

Non-negotiable mechanics:

- **Registrations are effects.** Every contribution goes through `ctx.effect()` / `ctx.on()` / a `register()` that returns a
  disposer, so reload and teardown unwind them predictably. A registry contribution must prove disposal through the HMR-safety
  test (dispose the fiber, observe removal).
- **Declare dependencies with `inject`**; load order is expressed through service requirements, not boot sequencing. Optional
  services are read with `ctx.get(name)` inside call paths (the property proxy is topology-sensitive); optional children use
  `ctx.inject(['svc'], childCtx => ...)` so the child activates only when the service is composed.
- **Waterfall listeners MUST call `next()`** to delegate; returning without it short-circuits (right for a policy owner, wrong
  for an observer). `agent/turn-stopping` is serial and has no `next()`.
- **Model-visible means logged.** Anything reaching a model request must be reconstructable from the session log; a new
  model-visible input requires a session event (`SessionEventMap` extension + `session.append`). Tool results reach the model
  through `output.render`; UI rendering is a separate pure concern (`presentCall`/`presentResult` card intents).
- **Prefer events for interception and policy; prefer service methods for direct capability calls.**
- **No hardcoded tunables.** Deployment-varying choices are validated `Config` fields changeable from cordis.yml; a
  `DEFAULT_*` constant is not configurability. Protocol constants, external specs, and security invariants stay fixed.
- **Explicit defaulting at package boundaries** — a default is an explicit `resolve(request): Spec` step in the owning
  implementation, never a hidden `?? default` inside `run()`.
- **Opaque cross-boundary ids are branded** (`Branded<B>` from `dsh-brand`), never bare `string`.
- **Type-trusted at typed same-process boundaries** — no runtime validation for values the static interface already requires;
  validate at parser/config, queued, model/tool JSON, durable/file, worker, process, and wire boundaries.

## Step 5 — Configure

- Declare `Config` with schemastery (`import z from '@deepseek-ai/schemastery'`; `export const Config: z<Config> = z.object({...})`)
  and a `Config` interface; validate fail-loud at load (llm-retry's `validateConfig`, plan-mode's `resolveConfig`).
- Misconfiguration fails loud at load when self-contained, otherwise at the earliest resolvable point — never silently skip a
  missing referent.
- cordis.yml rows take `config:`; `!!js` expressions interpolate (`config` against the plugin context, `disabled` against the
  loader context). Other metadata stays literal — use overlays when the environment selects plugins.

## Step 6 — Mount and run

- **In-tree (workspace package):** add the row to the profile's composition or a patch overlay, then run from source with
  `pnpm dsh --profile headless "task"` (needs `DEEPSEEK_API_KEY`; the source path maps workspace packages to TypeScript).
  Inspect the composed tree with `dsh --profile web --dump-config` — any row it prints can be replaced by a patch of your own.
- **Out-of-tree:** publish the plugin as an npm package, install it into `$DSH_HOME/profiles/<name>/package.json`
  (default home `~/.dsh`), and either list it in the profile's `cordis.patch.yml` or package it as a **bundle**
  (`"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }`) and list it in `dsh.profile.bundles`. Bare plugin specifiers
  resolve from the config directory; external callers without the Loader's native helper must install plugins where plain
  Node resolution finds them.
- Patch semantics: an id-targeted patch replaces the targeted row's **whole** config (restate kept fields); `insert:` adds
  rows. Layer order: bundles → profile `cordis.patch.yml` → home `cordis.patch.yml` → `--patch` overlay. A patch naming an
  absent id is a stderr warning.

## Step 7 — Test and verify

- Match evidence to the surface: focused vitest unit tests (`pnpm run test`), the per-file 100% coverage gate
  (`pnpm run test:coverage`), keyless snapshots for model- or user-visible output (`pnpm run test:snapshot`), real-API e2e
  (`pnpm run test:e2e`, self-skips without `DEEPSEEK_API_KEY`). Tier details:
  [references/testing-policy.md](references/testing-policy.md).
- **A product-visible plugin requires a non-unit REAL-composition test:** boot a test-only `cordis.yml` through the Loader
  and app/process; hand-built `ctx.plugin(...)` suites are insufficient. Mock only external services or nondeterministic
  inputs; assert model-visible, durable, or user-visible output. Keep opt-ins out of shipped defaults.
- Registry contributions prove disposal through the HMR-safety test.
- Full gates before push: `pnpm run doc-sync && pnpm run constraints && pnpm run typecheck && pnpm run lint && pnpm run build
  && pnpm run hygiene`; the pre-push skill (`dsh-pre-push-checks`, in the repo's `.agents/skills/`) selects the minimal
  relevant set.

## Step 8 — Document

- Package README documents service API, config keys, events, extension points, and design notes first; ends with the
  canonical Model Experience section (request context / what the model sees / token effect / KV cache effect) and
  `## Known Limitations and Deferred Work` (or a justified allowlist entry). README and JSDoc are part of the change.
  Structure: [references/package-checklist.md](references/package-checklist.md#5-package-readme).
- Non-trivial changes carry an Agent Note in the same PR; only mechanical/local edits are exempt (rules in the repo's
  `.agents/notes/README.md`).
- Every package owns `./invariant`: register the manifest name and check an owned event-stream or mutable-data relationship
  (or give empty installers a package-specific `No runtime invariant:` reason).
- Update affected docs together (subsystem pages for spine/seam vocabulary, config catalog is generated).

## Correctness checklist (the rules that bite)

- [ ] Function plugin: named exports only, **no default export** (Loader discards the namespace).
- [ ] All registrations return disposers; disposal is effect-based.
- [ ] Waterfall listeners call `next()` unless they own the decision.
- [ ] Model-visible inputs are logged session events; nothing reaches the model that replay cannot reconstruct.
- [ ] Optional services read via `ctx.get(name)`, not the property proxy.
- [ ] Config validated at load; no hardcoded deployment tunables.
- [ ] Tool `execute`: args validated upstream, readonly input, canonical JSON return, honor `exec.signal`, throw ⇒ `isError`,
      UI render intents are pure functions of args.
- [ ] Single `ctx` key, honest role suffix, package in the matching group.
- [ ] REAL-composition test for product-visible behavior; disposal (HMR-safety) test for registry contributions.
- [ ] README + JSDoc updated in the same change; Agent Note for non-trivial changes; `./invariant` owned.

## Reference implementations

| Pattern | Package |
|---|---|
| Function plugin + waterfall recovery + durable events | `packages/llm/llm-retry` |
| Service subclass + session events + prompt section + optional children + tool | `packages/plan/plan-mode` |
| Tool trio (Definition / provider / tool Consumer) | `packages/shell/shell` + `packages/shell/bash-local` + `packages/shell/tool-bash` |
| Provider registry | `packages/skill/skill` + `packages/skill/skill-filesystem` |
| Config-rich adapter | `packages/llm/llm-deepseek` |
| Mounted plugin tree, driven keyless | `examples/headless-agent` |
| Patch overlay over the web profile | `examples/web-cordis/cordis.yml` |
