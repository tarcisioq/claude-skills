# Build Artifacts

Audit of what actually ends up in `dist/` (or `build/`, `out/`, `.next/`, `.output/`) and gets copied to the public surface (CDN, npm registry, container image).

## Sourcemap policy decision matrix

| Project shape | Recommended sourcemap mode | Why |
|---|---|---|
| Public npm package (browser) | `'hidden'` + private upload to Sentry/Datadog | Consumers can debug their own bundles via the upstream tool, but the `.map` is not on the CDN |
| Public npm package (Node, server-only) | `inline` OK in dev, `false` or `'hidden'` in published artifact | Server-side stack traces only matter to operators; consumers do not typically debug into your dist |
| Internal SPA (private auth, SSO-gated) | `'hidden'` + upload to Sentry; CDN excludes `*.map` | Even auth-gated apps leak structure if `.map` is fetched alongside `.js` |
| Public SPA (marketing, e-commerce) | `'hidden'` + upload to Sentry; CDN excludes `*.map` | Same — sourcemap on CDN = engineering reverse trivially |
| CLI tool (npm-published, Node) | `false` (no sourcemap) | Stack traces map to user-readable code already; no consumer benefit |
| Spike / prototype | `inline` for dev, do not publish | Throwaway code; no decision needed |

**Hard rule:** if a `.js.map` is reachable at the same URL prefix as a `.js`, that is BLOCK regardless of what tool generated it.

## How sourcemaps leak even when "hidden"

The word `hidden` only means the `//# sourceMappingURL=` comment is omitted from the `.js`. The `.map` file is still emitted to disk. If the deploy step does an unfiltered `aws s3 sync dist/ s3://cdn/`, the `.map` ships anyway — just without the breadcrumb.

### Three layers to verify

```bash
# Layer 1 — local dist hygiene
$ find dist -name '*.js.map' | head
# Should: return matches (so you can upload them privately) — but NOT propagate to CDN

# Layer 2 — bundle does not point at the map
$ tail -c 80 dist/assets/*.js | grep -E 'sourceMappingURL'
# Should: return nothing (hidden mode)

# Layer 3 — CDN does not host the map
$ curl -sI https://cdn.example.com/assets/app.abc123.js.map
# Should: return 404 (or 403 with no body)
```

If layer 3 returns 200, that is BLOCK regardless of layers 1 and 2.

## Path leakage inside the `.map` itself

Even on a private bucket, the `.map` reveals project structure if `sources` paths are not normalized.

```bash
$ jq -r '.sources[]' dist/assets/app.abc123.js.map | head
# ❌ leaks
webpack:///./src/internal/auth-middleware.ts
/Users/dev/projects/internal-corp-tool/src/index.tsx
C:\\dev\\company-tools\\src\\admin.tsx

# ✅ safe
src/auth-middleware.ts
src/index.tsx
src/admin.tsx
```

### Bundler-specific config

| Bundler | Setting | Value |
|---|---|---|
| Webpack | `output.devtoolModuleFilenameTemplate` | `info => path.relative(rootDir, info.absoluteResourcePath)` |
| Rollup | `output.sourcemapPathTransform` | `(relativePath) => path.relative(rootDir, relativePath)` |
| Vite | `build.rollupOptions.output.sourcemapPathTransform` | (same as Rollup) |
| esbuild | `--source-root=` (empty) and `--source-root` flag in CLI | strips abs prefix |
| tsc (`tsconfig.json`) | `sourceRoot: ""` and `inlineSources: false` | normalizes paths |

## Build-time env-var inlining — the wildcard trap

`DefinePlugin` (Webpack), Vite `define`, esbuild `define`, and Rollup `@rollup/plugin-replace` substitute identifiers at compile time. The most dangerous shape is a wildcard substitution of `process.env`:

```ts
// vite.config.ts — ❌ BAKES the entire process.env into the bundle
export default defineConfig({
  define: { 'process.env': process.env }
});
```

```js
// webpack.config.js — ❌ same trap
new webpack.DefinePlugin({ 'process.env': JSON.stringify(process.env) });
```

**Why it is dangerous:** when the build runs in CI with all secrets in `process.env`, every secret becomes a literal string in the published JS.

### Safe patterns

```ts
// ✅ explicit allowlist of public-prefix vars
define: {
  'import.meta.env.VITE_API_URL': JSON.stringify(process.env.VITE_API_URL),
  'import.meta.env.VITE_PUBLIC_KEY': JSON.stringify(process.env.VITE_PUBLIC_KEY),
}
```

```js
// ✅ Webpack: filter to a known prefix
const publicEnv = Object.fromEntries(
  Object.entries(process.env).filter(([k]) => k.startsWith('PUBLIC_'))
);
new webpack.DefinePlugin({ 'process.env': JSON.stringify(publicEnv) });
```

### Audit commands

```bash
# Find wildcard `define` of process.env
grep -rE "define:\\s*\\{\\s*['\"]process\\.env['\"]\\s*:" .
grep -rE "DefinePlugin\\(\\{\\s*['\"]process\\.env['\"]" .

# Find any non-public-prefix env reference in client-bundled code
grep -rE "process\\.env\\.[A-Z_]+" src/ --include='*.{js,ts,jsx,tsx}' \
  | grep -vE "VITE_|NEXT_PUBLIC_|REACT_APP_|PUBLIC_|NODE_ENV"

# Confirm built bundle does not contain known server secret names
grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET|STRIPE_SECRET|AWS_SECRET)" dist/
```

## `dotenv` in client-bundled code

`dotenv` is a Node-only library that reads `.env` from disk at runtime. Loading it in code that the bundler resolves means the entire `.env` becomes a string in `dist/`.

### Audit

```bash
grep -rE "from\\s+['\"]dotenv|require\\(['\"]dotenv" src/ \
  --include='*.{js,ts,jsx,tsx,mjs,cjs}'
```

If any match is in a path the bundler entry resolves (not a `server/` or `api/` subtree explicitly excluded from the client config), it is BLOCK.

### Fix

Move `dotenv` to a server-only entrypoint. For Vite/Next/Astro use built-in `import.meta.env` / `process.env` with public-prefix discrimination.

## Console / debugger / debug logs in production bundle

```bash
# Audit — find debug constructs in built artifact
grep -rE "(console\\.(log|debug|info|warn|error|trace)|debugger;)" dist/ | head -20
```

### Bundler config to scrub at build time

| Tool | Setting |
|---|---|
| esbuild | `drop: ['console', 'debugger']` |
| Terser | `compress: { drop_console: true, drop_debugger: true }` |
| swc | `jsc.minify.compress.drop_console: true` |
| Vite | uses esbuild by default — pass `esbuild.drop` in `vite.config.ts` |

**Caveat:** `console.error` may legitimately be wanted in prod for surfacing to telemetry. Scope `drop` precisely (`drop: ['console.log', 'console.debug']` if your tool supports it; otherwise wrap in a logger).

## Bundle size and content inspection

```bash
# What is in the npm package
npm pack --dry-run

# What is in dist/ (size sorted)
find dist -type f -exec du -h {} + | sort -h | tail -20

# Visualize bundle composition (requires source-map-explorer)
npx source-map-explorer 'dist/assets/*.js'

# Inspect gzipped size (what the user actually downloads)
gzip -k -9 -c dist/assets/app.*.js | wc -c

# Check for embedded credentials / URLs / paths in bundle
grep -rE "(http[s]?://[^'\\\"\\s]+|/Users/|/home/[a-z]+/)" dist/ | head
```

## Minification audit

```bash
# Are identifiers minified? Sample first 2000 chars of largest bundle file
head -c 2000 dist/assets/$(ls dist/assets | head -1)

# ❌ readable identifiers, full function names → minification skipped or misconfigured
# ✅ single-char vars, mangled names, no whitespace → properly minified
```

If readable, check `mode: 'production'` (Webpack), `build.minify` (Vite — defaults to `'esbuild'` in prod), `minify: true` (esbuild), or `--minify` (CLI).

## Comment scrubbing in `.d.ts`

When TypeScript generates `.d.ts` files, JSDoc comments are preserved by default. Internal TODOs, FIXMEs, or hidden endpoint URLs leak.

```bash
# Audit
grep -rE "(TODO|FIXME|XXX|HACK|@internal|@private)" dist/**/*.d.ts | head
```

### Fix in `tsconfig.build.json`

```json
{
  "compilerOptions": {
    "removeComments": true,
    "stripInternal": true,
    "declaration": true
  }
}
```

Use `@internal` JSDoc tag in source to mark types/methods that should not appear in `.d.ts`. Verify with `arethetypeswrong --pack .`.

## Verification checklist

After any sourcemap or bundling change, re-verify:

```bash
# Sourcemaps are not on the CDN
curl -sI https://cdn/assets/app.*.js.map | head -1   # should 404

# Bundle has no sourceMappingURL pointing to a public path
tail -c 80 dist/assets/*.js | grep sourceMappingURL  # should return nothing

# Sourcemap (private upload only) does not contain absolute paths
jq -r '.sources[]' dist/assets/*.js.map | grep -E '^(webpack://|/[A-Z]?[a-z]+)' | head

# No server-only env names appear in built artifact
grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET)" dist/

# No console.log / debugger in built artifact
grep -rE "(console\\.log|debugger)" dist/
```

All five must return clean output before publish.
