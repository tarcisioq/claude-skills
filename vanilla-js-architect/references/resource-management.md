# Resource Management

Long-lived SDKs hold resources: network connections, observers, timers, listeners, blobs. Without explicit cleanup discipline, they leak. This file is the playbook.

## The cleanup contract

Every SDK that holds resources exposes:

1. A way to clean up explicitly: `destroy()`, `close()`, `disconnect()`, or via `Symbol.dispose`/`Symbol.asyncDispose`
2. Documentation of *what* gets released and what calling-after-destroy does
3. Tests proving the cleanup actually works (see `testing-libraries.md`)

```javascript
/**
 * Closes the client and releases all resources.
 *
 * After destroy:
 * - All in-flight requests are aborted with code "DESTROYED"
 * - Event listeners attached to external targets are removed
 * - Timers are cleared
 * - Subsequent method calls reject with code "DESTROYED"
 *
 * Idempotent: calling destroy() multiple times is safe.
 */
async destroy() { /* ... */ }
```

## `Symbol.dispose` and `using` — the modern primitive

ES2024 explicit resource management. Supported in browsers (2024+), Node 22+, Deno, Bun. **Ship this for any SDK that holds resources.**

```javascript
// ✅ Async resource — most SDKs
class Subscription {
  #closed = false;

  async close() {
    if (this.#closed) return;
    this.#closed = true;
    await this.#stream.close();
  }

  async [Symbol.asyncDispose]() {
    return this.close(); // delegate to manual close
  }
}

// Consumer:
{
  await using sub = await sdk.subscribe("topic");
  for await (const msg of sub) handle(msg);
} // ← `await sub[Symbol.asyncDispose]()` runs automatically here, including on throw
```

```javascript
// ✅ Sync resource — file handles, connections in a sync API
class Lock {
  #released = false;

  release() {
    if (this.#released) return;
    this.#released = true;
    this.#mutex.release();
  }

  [Symbol.dispose]() { return this.release(); }
}

{
  using lock = mutex.acquire();
  doCriticalSection();
} // ← `lock[Symbol.dispose]()` runs here
```

### When to provide BOTH manual + symbol method

Always. Some consumers run on environments without `using` syntax support, others use `await using` consistently. Provide the manual method as the canonical name, then delegate from the Symbol method.

```javascript
class Client {
  destroy()                       { /* the canonical implementation */ }
  [Symbol.dispose]()              { return this.destroy(); }
  // For async: [Symbol.asyncDispose]() { return this.destroy(); }
}
```

### `DisposableStack` and `AsyncDisposableStack`

When a class manages multiple internal disposables, use the stack to bundle cleanup.

```javascript
// ✅ Composite resource — stack tracks every sub-resource
import { AsyncDisposableStack } from "node:disposable"; // or polyfill

class SDK {
  #stack = new AsyncDisposableStack();

  async init() {
    this.#stack.use(this.#openConnection());     // disposable 1
    this.#stack.use(this.#startHeartbeat());      // disposable 2
    this.#stack.defer(async () => {               // arbitrary cleanup
      await this.#flushQueue();
    });
  }

  async [Symbol.asyncDispose]() {
    await this.#stack.disposeAsync();
  }
}
```

The stack ensures **LIFO cleanup** even if any individual disposer throws — exactly what you want for transactional cleanup semantics.

## `AbortSignal` as the universal cancellation primitive

`AbortSignal` isn't just for `fetch`. It's the canonical way to broadcast "stop everything" through an SDK.

```javascript
// ✅ Internal "destroy" signal that cascades to everything
class Client {
  #destroyController = new AbortController();
  get #destroySignal() { return this.#destroyController.signal; }

  async track(event, { signal } = {}) {
    // Compose: respect external signal AND internal destroy signal
    const composed = AbortSignal.any([signal, this.#destroySignal].filter(Boolean));
    return this.#send(event, { signal: composed });
  }

  // Periodic timer that respects destroy
  #startHeartbeat() {
    const tick = async () => {
      while (!this.#destroySignal.aborted) {
        await this.#ping();
        await sleep(30_000, { signal: this.#destroySignal });
      }
    };
    tick().catch(() => {}); // suppress AbortError on destroy
  }

  destroy() {
    if (this.#destroyController.signal.aborted) return; // idempotent
    this.#destroyController.abort(new SDKError("client destroyed", "DESTROYED"));
  }
}
```

**Pattern:** every async loop, timer, retry, or queue-drain should `signal?.throwIfAborted()` periodically and pass `signal` into anything it calls.

## Listener cleanup — the #1 leak source

```javascript
// ❌ — listeners outlive the SDK
class Client {
  constructor() {
    window.addEventListener("online", this.#onOnline.bind(this));
    document.addEventListener("visibilitychange", () => this.#onVisibility());
  }
  // No destroy() removes them
}

// ✅ — abort signal removes all listeners at once
class Client {
  #destroyController = new AbortController();

  constructor() {
    const { signal } = this.#destroyController;
    window.addEventListener("online", this.#onOnline.bind(this), { signal });
    window.addEventListener("offline", this.#onOffline.bind(this), { signal });
    document.addEventListener("visibilitychange", this.#onVisibility.bind(this), { signal });
  }

  destroy() {
    this.#destroyController.abort();
    // ALL listeners attached with `{ signal }` are removed automatically
  }
}
```

`addEventListener` with `{ signal }` is the cleanest pattern. One `abort()` removes every listener registered with that signal. No bookkeeping.

## Timer cleanup

```javascript
// ❌ — leaks if SDK destroyed before tick fires
const t = setTimeout(() => doWork(), 5000);

// ✅ — use AbortSignal-aware sleep, or track timer for cleanup
class Client {
  #destroySignal;

  async waitAndDo() {
    await sleep(5000, { signal: this.#destroySignal });
    if (this.#destroySignal.aborted) return;
    doWork();
  }
}

// Or for setInterval-like:
function intervalWithSignal(fn, ms, { signal }) {
  const id = setInterval(() => {
    if (signal.aborted) return clearInterval(id);
    fn();
  }, ms);
  signal.addEventListener("abort", () => clearInterval(id), { once: true });
  return id;
}
```

## Observer cleanup

`IntersectionObserver`, `MutationObserver`, `ResizeObserver`, `PerformanceObserver` all leak unless `disconnect()`-ed.

```javascript
class Widget {
  #observers = [];

  start(element) {
    const io = new IntersectionObserver(/* ... */);
    io.observe(element);
    this.#observers.push(io);

    const mo = new MutationObserver(/* ... */);
    mo.observe(element, { childList: true });
    this.#observers.push(mo);
  }

  destroy() {
    for (const obs of this.#observers) obs.disconnect();
    this.#observers.length = 0;
  }
}
```

## `WeakRef` and `FinalizationRegistry` — for caches that shouldn't pin objects

```javascript
// ✅ Cache that lets large objects be GC'd
class WeakCache {
  #refs = new Map();
  #registry = new FinalizationRegistry(key => this.#refs.delete(key));

  set(key, value) {
    this.#refs.set(key, new WeakRef(value));
    this.#registry.register(value, key);
  }

  get(key) {
    return this.#refs.get(key)?.deref(); // undefined if collected
  }
}
```

**When to use:**
- Cache of large objects (DOM nodes, blobs, big buffers)
- The consumer can drop the original reference, and you want the cache cleaned up
- A miss is fine (cache is a hint, not authoritative)

**When NOT to use:**
- Primitives (numbers, strings) — can't be `WeakRef`'d
- Cache that must keep the value alive (use `Map`)
- Hot path — `WeakRef.deref()` is slower than `Map.get()`
- Anything you'd notice if randomly evicted

**Don't use `FinalizationRegistry` to free critical resources.** Finalizers run "eventually" or never. Use it for hints (cache cleanup, telemetry), not for closing connections.

## Idempotent destroy

```javascript
// ✅ — safe to call multiple times, including after partial init
class Client {
  #destroyed = false;
  #destroyController = new AbortController();

  async destroy() {
    if (this.#destroyed) return;
    this.#destroyed = true;
    this.#destroyController.abort();
    // Cleanup individual resources — guard each in case init partially failed
    try { await this.#flushQueue(); } catch {}
    try { this.#observer?.disconnect(); } catch {}
    try { this.#worker?.terminate(); } catch {}
  }
}
```

Idempotency is critical: consumers should never have to track "did I already destroy this?". Multiple calls are harmless.

## Throw-after-destroy

```javascript
// ✅ — every public method checks destroyed state
class Client {
  #destroyed = false;

  #ensureAlive() {
    if (this.#destroyed) {
      throw new SDKError("client has been destroyed", "DESTROYED");
    }
  }

  async track(event) {
    this.#ensureAlive();
    // ...
  }
}
```

Document this in the JSDoc `@throws` for every public method.

## Cleanup ordering — `try/finally` is mandatory

```javascript
// ❌ — if `flush` throws, subsequent cleanup is skipped
async destroy() {
  await this.#flushQueue();
  this.#observer.disconnect();
  this.#worker.terminate();
}

// ✅ — every cleanup step in its own guard
async destroy() {
  try { await this.#flushQueue(); } catch {}
  try { this.#observer.disconnect(); } catch {}
  try { this.#worker.terminate(); } catch {}
}

// ✅ — even better: AsyncDisposableStack does this for you
async destroy() {
  await this.#stack.disposeAsync(); // LIFO, isolated errors
}
```

## Page-lifetime cleanup hooks

Browsers don't always call `destroy()` for you when the page closes. Your SDK can't either — but you can hook page-level events for *opportunistic* flush.

```javascript
// ✅ Best-effort flush on page hide / unload
class Analytics {
  constructor() {
    this.#destroyController = new AbortController();
    const { signal } = this.#destroyController;

    document.addEventListener("visibilitychange", () => {
      if (document.visibilityState === "hidden") {
        this.#flushBeacon(); // sendBeacon: survives page unload
      }
    }, { signal });

    window.addEventListener("pagehide", () => this.#flushBeacon(), { signal });
  }

  #flushBeacon() {
    const data = JSON.stringify(this.#queue);
    this.#queue.length = 0;
    navigator.sendBeacon?.(this.#endpoint, data);
  }
}
```

`navigator.sendBeacon` is the right primitive for "fire and forget at page-end" — fetch can be cancelled mid-flight by unload, beacon survives.

**Don't use `beforeunload`** for cleanup — it's unreliable on mobile Safari and modern Chrome (BFCache). Use `pagehide` + `visibilitychange`.

## Quick Reference

| Resource type | Cleanup primitive |
|---------------|-------------------|
| Generic | `destroy()` + `[Symbol.dispose]` / `[Symbol.asyncDispose]` |
| Composite | `AsyncDisposableStack` for LIFO |
| Listeners | `addEventListener(..., { signal })` + abort the signal |
| Timers | Track id → clear in destroy; or `sleep(ms, { signal })` |
| Observers | `disconnect()` on every observer in destroy |
| Async loops | `while (!signal.aborted)` + `signal?.throwIfAborted()` |
| Cache (releasable) | `WeakRef` + `FinalizationRegistry` (hints only) |
| Cache (must hold) | `Map` |
| Page-end flush | `pagehide` + `visibilitychange` + `sendBeacon` |
| Re-entrant destroy | `if (#destroyed) return;` early |
| Calls after destroy | Throw `SDKError` with `code: "DESTROYED"` |
| Cleanup ordering | `try/catch` per step OR `AsyncDisposableStack` |
