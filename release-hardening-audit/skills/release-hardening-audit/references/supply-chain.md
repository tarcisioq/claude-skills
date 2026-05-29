# Supply Chain

Audit of the dependency graph, the publish surface, and the integrity of the artifact in transit from build → registry → consumer.

## OIDC trusted publishing — the modern default

Long-lived `NPM_TOKEN` in CI secrets is the legacy pattern. npmjs.com supports OIDC trusted publishing since 2024. Each publish is short-lived, scoped to the workflow, and produces a verifiable provenance attestation.

### Setup

1. On npmjs.com → package settings → **Trusted Publishing** → add:
   - Repository: `tarcisioq/my-package`
   - Workflow filename: `release.yml`
   - Environment (optional): `production`
2. In the workflow:

```yaml
name: Release
on:
  push:
    tags: ['v*']

permissions:
  contents: read
  id-token: write   # required for OIDC

jobs:
  publish:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm run build
      - run: npm publish --provenance --access public
```

3. Remove `NPM_TOKEN` from repo secrets. Run a test publish; verify on the package page → "Provenance" section shows the workflow + commit SHA.

### Audit

```bash
# Find legacy long-lived token usage
grep -rE 'secrets\\.NPM_TOKEN|NODE_AUTH_TOKEN.*secrets\\.' .github/workflows/

# Confirm the workflow has id-token: write
grep -lE 'npm publish' .github/workflows/*.yml \
  | xargs grep -l 'id-token: write'
```

If the publish workflow lacks `id-token: write`, it is HARDEN (or BLOCK if the package is high-traffic and a token compromise would cascade).

## Lockfile hygiene

| Package manager | Lockfile | CI install command |
|---|---|---|
| npm | `package-lock.json` | `npm ci` (frozen, fails on drift) |
| yarn 1 | `yarn.lock` | `yarn install --frozen-lockfile` |
| yarn 2+ (berry) | `yarn.lock` | `yarn install --immutable` |
| pnpm | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| bun | `bun.lockb` | `bun install --frozen-lockfile` |

### Audit

```bash
# Lockfile present
test -f package-lock.json || test -f yarn.lock || test -f pnpm-lock.yaml || test -f bun.lockb

# Lockfile in sync with package.json
npm ci --dry-run 2>&1 | grep -i 'lock file'
# pnpm: pnpm install --lockfile-only --frozen-lockfile

# CI uses the frozen variant (audit workflow files)
grep -rE 'npm (install|i)( |$)' .github/workflows/ | grep -v 'npm ci'
# Each match is HARDEN — `npm install` mutates the lockfile in CI
```

## `package.json` `files` field — publish surface

The most common npm publish surface bug: maintainer expects `dist/` to ship; npm actually ships everything not in `.npmignore`.

### Order of precedence

1. `package.json` `files` field (allowlist) — if present, ONLY these files ship
2. `.npmignore` (denylist) — if `files` absent
3. `.gitignore` (fallback denylist) — if both absent
4. Plus always: `package.json`, `README*`, `LICENSE*`, `CHANGELOG*`, `LICENCE*`, `NOTICE*`

### Audit

```bash
jq '.files' package.json
# null → DENYLIST mode, fragile

npm pack --dry-run 2>&1
# Look for: .env*, tests/, __tests__/, *.test.*, *.spec.*, fixtures/, scripts/internal-*, .github/, internal/
```

### Fix

```json
{
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ]
}
```

Re-run `npm pack --dry-run` after every change to `files`, `dist/` structure, or build pipeline.

## Postinstall script audit

Lifecycle scripts (`preinstall`, `install`, `postinstall`, `prepare`) run on every `npm install`. Transitive deps with malicious or sloppy scripts can:

- Read `process.env` (CI secrets present at install time)
- Modify `node_modules` of OTHER packages (typosquatting amplification)
- Phone home

### Audit

```bash
# All transitive lifecycle scripts (this is the noisy one)
find node_modules -maxdepth 3 -name package.json -exec \
  jq -r 'select(.scripts.preinstall // .scripts.install // .scripts.postinstall // .scripts.prepare) | "\\(.name): preinstall=\\(.scripts.preinstall // \"-\") install=\\(.scripts.install // \"-\") postinstall=\\(.scripts.postinstall // \"-\") prepare=\\(.scripts.prepare // \"-\")"' {} \\; \
  | sort -u

# Or: install in a sandbox first, see what runs
npm install --foreground-scripts 2>&1 | grep -E '(preinstall|^>.*install|postinstall|prepare)'
```

### Defense

For consumers (recommend in README):
```bash
npm install --ignore-scripts
```

For maintainers — pnpm restricts by default to a `onlyBuiltDependencies` allowlist:
```yaml
# .npmrc
ignore-scripts=true
```

In CI:
```bash
npm ci --ignore-scripts
# then run only the scripts you trust:
npm rebuild  # for native deps you explicitly need built
```

## CVE scanning — thresholds

Three open-source tools, picked depending on signal-to-noise:

| Tool | Strengths | Weaknesses |
|---|---|---|
| `npm audit` | bundled, no install | high false-positive rate; lists DoS as CVEs; treats all transitive deps equally |
| `osv-scanner` (Google) | uses OSV.dev DB; per-language; lockfile-aware | newer, smaller advisory DB than Snyk |
| `retire.js` | targets known-vulnerable JS lib versions in source AND bundle | client-side focus; misses many Node-only CVEs |

### Recommended runbook

```bash
# Step 1 — OSV scanner (most accurate baseline)
osv-scanner --lockfile=package-lock.json

# Step 2 — npm audit, filter to high+critical with patch path
npm audit --omit=dev --audit-level=high --json | jq '.vulnerabilities | to_entries | map(select(.value.severity == "critical" or .value.severity == "high"))'

# Step 3 — retire.js on built bundle
npx retire --path dist/
```

### Severity mapping for the report

| CVE severity | Has patch | Severity in audit |
|---|---|---|
| Critical | Yes | BLOCK |
| Critical | No (zero-day) | HARDEN with mitigation note |
| High | Yes | HARDEN |
| High | No | NICE-TO-HAVE (track, watch advisory) |
| Medium / Low | Any | NICE-TO-HAVE |

## SBOM — Software Bill of Materials

Useful for downstream consumers who run their own CVE scanning. Two main formats:

| Tool | Format | Output |
|---|---|---|
| `cyclonedx-npm` | CycloneDX (XML/JSON) | `npx @cyclonedx/cyclonedx-npm --output-file sbom.json` |
| `syft` (Anchore) | SPDX or CycloneDX | `syft packages dir:. -o cyclonedx-json > sbom.json` |

Ship the SBOM as a release artifact (GitHub Releases, container image label, npm package side-channel).

## License audit

```bash
# License file present and matching package.json
ls LICENSE* LICENCE* 2>/dev/null
jq -r '.license' package.json

# Audit transitive dep licenses (catch GPL contamination, missing licenses)
npx license-checker --production --summary
npx license-checker --production --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0' --excludePrivatePackages
```

Document the license policy in `LICENSE` and surface deviations as NICE-TO-HAVE (or HARDEN if the project is corporate / closed-source consumed).

## `engines.node` declaration

```json
{
  "engines": {
    "node": ">=18.18.0"
  }
}
```

Without `engines.node`:
- Reproducibility breaks across local / CI / prod
- Native deps may be built against a Node version the runtime cannot use
- New consumers do not get a hint about supported runtimes

```bash
# Audit
jq '.engines.node // "MISSING"' package.json
```

NICE-TO-HAVE if missing; HARDEN if the package uses Node-version-specific APIs (`AbortSignal.timeout`, `structuredClone`, `node:test`).

## Dependency confusion / typosquatting

```bash
# Cross-check direct deps against known-popular package names (high-impact targets)
jq -r '.dependencies | keys[]' package.json | while read dep; do
  # Lookup actual download count via registry
  echo "${dep}: $(curl -s "https://api.npmjs.org/downloads/point/last-week/${dep}" | jq -r '.downloads // "NOT_FOUND"')"
done
```

Watch for: deps with very low download counts that share a prefix or near-edit-distance with a popular package (e.g., `react-dom` vs `reactdom`, `axios` vs `axoios`).

For private/scoped registries: ensure `@your-org/*` always resolves from the private registry, never falls through to public:

```ini
# .npmrc
@your-org:registry=https://npm.your-corp.com/
//npm.your-corp.com/:_authToken=${YOUR_NPM_TOKEN}
```

## SaaS alternatives (note, not default)

When open-source is insufficient — at organizational scale, with multiple repos and a need for cross-repo trend analysis — consider:

- **Snyk** (Open Source / Code) — broader CVE DB, license check, SBOM, IaC scanning
- **Socket.dev** — supply chain risk score per dep (typosquat, install scripts, network access in install)
- **GitHub Advanced Security** — Code Scanning + Secret Scanning + Dependabot alerts

Always start with open-source; recommend SaaS only when the project has documented organizational scale or compliance requirements.

## Verification commands

```bash
# OIDC publish workflow correctly configured
grep -lE 'npm publish' .github/workflows/*.yml | xargs grep -l 'id-token: write'

# Lockfile in sync, frozen install in CI
npm ci --dry-run >/dev/null && echo OK

# `files` field present and pack matches intent
jq -r '.files // "missing"' package.json
npm pack --dry-run

# No high/critical CVEs without patch
npm audit --omit=dev --audit-level=high

# License file present
ls LICENSE LICENSE.md LICENSE.txt 2>/dev/null
```
