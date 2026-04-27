# Release Engineering

Shipping an SDK is not `npm publish`. It's a recurring process with versioning discipline, pre-release channels, deprecation cadence, and rollback plans. This file is the playbook.

## Semver — operational rules

| Bump | When |
|------|------|
| **MAJOR** (`X.0.0`) | Any change a consumer must read about: removed methods, changed signatures, changed default behavior, dropped runtime support, renamed exports |
| **MINOR** (`x.X.0`) | New APIs, new options with safe defaults, new optional plugin hooks, new conditional exports — all additive |
| **PATCH** (`x.x.X`) | Bug fixes that don't change documented behavior, perf improvements, internal refactors, doc fixes |

### Concrete tells (mechanical, not abstract)

```
Renaming an exported symbol → MAJOR
Removing an exported symbol → MAJOR
Adding a required option    → MAJOR
Adding an optional option   → MINOR
Changing default value      → MAJOR (changes behavior for every consumer)
Adding new error code       → MINOR (consumers' error handlers gracefully ignore unknown codes)
Removing error code         → MAJOR (consumer's switch/case has a dead branch)
Tightening input validation → MAJOR (previously-accepted inputs now reject)
Loosening input validation  → MINOR (more inputs accepted, none rejected)
Bumping minimum Node/browser→ MAJOR
Adding new runtime support  → MINOR
Internal refactor (no API change) → PATCH
Dependency update — patch  → PATCH (if their semver holds)
Dependency update — major  → Treat as MAJOR (consumer's deduplication may break)
```

**Rule of thumb:** if a consumer's existing code might emit a different log line, throw an error it didn't before, or behave differently in any observable way, it's MAJOR. Be conservative.

## Changesets — the standard release tool

For multi-package or solo-package SDKs, [`@changesets/cli`](https://github.com/changesets/changesets) is the default.

```bash
npm install --save-dev @changesets/cli
npx changeset init
```

Workflow:

```bash
# 1. After making a change, document it
npx changeset
# → interactive: choose packages, choose bump (patch/minor/major), write summary

# 2. Commit the generated `.changeset/*.md` file
git add .changeset && git commit -m "feat: add retry option (changeset)"

# 3. On main, when ready to release:
npx changeset version  # bumps versions, updates CHANGELOG
npx changeset publish  # publishes to npm
```

The generated changeset file is the source of truth for the CHANGELOG. Reviewers can see the bump reasoning at PR time.

### Manual alternative (single-package, light)

If changesets feels heavy, the manual flow:

```bash
# 1. Update CHANGELOG.md by hand
# 2. Bump version
npm version patch  # or minor / major
# 3. Push tag
git push --follow-tags
# 4. CI publishes on tag push
```

## CHANGELOG discipline

`CHANGELOG.md` at the repo root, [Keep a Changelog](https://keepachangelog.com/) format. The CHANGELOG is **for consumers**, not for you.

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [2.1.0] — 2026-04-27

### Added
- `Client.subscribe(topic, { signal })` — async iteration over server-sent events. (#142)
- New plugin hook `afterError` runs after any retry-exhausted failure. (#138)

### Changed
- `Client.fetch` now retries on `5xx` by default (previously only `503`).
  Set `retry: { on: [503] }` to restore previous behavior.

### Deprecated
- `Client.send()` — use `Client.fetch()`. Will be removed in v3.0.

### Fixed
- Memory leak in `WeakCache` when keys were Symbols. (#141)

### Security
- Validate `postMessage` origin in iframe transport. (CVE-2026-XXXX)

## [2.0.0] — 2026-03-15
...
```

**Sections (in order):** Added, Changed, Deprecated, Removed, Fixed, Security. Skip empty sections.

## Pre-releases — `beta`, `next`, `alpha`

Use npm dist-tags to ship work-in-progress without affecting `latest`.

```bash
# Publish a pre-release version
npm version 2.1.0-beta.1
npm publish --tag beta

# Consumers opt in
npm install my-sdk@beta

# Promote beta to latest when ready
npm dist-tag add my-sdk@2.1.0 latest
```

### Common dist-tags

| Tag | Use |
|-----|-----|
| `latest` | Default install — production-ready |
| `next` | Upcoming major version, may have breaking changes |
| `beta` | Feature-complete, integration testing |
| `alpha` | Experimental, expect bugs |
| `rc` | Release candidate, final testing |
| `canary` | Auto-published from main on every commit |

**Versioning pre-releases:** `2.1.0-beta.1`, `2.1.0-beta.2`, `2.1.0-rc.1`, `2.1.0`. The pre-release suffix sorts before the release version.

```bash
# Iteration cycle
npm version prerelease --preid=beta  # 2.1.0-beta.1 → 2.1.0-beta.2
npm publish --tag beta
```

## Deprecation cadence

Three-step deprecation, never sudden removal.

### Step 1 — Deprecate in MINOR

```javascript
// In v2.5.0
const warned = new Set();
export function oldMethod() {
  if (!warned.has("oldMethod")) {
    warned.add("oldMethod");
    console.warn(
      "[my-sdk] oldMethod() is deprecated since v2.5.0. Use newMethod() instead. " +
      "Will be removed in v3.0."
    );
  }
  return newMethod();
}

/**
 * @deprecated since 2.5.0. Use {@link newMethod} instead.
 * Will be removed in v3.0.
 */
```

CHANGELOG entry under `### Deprecated`. README migration section added.

### Step 2 — Hold for at least one minor cycle

`oldMethod` lives unchanged through v2.6, v2.7. Consumers see the warning, plan migration.

### Step 3 — Remove in next MAJOR

```javascript
// In v3.0.0
// oldMethod removed entirely
```

CHANGELOG entry under `### Removed` with migration note.

**Don't skip step 2.** Removing in the next minor (v2.6) is technically a breaking change in disguise.

## Migration guides

For every MAJOR, ship a migration guide as `MIGRATING.md` or section in README.

```markdown
# Migrating from v1 to v2

## Breaking changes

### `Client.send()` removed
Use `Client.fetch()` instead. The signature is identical:

```diff
- await client.send("/path", { method: "POST", body });
+ await client.fetch("/path", { method: "POST", body });
```

### `apiKey` is now required
v1 allowed `null` for testing; v2 requires it. Pass an empty string for tests
and override the transport instead.

### Default `retry` count is now 3 (was 0)
If you depended on no retries, pass `retry: { attempts: 0 }`.
```

Concrete diffs > prose. Consumers read these in a hurry; help them grep.

## Rollback strategy

Things go wrong. Plan for it.

### npm `deprecate` — soft pull

```bash
# Mark a bad version as deprecated; consumers see a warning on install
npm deprecate my-sdk@2.1.0 "v2.1.0 has a critical bug; upgrade to 2.1.1"
```

Doesn't unpublish (consumers can still install if pinned). Triggers warnings.

### npm `unpublish` — only within 72 hours

```bash
npm unpublish my-sdk@2.1.0
```

After 72 hours, npm refuses unpublish (policy: don't break the dependency graph). Use `deprecate` instead.

### Reverting via new version

```bash
# v2.1.0 is broken
# Don't try to "fix" by republishing — version numbers are immutable
# Instead: ship a fix as 2.1.1
git revert <bad-commit>
npm version patch
npm publish
```

Consumers on `^2.1.0` get the fix automatically on next install.

## CI/CD — the publish workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write     # to create tags + GitHub releases
  id-token: write     # for npm provenance
  pull-requests: write # for changeset PR

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org

      - run: npm ci
      - run: npm run build
      - run: npm test
      - run: npm run test:size  # bundle regression check

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          publish: npm run release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

```json
// package.json
{
  "scripts": {
    "release": "changeset publish"
  }
}
```

This pattern is the industry standard. Changesets-action opens a "release PR" that consolidates pending changesets; merging it triggers publish.

## Release notes — beyond the CHANGELOG

GitHub Releases per tag, with:

- **Highlights** — top 1-3 user-visible changes
- **Migration notes** if MAJOR
- **Full changelog link**
- **SRI hashes** for IIFE/UMD builds (already covered in `distribution.md`)

```markdown
# v2.1.0

## Highlights
- New `subscribe()` API for server-sent events
- Memory leak in cache fixed
- TypeScript declarations now generated from JSDoc

## Migration notes
None — fully backward compatible.

## SRI
```
sha384-xxxxxxxxxxxx  dist/index.iife.js
```

[Full changelog](./CHANGELOG.md#210)
```

## Release cadence

Pick a rhythm and stick with it. SDKs that release sporadically lose trust.

| Cadence | When |
|---------|------|
| **Continuous** | Every merge to main → patch release | High-velocity SDK, mature CI |
| **Weekly minor** | Aggregate features each week | Active SDK |
| **Monthly minor** | Feature batching | Stable SDK |
| **Quarterly major** | Plan breaking changes | Mature SDK with many consumers |

Whatever you pick, document it. Consumers plan around your cadence.

## Versioning the unversionable — TypeScript types

`.d.ts` changes are part of your public API. Tightening a type narrows what compiled — that's a breaking change for TS consumers, even if runtime behavior is identical.

```typescript
// Before
function fetch(url: string | URL): Promise<Response>;

// After (still functionally compatible at runtime, but TS-breaking)
function fetch(url: string): Promise<Response>;
// ↑ TS users passing `URL` now get a compile error
```

Treat type tightening as MAJOR. Loosening is MINOR.

## Quick Reference

| Concern | Tool / pattern |
|---------|----------------|
| Versioning | Strict semver; conservative on MAJOR |
| Multi-PR releases | Changesets (`.changeset/*.md`) |
| CHANGELOG | Keep a Changelog format; consumer-oriented |
| Pre-release channel | `npm publish --tag beta` |
| Deprecation | 3-step: warn in MINOR → hold → remove in MAJOR |
| Migration guide | `MIGRATING.md` per MAJOR; concrete diffs |
| Rollback | `npm deprecate` (after 72h); ship fix as new patch |
| CI publish | `changesets/action` + npm provenance |
| Releases page | GitHub Releases per tag with highlights + SRI |
| Cadence | Documented and consistent |
| TS type changes | Tightening = MAJOR; loosening = MINOR |
