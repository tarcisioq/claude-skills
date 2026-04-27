# Types from JSDoc

Modern vanilla JS SDKs ship `.d.ts` declarations **without converting source to TypeScript**. Consumers get full editor intellisense and TS-project compatibility; you keep `.js` source. Best of both.

## Why this matters

- TS users represent the majority of bundler-using consumers. No types = "is this maintained?" signal
- Migrating to TS is invasive (build pipeline, tooling, mental model). JSDoc is incremental
- `.d.ts` from JSDoc is **lossless for what consumers see** — the source stays JS

## The minimal setup

```bash
npm install --save-dev typescript
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "./dist",
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmitOnError": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.js"],
  "exclude": ["node_modules", "tests"]
}
```

```json
// package.json
{
  "scripts": {
    "build:types": "tsc",
    "build": "rollup -c && tsc"
  },
  "exports": {
    ".": {
      "types":   "./dist/index.d.ts",
      "import":  "./dist/index.mjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

`types` **must** be the first key in conditional exports. TS resolution stops at the first match.

## JSDoc patterns — the canonical set

### Basic function annotations

```javascript
/**
 * Fetches a user by id.
 *
 * @param {string} id — non-empty user identifier
 * @param {Object} [options]
 * @param {AbortSignal} [options.signal] — cancel mid-flight
 * @returns {Promise<User>}
 * @throws {SDKError} with `code` "INVALID_ARGUMENT" if id is empty
 * @throws {SDKError} with `code` "NOT_FOUND" if user doesn't exist
 *
 * @example
 *   const user = await client.fetchUser("usr_123");
 */
async function fetchUser(id, { signal } = {}) {
  if (!id) throw new SDKError("id required", "INVALID_ARGUMENT");
  // ...
}
```

### `@typedef` for shared types

```javascript
/**
 * @typedef {Object} User
 * @property {string} id
 * @property {string} email
 * @property {Date} createdAt
 * @property {"active" | "suspended"} status
 */

/**
 * @typedef {Object} ClientOptions
 * @property {string} apiKey — your public-safe API key
 * @property {string} [baseUrl] — defaults to "https://api.example.com"
 * @property {number} [timeout] — defaults to 5000ms
 * @property {typeof fetch} [fetch] — for tests / custom transports
 */
```

Place `@typedef` blocks at the top of the file or in a dedicated `types.js`:

```javascript
// src/types.js
/**
 * @typedef {Object} User
 * @property {string} id
 * @property {string} email
 */

// Empty export — `types.js` exists only for the JSDoc declarations
export {};
```

```javascript
// Other files — import the typedef
/** @typedef {import("./types.js").User} User */

/** @returns {Promise<User>} */
async function fetchUser() { /* ... */ }
```

### Generics via `@template`

```javascript
/**
 * @template T
 * @param {T[]} items
 * @param {(item: T) => boolean} predicate
 * @returns {T | undefined}
 */
function findFirst(items, predicate) {
  for (const item of items) if (predicate(item)) return item;
}

// Constrained generics
/**
 * @template {{ id: string }} T
 * @param {T[]} items
 * @param {string} id
 * @returns {T | undefined}
 */
function findById(items, id) {
  return items.find(it => it.id === id);
}
```

### Function overloads via `@overload`

```javascript
/**
 * @overload
 * @param {string} key
 * @returns {Promise<string | null>}
 *
 * @overload
 * @param {string} key
 * @param {string} defaultValue
 * @returns {Promise<string>}
 *
 * @param {string} key
 * @param {string} [defaultValue]
 * @returns {Promise<string | null>}
 */
async function getValue(key, defaultValue) {
  const v = await storage.get(key);
  return v ?? defaultValue ?? null;
}
```

### Class with private fields

```javascript
/**
 * @typedef {Object} ClientOptions
 * @property {string} apiKey
 */

export class Client {
  /** @type {string} */
  #apiKey;

  /** @type {Array<unknown>} */
  #queue = [];

  /** @param {ClientOptions} options */
  constructor(options) {
    this.#apiKey = options.apiKey;
  }

  /**
   * @param {string} eventName
   * @returns {Promise<void>}
   */
  async track(eventName) {
    this.#queue.push(eventName);
  }
}
```

### Generic classes

```javascript
/**
 * @template T
 */
export class TypedQueue {
  /** @type {T[]} */
  #items = [];

  /** @param {T} item */
  push(item) {
    this.#items.push(item);
  }

  /** @returns {T | undefined} */
  pop() {
    return this.#items.shift();
  }
}

// Consumer (TS file):
const q = new TypedQueue<User>();
q.push({ id: "u1", email: "x" });  // ✅
q.push("hello");                    // ❌ Argument of type 'string' is not assignable
```

### Discriminated unions for error classes

```javascript
/**
 * @typedef {"INVALID_ARGUMENT" | "NETWORK" | "RATE_LIMITED" | "AUTH_FAILED" | "NOT_FOUND"} SDKErrorCode
 */

export class SDKError extends Error {
  /**
   * @param {string} message
   * @param {SDKErrorCode} code
   * @param {{ cause?: Error, details?: Record<string, unknown> }} [options]
   */
  constructor(message, code, { cause, details } = {}) {
    super(message, { cause });
    /** @type {"SDKError"} */
    this.name = "SDKError";
    /** @type {SDKErrorCode} */
    this.code = code;
    /** @type {Record<string, unknown> | undefined} */
    this.details = details;
  }
}

// Consumer code can switch on `code` with full type narrowing:
try {
  await client.fetchUser(id);
} catch (err) {
  if (err instanceof SDKError) {
    if (err.code === "RATE_LIMITED") retryLater();
    if (err.code === "NOT_FOUND") showNotFound();
  }
}
```

### Events — typed event maps

```javascript
/**
 * @typedef {Object} ClientEventMap
 * @property {Event} ready
 * @property {CustomEvent<{ error: SDKError }>} error
 * @property {CustomEvent<{ prev: string, next: string }>} state-change
 */

/**
 * @typedef {Object} TypedEventTarget
 * @property {<K extends keyof ClientEventMap>(type: K, listener: (event: ClientEventMap[K]) => void) => void} addEventListener
 * @property {<K extends keyof ClientEventMap>(type: K, listener: (event: ClientEventMap[K]) => void) => void} removeEventListener
 */

/** @extends {EventTarget} */
export class Client extends EventTarget {
  // Implementation extends EventTarget; consumers see the typed map
}
```

This is the trickiest pattern — typed event maps require careful shape. For complex APIs, a hand-written `.d.ts` augmentation alongside generated types is sometimes easier.

### `@deprecated`, `@since`, `@see`

```javascript
/**
 * @deprecated since 2.5.0. Use {@link fetchUser} instead.
 * Will be removed in v3.0.
 * @see fetchUser
 */
export function getUser(id) {
  return fetchUser(id);
}

/**
 * @since 2.0.0
 * @param {string} id
 */
export function fetchUser(id) { /* ... */ }
```

Surfaced by editors as strikethrough + tooltip.

## Strict mode — turn it on, fix what breaks

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "checkJs": true,        // type-check JSDoc as you write
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}
```

When `checkJs: true`, the TS compiler checks your JS source against JSDoc. Use it during development:

```bash
# Watch mode while writing
npx tsc --noEmit --watch
```

Fix the errors as they appear. The result is typed-as-tightly-as-TS without a TS migration.

## What can't JSDoc express?

A few advanced TS features have no clean JSDoc equivalent:

- **Conditional types** (`T extends X ? Y : Z`) — limited; use `@typedef {...}` with multiple definitions
- **Mapped types** (`{[K in keyof T]: ...}`) — partial support via `@typedef`
- **Type assertions** — `/** @type {Foo} */ (value)` works but is awkward
- **Module augmentation** — write a small hand-edited `.d.ts` alongside

For these cases, ship a hand-written supplementary `.d.ts` for the small surface that needs it.

```
dist/
├── index.mjs
├── index.d.ts        ← generated from JSDoc
└── advanced.d.ts     ← hand-written, narrowly scoped
```

## Validate the published types

After build, smoke-test that consumers will get sensible types:

```typescript
// tests/types.test.ts (run with `tsc --noEmit tests/types.test.ts`)
import { Client, SDKError, type User } from "my-sdk";

const client = new Client({ apiKey: "k" });

// Should compile:
const user: User = await client.fetchUser("u1");

// Should NOT compile:
// @ts-expect-error - missing apiKey
new Client({});

// @ts-expect-error - wrong code value
const err = new SDKError("x", "WRONG_CODE");

// @ts-expect-error - User doesn't have password
console.log(user.password);
```

The `@ts-expect-error` directives flip the assertion: the line MUST produce a TS error. If a future change accidentally allows it, tests fail.

## Common pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `@param {string} [foo]` with no `=` default | TS sees `string \| undefined` always | Add `= ""` or document with `\| undefined` |
| Returning `Promise` without `@returns` | Type becomes `Promise<any>` | Always `@returns {Promise<T>}` |
| `@type {Object}` | Becomes `{}` (empty interface) | Use `@type {Record<string, unknown>}` or named type |
| `@typedef` in every file | Drift between definitions | Centralize in `src/types.js`, import via `@typedef {import(...)}` |
| Missing `types` in `exports` | TS users see `any` | Add `"types": "./dist/index.d.ts"` first in conditional exports |
| `tsc` succeeds but `.d.ts` empty | Source files excluded | Check `include`/`exclude` in `tsconfig.json` |

## When to switch to TS source instead

JSDoc is the right answer for **most** SDKs. Switch to TS source when:

- Your contributors strongly prefer TS and would write nicer code
- You need conditional types or template literal types extensively
- Your audience is 100% TS users (rare)

JSDoc stays superior when:

- You want the source to remain runnable without compilation in dev
- You target a wide audience including no-build / `<script>` tag users
- You want zero-build distribution (publish source as-is for ESM consumers)

## Quick Reference

| Need | Tag |
|------|-----|
| Function param | `@param {Type} name` |
| Optional param | `@param {Type} [name]` |
| Return | `@returns {Type}` |
| Throws | `@throws {Type}` (free-form description) |
| Type alias | `@typedef {...} Name` |
| Generics | `@template T` (constrained: `@template {Constraint} T`) |
| Overloads | `@overload` (one per signature, then implementation) |
| Import a type | `@typedef {import("./other.js").Foo} Foo` |
| Cast | `/** @type {Foo} */ (value)` |
| Class field | `@type {T}` above the field |
| Deprecate | `@deprecated since X. Use Y.` |
| Example | `@example\n  code here` |
| Type-check JS as you write | `tsconfig: checkJs: true, strict: true` |
| Emit `.d.ts` | `tsconfig: emitDeclarationOnly: true, declaration: true` |
| Expose in package | `"types": "./dist/index.d.ts"` first in exports |
