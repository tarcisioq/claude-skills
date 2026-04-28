# API Design

How to design the *public surface* of a JavaScript SDK or library so consumers find it ergonomic, predictable, and stable across versions.

## Decision Matrix — Apply Before Writing the Method

### Naming

| The thing is… | Use |
|---------------|-----|
| An action / verb | `connect()`, `track()`, `flush()` — present-tense verb |
| A resource / noun | `connection`, `queue` — singular noun, no `get`/`set` prefix |
| A boolean state | `isConnected`, `hasPermission`, `canRetry` — `is`/`has`/`can` prefix |
| An async result | Same as sync; **never** suffix `Async` (Promise IS the signal) |
| An event name | `kebab-case` for DOM-style events (`state-change`); `camelCase` for emitter-style (`stateChange`) — pick one and stick |
| A constant | `SCREAMING_SNAKE_CASE` |
| A private | `#field` (modern) or `_field` for objects from factories |

### Parameter shape

| Method has… | Use |
|-------------|-----|
| 1 param, name obvious from method | Positional: `getUser(id)` |
| 2 params, both obvious | Positional: `add(a, b)` |
| 2+ params with order ambiguity | Options object: `track(name, { properties, signal })` |
| 3+ params | Always options object — no exceptions |
| Required + optional mix | Required positional, optional in trailing options object: `track(eventName, { properties, signal } = {})` |

### Failure signaling

| Failure characteristic | Use |
|------------------------|-----|
| Programmer error (bad args, misuse) | `throw new SDKError(msg, "INVALID_*", { details })` |
| Runtime error in async path (network/IO) | Reject with `SDKError` subclass |
| Both success and failure carry data | Return `{ ok: boolean, value?, error? }` |
| Operation may legitimately return nothing | Return `null` (NOT `undefined`, NOT a sentinel) |
| Operation always succeeds (truly) | Return the value, no wrapping |

### Sync vs async

| The operation… | Choose |
|----------------|--------|
| Touches the network, disk, IndexedDB, Crypto.subtle | Always `async` |
| Reads from cache that *might* miss to fetch | Always `async` |
| Pure computation, no I/O, no awaits | Sync |
| Could vary at runtime | **Always `async`** (consistency beats micro-opt) |
| Streaming/iterating | `async function*` |

### Mutation

| Method does… | Pattern |
|--------------|---------|
| Configures a builder | Return `this` for chaining (mutating) |
| Configures *immutably* | Return new instance with merged state |
| Modifies internal state | Return result, not `this` |
| Reads computed state | `get` accessor on the class |

## Naming Conventions

```javascript
// Verbs for actions, nouns for things, booleans prefixed with is/has/can
client.connect();        // ✅ verb
client.disconnect();     // ✅ verb
client.connection;       // ✅ noun (the resource)
client.isConnected;      // ✅ boolean
user.hasPermission();    // ✅ boolean predicate
config.canRetry;         // ✅ boolean

// ❌ Avoid
client.conn();           // abbreviated
client.doConnect();      // redundant prefix
client.connectedStatus;  // ambiguous (string? boolean?)
```

**Async methods**: do NOT suffix with `Async`. Returning a Promise is its own signal.

```javascript
// ✅
await client.fetch(id);

// ❌ — legacy .NET convention
await client.fetchAsync(id);
```

## Options Object Pattern

```javascript
// ✅ Object — order-independent, easy to extend without breaking callers
function createClient({
  apiKey,
  baseUrl = "https://api.example.com",
  timeout = 5000,
  retries = 3,
} = {}) { /* ... */ }

createClient({ apiKey: "x", timeout: 10_000 });

// ❌ Positional — adding a new option breaks every caller
function createClient(apiKey, baseUrl, timeout, retries, debug) { /* ... */ }
createClient("x", undefined, 10_000); // unreadable
```

**When positional args are OK**: 1-2 arguments where the meaning is obvious from the function name (`add(a, b)`, `getUser(id)`).

## Required vs Optional Parameters

```javascript
// Make required params explicit and fail fast
export function createClient(options) {
  if (!options?.apiKey) {
    throw new SDKError("apiKey is required", "INVALID_OPTIONS");
  }
  // ...
}

// Group required separately from options when the distinction matters
export function track(eventName, properties = {}) {
  if (typeof eventName !== "string") {
    throw new SDKError("eventName must be a string", "INVALID_ARGUMENT");
  }
  // ...
}
```

## Fluent Interfaces (Method Chaining)

```javascript
// Return `this` from configurator methods to enable chaining
export class QueryBuilder {
  #state = { filters: [], sort: null, limit: null };

  where(field, op, value) {
    this.#state.filters.push({ field, op, value });
    return this;
  }
  orderBy(field, dir = "asc") {
    this.#state.sort = { field, dir };
    return this;
  }
  limit(n) {
    this.#state.limit = n;
    return this;
  }
  async execute() {
    return runQuery(this.#state);
  }
}

// Usage:
const results = await new QueryBuilder()
  .where("status", "=", "active")
  .orderBy("name")
  .limit(20)
  .execute();
```

**Caveat**: chained methods that mutate `this` are not safe to share across consumers. If immutability matters, return a new instance instead:

```javascript
where(field, op, value) {
  return new QueryBuilder({
    ...this.#state,
    filters: [...this.#state.filters, { field, op, value }],
  });
}
```

## Sync vs Async Boundaries

```javascript
// Be consistent: a method either always returns a Promise or never does
// ❌ Sometimes-async — forces consumers to write `await` defensively everywhere
function getUser(id) {
  if (cache.has(id)) return cache.get(id);   // sync
  return fetch(`/users/${id}`).then(r => r.json()); // async
}

// ✅ Always async — predictable
async function getUser(id) {
  if (cache.has(id)) return cache.get(id);
  const res = await fetch(`/users/${id}`);
  return res.json();
}
```

## Error Contracts

```javascript
// 1. Define a base error class so consumers can `instanceof` filter
export class SDKError extends Error {
  constructor(message, code, { cause, details } = {}) {
    super(message, { cause });
    this.name = "SDKError";
    this.code = code;     // string enum, stable across versions
    this.details = details;
  }
}

// 2. Document codes as part of the public API
/**
 * @typedef {"INVALID_OPTIONS" | "NETWORK" | "RATE_LIMITED" | "AUTH_FAILED" | "TIMEOUT"} SDKErrorCode
 */

// 3. Subclass for errors with extra structure
export class HTTPError extends SDKError {
  constructor(status, body) {
    super(`HTTP ${status}`, "HTTP_ERROR", { details: { status, body } });
    this.status = status;
  }
}

// 4. Async APIs reject — never throw synchronously from an async function
async function fetchUser(id) {
  if (!id) throw new SDKError("id required", "INVALID_ARGUMENT"); // becomes a rejection
  // ...
}
```

**Never** throw strings or plain objects. Consumers can't `instanceof` them, can't pattern-match reliably, and stack traces are missing.

## Return Value Conventions

```javascript
// Return useful values, not just success/fail booleans
async function save(record) {
  const saved = await db.insert(record);
  return saved; // ✅ caller may need the generated ID, timestamps, etc.
}

// Use null for "not found", not undefined or sentinel values
async function findUser(id) {
  const user = await db.get(id);
  return user ?? null; // ✅ explicit: this method may return null
}

// Use Result-style objects only when both success and failure carry data
async function attemptPayment(amount) {
  try {
    const tx = await processPayment(amount);
    return { ok: true, transaction: tx };
  } catch (err) {
    return { ok: false, error: err.code, retryAfter: err.details?.retryAfter };
  }
}
```

## Versioning and Deprecation

```javascript
// Use semver strictly:
//   PATCH (1.2.X) — bug fixes, no API change
//   MINOR (1.X.0) — additive changes only
//   MAJOR (X.0.0) — any breaking change

// Deprecate before removing — give consumers a migration window
export function oldMethod(arg) {
  if (!oldMethod._warned) {
    console.warn("[my-sdk] oldMethod() is deprecated; use newMethod() instead. Will be removed in v3.0.");
    oldMethod._warned = true;
  }
  return newMethod({ legacy: arg });
}

// JSDoc deprecation tags — surfaced by editors and tooling
/**
 * @deprecated since 2.5.0. Use {@link newMethod} instead.
 */
export function oldMethod() { /* ... */ }
```

## Documenting the Public API

```javascript
/**
 * Tracks an event in the analytics queue.
 *
 * @param {string} eventName — non-empty event identifier (e.g. "page_view")
 * @param {object} [properties] — arbitrary key/value pairs attached to the event
 * @param {object} [options]
 * @param {AbortSignal} [options.signal] — cancel the in-flight delivery
 * @returns {Promise<void>} resolves once the event is durably queued
 * @throws {SDKError} with code "DESTROYED" if called after .destroy()
 * @throws {SDKError} with code "INVALID_ARGUMENT" if eventName is empty
 *
 * @example
 *   await client.track("checkout_completed", { revenue: 49.99 });
 */
async track(eventName, properties = {}, { signal } = {}) { /* ... */ }
```

Every public method gets at minimum: `@param`, `@returns`, `@throws` (with codes), and `@example`.

## Consistency Across the Surface

```javascript
// Pick one style and stick with it across the entire SDK.

// ✅ Consistent — all methods accept (id, options)
client.getUser(id, { signal });
client.deleteUser(id, { signal });
client.updateUser(id, { name: "x" }, { signal });

// ❌ Inconsistent — caller has to remember per-method conventions
client.getUser(id);                              // positional
client.deleteUser({ id });                       // options object
client.updateUser(id, "name", "x");              // positional with extras
```

Decide early on:
- Options object vs positional args
- Promise vs callback vs event-driven
- Throwing vs returning Result objects
- camelCase vs snake_case for option keys (camelCase is the JS convention)

## Backwards Compatibility Tactics

```javascript
// Add new optional params, never reorder or remove existing ones
function fetch(url, options = {}) {
  // adding new options.x is safe
}

// When changing behavior, opt-in via a flag
function fetch(url, { newBehavior = false } = {}) {
  if (newBehavior) return fetchV2(url);
  return fetchV1(url);
}

// In v3.0 (major bump), flip the default and remove the flag
function fetch(url) {
  return fetchV2(url);
}
```

## Quick Reference

| Concern | Choice | Notes |
|---------|--------|-------|
| Many params | Options object | Order-independent, extensible |
| 1-2 obvious params | Positional | `add(a, b)` is fine |
| Method chaining | Return `this` | Mutating; clone if immutability needed |
| Async signaling | Always Promise | Never sometimes-sync |
| Errors | Custom class + `code` | Consumers `instanceof` and switch on code |
| "Not found" | Return `null` | Never undefined for documented absence |
| Removal | Deprecate first | Console warning + JSDoc `@deprecated` |
| Versioning | Strict semver | Breaking change → MAJOR bump |
| Documentation | JSDoc on every export | `@param`, `@returns`, `@throws`, `@example` |
