# SDK Architecture

Patterns for designing the *internal structure* of a JavaScript SDK or library: how classes are organized, how state is encapsulated, how features are extended, and how lifecycle is managed.

## Decision Tree — Resolve Before Writing Code

### 1. Entry shape: class, factory, or plain functions?

```
Does the consumer need `instanceof` checks or to subclass?
├── YES → Class
└── NO  ↓
    Is there persistent state across calls (connection, queue, options)?
    ├── YES → Factory wrapping internal class
    │        (consumer doesn't see `new`, you keep flexibility)
    └── NO  → Plain named exports (no class, no factory)
             e.g. `export function parse(input) {}`
```

### 2. Composition vs inheritance

```
Default: composition.

Inherit ONLY when:
- You need `instanceof` for an ecosystem reason (extending Error, EventTarget)
- AND the relationship is genuine "is-a", not "has-a"
- AND you'll never go more than ONE level deep

Otherwise: compose with mixins, plugins, or feature-object decorators.
```

### 3. State management approach

```
How many states does this SDK go through?
├── 1-2 (just "alive" / "destroyed") → boolean flag (#destroyed)
├── 3-4 named states                  → string property + transition method
└── 5+ states OR illegal transitions  → explicit state machine
                                        (frozen TRANSITIONS map + #transition())
```

### 4. Extension points

```
Will third parties add behavior?
├── NO  → Don't build a plugin system. YAGNI. Add it in v2 if needed.
└── YES ↓
    What kind of extension?
    ├── Transform values in a pipeline → linear hooks (each plugin transforms)
    ├── Run side-effects at lifecycle  → event listeners (EventTarget)
    ├── Replace internal subsystem      → dependency injection via options
    └── Express-style chained ctrl flow → middleware with explicit `next()`
```

### 5. Singleton or multi-instance?

```
Default: multi-instance via factory or `new`.

Singleton ONLY when:
- The resource is genuinely process-wide (e.g. wraps `navigator.locks`)
- AND consumers can never legitimately want two

If singleton: gate behind `getShared()` + `resetShared()` so tests can isolate.
```

## Class-Based vs Factory-Based Entry Points

```javascript
// Class-based — when consumers benefit from `instanceof` checks or subclassing
export class Client {
  #apiKey;
  constructor({ apiKey }) {
    this.#apiKey = apiKey;
  }
}
const c = new Client({ apiKey: "x" });

// Factory-based — when you want to hide the class entirely or vary the returned shape
export function createClient({ apiKey }) {
  // Closure-based privacy, no `new` keyword for consumers
  return Object.freeze({
    fetch: (path) => fetch(path, { headers: { Authorization: apiKey } }),
  });
}
const c = createClient({ apiKey: "x" });

// Hybrid — class internally, factory externally (common in SDKs)
class InternalClient { /* ... */ }
export function createClient(options) {
  return new InternalClient(options);
}
```

**Rule of thumb**: Use a class when the SDK has identity, state, and a lifecycle. Use a factory when consumers don't need to know it's a class or when you want to swap implementations later without breaking the API.

## Encapsulation Strategies

```javascript
// 1. Private class fields — modern, recommended
class SDK {
  #state = "idle";
  #transport;

  constructor(opts) {
    this.#transport = createTransport(opts);
  }

  get state() { return this.#state; } // read-only public accessor
}

// 2. Closure-based privacy — when not using classes
export function createSDK(opts) {
  let state = "idle";
  const transport = createTransport(opts);

  return {
    get state() { return state; },
    start() { state = "running"; transport.connect(); },
  };
}

// 3. WeakMap — when you need private state external to the class
const privateState = new WeakMap();
export class SDK {
  constructor(opts) {
    privateState.set(this, { state: "idle", transport: createTransport(opts) });
  }
  get state() { return privateState.get(this).state; }
}
```

## Builder Pattern for Complex Construction

```javascript
// Use when an object has many optional parameters or staged setup
export class QueryBuilder {
  #filters = [];
  #sort = null;
  #limit = null;

  where(field, op, value) {
    this.#filters.push({ field, op, value });
    return this;
  }

  orderBy(field, direction = "asc") {
    this.#sort = { field, direction };
    return this;
  }

  take(n) {
    this.#limit = n;
    return this;
  }

  build() {
    return {
      filters: [...this.#filters],
      sort: this.#sort,
      limit: this.#limit,
    };
  }
}

// Usage:
const query = new QueryBuilder()
  .where("status", "=", "active")
  .orderBy("createdAt", "desc")
  .take(10)
  .build();
```

## Plugin / Middleware System

```javascript
// Plugin contract: plain object with a name + optional hooks
export class SDK {
  #plugins = [];
  #hooks = new Map();

  use(plugin) {
    if (!plugin?.name) throw new SDKError("plugin requires a name", "INVALID_PLUGIN");
    if (this.#plugins.some(p => p.name === plugin.name)) {
      throw new SDKError(`plugin "${plugin.name}" already registered`, "DUPLICATE_PLUGIN");
    }
    this.#plugins.push(plugin);
    plugin.install?.(this);
    return this;
  }

  // Hook: linear pipeline, each plugin can transform the value
  async runHook(name, value) {
    let current = value;
    for (const plugin of this.#plugins) {
      const fn = plugin[name];
      if (typeof fn === "function") {
        current = (await fn.call(plugin, current, this)) ?? current;
      }
    }
    return current;
  }
}

// Express-style middleware pipeline (next() chains explicitly)
class MiddlewareStack {
  #stack = [];

  use(fn) { this.#stack.push(fn); return this; }

  async dispatch(context) {
    let index = -1;
    const next = async (i) => {
      if (i <= index) throw new Error("next() called multiple times");
      index = i;
      const fn = this.#stack[i];
      if (!fn) return;
      await fn(context, () => next(i + 1));
    };
    await next(0);
    return context;
  }
}
```

## Lifecycle Events with EventTarget

```javascript
// Native EventTarget — no dependency, browser-standard
export class SDK extends EventTarget {
  async init() {
    this.dispatchEvent(new Event("initializing"));
    try {
      await this.#connect();
      this.dispatchEvent(new Event("ready"));
    } catch (err) {
      this.dispatchEvent(new CustomEvent("error", { detail: err }));
      throw err;
    }
  }

  destroy() {
    this.#cleanup();
    this.dispatchEvent(new Event("destroy"));
  }
}

// Consumer:
const sdk = new SDK();
sdk.addEventListener("ready", () => console.log("SDK ready"));
sdk.addEventListener("error", (e) => console.error(e.detail));
await sdk.init();

// Custom event emitter — when EventTarget feels too verbose
export class Emitter {
  #listeners = new Map();

  on(event, fn) {
    if (!this.#listeners.has(event)) this.#listeners.set(event, new Set());
    this.#listeners.get(event).add(fn);
    return () => this.off(event, fn); // returns disposer
  }

  off(event, fn) {
    this.#listeners.get(event)?.delete(fn);
  }

  emit(event, payload) {
    for (const fn of this.#listeners.get(event) ?? []) {
      try { fn(payload); }
      catch (err) { queueMicrotask(() => { throw err; }); } // never let one listener crash others
    }
  }
}
```

## Singleton vs Multi-Instance

```javascript
// ❌ Avoid module-scoped singletons — they break testability and multi-tenant usage
// sdk.js
const instance = new SDK({ apiKey: process.env.KEY });
export default instance; // hard to mock, can't have two clients with different keys

// ✅ Prefer factories or explicit instantiation
export function createSDK(options) {
  return new SDK(options);
}

// If you genuinely need a singleton, gate it behind a function so tests can reset it
let sharedInstance = null;
export function getSharedClient(options) {
  if (!sharedInstance) sharedInstance = new SDK(options);
  return sharedInstance;
}
export function resetSharedClient() { sharedInstance = null; }
```

## Composition Over Inheritance

```javascript
// ❌ Deep inheritance — fragile, hard to reuse
class Animal { /* ... */ }
class Dog extends Animal { /* ... */ }
class GuardDog extends Dog { /* ... */ }

// ✅ Compose behaviors via mixins or feature objects
const withLogging = (target) => ({
  ...target,
  log(msg) { console.log(`[${this.name}]`, msg); },
});

const withRetry = (target) => ({
  ...target,
  async retry(fn, attempts = 3) {
    for (let i = 0; i < attempts; i++) {
      try { return await fn(); }
      catch (err) { if (i === attempts - 1) throw err; }
    }
  },
});

export function createClient(options) {
  return withRetry(withLogging({
    name: options.name,
    fetch: (url) => fetch(url),
  }));
}
```

## Internal Module Layout

A typical SDK package structure:

```
src/
  index.js              # public entry — only re-exports the public API
  client.js             # main class
  errors.js             # SDKError + subclasses
  options.js            # option validation + defaults
  transport/
    http.js             # HTTP transport implementation
    websocket.js        # WS transport implementation
  plugins/
    plugin-base.js      # plugin interface contract
  internal/             # never imported by consumers
    queue.js
    backoff.js
    serializer.js
```

The `package.json` `exports` field exposes only `./` (index.js) and perhaps `./plugins`. Everything in `internal/` stays unreachable from outside the package.

## State Machines for Complex Lifecycles

```javascript
// When the SDK has more than 3-4 states, use an explicit state machine
const STATES = Object.freeze({
  IDLE: "idle",
  CONNECTING: "connecting",
  READY: "ready",
  RECONNECTING: "reconnecting",
  CLOSED: "closed",
});

const TRANSITIONS = {
  [STATES.IDLE]:         [STATES.CONNECTING],
  [STATES.CONNECTING]:   [STATES.READY, STATES.CLOSED],
  [STATES.READY]:        [STATES.RECONNECTING, STATES.CLOSED],
  [STATES.RECONNECTING]: [STATES.READY, STATES.CLOSED],
  [STATES.CLOSED]:       [],
};

export class Connection extends EventTarget {
  #state = STATES.IDLE;

  get state() { return this.#state; }

  #transition(next) {
    if (!TRANSITIONS[this.#state].includes(next)) {
      throw new SDKError(
        `invalid transition: ${this.#state} → ${next}`,
        "INVALID_TRANSITION"
      );
    }
    const prev = this.#state;
    this.#state = next;
    this.dispatchEvent(new CustomEvent("state-change", { detail: { prev, next } }));
  }
}
```

## Quick Reference

| Concern | Pattern | Use When |
|---------|---------|----------|
| Identity + state | Class with `#fields` | SDK has lifecycle and consumer holds a reference |
| Hidden internals | Factory + closure | Consumer should not see implementation details |
| Many optional params | Builder | More than ~5 options, staged construction |
| Cross-cutting hooks | Plugin pipeline | Allow third parties to extend behavior |
| Lifecycle notifications | `EventTarget` | Standard, no deps, DOM-style |
| Many states | State machine | More than 3-4 states with rules |
| Reuse behavior | Mixins / composition | Avoid deep inheritance trees |
| Cleanup | `destroy()` + WeakRef | Long-lived SDKs that hold resources |
