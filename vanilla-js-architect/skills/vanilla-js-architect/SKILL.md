---
name: vanilla-js-architect
description: Designs and implements browser-side SDKs, libraries, and frameworks in vanilla JavaScript. Use when building distributable JS packages, public API surfaces, plugin systems, or framework primitives — focuses on architecture, encapsulation, extensibility, and shippable distribution (ESM/UMD/IIFE). Not for Node.js services, TypeScript projects, or UI framework code (React/Vue/Svelte).
license: MIT
metadata:
  author: https://github.com/tarcisioq

  version: "2.2.0"
  domain: architecture
  triggers: SDK, library, framework, vanilla JavaScript, public API, plugin system, class design, ESM bundle, UMD, browser SDK, JS architecture
  role: architect
  scope: design-and-implementation
  output-format: code
---

# Vanilla JS Architect

## Skip This Skill If — Hard Gate

**Stop reading and refuse to engage if 2+ of these match the project.** This skill is for browser-side library authors; loading it on a backend, app, or pre-built UI wastes ~5–7K tokens before any source is read and produces guidance that doesn't translate.

| # | Programmatic tell | Why it disqualifies |
|---|-------------------|---------------------|
| 1 | `package.json` has **no `exports` field AND no `browser` field AND no `module` field** | Project isn't shipping a browser-consumable JS library — it's a service or app. |
| 2 | Source imports `express`, `fastify`, `koa`, `hapi`, `@nestjs/*`, `next`, `nuxt`, `astro`, `vite/server`, `node:http`/`node:https` for the public surface | Project is a server framework or full-stack app, not a client library. |
| 3 | **No bundler config** at the repo root (no `vite.config.*`, `rollup.config.*`, `webpack.config.*`, `tsup.config.*`, `esbuild.config.*`, `tsconfig.build.json`) | Nothing is being bundled for distribution; this is application code. |
| 4 | `package.json` `type` is unset or `"commonjs"` AND there is no `module`/`main`/`exports` ESM entry | Not shipping ESM, won't benefit from any of this skill's distribution rules. |
| 5 | Production runtime is `pm2` / `pm2-runtime` / `node <entry>` directly (no static-asset CDN, no `<script>` consumers) | This is a long-running server process, not a redistributable library. |
| 6 | The user's task is reviewing or refactoring an existing **Node.js service or backend API** | Wrong skill — even if the codebase has a couple of JS modules that look library-shaped, the prevailing concern is server discipline. Use the `solid` skill instead. |

**If you matched 2 or more:** stop. Reply briefly: *"This skill targets browser-side SDKs/libraries. The project signals a {server / app / pre-built-UI} project — `solid` covers cross-language engineering discipline (incl. cancellation in Q8) without the distribution rules that don't apply here. Want me to switch?"* Then wait for the user.

**If exactly 1 matches:** proceed but flag the friction (`"only Anti-pattern #9 / Rule 4 will likely fire — others are SDK-specific"`) so the user can redirect.

The "When NOT to Use" section below repeats this in prose for context after the gate. The gate above is the operative check.

## How to Use This Skill

This skill is structured for **lookup**, not linear reading. The flow:

1. Read this `SKILL.md` end-to-end once at the start of a task
2. Apply the **Core Workflow** to clarify what you're building
3. When you hit a specific decision, load the matching **reference file** (table below)
4. Before declaring done, run the **Review Checklist** at the end of this file

If you find yourself writing code without having loaded the relevant reference, stop and load it. Generic JS knowledge from training data tends to produce plausible-looking-but-wrong SDK code.

## When to Use This Skill

- Building a JavaScript SDK that wraps an HTTP/WebSocket API for browser consumers
- Designing a client library or framework primitive (router, state container, observable, etc.)
- Creating a plugin/extension system with lifecycle hooks
- Refactoring procedural JS into a coherent class-based or factory-based architecture
- Defining a public API surface (options pattern, fluent interface, event-driven)
- Preparing a JS package for distribution (ESM/UMD/IIFE bundles, `package.json exports`, tree-shaking)

## When NOT to Use

(Hard gate above is authoritative; this is the prose context.)

- Node.js backend services → use a backend-focused skill (`solid` for cross-language discipline including cancellation)
- TypeScript projects → use a TS-specific skill (this skill ships types via JSDoc, but the source is `.js`)
- UI framework code (React, Vue, Svelte components) → use `frontend-design` or a framework skill
- One-off scripts or application-internal utilities → overkill

## Top 5 Rules — Read This First

These are the rules that fire on almost every real SDK review. If you're short on time, anchor here before the Anti-Pattern Gallery and Decision Tree below.

| # | Rule | Tell to detect |
|---|------|----------------|
| 1 | **Throw typed errors with stable `code`** | `throw "..."`, `throw new Error("...")` without code, or `instanceof Error` checks at call sites — define `class SDKError extends Error` with `code` (Anti-pattern #3) |
| 2 | **Propagate `AbortSignal` through every long-running async path AND every layer of `await` between the public boundary and the I/O leaf** | Any `async` function that awaits, polls, retries, or schedules without an `{ signal }` option — and any internal `await fetch(...)`/`await sub-call(...)` without forwarding the signal (Anti-pattern #9) |
| 3 | **Named exports + `sideEffects` declared** | `export default { ... }` aggregates, missing `sideEffects` field in `package.json`, or imports running side-effects at module top — kill tree-shaking (Anti-patterns #2, #5) |
| 4 | **Clone consumer-provided objects before reading** | `options.x ??= default`, `Object.assign(opts, ...)`, mutating arrays/objects passed in — silent surprise for the caller (Anti-pattern #4) |
| 5 | **Async surface consistent across an API** | Two methods on the same class doing the same conceptual work but one returns `Promise` and the other doesn't — confuses consumers and prevents reliable awaiting (Anti-pattern #13) |

After these, the rest of the document covers full anti-pattern gallery, less-frequent smells, and references for deep dives.

## Anti-Patterns Gallery

These look correct but aren't. The training-data signal points at them — actively reject them.

### 1. Sometimes-sync, sometimes-async

```javascript
// ❌ — caller can't `await` reliably; cache hit returns plain value, miss returns Promise
function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  return fetch(`/u/${id}`).then(r => r.json());
}
```

**Tell:** non-`async` function that returns a Promise on some paths and a value on others.
**Fix:** declare `async` so every path returns a Promise; predictability beats the micro-optimization of avoiding the cache-hit microtask.

### 2. Default-export everything

```javascript
// ❌ — kills tree-shaking, locks consumers into one shape
export default { Client, SDKError, createClient, version: "1.0.0" };
```

**Tell:** `export default {` aggregating multiple symbols, or any `index.js` whose default export is an object literal.
**Fix:** use granular named exports (`export { Client } from "./client.js"`); reserve `default` for the single main SDK class when an `import Foo from "lib"` ergonomics is genuinely valuable.

### 3. Throwing strings or plain `Error`

```javascript
// ❌ — consumer can't switch on it, can't `instanceof`, no stack
throw "rate limited";
throw new Error("rate limited"); // marginally better, still no `code`
```

**Tell:** `throw <string>`, `throw new Error(...)` without a code, or call sites switching on `error.message` text.
**Fix:** define `class SDKError extends Error` with a stable `code` string + optional `details` (`throw new SDKError("rate limited", "RATE_LIMITED", { details: { retryAfter: 5 } })`); export the class so consumers can `instanceof` and switch on `code`.

### 4. Mutating consumer-provided options

```javascript
// ❌ — mutates caller's object; surprise side-effect
function createClient(options) {
  options.baseUrl ??= "https://api.example.com";
  return new Client(options);
}
```

**Tell:** `??=`, `||=`, `Object.assign(opts, ...)`, `arr.push(...)`, or any write to an argument the consumer passed in.
**Fix:** clone before reading (`const options = Object.freeze({ baseUrl: "...", ...userOptions })`); deep-clone arrays and nested objects too — `structuredClone(input)` for anything beyond a flat options bag.
**Scope:** SDK consumers (public API surface). Internal middleware in a server framework — Express `req.body`, Koa `ctx.state`, Fastify `request` — operates under different ownership: those objects ARE middleware-scoped scratch and mutating them is idiomatic. This rule fires when the object crossed the public API boundary, not when it's a runtime-internal scratchpad. (If you're reviewing internal middleware, you're probably in the wrong skill — see the Hard Gate at the top.)

### 5. Polyfill imported at module top

```javascript
// ❌ — every consumer ships the polyfill, even if they don't need it
import "core-js/stable";
export class Client { /* ... */ }
```

**Tell:** any side-effect import (`import "..."` without a binding) at module top, especially polyfills, registration calls, or auto-init logic.
**Fix:** put the side-effect behind a separate subpath (`"./polyfilled": "./dist/polyfilled.mjs"`, `"./auto-init": "./dist/auto-init.mjs"`) so consumers opt in; document required globals in the README. Mark the main entry `sideEffects: false` and the side-effect entries `sideEffects: ["./dist/auto-init.mjs"]`.

### 6. Positional args for many parameters

```javascript
// ❌ — adding `debug` later breaks every caller; positions become hieroglyphs
function createClient(apiKey, baseUrl, timeout, retries, debug, plugins) {}
createClient("k", undefined, 5000, undefined, false); // unreadable
```

**Tell:** > 2 positional parameters, or call sites passing `undefined` to skip an arg.
**Fix:** options object with destructured defaults (`function createClient({ apiKey, baseUrl, timeout = 5000, retries = 3 } = {}) {}`); adding a new option later is non-breaking for every existing caller.

### 7. `console.log` left in production code

```javascript
// ❌ — pollutes consumer console, can't disable
class SDK {
  track(event) {
    console.log("tracking", event);
    this.#queue.push(event);
  }
}
```

**Tell:** `console.log`/`info`/`debug` reachable from a public method without a debug gate; or a minifier configured with `drop_console: true` (silently kills `console.error` too).
**Fix:** gate verbose logging behind a `debug` flag (`if (this.#debug) console.debug("[my-sdk]", ...args)`) or inject a `logger` hook the consumer controls. For minifier `drop_console`, use `pure_funcs: ["console.log","console.info","console.debug"]` so `console.error`/`warn` survive.

### 8. Async function that throws synchronously

```javascript
// ❌ — non-async wrapper throws BEFORE returning the Promise; consumer's `.catch()` never sees it
function fetch(id) {
  if (!id) throw new SDKError("id required", "INVALID_ARG");
  return doFetchAsync(id);
}
```

**Tell:** non-`async` function that returns a Promise on the happy path but throws synchronously on input validation.
**Fix:** mark the wrapper `async` so any `throw` becomes a rejection (`async function fetch(id) { if (!id) throw new SDKError(...); return doFetchAsync(id); }`); consumers' `.catch()` and `try/await` then handle every failure mode uniformly.

### 9. Long-running async without `AbortSignal`

This is the highest-leverage SDK rule — consumer cancellation isn't optional. It applies to **every long-running async shape**, not just poll loops. Three concrete variants:

```javascript
// ❌ a — poll loop with no cancellation; consumer can't stop it on unmount
async function poll(url) {
  while (true) {
    await fetch(url);
    await sleep(1000);
  }
}

// ❌ b — public method accepts `signal`, but doesn't forward it to internal awaits → only the entry call cancels, the inner ones complete and leak
async function get(id, { signal } = {}) {
  signal?.throwIfAborted();
  const data = await fetch(`/u/${id}`); // signal not propagated
  return await postProcess(data);        // signal not propagated
}

// ❌ c — `setTimeout`-based "timeout" that races a Promise but doesn't cancel the underlying work (still runs, still consumes resources)
function withTimeout(promise, ms) {
  return Promise.race([promise, sleep(ms).then(() => { throw new Error("timeout"); })]);
}
```

**Tell:** any of: `async` function with awaits/loops/retries that doesn't accept `{ signal }`; method that accepts `signal` but doesn't forward it to inner `fetch`/`sleep`/sub-calls; "timeout" implemented with `Promise.race` instead of `AbortSignal.timeout`/`AbortSignal.any`.

**Tell — multi-layer (#9b sharpened):** when reviewing, **trace the signal across every layer of `await` between the public boundary and the I/O leaf**, not just the leaf. Walk the call graph: starting from the public method, follow each `await` and mark each intermediate function — any one that doesn't take a `signal` (or `{ signal }`) parameter silently disables cancellation downstream. A single missing layer makes every layer below it uncancellable, even if the leaf accepts a signal. Common shape: controller → service → adapter → fetch — the adapter often forgets to forward.

**Fix:** thread `AbortSignal` through every async layer from the public method to the I/O leaf (`fetch(url, { signal })`, `sleep(ms, { signal })`, every sub-method takes `{ signal }` in its options bag); compose multiple sources with `AbortSignal.any([consumer, AbortSignal.timeout(ms), this.#destroyController.signal])`; rely on `signal.throwIfAborted()` at top-of-method and reject naturally inside loops. See `references/async-and-cancellation.md`.

### 10. Plugin system without a contract

```javascript
// ❌ — plugins are arbitrary functions; no name, no lifecycle, no dedup
sdk.use(req => { req.headers.foo = "bar"; return req; });
```

**Tell:** `sdk.use(fn)` accepting bare functions with no required shape; no way to dedup, replace, or order plugins; no errors when a "plugin" returns the wrong type.
**Fix:** declare a `BasePlugin` contract with required `name` + named lifecycle hooks (`beforeRequest`, `afterResponse`, `onError`, `destroy`); validate the shape on `use()` and reject duplicates by `name`. See `references/sdk-architecture.md`.

### 11. Leaking Node.js APIs into browser builds

```javascript
// ❌ — `process`, `Buffer`, `require` blow up in browsers
const apiKey = process.env.API_KEY; // ReferenceError in browser
const buf = Buffer.from("x"); // ReferenceError
```

**Tell:** any reference to `process`, `Buffer`, `require`, `__dirname`, `fs`/`path`/`os` imports, or `globalThis.process` checks at module top.
**Fix:** use Web Standards (`new TextEncoder().encode("x")` for bytes, `crypto.subtle` for hashing, options-passed-by-consumer for config); for genuinely runtime-conditional code use feature detection (`typeof structuredClone === "function"`), never UA sniffing. See `references/cross-runtime.md`.

### 12. Mixing CommonJS and ESM without `exports` map

```javascript
// ❌ — package.json with "main" pointing at CJS and "module" at ESM, no `exports`
// Consumers' bundlers pick inconsistently; dual-package hazard creates duplicate state
```

**Tell:** `package.json` has `main` + `module` but no `exports` field; or `exports` exists but ships both CJS and ESM versions of the same module without isolation.
**Fix:** make `exports` the single source of truth (`{ ".": { "import": "./dist/index.mjs", "default": "./dist/index.mjs" } }`); when supporting CJS, version any module-scoped state via a separate file shared between both builds, never duplicated.

### 13. Inconsistent async surface within a single API

```javascript
// ❌ — the three `set*` methods do the same conceptual work (write to storage),
//      but two return Promise<void> and the third returns nothing/undefined.
//      Consumer code that does `await storage.setSession(id)` works for cookie but silently
//      no-ops the await for localStorage; switching backing stores becomes a breaking change.
class Storage {
  setSessionCookie(id) {
    document.cookie = `sid=${id}`;
    // returns undefined (fire-and-forget)
  }
  setSessionLocal(id) {
    localStorage.setItem("sid", id);
    // returns undefined (fire-and-forget)
  }
  async setSessionIDB(id) {
    await idb.put("sessions", id, "current");
    // returns Promise<void>
  }
}
```

**Tell:** two or more methods on the same class doing the same conceptual operation but with different return shapes — some sync/`undefined`, some `Promise`. Or: a method documented as async but whose only `await` is conditional (so it returns sync on the fast path).
**Fix:** make the *whole surface* async if any path is async (return `Promise<void>` from every variant, even when the underlying API is synchronous — the cost is one microtask). Document the contract in JSDoc and stick to it; the consumer should be able to swap backing stores without changing call sites. If sync is genuinely required for some calls (e.g. perf-critical hot path), name the methods differently (`setSessionSync` vs `setSession`) so consumers can't accidentally await a no-op.

## Decision Tree — High-Level Design

Use this *before* writing code. Resolves the ambiguous questions that lead to inconsistent output.

### Q1: Class or factory for the entry point?

| Consumer needs | Choose |
|----------------|--------|
| `instanceof` checks, subclassing, identity tracking | **Class** |
| Hidden implementation, swappable returned shape, no `new` keyword | **Factory** |
| Both ergonomics — class-like usage but want to control construction | **Factory wrapping internal class** |
| Truly stateless utility | **Plain named exports** (no class, no factory) |

### Q2: Sync, async, or both?

| The operation… | Choose |
|----------------|--------|
| Reads from an in-memory cache that *might* miss → fetch | **Always `async`** (never sometimes-sync) |
| Pure computation, no I/O, no awaits inside | **Sync** |
| Could complete synchronously *and* needs to await | **Always `async`** — predictability beats micro-optimization |
| Long-running with progress | **Async generator** (`async function*`) |

### Q3: How to signal failure?

| Failure characteristic | Choose |
|------------------------|--------|
| Programmer error (invalid args, misuse) | **Throw `SDKError` with `code`** |
| Network/IO failure in async path | **Reject Promise with `SDKError` subclass** |
| Both success and failure carry useful payload | **Return Result-style `{ ok, value, error }`** |
| Operation may legitimately have no result | **Return `null`** (never `undefined` for documented absence) |

Never throw strings, never throw plain `Error`, never sometimes-throw-sometimes-return-error.

### Q4: Multi-instance or singleton?

Default to **multi-instance via factory**. Only consider singleton if:
- The resource is genuinely process-wide (e.g. `navigator.locks`)
- AND consumers can never legitimately want two

If singleton, gate behind `getShared()` + `resetShared()` so tests can isolate.

### Q5: Inheritance or composition?

Default to **composition**. Inheritance only when:
- You need `instanceof` for ecosystem reasons (e.g. extending `EventTarget`, `Error`)
- AND the subclass relationship is genuine "is-a", not "has-a"

Never go more than one level deep.

## Core Workflow

1. **Clarify the consumer contract.** Who calls this code? What's the entry point (`new SDK()`, `createClient()`, `import { fn }`)? Which runtimes (browsers? Workers? Deno?) and bundlers must it support? Read existing `package.json` (`exports`, `browserslist`, `sideEffects`, `type`) before designing
2. **Design the public surface first.** Sketch the API shape (constructor signature, public methods, events, options object) *before* writing internals. Consumer ergonomics drive structure. Apply the Decision Tree above
3. **Architect for extension.** Decide which extension points exist (plugins, hooks, subclassing, event listeners) and which internals stay private (`#fields`, module-scoped closures, `WeakMap`)
4. **Implement.** Apply patterns from the reference files. Keep the public surface minimal and the internals replaceable. Pass `AbortSignal` through every async path
5. **Validate distribution.** Bundle size budget, tree-shake check, no leaked Node globals (`process`, `Buffer`), no `eval`/`new Function`/dynamic `require`. ESM verified to load in a real browser
6. **Smoke-test the built artifact.** After any change that touches `package.json`, `rollup.config.js`/bundler, exports map, minifier flags, or entry-point splitting: load the built file (`node --input-type=module -e 'import("./dist/index.mjs").then(m => console.log(Object.keys(m)))'` or open the UMD in a browser) and verify named exports, error subclasses, and any side-effect-free imports actually work. Tree-shake assumptions are false until empirically proven on the artifact you ship
7. **Test the contract.** Tests against the public API only. Cover: happy path, error path, plugin/hook integration, cancellation, memory leak, and at least one edge case per public method
8. **Run the Review Checklist** before declaring done

## Reference Guide

Load detailed guidance based on what you're about to do. The "Load when you see…" column is the operational tell.

| Topic | Reference | Load when you see… |
|-------|-----------|---------------------|
| SDK architecture | `references/sdk-architecture.md` | Designing classes, factories, plugin systems, lifecycle hooks, event-driven APIs, state machines |
| API design | `references/api-design.md` | Defining public method signatures, options pattern, fluent interfaces, error contracts, JSDoc, deprecation |
| Distribution | `references/distribution.md` | Bundling (ESM/UMD/IIFE), `package.json exports`, sideEffects, tree-shaking, browser targets, CDN, polyfills |
| Async + cancellation | `references/async-and-cancellation.md` | `AbortSignal` composition, retry/backoff, request dedup, concurrency limits, timeouts, async iteration, multi-layer signal trace |
| Idiomatic syntax | `references/idiomatic-syntax.md` | Choosing between `#private`/`WeakMap`/closure, `WeakRef`, `Symbol.dispose`/`using`, `structuredClone`, modern array methods |
| Browser primitives | `references/browser-primitives.md` | Storage abstraction, `postMessage` protocol, capability detection, `IntersectionObserver`/`MutationObserver` for SDK consumers |
| Consumer ergonomics | `references/consumer-ergonomics.md` | Subpath exports, deep import policy, import maps, naming for `import` paths |
| Testing libraries | `references/testing-libraries.md` | Writing tests for the SDK itself: contract tests, plugin tests, memory leak detection, bundle regression |
| Security | `references/security.md` | CSP-safe code, prototype pollution in option merge, `postMessage` origin, SRI, provenance, secret handling |
| Types from JSDoc | `references/types-from-jsdoc.md` | Shipping `.d.ts` from `.js` source via `tsc --emitDeclarationOnly --allowJs`, `@typedef`, `@template`, `@overload` |
| Resource management | `references/resource-management.md` | `dispose()`/`destroy()` discipline, `Symbol.dispose`/`using`, `FinalizationRegistry`, `WeakRef` cache, abort cascade |
| Observability | `references/observability.md` | Debug flag pattern, injectable telemetry/logger, `performance.mark`/`measure` namespacing, error reporting hooks |
| Release engineering | `references/release-engineering.md` | Changesets, dist-tags (`beta`/`next`), deprecation cadence, CHANGELOG discipline, `npm publish --provenance` |
| Cross-runtime | `references/cross-runtime.md` | Targeting browsers + Workers + Deno + Bun + Cloudflare Workers; Web Standards (`Request`, `URL`, `crypto.subtle`) vs Node-only APIs |

## Constraints

### MUST DO
- Treat the public API as a contract — changes are semver-breaking, design deliberately the first time
- Encapsulate internal state with `#private` fields, closures, or `WeakMap` — never expose mutable internals
- Provide a single, well-documented entry point per build target via `exports`
- Use named exports for tree-shakability; reserve `default` exports for the main SDK class only
- Accept an options object (not positional arguments) for any function with more than 2 parameters
- Surface errors via typed custom error classes (`class SDKError extends Error`) with stable `code` strings
- Make every long-running async operation cancellable via `AbortSignal` and propagate the signal **across every layer of `await`** — not just the I/O leaf
- Emit lifecycle events (`init`, `ready`, `error`, `destroy`) for any stateful SDK
- Document every public method with JSDoc (`@param`, `@returns`, `@throws`, `@example`)
- Keep the bundle small: lazy-load heavy features via dynamic `import()`, mark `sideEffects: false` when applicable
- Use Web Standards (`Request`, `Response`, `URL`, `crypto.subtle`, `TextEncoder`) for cross-runtime portability

### MUST NOT DO
- Leak Node.js APIs (`process`, `Buffer`, `fs`, `require`) into browser/cross-runtime builds
- Pollute global scope (`window.X = ...`) unless the build is explicitly an IIFE/UMD global
- Use `var`, `with`, `eval`, or `new Function()` (CSP-hostile, breaks bundlers)
- Mutate consumer-provided objects (options, callbacks, data) at the public API surface — clone defensively
- Throw synchronously from API surfaces shaped as async
- Ship code that requires polyfills the consumer didn't opt into; document required globals in README
- Couple the public API to a specific bundler, build tool, or framework
- Mix CommonJS and ESM in the same package without an `exports` map
- Use callback-only APIs for new code (provide Promise-returning versions)
- Block the main thread (heavy CPU → Web Worker; heavy I/O → batched/streamed)
- Leave `console.log` calls in production paths — gate behind debug flag or telemetry hook

## Output Templates

When implementing an SDK feature, deliver:

1. **Public API module** — entry point consumers `import` from, with named exports and JSDoc on every export
2. **Internal modules** — implementation split by concern (transport, queue, errors, plugins), not exposed in the package's `exports` map
3. **Custom error classes** — one base class + specific subclasses or `code` constants
4. **Tests against the public contract** — happy path, error path, cancellation, plugin/hook behavior, memory leak; never test private fields directly
5. **Usage example** — a short snippet in the JSDoc `@example` block or a README excerpt showing the canonical "first 5 minutes" experience
6. **Distribution checklist** — confirm `package.json` `exports`, `sideEffects`, `browserslist`, and bundle size targets are appropriate

## Review Checklist (run before declaring done)

Apply this list to every change. If any item is "no" or "not sure", fix before declaring complete.

### Public surface
- [ ] Every public method/export has JSDoc with `@param`, `@returns`, `@throws` (with codes), `@example`
- [ ] No positional args beyond 2 params; rest in options object
- [ ] Async APIs always return Promise (never sometimes-sync)
- [ ] All errors thrown/rejected are `SDKError` (or subclass) with stable `code` string
- [ ] Long-running async accepts and propagates `AbortSignal` **across every layer of `await`** between the public boundary and the I/O leaf
- [ ] No `console.log` in production paths (debug flag or telemetry hook only)

### Encapsulation
- [ ] Internal state is `#private`, closure-scoped, or in a `WeakMap`
- [ ] Internal modules not reachable via `exports` map
- [ ] Consumer-provided objects (options, payloads) are cloned, never mutated at the public API surface
- [ ] No global scope pollution (no `window.X = ...`) unless explicit IIFE/UMD build

### Cross-runtime safety
- [ ] No Node-only APIs in browser/cross-runtime paths (`process`, `Buffer`, `require`, `fs`)
- [ ] Web Standards used where possible (`fetch`, `Request`, `URL`, `crypto.subtle`, `TextEncoder`)
- [ ] No `eval` / `new Function` / `with` (CSP-safe)
- [ ] No top-level side effects (no I/O, no globals, no function calls at module top)

### Distribution
- [ ] `package.json` has `type: "module"`, `exports` map, `sideEffects` declared
- [ ] Tree-shaking verified (test app importing one export drops the rest)
- [ ] Bundle size within budget; lazy-load gates heavy features
- [ ] Source maps shipped
- [ ] Required globals (if any) documented in README

### Tests
- [ ] Contract tests (against public API, not internals)
- [ ] Cancellation test (abort mid-operation, verify cleanup propagates across all layers, not just the leaf)
- [ ] Memory leak test (long-running SDK doesn't grow unbounded)
- [ ] Plugin/hook integration test (if extension points exist)
- [ ] At least one edge-case test per public method

### Versioning + release
- [ ] `metadata.version` bumped per semver (PATCH/MINOR/MAJOR)
- [ ] CHANGELOG entry written
- [ ] If breaking: deprecation warnings issued in previous version, migration guide in README
- [ ] If new feature: example added to README
