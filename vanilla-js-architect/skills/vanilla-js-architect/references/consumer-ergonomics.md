# Consumer Ergonomics

How the SDK *feels* from the import statement onward. `distribution.md` covers *how to ship*; this file covers *what consumers type and what their bundler does with it*.

## The first import — design backwards from this

Before writing any internal code, write the import statement you want consumers to write. That dictates everything else.

```javascript
// What's the canonical first 5 minutes?
import { Client } from "my-sdk";
const client = new Client({ apiKey: "..." });
await client.fetch("/users/42");
```

Whatever you wrote there is now the contract:
- `my-sdk` resolves to a single ESM file
- `Client` is a named export, not a default
- `Client` is constructable with one options object
- `client.fetch` returns a Promise

Lock these decisions before writing internals.

## Subpath exports — designed entry points

The `package.json` `exports` field is the single most powerful tool for consumer ergonomics. It defines what consumers can import and gates everything else.

```json
{
  "name": "my-sdk",
  "type": "module",
  "exports": {
    ".":            "./dist/index.mjs",
    "./errors":     "./dist/errors.mjs",
    "./plugins":    "./dist/plugins/index.mjs",
    "./plugins/*":  "./dist/plugins/*.mjs",
    "./types":      "./dist/types.d.ts",
    "./package.json": "./package.json"
  }
}
```

```javascript
// Consumers can import:
import { Client } from "my-sdk";
import { SDKError, HTTPError } from "my-sdk/errors";
import { loggerPlugin } from "my-sdk/plugins";
import { authPlugin } from "my-sdk/plugins/auth"; // pattern wildcard
import meta from "my-sdk/package.json" with { type: "json" };

// But this throws — internals are not in `exports`:
import { internalQueue } from "my-sdk/internal/queue"; // ❌ Module not found
```

### When to add a subpath export

| Consumer situation | Add subpath? |
|--------------------|--------------|
| Tree-shake unused error classes from main bundle | `./errors` |
| Optional plugins consumer opts in to | `./plugins/*` |
| Type-only utilities (consumer's TS uses them) | `./types` |
| Heavy feature most consumers won't use | `./advanced` |
| Polyfilled build for legacy targets | `./polyfilled` |
| Internal helper (consumer should NEVER touch) | **No subpath** — leave unreachable |

### Anti-pattern: dumping everything into `index.mjs`

```javascript
// ❌ index.mjs that re-exports literally everything
export * from "./client.js";
export * from "./errors.js";
export * from "./plugins/index.js";
export * from "./advanced/index.js"; // ← every consumer pays for this
```

Result: consumers who only use `Client` still pay (in bundle size) for advanced features the bundler can't drop because of side effects elsewhere. Use subpath exports to split.

## Deep imports — when to allow, when to block

**Default policy: block deep imports into your package.** Only the paths in `exports` are reachable. Internal reorganizations don't break consumers.

```json
// ✅ Restrictive — internals unreachable
{
  "exports": {
    ".": "./dist/index.mjs",
    "./plugins": "./dist/plugins/index.mjs"
  }
}
```

```javascript
// Consumer doing this fails at install/build time:
import { TransportImpl } from "my-sdk/internal/transport"; // ❌ Module not found
```

**When to allow some deep imports (rare):** the SDK is a "kit" of independent utilities (e.g. `lodash-es`-style). Even then, `exports` should enumerate them — never use `"./*": "./dist/*"` in production.

## Conditional exports — runtime-aware entry points

```json
{
  "exports": {
    ".": {
      "browser":     "./dist/index.browser.mjs",
      "worker":      "./dist/index.worker.mjs",
      "deno":        "./dist/index.deno.mjs",
      "bun":         "./dist/index.mjs",
      "node":        "./dist/index.node.mjs",
      "default":     "./dist/index.mjs"
    },
    "./fetch": {
      "browser":     "./dist/fetch.browser.mjs",
      "node":        "./dist/fetch.node.mjs"
    }
  }
}
```

Bundlers and runtimes pick the matching condition. Order matters — most-specific first, `default` last as fallback. **Always include `default`** so unrecognized environments still get something working.

### Common conditions, in evaluation order

1. `worker` — Web Worker / Service Worker bundles
2. `browser` — bundlers targeting browsers
3. `deno`, `bun`, `react-native` — runtime-specific
4. `node` — Node.js
5. `import` — any ESM consumer (bundler condition)
6. `require` — any CJS consumer
7. `default` — fallback

```json
{
  "exports": {
    ".": {
      "types":   "./dist/index.d.ts",
      "browser": "./dist/index.browser.mjs",
      "import":  "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

`types` should always come **first** — TS resolution stops at the first match.

## Naming the package and its subpaths

Consumers see the package name everywhere. Choose carefully.

| Convention | Example | When |
|------------|---------|------|
| Plain name | `pino`, `zod` | Single-purpose, brand-distinct |
| Scoped | `@vendor/sdk` | Org-owned, namespace clarity |
| Scoped + suffix | `@vendor/sdk-core`, `@vendor/sdk-react` | Multi-package suite |
| `*-js` suffix | `tweetnacl-js` | Disambiguating from non-JS counterpart |

**Subpath naming rules:**
- Use kebab-case, never camelCase: `my-sdk/web-vitals`, not `my-sdk/webVitals`
- Avoid `/index` in subpaths; let resolver pick `index.mjs` from a `/dir` mapping
- Avoid `/src/...` paths in `exports` — exposes implementation detail
- Avoid pluralization inconsistency: pick one (`/plugin` vs `/plugins`) and stick with it across versions

## `imports` field — internal aliasing

Use `imports` (with `#` prefix) for **internal** aliases. Helps avoid `../../../` chains in source without leaking to consumers.

```json
{
  "imports": {
    "#errors":     "./src/errors.js",
    "#transport/*": "./src/transport/*.js"
  }
}
```

```javascript
// In your own source:
import { SDKError } from "#errors";
import { httpClient } from "#transport/http";

// Consumers cannot use these — `#` prefix is package-internal
```

**Watch out:** `imports` are resolved relative to the package's own `package.json`. Bundlers honor them at build time. Test suites (Vitest, etc.) need configuration to resolve them.

## Import maps — same syntax, different scope

`importmap` is a browser feature (and bundler convention) that lets the consumer rename packages globally. SDK authors don't write import maps; consumers do. But the SDK should be import-map-friendly.

```html
<!-- Consumer's HTML -->
<script type="importmap">
{
  "imports": {
    "my-sdk": "https://esm.sh/my-sdk@1",
    "my-sdk/errors": "https://esm.sh/my-sdk@1/errors"
  }
}
</script>
<script type="module">
  import { Client } from "my-sdk";
</script>
```

For this to work, the SDK must:
1. Have a clean `exports` map (every public path maps to an `.mjs`)
2. Not rely on internal deep imports between subpaths (each subpath is independently fetchable)
3. Reference external runtime deps via bare specifiers (so consumers can map them)

## CDN-friendly distribution

```html
<!-- ESM via CDN — modern -->
<script type="module">
  import { Client } from "https://esm.sh/my-sdk@1";
</script>

<!-- IIFE drop-in — legacy or no-build -->
<script src="https://unpkg.com/my-sdk@1/dist/index.iife.js"></script>
<script>
  const client = new MySDK.Client({ apiKey: "x" });
</script>
```

For CDN to work cleanly:
- `unpkg` and `jsdelivr` fields in `package.json` point at the IIFE build
- IIFE global name is the PascalCase of the package: `MySDK`, not `mySdk`
- Don't rely on `process.env.NODE_ENV` in CDN builds — it's `undefined`

## Versioning the import path

```javascript
// Consumers expect a stable import path across patch + minor versions
import { Client } from "my-sdk";

// Major version bumps with breaking changes can be exposed via package suffix:
import { Client as ClientV1 } from "my-sdk-v1"; // legacy
import { Client } from "my-sdk";                 // current

// Or distinct package names per major (most-conservative):
// my-sdk@1.x, my-sdk@2.x → consumer pins via package.json
```

Don't bake major versions into subpaths (`my-sdk/v2/Client`). Use semver via npm.

## ESM-only or dual ESM/CJS — pick one consciously

| Strategy | Pros | Cons |
|----------|------|------|
| **ESM-only** | Simpler, smaller, true tree-shaking, no dual-package hazard | Excludes legacy CJS consumers (some old Webpack/test setups) |
| **Dual ESM + CJS** | Maximum reach | Risk of double-loading state ("dual package hazard"); need careful `exports` |
| **CJS-only** | Old Node compat | Loses tree-shaking; rejected by modern bundlers' best paths |

**Recommendation for new SDKs in 2025+: ESM-only.** Document this clearly in the README. Modern bundlers (Vite, esbuild, Rollup, webpack 5+) and Node 18+ all handle ESM cleanly. CJS-only consumers are a shrinking minority.

If you do dual:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "default": "./dist/index.mjs"
    }
  }
}
```

Make sure both builds share **no module-level state**. The dual package hazard happens when two copies of your SDK (one ESM, one CJS) get loaded into the same process and each holds its own state.

## Top-level `await` — caveat for SDK authors

```javascript
// In an SDK module — top-level await blocks consumers' module graph
const config = await fetch("/config").then(r => r.json());
export const client = new Client(config);

// Consumer's import now waits on this network call before the page can run
```

**Rule:** never use top-level `await` in SDK code that runs at import time. Defer to a function or async factory.

```javascript
// ✅ Lazy
export async function init() {
  const config = await fetch("/config").then(r => r.json());
  return new Client(config);
}
```

## Side-effect-free module loading

```javascript
// ❌ Side effects at module top — blocks tree-shaking, runs even when unused
console.log("[my-sdk] loaded");
window.__sdkVersion = "1.0";
const cache = init();
import "./polyfills.js";

// ✅ Pure exports only
export const VERSION = "1.0";
export class Client { /* ... */ }
```

`package.json` declares this:

```json
{ "sideEffects": false }
```

If some files genuinely have side effects (CSS imports, polyfill bundles), enumerate them:

```json
{ "sideEffects": ["*.css", "./dist/polyfills.mjs"] }
```

## Quick Reference — the consumer ergonomics checklist

| Concern | What to do |
|---------|-----------|
| Single import path | One canonical entry per subpath; don't `*` re-export all of them |
| Deep imports | Block by default; only allow what's in `exports` |
| Runtime variance | Use conditional `exports` (`browser`, `worker`, `node`, `default`) |
| Tree-shaking | Named exports; `sideEffects: false`; no top-level work |
| Type-aware import | Put `types` first in conditional exports |
| Internal aliases | `imports` field with `#prefix` (consumer can't see) |
| CDN drop-in | `unpkg`/`jsdelivr` → IIFE; PascalCase global |
| Top-level await | Never in SDK module — defer to factory |
| ESM vs CJS | ESM-only by default; dual only if you must |
| Subpath naming | kebab-case, no `/index`, no `/src/` leaks |
| Major version path | Use semver/npm pin; don't put `/v2/` in path |
