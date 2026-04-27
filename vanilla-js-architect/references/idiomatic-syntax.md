# Idiomatic Syntax for SDK Authors

Modern JS syntax framed by **what's idiomatic when the consumer is another developer**. Generic ES2023 cheatsheets miss the SDK angle: when does this primitive prevent consumer foot-guns, and when does it leak implementation detail?

## Privacy: `#private` vs `WeakMap` vs closure

Three options. The wrong choice is a years-long regret.

| Need | Choose | Why |
|------|--------|-----|
| Modern browsers + Node 18+, you can use classes | **`#private` fields** | Engine-level enforcement, can't be reflected, syntax is clean |
| Pre-2022 environments OR you need to attach private state to objects you don't own | **`WeakMap`** | Works everywhere, GC-friendly, can target any object |
| No class at all — factory returning plain objects | **Closure** | Zero ceremony, captures lexically |

```javascript
// ✅ #private — modern default
class Client {
  #apiKey;
  #queue = [];
  constructor(opts) { this.#apiKey = opts.apiKey; }
  // External code CANNOT access #apiKey — even via reflection
}

// ✅ WeakMap — when you can't or don't want classes, or need to track private state of foreign objects
const state = new WeakMap();
export function createClient(opts) {
  const self = { /* public surface */ };
  state.set(self, { apiKey: opts.apiKey, queue: [] });
  return self;
}

// ✅ Closure — simplest when factory returns one object
export function createClient({ apiKey }) {
  let queue = [];
  return Object.freeze({
    track(event) { queue.push(event); },
    flush() { /* uses queue */ },
  });
}
```

**Anti-pattern: convention-only privacy (`_field`).** Just `#`. The underscore convention is a lie consumers will rely on, then break.

## `WeakRef` and `FinalizationRegistry` — long-lived caches without leaks

```javascript
// ✅ Cache that doesn't prevent GC of large objects
class WeakCache {
  #map = new Map(); // key → WeakRef<value>
  #registry = new FinalizationRegistry((key) => this.#map.delete(key));

  set(key, value) {
    this.#map.set(key, new WeakRef(value));
    this.#registry.register(value, key);
  }

  get(key) {
    return this.#map.get(key)?.deref(); // undefined if GC'd
  }
}

// Use case: SDK caches DOM nodes, blobs, large buffers — anything the consumer might
// release. WeakCache lets the consumer drop references and the cache cleans itself.
```

**When NOT to use `WeakRef`:** primitives, small objects, or anything where you actually need the cache to keep the value alive. `WeakRef` is a hint to the GC, not a guarantee — `deref()` can return `undefined` at any time.

## `Symbol.dispose` and `using` — explicit resource management

ES2024 (Stage 4). Browsers + Node 22+ support it. Major shift for SDKs that hold resources.

```javascript
// ✅ SDK exposes disposable resources
class DBTransaction {
  #conn;
  #committed = false;

  constructor(conn) { this.#conn = conn; }

  commit() { this.#committed = true; this.#conn.commit(); }

  [Symbol.dispose]() {
    if (!this.#committed) this.#conn.rollback();
    this.#conn.release();
  }
}

// Consumer code:
{
  using tx = sdk.transaction();
  tx.execute("INSERT ...");
  tx.commit();
} // ← `tx[Symbol.dispose]()` called automatically here, even on throw

// Async version:
class Subscription {
  async [Symbol.asyncDispose]() {
    await this.#stream.close();
  }
}

{
  await using sub = await sdk.subscribe("topic");
  for await (const msg of sub) handle(msg);
} // ← `await sub[Symbol.asyncDispose]()` runs here
```

**When to ship this in your SDK:** any resource with a clear "I'm done" moment — connections, file handles, subscriptions, transactions, observers. Always provide a manual `dispose()` *and* the symbol method, so consumers on older runtimes still work.

```javascript
class Subscription {
  async dispose() { await this.#stream.close(); }
  async [Symbol.asyncDispose]() { return this.dispose(); }
}
```

**Polyfill check:** check `typeof Symbol.dispose === "symbol"` before relying on `using` syntax in source. Symbols themselves are runtime-feature-gated; your build target dictates whether you can use the keyword.

## `structuredClone` — defensive copies done right

```javascript
// ❌ — JSON round-trip loses Date, Map, Set, undefined, RegExp; throws on cycles; costs O(n²) for big objects
const copy = JSON.parse(JSON.stringify(input));

// ❌ — spread is shallow; nested objects shared
const copy = { ...input };

// ✅ — structuredClone: deep, fast, handles Map/Set/Date/RegExp/typed arrays/cycles
const copy = structuredClone(input);

// Bonus: transferable objects (move ownership of ArrayBuffer to a Worker)
const buffer = new ArrayBuffer(1024);
const cloned = structuredClone(buffer, { transfer: [buffer] }); // buffer is now detached
```

Use `structuredClone` when receiving consumer-provided complex objects (event payloads, options with nested structures). Available in all modern browsers, Node 17+, Workers, Deno, Bun.

**Caveats:** does NOT clone functions, DOM nodes (limited), class instances (loses prototype), or `Error` (loses stack pre-Node-18). For class instances, you typically don't want a clone anyway — pass references or rebuild via constructor.

## `Object.freeze` and `Object.preventExtensions`

```javascript
// ✅ Freeze the consumer's view of internal structures they shouldn't mutate
class SDK {
  #plugins = [];
  get plugins() { return Object.freeze([...this.#plugins]); } // frozen snapshot, fresh array
}

// ✅ Freeze options after merge — prevents accidental mutation across the SDK lifetime
function createClient(userOptions = {}) {
  const options = Object.freeze({
    timeout: 5000,
    retries: 3,
    ...userOptions,
  });
  return new Client(options);
}
```

**Caveat:** `Object.freeze` is shallow. Nested objects stay mutable. For deep immutability use `structuredClone` + `freezeRecursively`, or better, design the API so consumers don't get nested mutable structures.

## Modern array methods — the ones SDK code actually uses

```javascript
// at(-1) — last element, no `arr[arr.length-1]` dance
const last = items.at(-1);

// findLast / findLastIndex — search from end (e.g. most recent matching event)
const lastError = events.findLast(e => e.type === "error");

// toSorted / toReversed / toSpliced / with — non-mutating versions
const sorted = items.toSorted((a, b) => a.priority - b.priority);
const updated = items.with(2, newItem);

// Object.groupBy — group an array by a key function (no Lodash needed)
const byStatus = Object.groupBy(orders, (o) => o.status);
// → { pending: [...], shipped: [...], cancelled: [...] }

// Map.groupBy — same but Map-keyed (objects as keys)
const byUser = Map.groupBy(events, (e) => e.user);

// Iterator helpers (Stage 4, browsers + Node 22+) — chainable lazy iteration
const top10Active = users
  .values()
  .filter(u => u.active)
  .map(u => u.name)
  .take(10)
  .toArray();
```

**Anti-pattern: pulling in Lodash for `groupBy`, `chunk`, `partition`.** All native now. Lodash adds ~20KB minified.

## Set operations — ES2025

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

a.union(b);              // Set { 1, 2, 3, 4 }
a.intersection(b);       // Set { 2, 3 }
a.difference(b);         // Set { 1 }
a.symmetricDifference(b);// Set { 1, 4 }
a.isSubsetOf(b);         // false
a.isSupersetOf(b);       // false
a.isDisjointFrom(b);     // false
```

Available in browsers from 2024-2025, Node 22+. Polyfill if you target older runtimes.

## Numeric separators and BigInt

```javascript
// Separators for readability — every digit-heavy SDK constant
const MAX_PAYLOAD = 1_048_576;    // 1 MiB
const RETRY_CAP_MS = 30_000;
const MASK = 0xFF_EC_DE_5E;

// BigInt — use when you have integers > 2^53 (Snowflake IDs, large counters, crypto)
const id = 9_007_199_254_740_993n;
const next = id + 1n;
// Cannot mix with Number: BigInt(123) + 456n
```

**Watch out:** `JSON.stringify(123n)` throws. SDK option types should not silently accept BigInt unless you handle it.

## Optional chaining + nullish coalescing patterns

```javascript
// ✅ Optional chaining for shallow checks
const userName = response?.user?.name ?? "Anonymous";

// ✅ Combined with logical assignment for one-shot defaults
options.timeout ??= 5000; // assign only if null/undefined

// ❌ Anti-pattern: optional chaining as error suppression
const result = sdk.getSomething?.()?.thing?.value;
// If `getSomething` doesn't exist on `sdk`, that's a bug — don't paper over it
```

**Rule:** use `?.` when the absence is genuinely valid (network response with optional fields). Don't use it to silence bugs.

## Logical assignment

```javascript
options.timeout ??= 5000;     // null/undefined only
config.cache ||= new Map();   // any falsy (0, "", false trigger this — usually wrong default)
user.name &&= sanitize(user.name); // only if truthy

// Use ??= as the default — `||=` is almost always a bug waiting (0 or "" gets replaced)
```

## Tagged templates for composable strings

```javascript
// ✅ Build query strings safely without concat hell
const sql = (strings, ...values) => ({
  text: strings.reduce((q, s, i) => q + s + (i < values.length ? `$${i + 1}` : ""), ""),
  values,
});

const query = sql`SELECT * FROM users WHERE id = ${userId} AND active = ${true}`;
// → { text: "SELECT * FROM users WHERE id = $1 AND active = $2", values: [userId, true] }
```

Useful pattern when your SDK builds structured strings (queries, URLs, signed payloads).

## Property shorthand and computed keys

```javascript
// ✅ Shorthand
const apiKey = "x";
const opts = { apiKey, baseUrl }; // equivalent to { apiKey: apiKey, baseUrl: baseUrl }

// ✅ Computed keys for dynamic property names
const event = "page_view";
const payload = { [event]: data, [`${event}_at`]: Date.now() };

// ✅ Method shorthand in objects
const sdk = {
  async fetch(url) { /* ... */ },     // not `fetch: async function fetch(url)`
  *iterate() { yield 1; yield 2; },   // generator method
};
```

## Quick Reference — the SDK syntax cheatsheet

| Need | Use |
|------|-----|
| Engine-enforced privacy | `#field` |
| Privacy on objects you don't own | `WeakMap` |
| Default-prone closures | `function createX() { let s; return { ... } }` |
| Long-lived cache that GC can clear | `WeakRef` + `FinalizationRegistry` |
| Resource with explicit cleanup | `[Symbol.dispose]` / `[Symbol.asyncDispose]` + `using` |
| Defensive deep copy | `structuredClone(input)` |
| Shallow immutable view | `Object.freeze` |
| Last element | `arr.at(-1)` |
| Find from end | `arr.findLast(fn)` |
| Non-mutating array op | `toSorted` / `toReversed` / `toSpliced` / `with` |
| Group an array | `Object.groupBy(arr, fn)` or `Map.groupBy` |
| Chainable lazy iteration | Iterator helpers (`.values().map().take().toArray()`) |
| Set union/intersection | `a.union(b)`, `a.intersection(b)` (ES2025) |
| Default if nullish | `??=` |
| Composable interpolation | tagged template literal |
