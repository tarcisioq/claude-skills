# Cross-Runtime Compatibility

A modern "browser SDK" runs in **far more places than browsers**: Web Workers, Service Workers, Cloudflare Workers, Deno, Bun, Node 18+, React Native (with `react-native-fetch`), and increasingly edge platforms (Vercel Edge, Netlify Edge, Fastly Compute@Edge). The good news: they all converge on **Web Standards**. The bad news: Node-only APIs leak silently from training data.

This file is the cross-runtime portability checklist.

## The Web Standards that work everywhere

These are available in browsers, Workers, Deno, Bun, Node 18+, and edge runtimes:

```javascript
// Networking
fetch, Request, Response, Headers, FormData, URL, URLSearchParams

// Streams
ReadableStream, WritableStream, TransformStream

// Encoding
TextEncoder, TextDecoder

// Crypto
crypto, crypto.subtle, crypto.getRandomValues, crypto.randomUUID

// Async
AbortController, AbortSignal, AbortSignal.timeout, AbortSignal.any
queueMicrotask, structuredClone

// Timing
performance, performance.now, performance.mark, performance.measure
setTimeout, setInterval, clearTimeout, clearInterval

// Data
ArrayBuffer, Uint8Array, DataView, Blob (most runtimes)
Map, Set, WeakMap, WeakRef, FinalizationRegistry

// Errors
Error, AggregateError, DOMException

// Events (in modern Node + Workers; partial in older)
Event, EventTarget, CustomEvent, MessageChannel, BroadcastChannel
```

**These are your primitives.** Build the SDK around them and it ports for free.

## What doesn't work — Node-only APIs to avoid

```javascript
// ❌ — Node-only, breaks in browsers/Workers/Deno
process              // env vars, argv, exit
Buffer               // use Uint8Array + TextEncoder/Decoder
require              // use ESM `import`
__dirname            // use `new URL("./", import.meta.url)`
__filename           // same
fs, path, os, child_process, http, https, net, tls, dgram  // Node-specific

// ⚠️ — works in some runtimes but not all
globalThis.window     // browser only
globalThis.document   // browser only (and DOM iframes)
localStorage          // browser + Deno (with --location); not Workers/Node
indexedDB             // browser; partial in Workers; absent in Node by default
WebSocket             // browser + Deno + Bun + Node 22+; not all Workers
```

If you must use a Node API, isolate it behind a runtime check or conditional export (see below).

## Runtime detection

Don't sniff user agents. Detect features.

```javascript
// ✅ Feature detection
const isBrowser = typeof document !== "undefined";
const isWorker = typeof WorkerGlobalScope !== "undefined" && self instanceof WorkerGlobalScope;
const isServiceWorker = typeof ServiceWorkerGlobalScope !== "undefined";
const isDeno = typeof Deno !== "undefined";
const isBun = typeof Bun !== "undefined";
const isNode = typeof process !== "undefined" && process.versions?.node != null && !isDeno && !isBun;
const isCloudflareWorker = typeof globalThis.Cloudflare !== "undefined" || (typeof navigator !== "undefined" && navigator.userAgent === "Cloudflare-Workers");

// ✅ Better — detect the API you need, not the runtime
const hasFetch = typeof fetch === "function";
const hasCrypto = typeof crypto?.subtle?.digest === "function";
const hasIndexedDB = typeof indexedDB !== "undefined";
const hasLocalStorage = (() => {
  try { return typeof localStorage !== "undefined" && (localStorage.x = "1", delete localStorage.x, true); }
  catch { return false; }
})();
```

The `hasLocalStorage` IIFE handles the case where `localStorage` exists but throws on access (Safari private mode, sandboxed iframes).

## Conditional exports for runtime branching

When the SDK has genuinely different code per runtime, use the `exports` field's conditions.

```json
{
  "exports": {
    ".": {
      "workerd":      "./dist/index.workerd.mjs",   // Cloudflare Workers
      "deno":         "./dist/index.deno.mjs",
      "bun":          "./dist/index.bun.mjs",
      "browser":      "./dist/index.browser.mjs",
      "node":         "./dist/index.node.mjs",
      "default":      "./dist/index.mjs"
    }
  }
}
```

Bundlers and runtimes pick the matching condition. Always include `default`.

**Use sparingly.** Most SDKs don't need this — Web Standards-only code with a single `index.mjs` is the goal. Conditional exports are a pressure valve for cases where there's no portable primitive.

### Common condition order (resolver-evaluated)

```
workerd → deno → bun → react-native → browser → node → import → require → default
```

Most-specific to least-specific, with `default` as the safety net.

## Storage portability

```javascript
// ✅ Pluggable storage that works everywhere
function defaultStorage() {
  if (typeof indexedDB !== "undefined") return idbStorage();
  if (hasLocalStorage()) return localStorageStorage();
  return memoryStorage();  // last resort: Workers without IDB, sandboxed iframes
}
```

**Cloudflare Workers note:** no IndexedDB, no localStorage. Use KV namespace or D1 — but those are not built-in to the runtime; they're consumer-bound. The SDK should accept a storage adapter (see `browser-primitives.md`).

## Networking portability

```javascript
// ✅ Use Web Standards everywhere
const url = new URL("/users", baseUrl);
url.searchParams.set("limit", "10");

const req = new Request(url, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload),
  signal,
});

const res = await fetch(req);
const data = await res.json();

// ❌ Node-only
const http = require("http");
http.request(...);
```

**Cloudflare Workers gotcha:** `fetch` works, but `cache: "no-store"` is the default; some `fetch` options behave differently. Test in `wrangler dev` to verify.

**React Native gotcha:** `fetch` exists but may not support all options (e.g. `keepalive`, streaming responses). Document caveats.

## Crypto portability

```javascript
// ✅ Web Crypto works in browsers, Workers, Deno, Bun, Node 19+ (Node 18 has it under different conditions)
const buf = new TextEncoder().encode(input);
const hash = await crypto.subtle.digest("SHA-256", buf);

// ✅ randomUUID works almost everywhere now (browsers Q3 2021+, Node 14.17+, Deno, Bun)
const id = crypto.randomUUID();

// ❌ Node-only
const nodeCrypto = require("crypto");
nodeCrypto.createHash("sha256").update(input).digest("hex");
```

For Node 16 and earlier (rare but exists), `crypto.subtle` is under `require("crypto").webcrypto.subtle`. If your `engines.node` targets that low, conditional export it. For Node 18+: just use `crypto`.

## Encoding portability

```javascript
// ✅ Web Standards
const bytes = new TextEncoder().encode("hello");
const text = new TextDecoder().decode(bytes);

// ❌ Node-only
const buf = Buffer.from("hello");
const text = buf.toString("utf8");
```

For Base64:

```javascript
// ✅ Universal (browsers, Workers, Deno, Bun, Node 16+)
const b64 = btoa(String.fromCharCode(...uint8));
const bytes = Uint8Array.from(atob(b64), c => c.charCodeAt(0));

// ✅ Even better — modern Uint8Array methods (Stage 4, 2024)
// Browsers: Chrome 133+, Firefox 138+; Node 22+; Deno 1.40+
const b64 = uint8.toBase64();
const bytes = Uint8Array.fromBase64(b64);
```

Document the minimum runtime if you use the new methods, or polyfill.

## Event APIs portability

```javascript
// ✅ EventTarget works in browsers, Workers, Deno, Bun, Node 16+
class SDK extends EventTarget {
  ready() { this.dispatchEvent(new Event("ready")); }
}

// ❌ — Node EventEmitter is NOT available in browsers/Workers/Deno
const { EventEmitter } = require("events");
```

Always extend `EventTarget`, not Node's `EventEmitter`. The API is slightly different (`addEventListener`/`dispatchEvent` vs `on`/`emit`) but works everywhere.

## Streams portability

Web Streams (`ReadableStream`, `WritableStream`, `TransformStream`) work in:

- Browsers (all modern)
- Cloudflare Workers, Deno, Bun
- Node 18+ (under the global; Node has parallel `stream` module too — don't mix)

```javascript
// ✅ Use the Web Streams everywhere
const transform = new TransformStream({
  transform(chunk, controller) {
    controller.enqueue(chunk.toUpperCase());
  },
});

const upperStream = response.body.pipeThrough(transform);
```

```javascript
// ❌ Node-only Readable
const { Readable } = require("stream");
```

Web Streams interop with Node streams is improving but still imperfect — write Web Streams in your SDK, document the conversion if consumers mix.

## Timer portability

`setTimeout` / `setInterval` work everywhere, **but their return type differs**:

- Browsers: `number`
- Node: `Timeout` object
- Workers/Deno/Bun: `number`

```javascript
// ✅ Treat the return as opaque
let timerId = setTimeout(fn, 1000);
clearTimeout(timerId); // works regardless of type

// ❌ — Node-specific timer methods
timerId.unref(); // Node only — undefined elsewhere
timerId.ref();
```

If you need Node's `unref()`, feature-detect:

```javascript
const t = setTimeout(fn, 1000);
if (typeof t.unref === "function") t.unref();
```

## File-like APIs

`Blob` and `File` work in browsers, Workers, Deno, Bun, and Node 18+. Reading files differs:

```javascript
// ✅ Browser/Worker — File from input or fetch
const file = inputElement.files[0];
const text = await file.text();

// ✅ Node — read from disk (Node-only path)
import { readFile } from "node:fs/promises";
const buffer = await readFile("./path"); // returns Buffer (Node) or Uint8Array (Bun)
const text = buffer.toString("utf8");
```

If your SDK reads files, accept a `Blob | File | ArrayBuffer | Uint8Array`. Don't accept paths — the consumer reads the file in their runtime.

## Environment variables

```javascript
// ❌ — Node-only `process.env`
const apiKey = process.env.API_KEY;

// ✅ — pass via constructor; consumer handles env
const sdk = new SDK({ apiKey: getApiKey() });

// Where consumers can use whatever their runtime provides:
// - Node: process.env
// - Deno: Deno.env.get
// - Workers: env.API_KEY (passed to the handler)
// - Browser: build-time substitution (Vite import.meta.env, etc.)
```

The SDK never reads env vars. Consumers pass them in.

## Globals — `globalThis` is the safe accessor

```javascript
// ✅ Works in every runtime
const fetchRef = globalThis.fetch;

// ❌ Runtime-specific globals
const fetchRef = window.fetch;   // browser-only
const fetchRef = global.fetch;   // Node-only (and deprecated)
const fetchRef = self.fetch;     // browser + Worker, not Node
```

`globalThis` was added precisely to unify these. Use it whenever you need a global by name.

## Testing cross-runtime claims

If your `package.json` says "works in Node, Deno, Bun, browsers" — your CI must prove it.

```yaml
# .github/workflows/ci.yml
strategy:
  matrix:
    runtime: [node-18, node-20, node-22, deno, bun]
jobs:
  test:
    steps:
      - run: npm test          # node
      - run: deno test ...     # deno
      - run: bun test          # bun
      - run: npx vitest --browser=chromium  # real browser
```

Skip the runtimes you don't claim to support. Don't claim what you don't test.

## Documenting supported runtimes

In the README, every SDK should have a table:

```markdown
## Supported runtimes

| Runtime | Min version | Notes |
|---------|-------------|-------|
| Browsers | Chrome 91+, Firefox 90+, Safari 15+, Edge 91+ | |
| Node.js | 18.0+ | Use `--experimental-fetch` on 17 (deprecated) |
| Deno | 1.30+ | |
| Bun | 1.0+ | |
| Cloudflare Workers | All | Set `compatibility_date` ≥ 2023-01-01 |
| React Native | 0.71+ | Requires `react-native-url-polyfill` |
```

Consumers grep this. Be specific. Don't say "modern browsers" — say which versions and why.

## Quick Reference

| Web Standard primitive | Avoid |
|------------------------|-------|
| `fetch`, `Request`, `Response`, `Headers` | `http.request`, `https.request` |
| `URL`, `URLSearchParams` | String concatenation |
| `crypto.subtle`, `crypto.randomUUID` | `require("crypto")` |
| `TextEncoder`, `TextDecoder` | `Buffer` |
| `EventTarget` | `EventEmitter` |
| `ReadableStream`, `WritableStream` | Node's `stream` module |
| `globalThis` | `window`, `global`, `self` |
| `AbortController`, `AbortSignal` | Custom cancellation tokens |
| `structuredClone` | `JSON.parse(JSON.stringify(...))` |
| Constructor injection for env | `process.env` |
| Constructor injection for storage | `localStorage` direct access |
| Conditional `exports` for branching | `if (typeof process !== "undefined")` runtime checks scattered |
