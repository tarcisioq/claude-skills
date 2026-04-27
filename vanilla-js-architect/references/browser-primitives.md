# Browser Primitives for SDK Authors

The browser-API surface that **library authors** actually need. Application-level concerns (Service Worker caching strategies, Canvas drawing, full PWA flows) live elsewhere — this file covers the primitives an SDK *uses internally* or *exposes to consumers as a stable abstraction*.

## Storage abstraction — never hardcode `localStorage`

Consumers run your SDK in environments where `localStorage` doesn't exist (Workers, sandboxed iframes, SSR-prepass) or shouldn't be used (sensitive data, cross-tab sync needs). Always abstract.

```javascript
// ✅ SDK accepts a storage adapter; defaults to localStorage but never assumes
export class SDK {
  #storage;
  constructor({ storage = defaultStorage() } = {}) {
    this.#storage = storage;
  }
}

// Storage interface (informal contract — document in JSDoc)
/**
 * @typedef {Object} Storage
 * @property {(key: string) => Promise<string|null>} get
 * @property {(key: string, value: string) => Promise<void>} set
 * @property {(key: string) => Promise<void>} delete
 */

function defaultStorage() {
  if (typeof localStorage === "undefined") return memoryStorage();
  return {
    async get(k) { return localStorage.getItem(k); },
    async set(k, v) { localStorage.setItem(k, v); },
    async delete(k) { localStorage.removeItem(k); },
  };
}

function memoryStorage() {
  const map = new Map();
  return {
    async get(k) { return map.get(k) ?? null; },
    async set(k, v) { map.set(k, v); },
    async delete(k) { map.delete(k); },
  };
}
```

**Consumer override pattern:**
```javascript
// In a Worker — provide your own
const sdk = new SDK({ storage: kvNamespaceStorage(env.KV) });
```

**Anti-pattern: shipping `localStorage` reads at module top-level.** Throws on import in non-browser runtimes. Always lazy.

## Storage tier choice — when to use what

| Storage | Sync/Async | Quota | Cross-tab | Persistence | Use for |
|---------|-----------|-------|-----------|-------------|---------|
| `localStorage` | Sync | ~5-10 MB | No (use `storage` event for cross-tab) | Persistent | Small string config, user prefs |
| `sessionStorage` | Sync | ~5-10 MB | No | Tab lifetime | Per-tab transient |
| Cookies | Sync | ~4 KB total | Yes | Configurable | Auth (HttpOnly), CSRF tokens |
| IndexedDB | Async | Large (10s of MB to GBs) | Yes (via BroadcastChannel) | Persistent | Structured data, blobs, queues |
| Cache API | Async | Large | Yes | Persistent | HTTP responses, SW caching |
| Memory | Sync | RAM | No | Lifetime of page | Hot caches, dedup maps |

**Default for SDK persistent state: IndexedDB via a thin wrapper.** Sync storage blocks the main thread on big values.

```javascript
// ✅ Minimal IndexedDB wrapper — no `idb` library needed for simple KV
async function openDB(name, version, upgrade) {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(name, version);
    req.onupgradeneeded = () => upgrade(req.result);
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
}

class IDBStorage {
  #dbPromise;
  constructor({ dbName = "sdk-storage", storeName = "kv" } = {}) {
    this.storeName = storeName;
    this.#dbPromise = openDB(dbName, 1, (db) => {
      if (!db.objectStoreNames.contains(storeName)) {
        db.createObjectStore(storeName);
      }
    });
  }

  async #tx(mode, fn) {
    const db = await this.#dbPromise;
    return new Promise((resolve, reject) => {
      const tx = db.transaction(this.storeName, mode);
      const store = tx.objectStore(this.storeName);
      const result = fn(store);
      tx.oncomplete = () => resolve(result?.result ?? result);
      tx.onerror = () => reject(tx.error);
    });
  }

  get(k)          { return this.#tx("readonly",  s => s.get(k)); }
  set(k, v)       { return this.#tx("readwrite", s => s.put(v, k)); }
  delete(k)       { return this.#tx("readwrite", s => s.delete(k)); }
}
```

## Cross-tab + cross-context messaging

### `BroadcastChannel` — same-origin tabs

```javascript
// ✅ Use BroadcastChannel for cross-tab coordination (auth, cache invalidation)
const channel = new BroadcastChannel("my-sdk-events");

channel.postMessage({ type: "auth:logout" });

channel.onmessage = (e) => {
  if (e.data.type === "auth:logout") sdk.clearSession();
};

// Cleanup
channel.close();
```

### `postMessage` protocol — cross-origin

When the SDK exposes a `postMessage` interface (iframe SDK, popup SDK, Worker), always validate `event.origin` and version the protocol.

```javascript
// ✅ Always validate origin; structure messages with `type` and `version`
const ALLOWED_ORIGIN = "https://app.example.com";
const PROTOCOL_VERSION = 1;

window.addEventListener("message", (event) => {
  // 1. Origin check FIRST — never trust `event.data`
  if (event.origin !== ALLOWED_ORIGIN) return;

  const msg = event.data;

  // 2. Version check — protocol can evolve
  if (msg?.protocol !== "my-sdk" || msg?.version !== PROTOCOL_VERSION) return;

  // 3. Now safe to dispatch
  handle(msg.type, msg.payload);
});

// Outgoing — always specify target origin (NEVER use "*" with sensitive data)
iframe.contentWindow.postMessage(
  { protocol: "my-sdk", version: PROTOCOL_VERSION, type: "init", payload },
  ALLOWED_ORIGIN
);
```

**Anti-patterns:**
- `event.origin === "*"` (always-true checks): there is no `*` origin
- `postMessage(data, "*")` with anything sensitive — leaks to whatever ends up loaded
- Trusting `event.source` without origin check
- No protocol version field: forward-compatibility is impossible

### `MessageChannel` — paired ports for SDK ↔ iframe RPC

```javascript
// ✅ Establish a private channel between SDK and iframe
const channel = new MessageChannel();
iframe.contentWindow.postMessage({ type: "handshake" }, ALLOWED_ORIGIN, [channel.port2]);

// SDK side keeps port1; iframe receives port2
channel.port1.onmessage = (e) => handle(e.data);
channel.port1.postMessage({ type: "request", id: 1, method: "fetchUser", args: [42] });
```

A typed RPC over `MessageChannel` is the standard pattern for embedded-SDK-in-iframe (auth widgets, payment flows, chat). Add an `id` to correlate requests/responses.

## Capability detection — never UA-sniff

```javascript
// ✅ Test for the API directly
const supportsFetch = typeof fetch === "function";
const supportsAbortSignalAny = typeof AbortSignal.any === "function";
const supportsDispose = typeof Symbol.dispose === "symbol";
const supportsWebWorkers = typeof Worker === "function";
const supportsCrypto = typeof crypto?.subtle?.digest === "function";

// ✅ Feature ladder pattern — pick the best implementation available
function makeHasher() {
  if (supportsCrypto) {
    return async (input) => {
      const buf = new TextEncoder().encode(input);
      const hash = await crypto.subtle.digest("SHA-256", buf);
      return [...new Uint8Array(hash)].map(b => b.toString(16).padStart(2, "0")).join("");
    };
  }
  // Fallback for environments without SubtleCrypto (very rare; old Workers)
  return async () => { throw new SDKError("hashing unavailable", "UNSUPPORTED"); };
}
```

**Anti-pattern:** `navigator.userAgent.includes("Chrome")`. UA strings are unreliable, spoofable, and miss every alternative runtime (Workers, Deno, Bun). Always feature-detect.

## Observers as SDK extension hooks

Use cases where SDKs expose observers to consumers (or use them internally).

### `IntersectionObserver` — visibility-driven SDK behavior

```javascript
// ✅ SDK that lazy-initializes when visible; cleans up when not
class LazyWidget {
  #observer;
  #cleanup;

  constructor(element) {
    this.#observer = new IntersectionObserver((entries) => {
      for (const entry of entries) {
        if (entry.isIntersecting) this.#init();
        else this.#suspend();
      }
    }, { rootMargin: "200px" }); // pre-warm before fully visible

    this.#observer.observe(element);
  }

  destroy() {
    this.#observer.disconnect();
    this.#cleanup?.();
  }
}
```

### `MutationObserver` — react to DOM changes consumer makes

```javascript
// ✅ SDK that auto-attaches behavior to dynamically-added nodes
class AutoAttachSDK {
  #observer;

  start(root = document.body) {
    this.#observer = new MutationObserver((mutations) => {
      for (const m of mutations) {
        for (const node of m.addedNodes) {
          if (node.nodeType === Node.ELEMENT_NODE && node.matches?.("[data-sdk-widget]")) {
            this.#attach(node);
          }
        }
      }
    });
    this.#observer.observe(root, { childList: true, subtree: true });
  }

  destroy() { this.#observer?.disconnect(); }
}
```

### `ResizeObserver` — element size changes

```javascript
// ✅ Cheaper than `window.resize` polling; per-element granularity
const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    layout(entry.target, width, height);
  }
});
```

**Universal pattern: every observer the SDK creates needs `disconnect()` at destroy time.** Memory leaks via dangling observers are the #1 SDK leak source.

## Network primitives — `fetch`, `Request`, `URL`

Use Web Standards everywhere; they work in browsers + Workers + Deno + Bun.

```javascript
// ✅ Build URLs with `URL` — never string concat
const url = new URL("/users", baseUrl);
url.searchParams.set("limit", "10");
url.searchParams.set("cursor", cursor);
await fetch(url, { signal });

// ✅ Build Request objects when you need to clone or pass around
const req = new Request(url, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
  signal,
});

// SDK can clone for retry without re-stringifying
const retryable = req.clone();
```

```javascript
// ❌ String concat — encoding bugs, double `?`, missing `=`
const url = `${baseUrl}/users?limit=${limit}&cursor=${cursor}`;
```

## Encoding and crypto

```javascript
// ✅ Web Standards — works everywhere
const bytes = new TextEncoder().encode("hello");        // → Uint8Array
const text = new TextDecoder().decode(bytes);            // → "hello"

// Hashing
const buf = new TextEncoder().encode(input);
const hash = await crypto.subtle.digest("SHA-256", buf); // → ArrayBuffer

// Random
const id = crypto.randomUUID();                          // → "xxxxxxxx-xxxx-..."
const bytes2 = crypto.getRandomValues(new Uint8Array(16)); // CSPRNG

// Base64 (modern, no atob/btoa quirks for binary)
const b64 = btoa(String.fromCharCode(...bytes));         // for ASCII strings
// For binary, use Uint8Array.fromBase64 / toBase64 (Stage 4, 2024)
```

**Anti-patterns:**
- `Math.random()` for IDs or tokens — not cryptographically random
- `Buffer.from(...).toString("base64")` — Node-only, breaks in browsers
- Custom UUID generators — `crypto.randomUUID()` is everywhere now

## What this file deliberately does NOT cover

- Service Worker registration and caching strategies — application-level, use a dedicated SW skill
- Full Canvas/WebGL APIs — unless your SDK is a graphics library
- Push notifications + permission flows — application UX, not SDK architecture
- General DOM manipulation — application code

If your SDK genuinely needs one of these, treat them as application primitives the consumer wires in, not as built-in SDK behavior.

## Quick Reference

| Primitive | Use for | Watch out for |
|-----------|---------|---------------|
| Storage abstraction | Pluggable persistence | Lazy-detect `localStorage`; provide memory fallback |
| `BroadcastChannel` | Cross-tab same-origin | `close()` on cleanup |
| `postMessage` + origin check | Cross-origin RPC | NEVER skip `event.origin` validation |
| `MessageChannel` | Private SDK ↔ iframe RPC | Add `id` for request correlation |
| Capability detection | Runtime branching | NEVER UA-sniff |
| `IntersectionObserver` | Visibility-driven init | `disconnect()` in destroy |
| `MutationObserver` | React to consumer DOM | Same — `disconnect()` |
| `URL` / `URLSearchParams` | Build URLs | Never string-concat with user input |
| `Request` / `Response` | Clone for retry | `body` consumed once unless cloned |
| `crypto.subtle` | Hashing, signing | Async; need `await`; Web-only Standard |
| `crypto.randomUUID` | IDs | Available in browsers Q3 2021+; check fallback for older |
