# Distribution

How to package and ship a JavaScript SDK so it works for the broadest range of consumers — bundlers, CDNs, `<script>` tags — without bloat.

## Build Targets Overview

| Format | Use Case | Loaded Via |
|--------|----------|------------|
| **ESM** (`.mjs`) | Modern bundlers (Vite, esbuild, Rollup, webpack 5+), native browser modules | `import` |
| **UMD** | Legacy bundlers, `<script>` with global fallback | `<script>` or `require()` |
| **IIFE** | `<script>` tag drop-in, exposes a single global | `<script>` |
| **CJS** (`.cjs`) | Legacy Node.js or older tooling (rarely needed for browser SDKs) | `require()` |

For a modern browser SDK, **ESM is mandatory** and **IIFE/UMD is optional** depending on whether you target `<script>`-tag users.

## Recommended `package.json`

```json
{
  "name": "my-sdk",
  "version": "1.0.0",
  "type": "module",
  "sideEffects": false,
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "browser": "./dist/index.mjs",
      "default": "./dist/index.mjs"
    },
    "./plugins/*": {
      "import": "./dist/plugins/*.mjs"
    },
    "./package.json": "./package.json"
  },
  "main": "./dist/index.mjs",
  "module": "./dist/index.mjs",
  "browser": "./dist/index.mjs",
  "unpkg": "./dist/index.iife.js",
  "jsdelivr": "./dist/index.iife.js",
  "files": ["dist", "README.md", "LICENSE"],
  "browserslist": ["> 0.5%", "last 2 versions", "not dead", "not IE 11"]
}
```

**Field-by-field:**
- `type: "module"` — every `.js` in your source is ESM
- `sideEffects: false` — bundlers can drop unused exports (critical for tree-shaking)
- `exports` — gates what consumers can import; everything not listed is unreachable
- `unpkg`/`jsdelivr` — CDNs use these for the `<script>` build
- `files` — restricts what npm publishes; keep `src/` out unless you publish source maps

## The `exports` Field — Encapsulation at the Package Level

```json
{
  "exports": {
    ".": "./dist/index.mjs",
    "./plugins": "./dist/plugins/index.mjs",
    "./errors": "./dist/errors.mjs"
  }
}
```

```javascript
// Consumer can do:
import { Client } from "my-sdk";
import { loggerPlugin } from "my-sdk/plugins";
import { SDKError } from "my-sdk/errors";

// But this throws — internals are not in `exports`:
import { secret } from "my-sdk/internal/queue"; // ❌ blocked
```

This is the most powerful encapsulation primitive in the JS package ecosystem. Use it.

## Conditional Exports

```json
{
  "exports": {
    ".": {
      "browser": "./dist/index.browser.mjs",
      "node": "./dist/index.node.mjs",
      "default": "./dist/index.mjs"
    },
    "./fetch": {
      "browser": "./dist/fetch.browser.mjs",
      "node": "./dist/fetch.node.mjs"
    }
  }
}
```

The bundler picks the right entry based on its environment target. Use this when the SDK has divergent implementations per environment (e.g. `fetch` exists natively in browsers but might need a polyfill on older Node).

## Tree-Shaking Checklist

```javascript
// ✅ Named exports — bundler can drop unused ones
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;

// ✅ No side effects at module top-level
// Don't do this:
console.log("loading");           // ❌ side effect
window.__sdk = "1.0";             // ❌ side effect
const cache = init();             // ❌ side effect (function call at top level)

// ✅ Lazy initialization
let _cache;
function getCache() {
  return (_cache ??= init());
}

// ✅ Mark side-effect-free files explicitly when you have some that aren't
{
  "sideEffects": ["./dist/polyfills.mjs", "*.css"]
}
```

**Test**: import a single named export from your built bundle into a test app and check the output bundle size. If unused exports show up, tree-shaking is broken.

## Bundling with Modern Tools

```javascript
// rollup.config.mjs — most common for libraries
import { defineConfig } from "rollup";
import terser from "@rollup/plugin-terser";

export default defineConfig([
  // ESM build (primary)
  {
    input: "src/index.js",
    output: { file: "dist/index.mjs", format: "es", sourcemap: true },
  },
  // IIFE build for <script> users (optional)
  {
    input: "src/index.js",
    output: {
      file: "dist/index.iife.js",
      format: "iife",
      name: "MySDK",
      sourcemap: true,
    },
    plugins: [terser()],
  },
]);
```

Other modern options: **tsup** (esbuild-based, zero-config), **unbuild** (Vite ecosystem), **microbundle** (small libs).

## Bundle Size Discipline

```javascript
// 1. Lazy-load heavy features
export class SDK {
  async useAdvancedFeature() {
    const { AdvancedFeature } = await import("./advanced.js");
    return new AdvancedFeature();
  }
}

// 2. Avoid pulling in large dependencies; reach for native browser APIs first
// ❌ import _ from "lodash";
// ✅ const grouped = Object.groupBy(items, x => x.category);

// 3. Externalize peer dependencies in your bundler config
// rollup.config.mjs
export default {
  external: ["some-peer-dep"], // not bundled, consumer provides
};

// 4. Track size with size-limit or bundlejs.com
// package.json
{
  "scripts": {
    "size": "size-limit"
  },
  "size-limit": [
    { "path": "dist/index.mjs", "limit": "10 kB" }
  ]
}
```

## Polyfills and Browser Targets

```javascript
// ❌ Don't auto-polyfill — bloats every consumer's bundle
import "core-js/stable"; // forces every importer to ship 80kB of polyfills

// ✅ Document required globals in README, let consumers polyfill if needed
/**
 * Requires: fetch, AbortController, Promise.allSettled
 * For pre-2020 browser support, polyfill before importing.
 */
export class Client { /* ... */ }

// If you must polyfill, do it in a separate, opt-in entry:
// package.json
{
  "exports": {
    ".": "./dist/index.mjs",
    "./polyfilled": "./dist/index-polyfilled.mjs"
  }
}
```

Set `browserslist` to declare your support matrix and use it consistently for transpilation:

```json
{
  "browserslist": [
    "Chrome >= 90",
    "Firefox >= 90",
    "Safari >= 15",
    "Edge >= 90"
  ]
}
```

## Source Maps

```javascript
// Always ship source maps for production debuggability
// rollup.config.mjs
output: {
  sourcemap: true, // generates .mjs.map alongside .mjs
}

// Reference them in package.json
{
  "files": ["dist/**/*.mjs", "dist/**/*.mjs.map"]
}
```

## CDN Usability

```html
<!-- IIFE build via CDN -->
<script src="https://unpkg.com/my-sdk@1/dist/index.iife.js"></script>
<script>
  const client = new MySDK.Client({ apiKey: "x" });
</script>

<!-- ESM via CDN (modern browsers) -->
<script type="module">
  import { Client } from "https://esm.sh/my-sdk@1";
  const client = new Client({ apiKey: "x" });
</script>

<!-- Or with import maps for cleaner imports -->
<script type="importmap">
{ "imports": { "my-sdk": "https://esm.sh/my-sdk@1" } }
</script>
<script type="module">
  import { Client } from "my-sdk";
</script>
```

## Subresource Integrity (SRI) for CDN consumers

When you ship to CDNs, give consumers SRI hashes so script tags survive a CDN compromise.

```html
<!-- Consumer pins a specific build with its hash -->
<script
  src="https://cdn.jsdelivr.net/npm/my-sdk@1.2.3/dist/index.iife.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous"
></script>
```

Generate hashes at release time:

```bash
# Generate SRI hash for a built file
cat dist/index.iife.js | openssl dgst -sha384 -binary | openssl base64 -A
# Or via tool:
npx srihash dist/index.iife.js
```

Add the hash to your README + CHANGELOG for each released version. jsdelivr surfaces them automatically at `https://www.jsdelivr.com/package/npm/my-sdk?tab=files`.

## Supply chain — `npm publish --provenance`

```bash
# Sign and attest the package on publish (npm 9.5+)
npm publish --provenance --access public
```

Provenance attaches a verifiable build attestation to the package on the npm registry, signed by the CI provider. Requires:
- Publishing from GitHub Actions (or supported OIDC provider)
- `id-token: write` permission in the workflow
- npm package access set to public (or use scoped private with paid provenance)

Example workflow snippet:

```yaml
# .github/workflows/publish.yml
permissions:
  contents: read
  id-token: write   # required for provenance
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org
      - run: npm ci
      - run: npm run build
      - run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

Consumers can then verify with:

```bash
npm audit signatures
```

**Why this matters:** without provenance, a stolen npm token can publish malware as you. With provenance, the registry refuses to accept publishes that don't match the attested build.

## Pre-Publish Checklist (consolidated)

### Files
- [ ] `npm pack --dry-run` — only intended files (`dist/`, `README.md`, `LICENSE`, `package.json`)
- [ ] No `src/` shipped unless you also ship source maps that reference it
- [ ] No `.env`, `.npmrc`, secrets, test fixtures, dev scripts in the tarball

### Bundle hygiene
- [ ] No leaked Node APIs in browser/cross-runtime builds (grep for `process.`, `Buffer`, `require(`)
- [ ] No `eval`, `new Function`, `with` (CSP-hostile)
- [ ] No `console.log` in production paths (only debug-flag-gated)
- [ ] No top-level side effects (`grep -E "^[^/].*\(\)" dist/index.mjs` should return nothing meaningful)

### Tree-shaking
- [ ] `sideEffects: false` (or explicit list)
- [ ] Test consumer importing one named export drops the rest from final bundle
- [ ] Bundle size within `size-limit` budget

### `package.json`
- [ ] `type: "module"` if ESM
- [ ] `exports` map gates internals
- [ ] `unpkg` / `jsdelivr` point at IIFE build
- [ ] `browserslist` declares target matrix
- [ ] `peerDependencies` enumerated (not bundled)
- [ ] `engines.node` set if any tooling needs it

### Versioning
- [ ] `version` bumped per semver
- [ ] CHANGELOG entry written
- [ ] If breaking: deprecation warnings shipped in previous minor; migration guide in README

### Distribution + supply chain
- [ ] Source maps included and listed in `files`
- [ ] `npm publish --provenance --access public` (or equivalent OIDC flow)
- [ ] SRI hashes generated and added to README/CHANGELOG for IIFE/UMD builds
- [ ] `LICENSE` file present
- [ ] README has install + 5-minute quickstart + supported runtimes table
- [ ] `npm publish --dry-run` reviewed

## Quick Reference

| Field | Purpose |
|-------|---------|
| `type: "module"` | Source files are ESM |
| `exports` | Public API gate (modern resolvers honor this) |
| `main` / `module` / `browser` | Legacy resolver fallbacks |
| `unpkg` / `jsdelivr` | CDN entry points |
| `sideEffects: false` | Enable aggressive tree-shaking |
| `files` | Whitelist for `npm publish` |
| `browserslist` | Declare target browser matrix |
| `engines` | Declare required Node version (if any tool needs it) |
| `peerDependencies` | Consumer must provide (don't bundle) |
