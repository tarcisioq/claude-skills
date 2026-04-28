# Testing Libraries

Testing an SDK is **fundamentally different** from testing application code. The unit under test is the *public contract*, not the implementation. Internal refactors must not break tests; internal bugs that escape the public contract must be impossible.

## Test pyramid for SDKs

```
                    ┌──────────────────────────┐
                    │   1-2 e2e in real browser │  Playwright / Vitest browser mode
                    │   (smoke + integration)   │  — slow, expensive, irreplaceable
                    └──────────────────────────┘
                  ┌──────────────────────────────┐
                  │   Contract tests (the bulk)  │  Vitest / Node test runner
                  │   — public API, all paths   │  — fast, the source of truth
                  └──────────────────────────────┘
              ┌──────────────────────────────────────┐
              │   Internal unit tests (sparse)        │  Only for non-trivial pure
              │   — only for complex pure helpers     │  logic with edge cases
              └──────────────────────────────────────┘
```

**Inversion vs application code:** in apps, you write many unit tests + few integration. In SDKs, **most tests target the public contract** (effectively integration-level), few are internal. Reason: internal tests bind to internal structure; refactoring breaks them.

## Contract test discipline

```javascript
// ✅ Test the documented public behavior; agnostic to internals
import { describe, it, expect, vi } from "vitest";
import { Client, SDKError } from "my-sdk";

describe("Client.fetch", () => {
  it("returns parsed JSON on 200", async () => {
    const fetch = vi.fn().mockResolvedValue(new Response('{"id":1}'));
    const client = new Client({ apiKey: "k", fetch });
    expect(await client.fetch("/x")).toEqual({ id: 1 });
  });

  it("rejects with SDKError code=HTTP_ERROR on non-2xx", async () => {
    const fetch = vi.fn().mockResolvedValue(new Response("oops", { status: 500 }));
    const client = new Client({ apiKey: "k", fetch });
    await expect(client.fetch("/x")).rejects.toMatchObject({
      name: "SDKError",
      code: "HTTP_ERROR",
    });
  });

  it("propagates AbortSignal — rejects with the signal's reason", async () => {
    const controller = new AbortController();
    const fetch = vi.fn().mockImplementation((_, { signal }) => new Promise((_, reject) => {
      signal.addEventListener("abort", () => reject(signal.reason));
    }));
    const client = new Client({ apiKey: "k", fetch });
    const promise = client.fetch("/x", { signal: controller.signal });
    queueMicrotask(() => controller.abort(new Error("user cancelled")));
    await expect(promise).rejects.toThrow("user cancelled");
  });
});
```

**Anti-pattern: testing private fields.**

```javascript
// ❌ Brittle — refactor renaming `#queue` to `#events` breaks the test
expect(client._queue.length).toBe(1);

// ✅ Observe via the contract — what does a consumer see?
await client.flush();
expect(serverReceived).toHaveLength(1);
```

## Inject dependencies — never mock at module level

The SDK's constructor accepts seams (fetch, storage, clock) so tests can substitute deterministic ones. **No `vi.mock("fs")` or `jest.mock("./transport")`.** Module mocks are coupling-by-stealth.

```javascript
// ✅ Constructor injection
export class Client {
  constructor({
    apiKey,
    fetch = globalThis.fetch,
    storage = defaultStorage(),
    now = () => Date.now(),
  } = {}) { /* ... */ }
}

// In tests:
const client = new Client({
  apiKey: "k",
  fetch: vi.fn(),
  storage: memoryStorage(),
  now: () => 1_700_000_000_000,
});
```

This is also good design — it benefits production (consumers can swap transports for their own runtime).

## The contract test matrix

For every public method, verify these axes:

| Axis | What to test |
|------|--------------|
| Happy path | Canonical inputs → expected output |
| Error path | Each documented `code` produces the right `SDKError` subclass |
| Cancellation | `AbortSignal` mid-operation → reject with signal reason, no cleanup leak |
| Idempotency | Calling twice in row produces consistent state (or documented behavior) |
| Concurrency | Two parallel calls don't corrupt internal state |
| Boundary | Empty/null/max-sized inputs |
| Plugin/hook | If extension points exist, test plugin install + hook invocation |
| Lifecycle | `destroy()` after operations is clean; calling after destroy throws documented `code` |

If a public method lacks tests on any axis above, add them before declaring done.

## Cross-environment testing

```javascript
// ✅ Vitest browser mode — runs the same tests in a real browser
// vitest.config.js
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: {
    browser: {
      enabled: true,
      provider: "playwright",
      instances: [
        { browser: "chromium" },
        { browser: "firefox" },
        { browser: "webkit" },
      ],
    },
  },
});
```

```bash
# Cross-runtime sanity — same test file, multiple runtimes
node --test tests/contract.test.js
deno test tests/contract.test.js
bun test tests/contract.test.js
```

If your SDK claims cross-runtime support, you must run the contract suite in each runtime. CI matrix is non-optional.

## Memory leak detection

Long-lived SDKs accumulate listeners, timers, observers, and cached objects. Without explicit tests, leaks ship.

```javascript
// ✅ Pattern: create N instances, destroy them, verify no growth
import { describe, it, expect } from "vitest";

describe("Client lifecycle", () => {
  it("destroy() releases all references", async () => {
    if (!globalThis.gc) return; // run with `node --expose-gc` or `vitest --expose-gc`

    const refs = new Set();
    for (let i = 0; i < 100; i++) {
      const c = new Client({ apiKey: "k" });
      refs.add(new WeakRef(c));
      c.destroy();
    }

    // Force GC and yield to let it run
    globalThis.gc();
    await new Promise(r => setTimeout(r, 50));
    globalThis.gc();

    const alive = [...refs].filter(ref => ref.deref() !== undefined);
    expect(alive.length).toBe(0);
  });

  it("destroy() removes all event listeners from external EventTargets", () => {
    const target = new EventTarget();
    const before = countListeners(target); // see helper below
    const client = new Client({ apiKey: "k", externalTarget: target });
    client.destroy();
    expect(countListeners(target)).toBe(before);
  });
});

// Helper: track listener count via spying on add/remove
function countListeners(target) {
  if (!target.__listeners) {
    target.__listeners = 0;
    const origAdd = target.addEventListener.bind(target);
    const origRem = target.removeEventListener.bind(target);
    target.addEventListener = (...args) => { target.__listeners++; origAdd(...args); };
    target.removeEventListener = (...args) => { target.__listeners--; origRem(...args); };
  }
  return target.__listeners;
}
```

**Even simpler smoke check:** instantiate-and-destroy a few times, watch heap snapshots in DevTools. CI doesn't catch every leak; periodic manual inspection helps.

## Bundle size regression

```json
// package.json
{
  "scripts": {
    "size": "size-limit",
    "test:size": "size-limit --silent"
  },
  "size-limit": [
    { "path": "dist/index.mjs",         "limit": "10 kB" },
    { "path": "dist/index.iife.js",     "limit": "12 kB", "gzip": true },
    {
      "name": "Client only (tree-shaken)",
      "path": "dist/index.mjs",
      "import": "{ Client }",
      "limit": "6 kB"
    }
  ]
}
```

Run on CI on every PR. The third entry above is the **most important**: it verifies tree-shaking actually works by importing one symbol and measuring the resulting bundle.

```yaml
# .github/workflows/size.yml
- run: npm run test:size
- uses: andresz1/size-limit-action@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

Bot comments on PRs with size diff. Catches accidental imports of large utilities.

## Tree-shaking validation

```javascript
// tests/tree-shaking.test.js
import { build } from "esbuild";
import { describe, it, expect } from "vitest";

describe("tree-shaking", () => {
  it("importing only Client drops other named exports", async () => {
    const result = await build({
      stdin: {
        contents: `import { Client } from "../dist/index.mjs"; new Client({ apiKey: "k" });`,
        resolveDir: import.meta.dirname,
      },
      bundle: true,
      minify: true,
      format: "esm",
      write: false,
    });
    const code = result.outputFiles[0].text;
    expect(code).not.toMatch(/SDKError/);
    expect(code).not.toMatch(/loggerPlugin/);
  });
});
```

## Plugin / hook integration tests

```javascript
// ✅ Verify plugins install, hooks run in order, plugins compose
describe("plugin system", () => {
  it("hooks run in install order", async () => {
    const calls = [];
    const sdk = new Client({ apiKey: "k" });
    sdk.use({ name: "a", beforeRequest: (req) => { calls.push("a"); return req; } });
    sdk.use({ name: "b", beforeRequest: (req) => { calls.push("b"); return req; } });

    await sdk.fetch("/x");
    expect(calls).toEqual(["a", "b"]);
  });

  it("rejects duplicate plugin names", () => {
    const sdk = new Client({ apiKey: "k" });
    sdk.use({ name: "a" });
    expect(() => sdk.use({ name: "a" }))
      .toThrow(expect.objectContaining({ code: "DUPLICATE_PLUGIN" }));
  });

  it("plugin error in beforeRequest rejects the operation", async () => {
    const sdk = new Client({ apiKey: "k" });
    sdk.use({ name: "x", beforeRequest: () => { throw new Error("plugin boom"); } });
    await expect(sdk.fetch("/x")).rejects.toThrow("plugin boom");
  });
});
```

## Time and randomness — make them seams

```javascript
// ✅ Inject `now` and `random` so tests are deterministic
export class RetryQueue {
  constructor({ now = () => Date.now(), random = Math.random } = {}) {
    this.#now = now; this.#random = random;
  }

  // Use this.#now() instead of Date.now() everywhere internally
}

// In tests — fake clock
let time = 0;
const queue = new RetryQueue({
  now: () => time,
  random: () => 0.5, // deterministic jitter
});
```

`vi.useFakeTimers()` works for `setTimeout`/`setInterval` but not `Date.now()` reliably across runtimes. Inject the seam.

## What NOT to test

- Private fields directly (covered above)
- Implementation file structure (`expect(transport.constructor.name).toBe("HttpTransport")`)
- Error messages verbatim (test `code`, not human-facing strings — they change)
- Third-party libraries (don't re-test `fetch`)
- Trivial getters/setters with no logic

## Test file layout

```
package/
├── src/
│   └── client.js
├── tests/
│   ├── contract.test.js          ← public API, the bulk of tests
│   ├── lifecycle.test.js         ← destroy, GC, listener cleanup
│   ├── plugins.test.js           ← extension point tests
│   ├── tree-shaking.test.js      ← bundle composition tests
│   ├── cross-runtime.test.js     ← run in node + deno + bun
│   └── e2e.browser.test.js       ← real-browser smoke (Playwright)
├── tests-internal/               ← OPTIONAL: only for non-trivial pure helpers
│   └── backoff.test.js
└── vitest.config.js
```

`tests-internal/` is sparse on purpose. Most tests live in `tests/`.

## Quick Reference

| Concern | What to do |
|---------|-----------|
| Test public, not internal | Tests import from package root; never reach into `src/` |
| Inject dependencies | Constructor seams (`fetch`, `storage`, `now`); no module mocks |
| Test all axes | Happy, error code, cancellation, lifecycle, plugin, boundary |
| Cross-runtime claim → CI matrix | Vitest browser, Deno, Bun, Node; same test file |
| Detect leaks | `WeakRef` after destroy + `globalThis.gc()` |
| Bundle regression | `size-limit` on each PR; include tree-shake import test |
| Tree-shaking proof | `esbuild` programmatic build + check exports dropped |
| Time/random | Inject seams; never `vi.useFakeTimers()` for SDKs |
| What NOT to test | Internals, error strings, file structure, third-party code |
