# Async and Cancellation

SDK-specific async patterns. Generic JS async knowledge from training data is not enough — this file encodes what *consumers of an SDK* need: predictable cancellation, sane retry, no leaks.

## Universal cancellation: `AbortSignal` everywhere

Every long-running async API in the SDK accepts `signal` in its options. No exceptions.

```javascript
// ✅ Pattern: accept signal, propagate to fetch (and to anything async you call)
async fetch(path, { signal } = {}) {
  signal?.throwIfAborted(); // fail fast if already aborted
  const res = await fetch(`${this.#baseUrl}${path}`, { signal });
  if (!res.ok) throw new SDKError(`HTTP ${res.status}`, "HTTP_ERROR");
  return res.json();
}

// ❌ Pattern: no signal — consumer can't cancel, can't time out, leaks if component unmounts
async fetch(path) {
  const res = await fetch(`${this.#baseUrl}${path}`);
  return res.json();
}
```

### `AbortSignal.timeout(ms)` — built-in timeout

```javascript
// ✅ Built-in (browsers + Node 17.3+ + Workers + Deno + Bun) — no manual setTimeout dance
const data = await sdk.fetch("/x", { signal: AbortSignal.timeout(5000) });
```

### `AbortSignal.any([...])` — compose multiple signals

The most underused primitive in JS. Combines consumer's signal with an internal one (timeout, destroy lifecycle, etc.).

```javascript
// ✅ SDK adds its own deadline AND respects consumer's signal
async fetch(path, { signal } = {}) {
  const composed = AbortSignal.any([
    signal,                              // consumer-provided
    AbortSignal.timeout(this.#timeout),  // SDK's own deadline
    this.#destroySignal,                 // SDK's destroy() lifecycle
  ].filter(Boolean));
  return fetch(`${this.#baseUrl}${path}`, { signal: composed });
}

// Internal: AbortController fires when SDK is destroyed
class SDK {
  #destroyController = new AbortController();
  get #destroySignal() { return this.#destroyController.signal; }
  destroy() { this.#destroyController.abort(new SDKError("destroyed", "DESTROYED")); }
}
```

### Pre-aborted check — fail fast

```javascript
// ✅ Always check at entry — saves wasted work
async doWork({ signal } = {}) {
  signal?.throwIfAborted();
  // ...
}
```

### Polyfill check (if you support pre-2023 environments)

`AbortSignal.any` arrived in browsers Q3 2023. If your `browserslist` includes older browsers, polyfill or feature-detect:

```javascript
const anySignal = (signals) => {
  if (typeof AbortSignal.any === "function") return AbortSignal.any(signals);
  // Fallback
  const controller = new AbortController();
  const onAbort = (e) => controller.abort(e.target.reason);
  for (const s of signals) {
    if (s?.aborted) { controller.abort(s.reason); break; }
    s?.addEventListener("abort", onAbort, { once: true });
  }
  return controller.signal;
};
```

## Retry with exponential backoff + jitter

```javascript
// ✅ Full-jitter exponential backoff. Caps. Respects signal. Knows what's retryable.
async function retry(fn, {
  attempts = 3,
  baseMs = 200,
  capMs = 10_000,
  signal,
  isRetryable = (err) => err instanceof SDKError && ["NETWORK", "TIMEOUT", "RATE_LIMITED"].includes(err.code),
} = {}) {
  let lastErr;
  for (let i = 0; i < attempts; i++) {
    signal?.throwIfAborted();
    try {
      return await fn({ attempt: i });
    } catch (err) {
      lastErr = err;
      if (!isRetryable(err) || i === attempts - 1) throw err;

      // Honor server hint if present
      const hintMs = err.details?.retryAfter ? err.details.retryAfter * 1000 : null;

      // Full-jitter: random in [0, backoff]
      const backoff = Math.min(capMs, baseMs * 2 ** i);
      const delay = hintMs ?? Math.random() * backoff;

      await sleep(delay, { signal });
    }
  }
  throw lastErr;
}

// Cancellable sleep (better than `setTimeout` because aborts cleanly)
function sleep(ms, { signal } = {}) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(signal.reason);
    const t = setTimeout(resolve, ms);
    signal?.addEventListener("abort", () => {
      clearTimeout(t);
      reject(signal.reason);
    }, { once: true });
  });
}
```

**Anti-pattern: equal-jitter or no-jitter retries.** Without jitter, every client retries at the same time after a server hiccup → thundering herd → server stays down. Always full-jitter for SDKs.

**Anti-pattern: retrying on programmer errors.** `INVALID_OPTIONS`, `INVALID_ARGUMENT`, `AUTH_FAILED` should NOT retry. Be explicit about which `code`s are retryable.

## Request deduplication (in-flight coalescing)

When the same request fires N times concurrently, run it once and share the Promise.

```javascript
// ✅ Coalesce concurrent identical requests
class SDK {
  #inflight = new Map(); // key → Promise

  async getUser(id, { signal } = {}) {
    const key = `getUser:${id}`;

    if (this.#inflight.has(key)) {
      // Reuse the in-flight Promise — but still respect this caller's signal
      return this.#raceWithSignal(this.#inflight.get(key), signal);
    }

    const promise = this.#fetch(`/users/${id}`, { signal })
      .finally(() => this.#inflight.delete(key));

    this.#inflight.set(key, promise);
    return promise;
  }

  #raceWithSignal(promise, signal) {
    if (!signal) return promise;
    return new Promise((resolve, reject) => {
      promise.then(resolve, reject);
      signal.addEventListener("abort", () => reject(signal.reason), { once: true });
    });
  }
}
```

**Caveat:** dedup is correct only when the operation is **idempotent and has no side effects on the SDK's local state**. Don't dedup `track(event)` — each call must produce its own queued event.

## Concurrency limit (queue with backpressure)

Browsers cap fetches per origin (~6). Caller-side queueing avoids 1000s of pending requests.

```javascript
// ✅ Bounded concurrency. Order-preserving. Cancellable per task.
class ConcurrencyLimiter {
  #max;
  #running = 0;
  #queue = [];

  constructor(max = 6) { this.#max = max; }

  run(fn, { signal } = {}) {
    return new Promise((resolve, reject) => {
      const task = { fn, signal, resolve, reject };
      if (signal?.aborted) return reject(signal.reason);
      signal?.addEventListener("abort", () => {
        const idx = this.#queue.indexOf(task);
        if (idx !== -1) {
          this.#queue.splice(idx, 1);
          reject(signal.reason);
        }
      }, { once: true });

      this.#queue.push(task);
      this.#drain();
    });
  }

  async #drain() {
    while (this.#running < this.#max && this.#queue.length > 0) {
      const task = this.#queue.shift();
      if (task.signal?.aborted) {
        task.reject(task.signal.reason);
        continue;
      }
      this.#running++;
      try {
        task.resolve(await task.fn());
      } catch (err) {
        task.reject(err);
      } finally {
        this.#running--;
        this.#drain();
      }
    }
  }
}

// Usage
const limiter = new ConcurrencyLimiter(4);
const results = await Promise.all(
  urls.map(url => limiter.run(() => fetch(url), { signal }))
);
```

## Async iteration for streaming + pagination

```javascript
// ✅ Async generator — caller controls the rhythm via `for await`
async function* paginate(baseUrl, { signal } = {}) {
  let cursor = null;
  do {
    signal?.throwIfAborted();
    const url = cursor ? `${baseUrl}?cursor=${cursor}` : baseUrl;
    const res = await fetch(url, { signal });
    const page = await res.json();
    yield* page.items;
    cursor = page.nextCursor;
  } while (cursor);
}

// Consumer:
for await (const item of paginate("/api/items", { signal })) {
  process(item);
  if (someCondition) break; // generator's `finally` runs cleanup
}
```

**Why generators beat callback APIs:** consumer can `break`, `return`, or throw inside the loop, and the generator's `finally` runs cleanup. Backpressure is automatic — the generator pauses while the consumer is busy.

```javascript
// Generator with cleanup on early termination
async function* withCleanup(source, { signal } = {}) {
  const subscription = subscribe(source);
  try {
    while (!signal?.aborted) {
      const value = await subscription.next();
      if (value === null) return;
      yield value;
    }
  } finally {
    subscription.unsubscribe(); // runs even on early `break`
  }
}
```

## Promise combinators — when to use which

| Combinator | Resolves when… | Rejects when… | Use for |
|------------|----------------|---------------|---------|
| `Promise.all` | all fulfill | any rejects (others keep running) | Need every result; one failure is total failure |
| `Promise.allSettled` | all settle | never | Need every outcome regardless of failures (parallel API hits, dashboard) |
| `Promise.race` | first settles | first rejects | Timeout race (legacy — prefer `AbortSignal.timeout`) |
| `Promise.any` | first fulfills | all reject | Mirror/fallback (try N CDNs, return first 200) |

**Critical caveat for `Promise.all`:** when one promise rejects, the others are *not* cancelled by default. Pair with `AbortController` + `AbortSignal.any` if you want true fail-fast cancellation.

```javascript
// ✅ All-or-nothing with cancel cascade
async function allOrCancel(makeTasks) {
  const controller = new AbortController();
  const tasks = makeTasks(controller.signal);
  try {
    return await Promise.all(tasks);
  } catch (err) {
    controller.abort(err); // cancel siblings
    throw err;
  }
}

await allOrCancel((signal) => [
  fetch("/a", { signal }),
  fetch("/b", { signal }),
  fetch("/c", { signal }),
]);
```

## Microtasks, macrotasks, and yielding

```javascript
// queueMicrotask — runs before next render, after current sync code
queueMicrotask(() => doSomethingAfterCurrentTick());

// Yield to event loop without 4ms minimum of setTimeout(0)
function yieldToMain() {
  return new Promise(resolve => {
    if (typeof scheduler !== "undefined" && scheduler.yield) {
      scheduler.yield().then(resolve); // ✅ modern, prioritizes user input
    } else {
      setTimeout(resolve, 0);          // fallback
    }
  });
}

// Process large arrays without blocking
async function processInBatches(items, fn, { batchSize = 100, signal } = {}) {
  const out = [];
  for (let i = 0; i < items.length; i += batchSize) {
    signal?.throwIfAborted();
    const batch = items.slice(i, i + batchSize);
    out.push(...batch.map(fn));
    if (i + batchSize < items.length) await yieldToMain();
  }
  return out;
}
```

`scheduler.yield()` is browser-only (Chrome 129+, behind flag elsewhere) and prioritizes user input. Falls back to `setTimeout(0)` cleanly.

## Promise lifecycle: catch unhandled rejections in fire-and-forget

```javascript
// ❌ — fire-and-forget without `.catch` produces unhandledrejection events
emitter.on("event", () => sdk.track(event)); // if track rejects, console pollution

// ✅ — explicit catch, route to telemetry
emitter.on("event", () => {
  sdk.track(event).catch(err => sdk.options.onError?.(err));
});
```

## Quick Reference

| Pattern | Use when | Gotcha |
|---------|----------|--------|
| `signal?.throwIfAborted()` | Every async entry point | Must come before any `await` to fail fast |
| `AbortSignal.timeout(ms)` | Per-operation deadline | Reason is `TimeoutError`; document this |
| `AbortSignal.any([...])` | Compose consumer + SDK signals | Polyfill needed for browsers <2023 |
| Full-jitter retry | Network/rate-limit errors | Never retry programmer errors |
| Request dedup via `Map` | Idempotent reads | NOT for state-mutating writes |
| Concurrency limiter | Bulk operations | Cancel queued tasks on signal abort |
| `async function*` | Pagination, streams | `try/finally` for cleanup on early break |
| `Promise.allSettled` | "Never throw" parallel | Caller must inspect each `status` |
| `scheduler.yield()` | CPU-heavy work | Browser-only; fall back to `setTimeout(0)` |
