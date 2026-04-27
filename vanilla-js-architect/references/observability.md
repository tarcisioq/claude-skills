# Observability

How an SDK should expose its internals for debugging, telemetry, and production monitoring **without imposing tools on the consumer**. The consumer chooses what to log, where to send metrics, how to report errors. The SDK provides the hooks.

## Core principle: silent by default, observable on opt-in

The default state of any SDK is *quiet*. No `console.log`, no telemetry pings, no auto-instrumentation. Anything noisy is opt-in via constructor options.

```javascript
// ✅ Silent by default
const client = new Client({ apiKey: "k" }); // doesn't log anything

// Opt in to debug logging
const client = new Client({ apiKey: "k", debug: true });

// Or pipe to your own logger
const client = new Client({
  apiKey: "k",
  logger: { debug: pino.debug, info: pino.info, warn: pino.warn, error: pino.error },
});
```

## Debug flag pattern

```javascript
// ✅ Single boolean → routes through one internal #log method
class Client {
  #debug;
  #logger;

  constructor({ debug = false, logger = null } = {}) {
    this.#debug = debug;
    this.#logger = logger;
  }

  #log(level, message, data) {
    // 1. If consumer provided a logger, use it
    if (this.#logger) {
      this.#logger[level]?.(message, data);
      return;
    }
    // 2. Otherwise, debug flag gates console output (debug-level only)
    if (this.#debug && level === "debug") {
      console.debug(`[my-sdk]`, message, data);
    }
    // 3. Warnings and errors always go to console (they indicate real problems)
    if (level === "warn") console.warn(`[my-sdk]`, message, data);
    if (level === "error") console.error(`[my-sdk]`, message, data);
  }
}
```

**Anti-patterns:**
- `console.log` directly anywhere in production code paths
- `if (DEBUG) console.log(...)` — use the wrapper, not scattered checks
- Multiple debug levels via env vars — the SDK doesn't read env, the consumer does

## Logger interface — duck-typed, dependency-free

```javascript
/**
 * @typedef {Object} Logger
 * @property {(message: string, data?: unknown) => void} [debug]
 * @property {(message: string, data?: unknown) => void} [info]
 * @property {(message: string, data?: unknown) => void} [warn]
 * @property {(message: string, data?: unknown) => void} [error]
 */
```

Match the shape of common loggers (`pino`, `bunyan`, `winston`, console-like). All methods optional — degrade gracefully if consumer's logger doesn't implement `debug`.

```javascript
// Consumer wires up their existing logger
import pino from "pino";
const log = pino();

const sdk = new Client({
  apiKey: "k",
  logger: { debug: log.debug.bind(log), info: log.info.bind(log), warn: log.warn.bind(log), error: log.error.bind(log) },
});

// Or trivially:
const sdk = new Client({ apiKey: "k", logger: console }); // console-as-logger works
```

## Logging redaction — never leak secrets

Already covered in `security.md`, but the rule lives here too:

```javascript
// ✅ Always redact known-sensitive headers + query params before logging
function redactRequest(req) {
  const headers = { ...Object.fromEntries(req.headers || []) };
  if (headers.authorization) headers.authorization = "[REDACTED]";
  if (headers["x-api-key"])  headers["x-api-key"]  = "[REDACTED]";

  const url = new URL(req.url);
  if (url.searchParams.has("token")) url.searchParams.set("token", "[REDACTED]");

  return { method: req.method, url: url.toString(), headers };
}

#log("debug", "request", redactRequest(req));
```

**Test for accidental leaks** — see `testing-libraries.md`. Setup spies on console; assert no test produces output containing test secrets.

## Telemetry hooks — let consumers wire their own

The SDK doesn't ship a telemetry SDK. It exposes events the consumer can pipe to *their* tools.

```javascript
// ✅ Generic telemetry hook
class Client {
  #onTelemetry;

  constructor({ onTelemetry } = {}) {
    this.#onTelemetry = onTelemetry;
  }

  #emit(event, data) {
    try {
      this.#onTelemetry?.({ name: event, timestamp: Date.now(), ...data });
    } catch (err) {
      // Telemetry failures must never break the SDK
      console.error("[my-sdk] telemetry hook threw:", err);
    }
  }

  async fetch(url) {
    const start = performance.now();
    try {
      const result = await this.#doFetch(url);
      this.#emit("request_success", { url, duration: performance.now() - start });
      return result;
    } catch (err) {
      this.#emit("request_failure", { url, duration: performance.now() - start, code: err.code });
      throw err;
    }
  }
}

// Consumer:
const client = new Client({
  onTelemetry: (event) => {
    if (event.name === "request_failure") sentry.captureMessage(event.name, { extra: event });
    statsd.timing("sdk.request", event.duration);
  },
});
```

**Why this matters:** dependents may be on Datadog, New Relic, OpenTelemetry, custom backends. Don't make them strip out your built-in metrics.

## Error reporting hook — separate from telemetry

```javascript
// ✅ onError receives every error the SDK encounters internally
class Client {
  #onError;

  constructor({ onError } = {}) {
    this.#onError = onError;
  }

  #reportError(err, context = {}) {
    try {
      this.#onError?.(err, context);
    } catch {} // never throw from error reporting
  }

  // Internal error in a background task
  async #backgroundFlush() {
    try {
      await this.#sendQueued();
    } catch (err) {
      this.#reportError(err, { source: "backgroundFlush" });
    }
  }
}

// Consumer:
const client = new Client({
  onError: (err, ctx) => Sentry.captureException(err, { tags: ctx }),
});
```

**Why separate from telemetry:** errors deserve their own routing. Some consumers want telemetry → Datadog and errors → Sentry. Two hooks, no coupling.

## `performance.mark` and `performance.measure`

Browser-built-in performance instrumentation, free, available everywhere. Use namespaced names so consumers can identify your marks.

```javascript
// ✅ Namespaced marks — show up in DevTools Performance panel and PerformanceObserver
class Client {
  async fetch(url) {
    performance.mark("my-sdk:fetch:start", { detail: { url } });
    try {
      const result = await this.#doFetch(url);
      performance.mark("my-sdk:fetch:end");
      performance.measure("my-sdk:fetch", "my-sdk:fetch:start", "my-sdk:fetch:end");
      return result;
    } catch (err) {
      performance.mark("my-sdk:fetch:error");
      performance.measure("my-sdk:fetch:failed", "my-sdk:fetch:start", "my-sdk:fetch:error");
      throw err;
    }
  }
}

// Consumer can observe:
const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntriesByName("my-sdk:fetch", "measure")) {
    console.log(`SDK fetch took ${entry.duration.toFixed(1)}ms`);
  }
});
obs.observe({ type: "measure", buffered: true });
```

**Naming convention:** `<sdk-prefix>:<feature>:<lifecycle>`. Always prefix with the SDK name so consumers can filter by name pattern.

**When to skip `performance.mark`:** server-side runtimes that don't have `performance` (rare; modern Node/Workers/Deno/Bun do) — feature-detect:

```javascript
const perf = globalThis.performance;
if (perf?.mark) perf.mark("my-sdk:start");
```

## Tracing context propagation

If your SDK makes HTTP requests, propagate W3C trace context headers when present. Lets distributed tracing work end-to-end without the SDK becoming an OTel client.

```javascript
// ✅ Propagate traceparent if available; don't generate one if not
async fetch(url, { signal } = {}) {
  const headers = new Headers();

  // If the consumer set a tracing context, forward it
  const ctx = getActiveTraceContext?.(); // consumer-provided hook
  if (ctx?.traceparent) headers.set("traceparent", ctx.traceparent);
  if (ctx?.tracestate)  headers.set("tracestate",  ctx.tracestate);

  return fetch(url, { headers, signal });
}
```

The SDK takes a `getActiveTraceContext` option (or detects via OTel API if present). It does NOT bundle OTel as a dependency.

## Health and metrics surface

Expose introspection methods for runtime status — useful for debugging and consumer-built dashboards.

```javascript
class Client {
  /** @returns {{ queueSize: number, inflight: number, lastError: SDKError | null }} */
  getStatus() {
    return Object.freeze({
      queueSize: this.#queue.length,
      inflight: this.#inflight.size,
      lastError: this.#lastError,
    });
  }
}

// Consumer dashboard:
setInterval(() => updateUI(client.getStatus()), 1000);
```

**Anti-pattern:** auto-pushing metrics to a fixed endpoint. The SDK exposes; the consumer pulls.

## Logging levels — what goes where

| Level | Use for | Default visibility |
|-------|---------|---------------------|
| `debug` | Internal state changes, request/response bodies, retry decisions | Off (debug flag or logger) |
| `info` | Lifecycle events the consumer might care about (init, ready, destroy) | Off by default; logger only |
| `warn` | Non-fatal: deprecated API used, retry happening, slow operation | Always visible (console.warn) |
| `error` | Errors the consumer might miss (background task failures) | Always visible (console.error) |

`debug` and `info` are noise unless asked for. `warn` and `error` are signals the consumer should see — but route through the logger if provided so they don't double-log.

## Deprecation warnings

```javascript
// ✅ One warning per deprecated API per process — not on every call
const warned = new Set();

function warnDeprecated(api, version, replacement) {
  if (warned.has(api)) return;
  warned.add(api);
  console.warn(
    `[my-sdk] ${api} is deprecated since v${version}. Use ${replacement} instead.`
  );
}

export function oldMethod() {
  warnDeprecated("oldMethod()", "2.5.0", "newMethod()");
  return newMethod();
}
```

Surface in `@deprecated` JSDoc tag too — editors render them with strikethrough.

## What this file deliberately doesn't cover

- Building a full distributed tracing system — bundle OpenTelemetry separately if needed
- Logging libraries themselves — the SDK takes one, doesn't ship one
- Metrics aggregation — the consumer aggregates; SDK emits raw
- Error tracking dashboards — consumer wires their own (Sentry, Bugsnag, custom)

The SDK's job: emit structured signals. Consumer's job: route them.

## Quick Reference

| Need | Pattern |
|------|---------|
| Default behavior | Silent — no `console.log` |
| Opt-in debug | `debug: true` constructor option |
| Pluggable logger | `logger: { debug, info, warn, error }` |
| Sensitive headers | Redact in log path: Authorization, X-Api-Key |
| Telemetry events | `onTelemetry({ name, timestamp, ...data })` hook |
| Error reporting | `onError(err, context)` hook |
| Performance | `performance.mark("my-sdk:feature:lifecycle")` |
| Trace propagation | Forward `traceparent`/`tracestate` if consumer supplies |
| Status introspection | `getStatus()` returning frozen snapshot |
| Deprecation | One warning per deprecated API + `@deprecated` JSDoc |
| Telemetry/error hook errors | Catch and console.error — never bubble up |
