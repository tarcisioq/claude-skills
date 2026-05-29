# Tools and Checks

Operational matrix of audit tools — install, run, threshold, CI integration. Open-source first; SaaS noted as alternative.

## Tool matrix

| Tool | Purpose | Speed | False-positive rate | When to use |
|---|---|---|---|---|
| `gitleaks` | Secret scan in source + git history | Fast | Medium | Layer 3 secrets-hygiene; pre-commit hook |
| `trufflehog` | Secret scan with API verification (live secrets only) | Slow (network) | Very low (verified) | Prioritize incident response after gitleaks |
| `osv-scanner` | CVE scan via OSV.dev (lockfile-aware) | Fast | Low | Primary CVE check; replaces npm audit |
| `npm audit` | CVE scan via npm advisory DB | Fast | High (DoS noise) | Cross-check against osv-scanner |
| `retire.js` | Known-vulnerable JS lib detection in source AND bundle | Fast | Low | Audit built `dist/` for old library versions |
| `semgrep` | Static analysis with rule packs | Fast | Medium | Custom rules; complements CVE scanners |
| `eslint-plugin-security` | ESLint plugin for unsafe Node patterns | Fast (in editor) | Medium | Continuous in IDE / pre-commit |
| `publint` | `package.json` correctness for npm packages | Fast | Low | Every npm publish |
| `arethetypeswrong` | `.d.ts` consumer compatibility | Fast | Low | Every npm publish (TS projects) |
| `source-map-explorer` | Visualize bundle composition | Medium | N/A | Bundle audit; find unexpectedly-shipped modules |
| `npm pack --dry-run` | What ships to registry | Fast | N/A | Every npm publish |
| `audit-ci` | npm audit + threshold-based CI gate | Fast | High | CI gate (open-source variant) |
| `license-checker` | Transitive license audit | Medium | Low | Pre-release; license-policy enforcement |
| `cyclonedx-npm` | Generate CycloneDX SBOM | Medium | N/A | Release artifact |

## Install — one-shot, no global pollution

```bash
# Most checks via npx (no install)
npx gitleaks detect --source .
npx -y osv-scanner --lockfile=package-lock.json
npx retire --path dist/
npx publint
npx -y @arethetypeswrong/cli --pack .
npx source-map-explorer 'dist/assets/*.js'
npx audit-ci --high
npx license-checker --production --summary

# semgrep (Python; needs system install)
brew install semgrep
# or: pip install semgrep

# trufflehog (Go binary)
brew install trufflehog
# or: curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh
```

## Run — first-audit runbook

Run in this order. Each takes seconds; the full sequence is under 5 minutes for a typical repo.

### 1. Discover surface

```bash
cat package.json | jq '{name,version,main,module,exports,types,files,scripts,engines}'
npm pack --dry-run 2>&1
find . -maxdepth 3 \( -name 'vite.config.*' -o -name 'webpack.config.*' -o -name 'rollup.config.*' -o -name 'tsup.config.*' -o -name 'next.config.*' -o -name 'tsconfig*.json' -o -name 'Dockerfile' \) 2>/dev/null
ls .github/workflows/*.yml 2>/dev/null
```

### 2. Secrets — git history + working tree

```bash
gitleaks detect --source . --no-git --redact
gitleaks detect --source . --log-opts="--all" --redact
git log --all --full-history --diff-filter=A --name-only --pretty=format: -- '.env*' | sort -u
```

### 3. Dependencies — CVE + license

```bash
osv-scanner --lockfile=package-lock.json
npm audit --omit=dev --audit-level=high --json | jq '.metadata.vulnerabilities'
npx license-checker --production --onlyAllow 'MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC;CC0-1.0'
```

### 4. Built artifact — what ships

```bash
# What npm would publish
npm pack --dry-run 2>&1 | tee /tmp/npm-pack.txt
grep -E '\\.(env|test|spec|fixture)|tests/|scripts/internal' /tmp/npm-pack.txt   # should be empty

# Sourcemaps
find dist -name '*.js.map' -exec jq -r '.sources[]' {} \\; | grep -E '^(webpack://|/[A-Z]?[a-z]+)' | head

# Secrets in built artifact
grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET|sk_live_)" dist/ build/ 2>/dev/null

# Old / known-vulnerable JS libs in bundle
npx retire --path dist/

# .d.ts internals
grep -rE "(TODO|FIXME|@internal|^_[A-Z])" dist/**/*.d.ts 2>/dev/null

# Bundle composition (if size or unexpected deps suspected)
npx source-map-explorer 'dist/assets/*.js'
```

### 5. Publish surface (TS + npm)

```bash
npx publint
npx -y @arethetypeswrong/cli --pack .
```

### 6. CI/CD — workflow audit

```bash
# Permissions
for f in .github/workflows/*.yml; do
  grep -q '^permissions:' "$f" || echo "MISSING permissions: $f"
done

# pull_request_target + untrusted checkout
grep -lE 'pull_request_target' .github/workflows/*.yml \
  | xargs grep -lE 'ref:.*github\\.event\\.pull_request\\.(head\\.sha|head\\.ref)' 2>/dev/null

# Long-lived cloud keys
grep -rE 'AWS_ACCESS_KEY_ID|AZURE_CLIENT_SECRET|GOOGLE_APPLICATION_CREDENTIALS|secrets\\.NPM_TOKEN' .github/workflows/

# Third-party actions not pinned to SHA
grep -hE '^\\s+- uses: ' .github/workflows/*.yml \
  | grep -vE '(actions/|github/|google-github-actions/|aws-actions/|azure/)' \
  | grep -vE '@[0-9a-f]{40}\\b'

# Shell injection from event payloads
grep -rE 'run:.*\\$\\{\\{\\s*github\\.event\\.' .github/workflows/
```

### 7. Headers (live deployment)

```bash
curl -sI https://app.example.com/ \
  | grep -iE '(strict-transport|x-content-type|x-frame|referrer-policy|content-security|permissions-policy)'

curl -sI -H "Origin: https://attacker.example.com" https://app.example.com/api/ \
  | grep -iE '^access-control-(allow|expose)-'
```

## Threshold configuration

### `audit-ci` (CVE gate in CI)

```json
// audit-ci.json
{
  "low": false,
  "moderate": false,
  "high": true,
  "critical": true,
  "allowlist": []
}
```

```yaml
# .github/workflows/ci.yml
- run: npx audit-ci --config audit-ci.json
```

### `gitleaks` (pre-commit + CI)

```toml
# .gitleaks.toml
[allowlist]
description = "Test fixtures with placeholder credentials"
paths = [
  '''tests/fixtures/.*\\.json''',
  '''.*\\.example''',
]

[[rules]]
id = "company-internal-key"
description = "Company internal API key"
regex = '''ck_[A-Za-z0-9]{32,}'''
keywords = ["ck_"]
```

### `semgrep` rule packs

```bash
# Use community-maintained packs
semgrep --config=p/javascript --config=p/typescript --config=p/owasp-top-ten

# Or curated for Node.js security
semgrep --config=p/nodejs
semgrep --config=p/expressjs
```

## CI integration — minimum gates

```yaml
# .github/workflows/security.yml
name: Security audit

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # gitleaks full history

      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }

      - run: npm ci --ignore-scripts

      - name: Secrets — gitleaks
        uses: gitleaks/gitleaks-action@v2

      - name: CVE — osv-scanner
        run: npx -y osv-scanner --lockfile=package-lock.json

      - name: CVE — npm audit (high+critical only)
        run: npx -y audit-ci --high

      - name: Build
        run: npm run build

      - name: Built artifact secrets sweep
        run: |
          if grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET|sk_live_)" dist/; then
            echo "::error::secrets found in dist/"
            exit 1
          fi

      - name: Sourcemap path leakage
        run: |
          for m in $(find dist -name '*.js.map'); do
            if jq -r '.sources[]' "$m" | grep -qE '^(webpack://|/[A-Z]?[a-z]+)'; then
              echo "::error::absolute path in $m"
              exit 1
            fi
          done

      - name: Publish surface — publint
        run: npx -y publint

      - name: Publish surface — types correctness
        run: npx -y @arethetypeswrong/cli --pack .
```

## Pre-commit hook (Husky)

```bash
npm install -D husky lint-staged
npx husky init

# .husky/pre-commit
#!/usr/bin/env sh
npx lint-staged
npx gitleaks protect --staged --redact
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,jsx,tsx}": ["eslint --fix", "prettier --write"]
  }
}
```

## SaaS alternatives

When org-scale or compliance requires central reporting:

| Need | OSS | SaaS |
|---|---|---|
| CVE scanning | osv-scanner + npm audit | Snyk Open Source, GitHub Dependabot, Socket.dev |
| Secret scanning | gitleaks, trufflehog | GitHub Advanced Security (Secret Scanning), GitGuardian |
| SAST (code patterns) | semgrep + community rules | Snyk Code, GitHub CodeQL |
| Supply chain risk | (manual review) | Socket.dev, Phylum |
| License audit | license-checker | FOSSA, Snyk |
| SBOM generation | cyclonedx-npm, syft | Anchore Enterprise |

Always start with OSS. Recommend SaaS only when a documented organizational requirement justifies the lock-in.

## Tool selection summary

```
First audit on a new repo (under 5 min):
  1. gitleaks detect --source . --no-git --redact
  2. osv-scanner --lockfile=package-lock.json
  3. npm pack --dry-run
  4. retire --path dist/   (after build)
  5. publint && arethetypeswrong --pack .

Continuous in CI:
  1. gitleaks-action
  2. osv-scanner OR audit-ci --high
  3. publint + arethetypeswrong (npm packages)
  4. custom dist/ secret sweep
  5. workflow audit script (the run-7 above)

Pre-commit:
  1. gitleaks protect --staged
  2. eslint --fix
```
