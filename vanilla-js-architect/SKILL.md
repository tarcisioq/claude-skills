---
name: vanilla-js-architect
description: Designs and implements browser-side SDKs, libraries, and frameworks in vanilla JavaScript. Use when building distributable JS packages, public API surfaces, plugin systems, or framework primitives — focuses on architecture, encapsulation, extensibility, and shippable distribution (ESM/UMD/IIFE). Not for Node.js services, TypeScript projects, or UI framework code (React/Vue/Svelte).
license: MIT
metadata:
  author: https://github.com/tarcisioq
  version: "2.0.0"
  domain: architecture
  triggers: SDK, library, framework, vanilla JavaScript, public API, plugin system, class design, ESM bundle, UMD, browser SDK, JS architecture
  role: architect
  scope: design-and-implementation
  output-format: code
---

# Vanilla JS Architect

Specialist for designing and shipping browser-side SDKs, libraries, and frameworks in pure JavaScript. Optimized for code that *other developers consume* — emphasis on stable APIs, clear extension points, small bundle size, and predictable behavior across browsers and Web-Standard runtimes (browsers, Workers, Deno, Bun, Cloudflare Workers).

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

- Node.js backend services → use a backend-focused skill
- TypeScript projects → use a TS-specific skill (this skill ships types via JSDoc, but the source is `.js`)
- UI framework code (React, Vue, Svelte components) → use `frontend-design` or a framework skill
- One-off scripts or application-internal utilities → overkill

## Core Workflow

1. **Clarify the consumer contract.** Who calls this code? What's the entry point (`new SDK()`, `createClient()`, `import { fn }`)? Which runtimes (browsers? Workers? Deno?) and bundlers must it support? Read existing `package.json` (`exports`, `browserslist`, `sideEffects`, `type`) before designing
2. **Design the public surface first.** Sketch the API shape (constructor signature, public methods, events, options object) *before* writing internals. Consumer ergonomics drive structure. Apply the Decision Tree below
3. **Architect for extension.** Decide which extension points exist (plugins, hooks, subclassing, event listeners) and which internals stay private (`#fields`, module-scoped closures, `WeakMap`)
4. **Implement.** Apply patterns from the reference files. Keep the public surface minimal and the internals replaceable. Pass `AbortSignal` through every async path
5. **Validate distribution.** Bundle size budget, tree-shake check, no leaked Node globals (`process`, `Buffer`), no `eval`/`new Function`/dynamic `require`. ESM verified to load in a real browser
6. **Test the contract.** Tests against the public API only. Cover: happy path, error path, plugin/hook integration, cancellation, memory leak, and at least one edge case per public method
7. **Run the Review Checklist** before declaring done

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

## Anti-Patterns Gallery

These look correct but aren't. The training-data signal points at them — actively reject them.

### 1. Sometimes-sync, sometimes-async

```javascript
// ❌ — caller can't `await` reliably; cache hit returns plain value, miss returns Promise
function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  return fetch(`/u/${id}`).then(r => r.json());
}

// ✅ — always async; predictable
async function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  const r = await fetch(`/u/${id}`);
  return r.json();
}
```

### 2. Default-export everything

```javascript
// ❌ — kills tree-shaking, locks consumers into one shape
export default { Client, SDKError, createClient, version: "1.0.0" };

// ✅ — named exports, granular
export { Client } from "./client.js";
export { SDKError } from "./errors.js";
export { createClient } from "./factory.js";
```

### 3. Throwing strings or plain `Error`

```javascript
// ❌ — consumer can't switch on it, can't `instanceof`, no stack
throw "rate limited";
throw new Error("rate limited"); // marginally better, still no `code`

// ✅ — typed error with stable code
throw new SDKError("rate limited", "RATE_LIMITED", { details: { retryAfter: 5 } });
```

### 4. Mutating consumer-provided options

```javascript
// ❌ — mutates caller's object; surprise side-effect
function createClient(options) {
  options.baseUrl ??= "https://api.example.com";
  return new Client(options);
}

// ✅ — clone defensively, freeze internally
function createClient(userOptions = {}) {
  const options = Object.freeze({
    baseUrl: "https://api.example.com",
    ...userOptions,
  });
  return new Client(options);
}
```

### 5. Polyfill imported at module top

```javascript
// ❌ — every consumer ships the polyfill, even if they don't need it
import "core-js/stable";
export class Client { /* ... */ }

// ✅ — separate opt-in entry; document required globals in README
// package.json:
// "exports": { ".": "./dist/index.mjs", "./polyfilled": "./dist/polyfilled.mjs" }
```

### 6. Positional args for many parameters

```javascript
// ❌ — adding `debug` later breaks every caller; positions become hieroglyphs
function createClient(apiKey, baseUrl, timeout, retries, debug, plugins) {}
createClient("k", undefined, 5000, undefined, false); // unreadable

// ✅ — options object
function createClient({ apiKey, baseUrl, timeout = 5000, retries = 3 } = {}) {}
```

### 7. `console.log` left in production code

```javascript
// ❌ — pollutes consumer console, can't disable
class SDK {
  track(event) {
    console.log("tracking", event);
    this.#queue.push(event);
  }
}

// ✅ — gated by debug flag, default off; or telemetry hook injected by consumer
class SDK {
  #debug;
  constructor({ debug = false } = {}) { this.#debug = debug; }
  track(event) {
    if (this.#debug) this.#log("tracking", event);
    this.#queue.push(event);
  }
  #log(...args) { console.debug("[my-sdk]", ...args); }
}
```

### 8. Async function that throws synchronously

```javascript
// ❌ — looks fine, but consumer might do `sdk.fetch().catch(handle)` expecting rejection
async function fetch(id) {
  if (!id) throw new SDKError("id required", "INVALID_ARG"); // becomes rejection — OK
  // BUT: outside `async`, raw `throw` before any `await` happens synchronously
}

// In a non-async wrapper:
function fetch(id) {
  if (!id) throw new SDKError("id required", "INVALID_ARG"); // ❌ sync throw from "async-shaped" API
  return doFetchAsync(id);
}

// ✅ — keep the throw inside an async function so it always rejects
async function fetch(id) {
  if (!id) throw new SDKError("id required", "INVALID_ARG");
  return doFetchAsync(id);
}
```

### 9. Long-running async without `AbortSignal`

```javascript
// ❌ — consumer can't cancel, can't time out, leaks if component unmounts
async function poll(url) {
  while (true) {
    await fetch(url);
    await sleep(1000);
  }
}

// ✅ — accept and propagate signal
async function poll(url, { signal } = {}) {
  while (!signal?.aborted) {
    signal?.throwIfAborted();
    await fetch(url, { signal });
    await sleep(1000, { signal });
  }
}
```

### 10. Plugin system without a contract

```javascript
// ❌ — plugins are arbitrary functions; no name, no lifecycle, no dedup
sdk.use(req => { req.headers.foo = "bar"; return req; });

// ✅ — plugins are named objects with declared hooks; SDK validates and calls them
sdk.use({
  name: "auth",
  beforeRequest(req) { req.headers.Authorization = `Bearer ${this.token}`; return req; },
});
```

### 11. Leaking Node.js APIs into browser builds

```javascript
// ❌ — `process`, `Buffer`, `require` blow up in browsers
const apiKey = process.env.API_KEY; // ReferenceError in browser
const buf = Buffer.from("x"); // ReferenceError

// ✅ — use Web Standards
const apiKey = options.apiKey; // pass via constructor
const buf = new TextEncoder().encode("x"); // Uint8Array
```

### 12. Mixing CommonJS and ESM without `exports` map

```javascript
// ❌ — package.json with "main" pointing at CJS and "module" at ESM, no `exports`
// Consumers' bundlers pick inconsistently; dual-package hazard creates duplicate state

// ✅ — single source of truth via `exports`
{
  "type": "module",
  "exports": { ".": { "import": "./dist/index.mjs", "default": "./dist/index.mjs" } }
}
```

## Reference Guide

Load detailed guidance based on what you're about to do. The "Load when you see…" column is the operational tell.

| Topic | Reference | Load when you see… |
|-------|-----------|---------------------|
| SDK architecture | `references/sdk-architecture.md` | Designing classes, factories, plugin systems, lifecycle hooks, event-driven APIs, state machines |
| API design | `references/api-design.md` | Defining public method signatures, options pattern, fluent interfaces, error contracts, JSDoc, deprecation |
| Distribution | `references/distribution.md` | Bundling (ESM/UMD/IIFE), `package.json exports`, sideEffects, tree-shaking, browser targets, CDN, polyfills |
| Async + cancellation | `references/async-and-cancellation.md` | `AbortSignal` composition, retry/backoff, request dedup, concurrency limits, timeouts, async iteration |
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
- Make every long-running async operation cancellable via `AbortSignal` and propagate the signal everywhere
- Emit lifecycle events (`init`, `ready`, `error`, `destroy`) for any stateful SDK
- Document every public method with JSDoc (`@param`, `@returns`, `@throws`, `@example`)
- Keep the bundle small: lazy-load heavy features via dynamic `import()`, mark `sideEffects: false` when applicable
- Use Web Standards (`Request`, `Response`, `URL`, `crypto.subtle`, `TextEncoder`) for cross-runtime portability

### MUST NOT DO
- Leak Node.js APIs (`process`, `Buffer`, `fs`, `require`) into browser/cross-runtime builds
- Pollute global scope (`window.X = ...`) unless the build is explicitly an IIFE/UMD global
- Use `var`, `with`, `eval`, or `new Function()` (CSP-hostile, breaks bundlers)
- Mutate consumer-provided objects (options, callbacks, data) — clone defensively
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
- [ ] Long-running async accepts and propagates `AbortSignal`
- [ ] No `console.log` in production paths (debug flag or telemetry hook only)

### Encapsulation
- [ ] Internal state is `#private`, closure-scoped, or in a `WeakMap`
- [ ] Internal modules not reachable via `exports` map
- [ ] Consumer-provided objects (options, payloads) are cloned, never mutated
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
- [ ] Cancellation test (abort mid-operation, verify cleanup)
- [ ] Memory leak test (long-running SDK doesn't grow unbounded)
- [ ] Plugin/hook integration test (if extension points exist)
- [ ] At least one edge-case test per public method

### Versioning + release
- [ ] `metadata.version` bumped per semver (PATCH/MINOR/MAJOR)
- [ ] CHANGELOG entry written
- [ ] If breaking: deprecation warnings issued in previous version, migration guide in README
- [ ] If new feature: example added to README
