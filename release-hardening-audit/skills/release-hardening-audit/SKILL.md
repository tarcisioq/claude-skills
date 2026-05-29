---
name: release-hardening-audit
description: Pre-publish security hardening audit for JavaScript / TypeScript / Node.js projects. Use when preparing a release, PR-reviewing a release branch, auditing a project for production readiness, or investigating why a published artifact leaked something it shouldn't have. Catches non-obvious release-blocking issues — sourcemap leakage on CDN, build-time env-var inlining of server secrets, npm publish surface drift (`files` field gaps, `.d.ts` internal leaks), postinstall risks in transitive deps, GitHub Actions privilege escalation, OIDC vs long-lived `NPM_TOKEN`. Workflow is interactive: presents findings as a decision table with effort and auto-fix capability, awaits explicit user confirmation, applies in-repo code fixes, summarizes affected files, and surfaces external actions (CDN/cloud config, secret rotation, npm OIDC setup) separately. Not for runtime vulnerability scanning of code under change — pair with `/security-review` for that. Not for SDK API design (`vanilla-js-architect`) or engineering discipline review (`solid`).
license: MIT
metadata:
  author: https://github.com/tarcisioq
  version: "0.3.0"
  domain: security
  triggers: release audit, pre-publish audit, security audit, sourcemap leak, npm publish, supply chain audit, CSP audit, CORS audit, SRI audit, build determinism, GitHub Actions hardening, dist audit, postinstall audit, env var inlining, secrets in bundle, helmet audit, JWT verify, trust proxy, OIDC publish, files field, .d.ts leak, pull_request_target, monorepo audit, lambda hardening, cloudflare workers hardening, serverless audit
  role: auditor
  scope: audit-confirm-apply-report
  output-format: decision-table-then-applied-changes
---

# Release Hardening Audit

## Skip This Skill If — Hard Gate

**Stop reading and refuse to engage if any of these match.** This skill is for JS/TS/Node.js projects shipping an artifact (npm package, browser bundle on CDN, container image, or deployable Node service). On non-JS projects or on code that does not ship to a release surface, the gallery and references do not apply.

| # | Programmatic tell | Why it disqualifies |
|---|-------------------|---------------------|
| 1 | **No `package.json` *anywhere* in the tree** (root or any subdirectory) | Not a JS/TS/Node project — refuse. |
| 2 | Repo is `pyproject.toml` / `go.mod` / `Cargo.toml` / `pom.xml` / `Gemfile` (no `package.json`) | Wrong ecosystem — refuse. |
| 3 | The task is **runtime vulnerability scanning of changed code in a PR** (SQL injection / XSS / RCE in modified files) | Use `/security-review` — purpose-built for code-level vuln scanning with high-confidence FP filtering. This skill audits the **shipped artifact and surface**, not new code. |
| 4 | Project is a **spike / prototype / REPL exercise** with no `dist/`, no build script, no `Dockerfile`, no `.github/workflows/*.yml`, and no published consumers | Nothing to harden — no release surface exists. |
| 5 | The same audit ran on the current commit and nothing changed in `package.json`, build config, deps, or `.github/workflows/` | Cache the prior result. Tell the user; ask before re-running. |

**If any tell matches:** stop. Reply briefly explaining which gate fired and recommend the right alternative. Do not proceed with a partial audit.

### Monorepo / umbrella handling

If the **root has no `package.json`** but **subprojects do** (pnpm/npm/yarn workspaces, lerna, Nx, Turborepo, or simple folder co-location like `client/` + `server/`), do **not** refuse — the project has release surfaces, just not collapsed under a single root manifest.

Discipline:

1. **Enumerate subproject release surfaces:** `find . -maxdepth 4 -name 'package.json' -not -path '*/node_modules/*' -not -path '*/dist/*' -not -path '*/build/*'`. Each result is a candidate surface.
2. **Run Phase 1 (Discover) per subproject** — each may have its own `dist/`, build config, deploy pipeline, framework. Surface models do not merge.
3. **Walk the same gallery** against each subproject's surface model. Findings are tagged with the subproject path: `client:B1`, `identify:H4`, etc.
4. **Merge in Phase 3** — single decision table covering all subprojects, but each row carries its subproject prefix so the user can triage per surface.
5. **Apply (Phase 5) per subproject** — file edits stay scoped to the owning subproject. Verification commands run from the subproject root.

The umbrella itself (root `README.md`, contract docs, shared CI workflows in `.github/workflows/`) is still audited for #7, #8, #9 as a single workflows surface — workflows live at the repo root regardless of the per-subproject layout.

## How to Use This Skill

This skill is structured for **lookup**, not linear reading. The audit is **interactive** — never apply a fix without explicit user confirmation. The flow:

1. Read this `SKILL.md` end-to-end at the start of an audit
2. Run **Phase 1 (Discover)** to map the actual release surface of *this* project
3. Run **Phase 2 (Audit)** — walk Anti-Pattern Gallery entries selectively against the discovered surface
4. Load matching **reference files** on demand when a finding needs tooling commands or deeper context
5. Run **Phase 3 (Present)** — emit the decision table + detailed cards (the user reads this and decides)
6. Run **Phase 4 (Confirm)** — pause and ask the user which findings to apply; wait for explicit response
7. Run **Phase 5 (Apply)** — execute confirmed in-repo code fixes; defer external actions
8. Run **Phase 6 (Report)** — summarize affected files, verification results, and remaining user actions

If you start reporting findings without having run Phase 1, stop. The gallery only matters against the actual surface — auditing without discovery produces generic noise. **If you start applying fixes without having run Phase 4, stop.** The interactive gate is non-negotiable: even BLOCK findings wait for user consent.

## When to Use This Skill

- Preparing a public release of an npm package, browser SDK, or Node service
- PR review on a release branch / version bump / "ready to ship" PR
- Periodic project health check before a milestone deploy
- Onboarding a project to ensure the release pipeline is hardened
- Investigating a supply chain or release-surface concern reported in production

## When NOT to Use

(Hard gate above is authoritative; this is the prose context.)

- Code-level vuln scan in changed PR code → use `/security-review`
- React / Vue / Svelte component design → use `frontend-design` (this skill audits the build output of those apps, not component design)
- SDK architecture / public API design → use `vanilla-js-architect`
- Engineering discipline / refactor / TDD review → use `solid`

## Severity Model

Findings are graded with **release-blocking semantics**, not CVSS. The user reads severity to decide what to do tonight vs this sprint.

| Severity | Meaning | Examples |
|---|---|---|
| **BLOCK** | Do not publish until fixed. Concrete risk: secret leakage, exploit surface, irreversible exposure. | Sourcemap with `webpack://` paths uploaded publicly; `process.env.DATABASE_URL` inlined in client bundle; `.env.production` shipped in `dist/`; `pull_request_target` + checkout of untrusted ref; CVE Critical with patch path available. |
| **HARDEN** | Ship this release, fix in next cycle. Defense-in-depth gap. | Workflow without explicit `permissions:`; long-lived `NPM_TOKEN` instead of OIDC; Express without `helmet()`; `crypto.createHash` for password hashing where `argon2` should be. |
| **NICE-TO-HAVE** | Quality improvement. Track but do not gate. | SBOM missing; license file unset; `engines.node` undeclared; deps with low/medium CVEs and no patch path. |

**Do not promote a NICE-TO-HAVE to BLOCK** because the project has many of them. Severity reflects ship-time risk, not aggregate hygiene.

## Top 5 Rules — Read This First

These fire on almost every JS/TS/Node release audit. If short on time, anchor here before walking the full gallery.

| # | Rule | Tell to detect |
|---|------|----------------|
| 1 | **Sourcemaps must not ship to public CDN as `.js.map` siblings** | `npm pack --dry-run` lists `*.js.map`; OR `dist/**/*.js` ends with `//# sourceMappingURL=` AND deploy step copies `dist/` unfiltered to CDN (Anti-pattern #1, #2). |
| 2 | **Build-time `define`/`DefinePlugin`/`dotenv` must not bake server-only env vars into client bundle** | Vite/Webpack config uses `define: { 'process.env': process.env }` (wildcard); OR `process.env.<NON_PUBLIC_PREFIX>` in client-bundled code; OR `dotenv` imported in client entrypoint (Anti-pattern #3, #4). |
| 3 | **`package.json` `files` field must be present and audited via `npm pack --dry-run`** | `files` field absent; OR `npm pack --dry-run` lists test fixtures, `.env.example`, internal docs, repo root files (Anti-pattern #5). |
| 4 | **GitHub Actions workflows must declare `permissions:` explicitly and avoid `pull_request_target` + untrusted checkout** | `.github/workflows/*.yml` without a `permissions:` block; OR `pull_request_target` + `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` (Anti-pattern #7, #8). |
| 5 | **`npm publish` must use OIDC trusted publishing (provenance), not a long-lived `NPM_TOKEN`** | Workflow uses `secrets.NPM_TOKEN` without `permissions: { id-token: write }` and without `--provenance` (Anti-pattern #9). |

## Anti-Patterns Gallery

Each entry is a finding template. The `Tell:` line is a programmatic check — when auditing, run it verbatim.

### 1. Public sourcemaps shipping with the bundle (BLOCK)

```bash
# ❌ — `.js.map` siblings of every bundle file in dist/, copied to CDN as-is
$ ls dist/
  app.abc123.js  app.abc123.js.map  vendor.def456.js  vendor.def456.js.map
$ tail -c 80 dist/app.abc123.js
  //# sourceMappingURL=app.abc123.js.map
```

**Tell:** `find dist -name '*.js.map'` returns results AND the deploy step does `aws s3 sync dist/ ...` (or equivalent unfiltered copy) AND `tail -c 80 dist/<entry>.js` ends with `//# sourceMappingURL=`.
**Fix:** Set `build.sourcemap: 'hidden'` (Vite) / `devtool: 'hidden-source-map'` (Webpack) / `sourcemap: 'hidden'` (Rollup, esbuild). Upload `.map` files to a private bucket / Sentry / Datadog separately. Either delete `*.map` before the public sync or exclude them. Verify: `tail -c 80 dist/app.*.js` returns no `sourceMappingURL` directive in the public artifact AND `find <cdn-mirror> -name '*.js.map'` returns nothing.

### 2. Sourcemap with `webpack://` paths or filesystem-absolute paths (BLOCK)

```bash
# ❌ — even when uploaded privately, `.map` reveals project structure
$ jq -r '.sources[]' dist/app.abc123.js.map | head -3
  webpack:///./src/internal/auth-middleware.ts
  webpack:///./src/secrets/api-keys.ts
  /Users/dev/projects/internal-corp-tool/src/index.tsx
```

**Tell:** `jq -r '.sources[]' <map>` shows `webpack://`, `vite://`, or absolute paths starting with `/Users/`, `/home/`, `C:\\`, or any internal filesystem prefix.
**Fix:** Configure relative path mapping — `output.devtoolModuleFilenameTemplate` (Webpack), `sourcemapPathTransform` (Rollup/Vite), `--source-root` (esbuild). Strip absolute paths and `webpack://` prefix. Re-verify with `jq -r '.sources[]'`.

### 3. Server-only env vars inlined in client bundle (BLOCK)

```ts
// ❌ — Vite config — wildcard `define` bakes ALL process.env into bundle
// vite.config.ts
export default defineConfig({
  define: { 'process.env': process.env }  // ← every server env var now in dist/
});
```

```bash
# ❌ — DATABASE_URL ends up grep-able in the public bundle
$ grep -o 'postgres://[^"]*' dist/assets/*.js
  postgres://prod-user:hunter2@db.internal:5432/app
```

**Tell:** vite.config / webpack.config / rollup.config has `define` with `process.env` as a whole, OR with any specific `process.env.<NAME>` whose name is NOT prefixed `VITE_` / `NEXT_PUBLIC_` / `REACT_APP_` / `PUBLIC_`. Verify in built output: `grep -E '(DATABASE_URL|API_SECRET|PRIVATE_KEY|TOKEN|PASSWORD|SECRET)' dist/**/*.js`.
**Fix:** Replace wildcard with explicit allowlist of public-prefix vars only. Audit `import.meta.env` / `process.env` references in source for any non-public-prefix names that read in client paths and move them to server-only modules. Re-grep dist after rebuild.

### 4. `dotenv` loaded in client-bundled code (BLOCK)

```ts
// ❌ — `dotenv` imported in a module that gets bundled for the browser
import 'dotenv/config';                  // ← bakes the entire .env into the bundle
import { Client } from './sdk-client.js';
```

**Tell:** `grep -rE "from\\s+['\"]dotenv|require\\(['\"]dotenv" src/ --include='*.{js,ts,jsx,tsx,mjs,cjs}'` returns matches in any file under a path that the bundler entrypoint resolves (i.e., not isolated to a `server/` or `api/` subtree explicitly excluded from the client bundle).
**Fix:** Move `dotenv` import to server-only entrypoint. For Vite/Next/Astro use the framework's built-in env handling with public-prefix discrimination (`VITE_*`, `NEXT_PUBLIC_*`). Verify the built bundle does not contain server var names: `grep -E '<server-var-pattern>' dist/`.

### 5. `package.json` ships test fixtures, `.env.example`, and repo root (HARDEN)

```bash
# ❌ — no `files` field; npm pack ships everything not in .npmignore
$ jq '.files' package.json
  null
$ npm pack --dry-run 2>&1 | grep -E '\\.(env|test|spec|fixture)|tests/|scripts/|internal/'
  npm notice 1.2kB .env.example
  npm notice 4.5kB tests/fixtures/sample-payload.json
  npm notice 3.1kB scripts/internal-deploy.sh
```

**Tell:** `jq '.files' package.json` returns `null` OR `npm pack --dry-run` output includes any of: `.env*`, `tests/`, `__tests__/`, `*.test.*`, `*.spec.*`, `fixtures/`, `internal/`, `scripts/internal-*`, `docs/internal/`, `.github/`.
**Fix:** Declare `files: ["dist", "README.md", "LICENSE"]` (allowlist, not ignorelist). Re-run `npm pack --dry-run`; confirm the listing is exactly the public surface intended. For TS projects also run `arethetypeswrong --pack .` to check `.d.ts` correctness.

### 6. Postinstall scripts in transitive deps execute on consumer install (HARDEN)

```bash
# ❌ — install runs arbitrary code from deep in the dep tree
$ npm install
> some-popular-package@4.2.1 postinstall
> node ./scripts/post-install-helper.js
```

**Tell:** `npm install --dry-run --foreground-scripts 2>&1 | grep -E '(preinstall|install|postinstall)'` shows scripts from packages NOT in direct dependencies (the script source path is in `node_modules/<some-transitive>/`). OR `find node_modules -maxdepth 3 -name package.json -exec jq -r 'select(.scripts.postinstall // .scripts.install // .scripts.preinstall) | "\(.name): \(.scripts)"' {} \\;` returns transitive deps with lifecycle scripts.
**Fix:** For consumers: document `npm install --ignore-scripts` and provide an explicit allowlist of trusted postinstall scripts. For maintainers: `npm config set ignore-scripts true` in CI, or use pnpm's `onlyBuiltDependencies` allowlist. Audit each transitive postinstall script source before allowing.

### 7. GitHub Actions workflow without explicit `permissions:` (HARDEN)

```yaml
# ❌ — no `permissions:` block; legacy repos default to write-all
# .github/workflows/release.yml
on: [push]
jobs:
  release:
    runs-on: ubuntu-latest
    steps: [...]
```

**Tell:** `for f in .github/workflows/*.yml; do grep -L '^permissions:' "$f"; done` lists workflows without a top-level `permissions:` AND each job in those workflows lacks `permissions:` too.
**Fix:** Add `permissions: { contents: read }` at workflow level, then escalate per-job (`permissions: { contents: write, id-token: write }`) only where actually needed. Also update the repo-level default at `Settings → Actions → General → Workflow permissions → Read repository contents and packages permissions` (this hardens repos created before 2023-02 where the default is still write-all).

### 8. `pull_request_target` + checkout of untrusted ref (BLOCK)

```yaml
# ❌ — PR code runs with secrets and write access
on:
  pull_request_target:
    types: [opened, synchronize]
jobs:
  ci:
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # ← attacker-controlled
      - run: npm install && npm test                      # ← runs attacker's scripts
```

**Tell:** `grep -l 'pull_request_target' .github/workflows/*.yml | xargs grep -lE 'ref:.*github\\.event\\.pull_request\\.(head\\.sha|head\\.ref)'` returns any file. OR any combination of `pull_request_target` trigger + subsequent `npm install`/`npm test`/`npm run build`/`yarn`/`pnpm` step.
**Fix:** Use `pull_request` trigger (no secrets, isolated context) for CI on PRs from forks. Reserve `pull_request_target` for label-bot / triage workflows that do not run untrusted code. If you must run on PR contents with elevated permissions, gate behind a label-with-approval pattern (e.g. `if: contains(github.event.pull_request.labels.*.name, 'safe-to-test')`) AND check out the trusted base, not the head.

### 9. `npm publish` via long-lived `NPM_TOKEN` instead of OIDC trusted publishing (HARDEN)

```yaml
# ❌ — token is reusable, exfiltratable, manually rotated
- run: npm publish
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Tell:** `.github/workflows/*.yml` references `secrets.NPM_TOKEN` (or any `secrets.*NPM*`) AND does NOT declare `permissions: { id-token: write }` AND does NOT pass `--provenance` to `npm publish`.
**Fix:**
```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: actions/setup-node@v4
    with: { registry-url: 'https://registry.npmjs.org' }
  - run: npm publish --provenance --access public
```
Configure trusted publishing on npmjs.com → package settings → Trusted Publishing → add the GitHub repo + workflow + environment.

### 10. TypeScript `.d.ts` ships internal JSDoc and types (HARDEN)

```bash
# ❌ — JSDoc with TODOs and internal type names visible to consumers
$ head -10 dist/index.d.ts
  /** TODO: deprecate before 2.0; uses internal endpoint /v0/internal/keys */
  export interface InternalAuthState {
    secretRotationKey: string;            // ← shape of internal state leaks
  }
```

**Tell:** `tsconfig.build.json` (or whatever tsconfig drives the build) has `removeComments: false` (or unset; default `false`) AND `declaration: true`. OR `arethetypeswrong --pack .` reports issues. OR `grep -E 'TODO|FIXME|XXX|HACK|@internal' dist/**/*.d.ts` returns matches. OR `dist/**/*.d.ts` exports types named `^_`, `^Internal`, `^Private`.
**Fix:** Set `removeComments: true` in build tsconfig. Use `@internal` JSDoc tag + `tsc --stripInternal` (or API Extractor) to gate what ships in `.d.ts`. Re-verify with `arethetypeswrong --pack .` and re-grep dist.

### 11. Express app without `helmet()` or with permissive defaults (HARDEN)

```js
// ❌ — no security headers; X-Powered-By: Express leaks framework
const app = express();
app.use(express.json());
app.use(routes);
```

```js
// ❌ — helmet present but CSP disabled wholesale
app.use(helmet({ contentSecurityPolicy: false }));
```

**Tell:** the file that creates the Express/Fastify/Koa app does not call `helmet()` (or framework equivalent). OR `grep -E "helmet\\(\\{[^)]*contentSecurityPolicy:\\s*false" src/`.
**Fix:** `app.use(helmet())` (defaults are good for most APIs). For SPAs needing inline scripts, use `helmet.contentSecurityPolicy({ directives: { ... } })` with nonces or hashes — never `false`. Verify with `curl -I https://app/api/ | grep -E '(Strict-Transport|X-Content-Type|Referrer-Policy|Content-Security)'`.

**Surface adaptation (non-Express runtimes):**
- **AWS Lambda + API Gateway HTTP API v2** — set headers on the handler return value. Tell: handler returns `{ statusCode, body }` without a `headers` map, or with a map missing `X-Content-Type-Options`, `Strict-Transport-Security`, `Referrer-Policy`, `Cache-Control`. Fix: `return { statusCode, headers: { 'X-Content-Type-Options': 'nosniff', 'Strict-Transport-Security': 'max-age=63072000; includeSubDomains; preload', 'Referrer-Policy': 'strict-origin-when-cross-origin', 'Cache-Control': 'no-store' }, body }`.
- **Cloudflare Workers (Hono)** — `app.use('*', secureHeaders())` from `hono/secure-headers`. Tell: Hono `app` declared without a `secureHeaders` import.
- **Next.js (App Router middleware or API routes)** — set the four headers in `middleware.ts` via `response.headers.set(...)` (covers all routes) OR per-route via `Response.headers`. Tell: no `middleware.ts` setting security headers AND no per-route header setting.
- **Standalone Node `http.createServer`** — `res.setHeader('X-Content-Type-Options', 'nosniff')` (× 4) before `res.writeHead`. Tell: handler writes status without the four headers.

### 12. CORS `origin: true` + `credentials: true` (BLOCK)

```js
// ❌ — echoes the requesting origin AND sends cookies → cross-site CSRF
app.use(cors({ origin: true, credentials: true }));
```

**Tell:** `grep -E "cors\\(\\{[^)]*credentials:\\s*true" src/` AND in the same options block: `origin: true` OR `origin: '\\*'` OR `origin: (req, cb) => cb(null, true)`.
**Fix:** Replace with explicit allowlist: `origin: ['https://app.example.com', 'https://admin.example.com']`. For wildcard subdomain patterns, validate via function: `origin: (origin, cb) => allowlist.includes(origin) ? cb(null, true) : cb(new Error('not allowed'))`.

**Surface adaptation (non-Express runtimes):**
- **AWS Lambda + API Gateway** — CORS is configured on the Gateway (SAM `Cors:` block, OpenAPI `x-amazon-apigateway-cors`, or AWS console). Tell: gateway config has `AllowOrigins: ['*']` AND `AllowCredentials: true` together. Fix: explicit `AllowOrigins: ['https://app.example.com', ...]` allowlist; reject the preflight at the gateway when origin is not allowed.
- **Cloudflare Workers (Hono)** — `app.use('*', cors())`. Tell: `cors({ origin: '*', credentials: true })` or origin-reflection function with `credentials: true`. Fix: explicit `origin: ['https://app.example.com', ...]` array OR validation function returning a single allowed origin per request.
- **Next.js (middleware or API routes)** — manual `Access-Control-*` response headers. Tell: code sets `Access-Control-Allow-Origin: *` AND `Access-Control-Allow-Credentials: true` on the same response. Fix: explicit allowlist; reject preflight when origin not in allowlist.
- **Standalone Node** — manual `Access-Control-*` response headers; same Tell / Fix as Next.

### 13. Cookies missing security flags (HARDEN)

```js
// ❌ — session cookie can be read by JS, sent over HTTP, used cross-site
res.cookie('session', token);  // no Secure, no HttpOnly, no SameSite
```

**Tell:** `grep -nE "(res\\.cookie|setCookie|session\\(\\{|cookie-session|express-session|iron-session)" src/**/*.{js,ts}` and inspect each call for missing `secure: true`, `httpOnly: true`, `sameSite: 'strict' | 'lax'`.
**Fix:** Set `{ secure: true, httpOnly: true, sameSite: 'lax' }` on every cookie holding session/auth state. For first-party-only cookies, prefer the `__Host-` prefix (forces `secure`, `path=/`, no `domain`). Validate with `curl -I` against the deployed app.

**Surface adaptation (cookie API differs by runtime; flags don't):**
The flags themselves (`Secure`, `HttpOnly`, `SameSite=Lax|Strict`, `__Host-` prefix) are HTTP semantics — identical across runtimes. Only the API for setting them changes:
- **AWS Lambda + API Gateway HTTP API v2** — `return { ..., cookies: ['name=val; Secure; HttpOnly; SameSite=Lax'] }` (HTTP API v2) or `multiValueHeaders["Set-Cookie"]: [...]` (REST API).
- **Cloudflare Workers** — `response.headers.append('Set-Cookie', 'name=val; Secure; HttpOnly; SameSite=Lax')`.
- **Next.js** — `cookies().set({ name, value, secure: true, httpOnly: true, sameSite: 'lax' })` (App Router) or `res.setHeader('Set-Cookie', ...)` (Pages API).
- **Standalone Node** — `res.setHeader('Set-Cookie', ['name=val; Secure; HttpOnly; SameSite=Lax'])`.

### 14. Express body parsing without `limit` (HARDEN)

```js
// ❌ — default for express.json is 100kb but easy to forget; older shims accept unlimited
app.use(express.json());
app.use(express.urlencoded());
```

**Tell:** `grep -E "express\\.(json|urlencoded|raw|text)\\(" src/` and check each call for an explicit `{ limit: '...' }` option.
**Fix:** Set explicit limits: `app.use(express.json({ limit: '100kb' }))`. Increase only on routes that legitimately need it. Reject oversized payloads at the boundary, not in handlers.

**Surface adaptation (non-Express runtimes):**
- **AWS Lambda + API Gateway HTTP API v2** — Tell: handler does `JSON.parse(event.body)` without `if ((event.body?.length ?? 0) > N) return { statusCode: 413 }` first. (API Gateway has a 6 MB hard limit on payloads; the app-layer guard should be tighter — typical 256 KB to 1 MB depending on use case.) Fix: validate `event.body.length` before parsing; return `413 Payload Too Large` on overage.
- **Cloudflare Workers** — Tell: handler does `await request.json()` without checking `request.headers.get('content-length')` first. Fix: `if (Number(request.headers.get('content-length') ?? 0) > N) return new Response(null, { status: 413 })`.
- **Next.js API routes** — Pages Router: `export const config = { api: { bodyParser: { sizeLimit: '100kb' } } }` per route. App Router: check content-length manually in the route handler. Tell: route uses `request.json()` / `req.body` without `sizeLimit` config and without manual length check.
- **Standalone Node `http.createServer`** — Tell: handler accumulates `req` chunks in a `data` event without tracking total bytes. Fix: increment `bytesRead` on each chunk; if `bytesRead > N`, `req.destroy()` and respond with `413` — never let an unbounded buffer grow.

### 15. Password hashing with `crypto.createHash('sha256')` (BLOCK)

```js
// ❌ — fast hash, no salt, brute-forceable on commodity hardware
const hash = crypto.createHash('sha256').update(password).digest('hex');
```

**Tell:** `grep -nE "createHash\\(['\"]?(md5|sha1|sha256|sha512)" src/**/*.{js,ts}` in any file path matching `auth/`, `password/`, `login/`, `signup/`, `user/`, `account/`, `credential/`.
**Fix:** Replace with `argon2` (preferred) or `bcrypt` or Node's built-in `crypto.scrypt`. Use `argon2id` with the `argon2` library defaults. Migrate existing hashes by double-wrapping on next successful login (`new = argon2(old_sha256_hash)`), marking the row as upgraded.

### 16. JWT verification without explicit `algorithms` (BLOCK)

```js
// ❌ — accepts whatever the token claims, including `alg: none`
jwt.verify(token, secret);
```

**Tell:** `grep -nE "(jwt|jose)\\.verify\\(" src/**/*.{js,ts}` and inspect each call for an explicit `algorithms: [...]` argument.
**Fix:** `jwt.verify(token, secret, { algorithms: ['HS256'] })` — list ONLY the algorithm actually used. For asymmetric: `algorithms: ['RS256']` or `['ES256']`. Never accept `none`. Add a regression test that submits `alg: none` and confirms rejection.

### 17. `Math.random()` used for tokens / session IDs / nonces (BLOCK)

```js
// ❌ — predictable; not cryptographically secure
const sessionId = Math.random().toString(36).slice(2);
const csrfToken = Math.random().toString(16);
```

**Tell:** `grep -nE "Math\\.random\\(\\)" src/**/*.{js,ts}` in any file path matching `auth/`, `token/`, `session/`, `csrf/`, `nonce/`, `id/`, `uuid/`, `random/`.
**Fix:** Use `crypto.randomUUID()` for IDs, `crypto.randomBytes(32).toString('hex')` for tokens, or Web Crypto `crypto.getRandomValues(new Uint8Array(32))` for browser-side. Reserve `Math.random()` for non-security uses (animations, A/B routing).

### 18. Express trusts `X-Forwarded-For` without `trust proxy` configured (HARDEN)

```js
// ❌ — req.ip uses socket address (not real client) OR trusts spoofable header
const app = express();
app.use(rateLimit({ windowMs: 60_000, max: 100 }));  // keyed by req.ip
```

**Tell:** `grep -nE "(req\\.ip|req\\.ips|X-Forwarded-For)" src/**/*.{js,ts}` returns matches AND `grep -nE "(trust proxy|app\\.set\\(['\"]trust proxy)" src/**/*.{js,ts}` returns nothing OR returns blanket `app.set('trust proxy', true)` (= spoofable).
**Fix:** Configure `app.set('trust proxy', N)` with N = exact number of trusted proxies between client and app (1 for single Cloudflare/ALB layer, 2 for Cloudflare → ALB, etc.). Or use specific subnet trust: `app.set('trust proxy', 'loopback, 192.168.1.0/24')`. Validate: behind real proxy, `req.ip` should be the actual client IP.

**Surface adaptation (the trustable client-IP source differs by runtime):**
- **AWS Lambda + API Gateway HTTP API v2** — `event.requestContext.http.sourceIp` is set by API Gateway (controlled, trustable). Tell: rate limiter or audit log keys off `event.headers['x-forwarded-for']` (parseable / spoofable from the leftmost token) instead of `requestContext.http.sourceIp`. Fix: use `event.requestContext.http.sourceIp`.
- **Cloudflare Workers** — `request.headers.get('cf-connecting-ip')` is set by Cloudflare's edge and not modifiable from outside Cloudflare's network. Tell: handler uses `request.headers.get('x-forwarded-for')` instead. Fix: prefer `cf-connecting-ip`.
- **Next.js (Vercel)** — `request.headers.get('x-forwarded-for')` is set by Vercel; the **first** comma-separated token is the trustable client IP (Vercel rewrites it). Tell: code reads raw socket IP or uses the wrong token index. Fix: `request.headers.get('x-forwarded-for')?.split(',')[0].trim()`.
- **Standalone Node behind a proxy** — same model as Express: trust exactly N hops. Validate the `x-forwarded-for` chain length against expected proxy count, or use a typed proxy library.

### 19. Banner / build artifact embeds non-deterministic timestamp (NICE-TO-HAVE)

```js
// ❌ — every build emits a different banner because new Date() runs at config-eval time
// rollup.config.js
import pkg from './package.json' assert { type: 'json' };
const banner = `/* ${pkg.name} ${pkg.version} — Build ${new Date().toISOString().slice(0,10)} */`;
export default { plugins: [terser({ format: { preamble: banner } })] };
```

**Tell:** `grep -rE "new Date\(\)|Date\.now\(\)|Build [0-9]{4}-" rollup.config.* webpack.config.* vite.config.* tsup.config.* esbuild.config.* 2>/dev/null` returns matches in any banner / version constant / license-file template path. OR rebuilding the same source twice produces different `dist/` bytes for the same logical output: `npm run clean && npm run build && md5sum dist/*.js > /tmp/a; npm run clean && npm run build && md5sum dist/*.js > /tmp/b; diff /tmp/a /tmp/b` shows differences.
**Fix:** Parameterize the banner via `pkg.version` only (drop the date entirely). For audit traceability, accept a `BUILD_DATE` env var emitted by CI: `const banner = \`/* ${pkg.name} ${pkg.version}${process.env.BUILD_DATE ? ' — Build ' + process.env.BUILD_DATE : ''} */\`` — local rebuilds stay deterministic; CI rebuilds at the same commit produce the same date → same bytes. Verify reproducibility: the `md5sum`-diff command above returns empty.

### 20. CDN-distributed bundle published without SRI guidance (NICE-TO-HAVE)

```html
<!-- ❌ — consumer page loads the bundle without integrity check -->
<!-- CDN compromise OR a swapped object on the bucket = silent code injection on every consumer -->
<script src="https://cdn.example.com/sdk/v1.2.3/sdk.min.js"></script>
```

**Tell:** project ships `dist/*.min.js` AND has a CDN deploy step (`aws s3 sync dist/ s3://...`, `gh-pages`, `wrangler r2 object put`, `vercel deploy`, Cloudflare/CloudFront origin) AND none of: README/docs include a `<script integrity="sha384-..." crossorigin="anonymous">` snippet; build emits SHA-384 hashes (`openssl dgst -sha384 -binary <bundle> | openssl base64 -A`); SRI hashes are published alongside the bundle.
**Fix:** Emit SHA-384 per bundle as a postbuild step:
```bash
# postbuild
for f in dist/*.min.js; do
  hash=$(openssl dgst -sha384 -binary "$f" | openssl base64 -A)
  printf '{"file":"%s","integrity":"sha384-%s"}\n' "$(basename "$f")" "$hash" >> dist/sri.jsonl
done
```
Document the consumer snippet in distribution README:
```html
<script src="https://cdn.example.com/sdk/v1.2.3/sdk.min.js"
        integrity="sha384-<hash-from-sri.jsonl>"
        crossorigin="anonymous"></script>
```
Set `Cross-Origin-Resource-Policy: cross-origin` on the CDN object (otherwise browsers may CORS-block the script before SRI runs and the consumer never gets the integrity guarantee). Verify: `curl -sI https://cdn.example.com/sdk/v1.2.3/sdk.min.js | grep -i 'cross-origin-resource-policy'` returns `cross-origin`; loading the consumer page with a deliberately wrong `integrity` string produces a console error and refuses to execute the bundle.

### Note on surface adaptation in this gallery

Cards **#11, #12, #13, #14, and #18** carry an explicit **Surface adaptation** block with Lambda / Cloudflare Workers / Next.js / standalone Node analogs — they are runtime-coupled. Cards **#15, #16, #17** (password hashing, JWT verify, `Math.random` for tokens) are **runtime-agnostic** — primitives are JavaScript / Node language-level, not framework-specific. Cards **#1–#10** cover bundle / CI / publish hygiene; cards **#19–#20** cover browser-bundle distribution. Pick the cards that match your discovered surface (see Phase 2 table below).

## Audit Workflow

The workflow has **6 phases**. Phases 1–3 produce a report; Phases 4–6 are the interactive remediation. Never collapse them — applying fixes before user confirmation violates the contract.

### Phase 1 — Discover (always run first)

The release surface determines which gallery entries apply.

| Step | Command | What you learn |
|------|---------|----------------|
| 1 | `cat package.json \| jq '{name,version,main,module,exports,types,files,scripts,engines}'` | Public surface declarations + script entrypoints + Node version constraint |
| 2 | `npm pack --dry-run 2>&1` | Exactly what would ship to npm registry — compare against `files` field intent |
| 3 | `find . -maxdepth 3 \\( -name 'vite.config.*' -o -name 'webpack.config.*' -o -name 'rollup.config.*' -o -name 'tsup.config.*' -o -name 'esbuild.config.*' -o -name 'next.config.*' -o -name 'tsconfig*.json' -o -name 'Dockerfile' -o -name 'template.yml' -o -name 'template.yaml' -o -name 'serverless.yml' -o -name 'wrangler.toml' -o -name 'wrangler.json' \\) 2>/dev/null` | Build pipeline shape — which configs to inspect for `define`, `sourcemap`, `outDir`. Also surfaces serverless / edge framework: SAM (`template.yml`/`yaml` → `framework: 'lambda'`), Cloudflare Workers (`wrangler.toml`/`json` → `framework: 'workers'`), Serverless Framework (`serverless.yml` → typically `framework: 'lambda'`) |
| 4 | `ls .github/workflows/*.yml 2>/dev/null` and `cat` each | CI/CD surface for permissions, triggers, secrets usage |
| 5 | `cat .gitignore .npmignore 2>/dev/null \| grep -E '(env\|secret\|credential\|token)'` | Secret-handling intent declared by maintainer |
| 6 | `ls -la .env* 2>/dev/null` and `git log --all --full-history --diff-filter=A -- '.env*' 2>/dev/null` | `.env` files in working tree + ever-committed secret files |
| 7 | If `dist/` or `build/` exists: `find dist build -type f 2>/dev/null \| head -50` and `find dist build \\( -name '*.map' -o -name '*.env*' -o -name '*.test.*' \\) 2>/dev/null` | Built artifact contents — sourcemaps and accidentally shipped files |
| 8 | `grep -rE 'aws s3 sync\|gh-pages\|wrangler.+(deploy\|publish)\|vercel deploy\|cloudfront\|netlify deploy' .github/workflows/ scripts/ Makefile package.json 2>/dev/null` | CDN distribution? Sets `distributed_via_cdn: true` in the surface model. Drives card #20 (SRI) applicability |

Record the **surface model**: `{ kind: 'npm-library' | 'browser-spa' | 'node-service' | 'cli' | 'monorepo', publishes_to_npm: bool, has_dist: bool, distributed_via_cdn: bool, has_workflows: bool, has_dotenv: bool, framework: 'express' | 'fastify' | 'koa' | 'next' | 'lambda' | 'workers' | 'node-http' | 'none' }`. Phase 2 walks gallery entries selectively against this model. **For monorepos / umbrellas:** record one surface model per subproject containing a `package.json` (see "Monorepo / umbrella handling" near the top of this skill).

### Phase 2 — Audit (walk gallery against discovered surface)

For each gallery entry, run its `Tell:` check against the actual project state. Use tools, not pattern-match-from-memory.

| Surface | Gallery entries to walk |
|---------|-------------------------|
| `publishes_to_npm: true` | #1 sourcemap, #2 sourcemap paths, #5 files field, #6 postinstall, #9 OIDC publish, #10 .d.ts |
| `has_dist && bundler config present` (browser bundle) | #1, #2, #3 env inlining, #4 dotenv, #10 .d.ts (if TS), #19 build-determinism |
| `has_dist && distributed_via_cdn: true` (browser bundle on CDN) | + #20 SRI |
| `framework: express \| fastify \| koa` (traditional Node service) | #11 helmet, #12 CORS, #13 cookies, #14 body limit, #15-17 crypto, #18 trust proxy — apply via the **literal** Express idioms in each card |
| `framework: lambda \| workers \| next \| node-http` (Node service on serverless / edge / standalone) | Same cards #11, #12, #13, #14, #18 — apply via the **Surface adaptation** block in each card. Cards #15-#17 (crypto primitives) are runtime-agnostic and apply unchanged |
| `has_workflows: true` | #7 permissions, #8 pull_request_target, #9 OIDC publish |
| `has_dotenv: true` | Phase 1 step 6 + `references/secrets-hygiene.md` |

For each match, capture into a **finding record**: id (B*/H*/N*), severity, gallery entry, file path + line, evidence (command output snippet), risk, fix snippet, verification command, effort estimate, auto-fix capability, external setup required (if any).

### Phase 3 — Present (decision table + detailed cards)

Output two artifacts in the chat. The user reads both and decides.

**Effort scale (mechanical):**
- **XS** — single-line config edit OR adding a missing field. < 5 min.
- **S** — small config + workflow change OR moving a few imports. < 30 min.
- **M** — refactor across 3+ files OR coordinated change to build pipeline + deploy script. < 2 h.
- **L** — pipeline migration, secret rotation cascade, framework swap. > 2 h.

**Auto-fix capability:**
- **✓ Yes** — Claude can apply via `Edit` / `Write` / `Bash` (single repo state change).
- **⚠ Partial** — Claude applies the in-repo portion; user does an external action (redeploy / rotate / configure dashboard). The card lists what remains.
- **✗ No** — only an external action; Claude cannot apply (e.g., npm OIDC dashboard config, secret rotation at the provider).

#### Artifact A — Glanceable decision table

```markdown
## Findings — <project name> @ <commit SHA>

**Surface:** <surface model>  ·  **Audit date:** <YYYY-MM-DD>
**Counts:** 🔴 N BLOCK · 🟡 N HARDEN · 🟢 N NICE-TO-HAVE

| # | Sev | Finding | Location | Effort | Auto-fix | External setup | Recommendation |
|---|-----|---------|----------|--------|----------|----------------|----------------|
| B1 | 🔴 | Public sourcemaps shipping with bundle | `dist/*.map` (12 files); `.github/workflows/deploy.yml:23` | S | ⚠ Partial | None — but redeploy required to verify | Apply now |
| B2 | 🔴 | `process.env.DATABASE_URL` inlined in client bundle | `vite.config.ts:8`; `dist/assets/app.*.js` | S | ✓ Yes | None | Apply now |
| H1 | 🟡 | No `permissions:` in release workflow | `.github/workflows/release.yml` | XS | ✓ Yes | None | Apply now |
| H2 | 🟡 | `NPM_TOKEN` instead of OIDC trusted publishing | `.github/workflows/release.yml:18` | M | ⚠ Partial | npmjs.com → Trusted Publishing config | Apply config now; complete OIDC setup before next publish |
| N1 | 🟢 | `engines.node` undeclared | `package.json` | XS | ✓ Yes | None | Apply (optional) |
```

The table is the **at-a-glance scan**. The cards (Artifact B) carry evidence and full fix.

#### Artifact B — Detailed cards (one per finding)

```markdown
### B1 — Public sourcemaps shipping with the bundle

* **Severity:** 🔴 BLOCK
* **Location:** `dist/*.map` (12 files); `.github/workflows/deploy.yml:23`
* **Evidence:**
  ```
  $ find dist -name '*.js.map' | wc -l
  12
  $ tail -c 80 dist/assets/app.*.js
  //# sourceMappingURL=app.abc123.js.map
  $ grep -A1 'aws s3 sync' .github/workflows/deploy.yml
  - run: aws s3 sync dist/ s3://cdn-prod/        # ← no exclusion of *.map
  ```
* **Risk:** Engenharia reversa trivial — DevTools reconstrói código original com nomes, comentários e estrutura de arquivos. Atacantes / concorrentes mapeiam toda a lógica de negócio shipada no bundle público.
* **Fix:**
  ```ts
  // vite.config.ts
  export default defineConfig({
    build: { sourcemap: 'hidden' }   // ← was: true
  });
  ```
  ```yaml
  # .github/workflows/deploy.yml
  - run: aws s3 sync dist/ s3://cdn-prod/ --exclude '*.map'
  ```
* **Verification:**
  ```
  tail -c 80 dist/assets/*.js | grep sourceMappingURL  # should return nothing
  curl -sI https://cdn-prod/assets/app.*.js.map        # should 404
  ```
* **Decision factors:**
  - **Effort:** S (~15 min — config + workflow update)
  - **Auto-fix:** ⚠ Partial — Claude edits `vite.config.ts` and `deploy.yml`; user must redeploy and run the second verification command against the live CDN
  - **External setup:** None (all changes are in-repo)
  - **Reversibility:** trivial (revert the two diffs)
  - **Blast radius:** sourcemaps stop reaching CDN; private upload to Sentry/Datadog continues to work for internal debugging if configured (separate concern, see `references/build-artifacts.md`)
  - **Recommendation:** Apply now. Sourcemap leakage on a public bundle is BLOCK regardless of project size or audience.
```

Repeat per finding. Order: BLOCK first (B1, B2, ...), then HARDEN (H1, H2, ...), then NICE-TO-HAVE (N1, ...).

### Phase 4 — Confirm (interactive gate — non-negotiable)

After presenting Artifacts A and B, **pause** and prompt the user with this exact menu:

```
Quais findings aplicar?
  • all              → todos auto-fixáveis (✓ e parte ✓ dos ⚠)
  • block            → apenas BLOCK
  • block+harden     → BLOCK e HARDEN
  • <ids>            → lista específica, ex: "B1, H1, N3"
  • none / skip      → não aplicar nada; manter só o relatório
```

Wait for an explicit user response. Do not auto-apply on assumed consent — even if every finding is BLOCK, even if the user said "audit and fix" in the original prompt, the Phase 3 report is the new contract and Phase 4 is its gate.

If the response is unclear, ask once more for clarification. Do not guess.

### Phase 5 — Apply (only confirmed findings)

For each confirmed ID, in BLOCK → HARDEN → NICE order:

| Auto-fix | Action |
|---|---|
| `✓ Yes` | Apply the fix via `Edit` / `Write` / `Bash`. Run the **verification command** immediately after. Capture pass/fail. |
| `⚠ Partial` | Apply the in-repo portion (file edits, lockfile changes, workflow updates). Run any in-repo verification. Add a "remaining user action" entry describing the external step required. |
| `✗ No` | Skip; cannot be applied by Claude. Add to "remaining user action" list with the exact steps the user must take. |

Per-finding tracking record: `{ id, status: 'applied' | 'partial' | 'skipped', files: [...], verification: 'pass' | 'fail' | 'not-verified', remaining_action: '...' | null }`.

If a verification fails after an apply, **do not retry blindly**. Mark the fix as `applied but verification failed`, surface in Phase 6, and let the user investigate. A failing verification means the model of the project is wrong, not that the fix should be force-pushed.

### Phase 6 — Report changes applied

Final output. Replaces the Phase 3 report only if the user actually applied something; otherwise append to it.

```markdown
## Changes applied

| # | Status | Finding | Files modified | Verification |
|---|--------|---------|----------------|--------------|
| B1 | ⚠ Partial | Public sourcemaps shipping with bundle | `vite.config.ts`, `.github/workflows/deploy.yml` | ✓ Pass (in-repo); ⏳ pending CDN redeploy |
| B2 | ✓ Applied | `DATABASE_URL` inlined in client bundle | `vite.config.ts` | ✓ Pass (`grep DATABASE_URL dist/` returns empty) |
| H1 | ✓ Applied | No `permissions:` in release workflow | `.github/workflows/release.yml` | ✓ Pass |
| H2 | ⚠ Partial | `NPM_TOKEN` → OIDC | `.github/workflows/release.yml` | ✓ Pass (in-repo); ⏳ pending npmjs.com Trusted Publishing config |
| N1 | ✓ Applied | `engines.node` undeclared | `package.json` | ✓ Pass |

## Files modified (4)

- `vite.config.ts` — B1 (`sourcemap: 'hidden'`), B2 (removed wildcard `define: { 'process.env': ... }`)
- `.github/workflows/deploy.yml` — B1 (added `--exclude '*.map'` to S3 sync)
- `.github/workflows/release.yml` — H1 (added top-level `permissions:`), H2 (added `id-token: write` + `--provenance` flag)
- `package.json` — N1 (added `engines.node: '>=18.18.0'`)

## Remaining user actions (2)

1. **B1 — Redeploy and verify CDN drops sourcemaps**
   - Trigger your normal release pipeline OR run `aws s3 sync dist/ s3://cdn-prod/ --exclude '*.map' --delete` manually
   - Verify: `curl -sI https://cdn-prod/assets/app.*.js.map` should return 404
2. **H2 — Configure npm Trusted Publishing**
   - On npmjs.com → package settings → Trusted Publishing → add: repo `<owner>/<repo>`, workflow `release.yml`, environment (optional)
   - Remove the legacy `NPM_TOKEN` from repo secrets after the next successful OIDC publish
   - Verify: next publish succeeds AND the package page on npmjs.com shows a "Provenance" attestation

## Skipped findings (0)

(none in this run)

## Verification commands — full re-run

After completing remaining actions, re-run these to confirm closure:

```bash
# B1
tail -c 80 dist/assets/*.js | grep sourceMappingURL   # nothing
curl -sI https://cdn-prod/assets/app.*.js.map         # 404

# B2
grep -E "(DATABASE_URL|API_SECRET|PRIVATE_KEY)" dist/  # nothing

# H1
grep -q '^permissions:' .github/workflows/release.yml  # exit 0

# H2
grep -lE 'npm publish' .github/workflows/*.yml | xargs grep -l 'id-token: write'

# N1
jq -r '.engines.node // "MISSING"' package.json        # not "MISSING"
```
```

If the user said `none` / `skip` in Phase 4: do **not** produce Phase 6. The Phase 3 report stands as the deliverable.

## Reference Guide

Load on demand when a gallery entry needs deeper context or tooling specifics.

| Reference | Load when... |
|-----------|--------------|
| `references/build-artifacts.md` | Auditing sourcemaps, env inlining, console scrubbing, bundle hygiene, paths leakage, minification config |
| `references/distribution-config.md` | Auditing CSP, SRI, security headers, CORS, cookies, helmet config, body limits, trust proxy |
| `references/supply-chain.md` | Auditing npm provenance, lockfile, postinstall, CVE thresholds, SBOM, license, `engines.node`, `files` field |
| `references/secrets-hygiene.md` | Auditing `.env` files, `.gitignore` patterns, git history scan, `.npmrc`, CI secret leakage, public docs |
| `references/cicd-hardening.md` | Auditing GitHub Actions permissions, `pull_request_target`, OIDC, action pinning, workflow injection |
| `references/tools-and-checks.md` | Need exact install + run commands for `gitleaks`, `osv-scanner`, `npm audit`, `retire.js`, `semgrep`, `publint`, `arethetypeswrong`, `source-map-explorer` |

## Constraints

### MUST DO

- Run **Phase 1 (Discover)** before reporting any finding. The surface model determines applicable gallery entries.
- For every finding, capture: file path, line (if applicable), severity (BLOCK / HARDEN / NICE-TO-HAVE), `Tell:` evidence (command output snippet), Fix snippet, verification command, **effort estimate** (XS/S/M/L), **auto-fix capability** (✓ / ⚠ / ✗), and **external setup required** (or "None").
- **Present Phase 3 (decision table + cards) BEFORE Phase 5.** The user reads the report and decides. Even BLOCK findings wait for confirmation.
- **Pause at Phase 4 (Confirm).** Wait for an explicit user response specifying which findings to apply (`all` / `block` / `block+harden` / `<id list>` / `none`). Do not infer consent from the original prompt.
- In Phase 5, run the **verification command immediately after each applied fix.** Capture pass / fail / not-verified per finding.
- In Phase 6, produce the **Files modified table with per-finding attribution** — each modified file lists which finding(s) caused which line of change.
- Treat `BLOCK` literally. If preparing a release, BLOCK = do not publish until resolved. Surface this prominently in the Phase 3 report and in the Phase 4 prompt.
- Inspect the **actual built artifact** (`dist/`, `build/`, `npm pack --dry-run` output) for surface-related findings. Source-only audit misses inlining, sourcemaps, and ship-list issues.
- For npm packages: run `npm pack --dry-run` and `arethetypeswrong --pack .` (when `.d.ts` is shipped) as part of every audit.
- Pin every recommended GitHub Action to a commit SHA (per `references/cicd-hardening.md`).

### MUST NOT DO

- Do NOT apply any fix before user confirmation in Phase 4. Even BLOCK findings wait. Even if the user originally said "audit and fix", the Phase 3 report becomes the new contract.
- Do NOT auto-execute external actions in Phase 5: npm OIDC dashboard config, secret rotation at the provider, branch protection settings, CDN policy changes, npmjs.com Trusted Publishing setup. Apply only the in-repo portion; surface the external step as a "remaining user action" in Phase 6.
- Do NOT mark a finding as `✓ Applied` if the verification failed. Mark it `applied but verification failed` and surface in Phase 6 — let the user investigate.
- Do NOT retry a failing verification by re-applying the same fix. A failing verification means the project model is wrong, not that the fix should be force-pushed.
- Do NOT collapse Phases 3 and 5 ("here's the report, fixes already applied"). The interactive gate is the contract.
- Do NOT run a code-level vuln scan as part of this audit (no SQL injection / XSS analysis of changed files). That is `/security-review`'s scope.
- Do NOT recommend SaaS scanners (Snyk, Socket.dev) as defaults — open-source first (gitleaks, osv-scanner, npm audit, retire.js). Note SaaS as alternatives only.
- Do NOT promote NICE-TO-HAVE findings to BLOCK because the project has many.
- Do NOT propose fixes that change the build pipeline beyond the audit's scope. If the fix would require a CommonJS→ESM migration, flag it but stop short of executing.
- Do NOT skip the verification command in any finding. The fix must be confirmable post-audit.
- Do NOT load reference files speculatively — only when a finding from the gallery needs specific tooling commands the SKILL.md does not carry.

## Output Templates

The output formats are defined inline in the workflow:

- **Phase 3 (Present)** — Artifact A (decision table) + Artifact B (detailed cards). See examples in Phase 3 above.
- **Phase 6 (Report changes applied)** — Changes-applied table + Files-modified list + Remaining-user-actions list + Verification re-run block. See example in Phase 6 above.

Do not duplicate the templates here — single source of truth lives in the Workflow section.

## Review Checklist

Before declaring an audit complete, confirm:

**Discovery and audit (Phases 1–2):**
- [ ] Phase 1 (Discover) ran and the surface model is recorded in the output
- [ ] Every gallery entry applicable to the surface model was checked (Phase 2 surface→entries table)
- [ ] Severity is graded against ship-time risk (BLOCK / HARDEN / NICE-TO-HAVE), not CVSS or "feeling"
- [ ] Report does NOT include code-level vuln findings (those go to `/security-review`)
- [ ] If `npm pack --dry-run` was applicable, its output is attached or referenced
- [ ] If GitHub Actions workflows are present, every workflow file was inspected for #7, #8, #9
- [ ] If a `.env` file appears in working tree or git history, secrets-hygiene findings are surfaced
- [ ] References were loaded only on demand, not speculatively

**Presentation (Phase 3):**
- [ ] Decision table (Artifact A) presented with: id, severity icon, finding, location, effort, auto-fix, external setup, recommendation
- [ ] Detailed cards (Artifact B) presented with: severity, location, evidence, risk, fix, verification, decision factors
- [ ] Every finding carries effort estimate (XS/S/M/L) and auto-fix capability (✓/⚠/✗)

**Confirmation (Phase 4):**
- [ ] Explicit user response received specifying which findings to apply (`all` / `block` / `block+harden` / `<id list>` / `none`)
- [ ] No fix applied before the user responded
- [ ] Ambiguous response prompted one clarification round, not silent guessing

**Application (Phase 5):**
- [ ] Each applied fix has its verification command run; result captured (pass / fail / not-verified)
- [ ] `⚠ Partial` findings: in-repo portion applied; external action recorded
- [ ] `✗ No` findings: skipped with the manual steps recorded
- [ ] No verification failure was force-pushed past — failures are surfaced for user investigation

**Reporting (Phase 6):**
- [ ] Files-modified table includes per-finding attribution (which finding caused which line in each file)
- [ ] Remaining-user-actions list is complete and actionable (not just "configure OIDC" — include the click path)
- [ ] Verification re-run block contains every command needed to confirm closure end-to-end
- [ ] If user said `none` in Phase 4: Phase 6 not produced; Phase 3 report stands as the deliverable
