# Secrets Hygiene

Audit of where secrets live, where they leak, and where they get committed.

## Layered audit — six places to look

| Layer | What you scan | Tool |
|---|---|---|
| 1. Working tree | `.env*` files present and not gitignored | `ls -la .env*` + `.gitignore` grep |
| 2. Git history | `.env*` ever committed (then later deleted) | `git log --all --diff-filter=A -- '.env*'` |
| 3. Full history scan | Any leaked credential pattern in any past commit | `gitleaks` or `trufflehog` |
| 4. CI/CD logs | `echo $SECRET`, debug logs that print env | log audit + workflow grep |
| 5. Built artifact | Secrets baked into `dist/` via build inlining | `grep` on `dist/` (see `build-artifacts.md`) |
| 6. Public docs | README / examples / `.env.example` with real-looking creds | `grep` on docs/ + sample files |

Skipping any layer means a real leak can persist undetected.

## Layer 1 — `.env*` working tree audit

```bash
# Files present
ls -la .env* 2>/dev/null
```

### `.gitignore` patterns required

```
# .env files (canonical pattern)
.env
.env.local
.env.*.local
.env.development.local
.env.production.local
.env.test.local

# Secrets / credentials
*.pem
*.key
*.p12
*.pfx
credentials.json
.npmrc
.aws/credentials
service-account*.json
```

### Audit

```bash
# Are .env files actually ignored?
for f in .env .env.local .env.production.local; do
  git check-ignore -v "$f" 2>/dev/null || echo "NOT IGNORED: $f"
done

# Are sensitive cred file patterns covered?
for pat in '*.pem' '*.key' 'credentials.json' '.npmrc' 'service-account*.json'; do
  git check-ignore -v "$pat" 2>/dev/null || echo "NOT IGNORED: $pat"
done
```

### `.env.example` template

The right pattern: ship a placeholder file with NO real values, listing every var the app expects.

```bash
# .env.example — committed
DATABASE_URL=postgres://user:pass@host:5432/db
API_SECRET=change-me-in-prod
JWT_SECRET=change-me-in-prod
STRIPE_SECRET_KEY=sk_test_...
```

Audit: `.env.example` exists AND every key in it has a placeholder value, never a real-looking one (no real keys, no `prod-*` hostnames).

## Layer 2 — git history for `.env*` adds

```bash
# Files ever committed at any point in history (even if later deleted)
git log --all --full-history --diff-filter=A --name-only --pretty=format: -- '.env*' | sort -u
git log --all --full-history --diff-filter=A --name-only --pretty=format: -- '*.pem' '*.key' 'credentials*.json' '.npmrc' | sort -u
```

If anything returns:

1. The secret is permanently in git history (force-push does not help — assume copied/forked).
2. **Rotate the secret immediately**. Do not "remove from history" first.
3. After rotation, optionally rewrite history with `git filter-repo` to scrub the file (cosmetic; the secret was already exposed).

## Layer 3 — full history credential scan

```bash
# gitleaks (preferred — built-in patterns + custom rules)
gitleaks detect --source . --report-format=json --report-path=gitleaks.json
gitleaks detect --source . --log-opts="--all"   # includes all branches and history

# trufflehog (alternative — verifies live secrets via API calls)
trufflehog git file://. --only-verified
```

### Verified vs unverified findings

- **gitleaks**: pattern-based; reports anything matching a regex (high recall, some false positives).
- **trufflehog --only-verified**: calls the actual provider API; reports only live, working secrets. Use for prioritization — verified is BLOCK.

### Custom rules

For org-specific secret formats (internal API key prefix, custom JWT signing key), add to `.gitleaks.toml`:

```toml
[[rules]]
id = "company-api-key"
description = "Company API Key"
regex = '''ck_[A-Za-z0-9]{32,}'''
keywords = ["ck_"]
```

## Layer 4 — CI/CD secret leakage

```bash
# Workflows that echo, print, or log env vars
grep -rE '(echo\\s+\\$[A-Z_]+|printenv|env\\s*$|set -x)' .github/workflows/

# Workflows where a step uses `with:` to pass a secret as a non-secret input (gets logged)
grep -rE 'with:.*secrets\\.' .github/workflows/

# Matrix builds that interpolate secrets into job names (logged)
grep -rE 'name:.*\\$\\{\\{\\s*secrets\\.' .github/workflows/
```

### Defense

- Always use `${{ secrets.NAME }}` in `env:` blocks, never inlined into `run:` shell strings.
- Use `::add-mask::` for derived secrets:
  ```yaml
  - run: echo "::add-mask::${{ steps.derive.outputs.token }}"
  ```
- Disable command echo: avoid `set -x` and `set -v`.
- Pin runner Node/Python/etc. so a compromised runner image cannot exfiltrate.

## Layer 5 — secrets in built artifact

Covered in `build-artifacts.md`. Key audit:

```bash
grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET|AWS_SECRET_ACCESS_KEY|STRIPE_SECRET|sk_live_|password=)" dist/ build/ | head
```

## Layer 6 — public docs scrubbing

```bash
# Real-looking credentials in README / examples
grep -rEn "(sk_live_[A-Za-z0-9]+|AKIA[A-Z0-9]{16}|ghp_[A-Za-z0-9]{36}|xox[bp]-[A-Za-z0-9-]+|eyJ[A-Za-z0-9_-]{20,}\\.eyJ)" \
  README.md docs/ examples/ 2>/dev/null

# Internal hostnames in public docs
grep -rEn "(internal\\.example\\.com|prod-[a-z0-9-]+|admin\\.example\\.com)" README.md docs/ 2>/dev/null
```

## `.npmrc` token handling

```ini
# ❌ — committed file with real auth token
//registry.npmjs.org/:_authToken=npm_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Right place for `.npmrc` auth

| Location | Use |
|---|---|
| `~/.npmrc` (user home) | Local dev — auth tokens for personal use |
| CI environment variable | `NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}` (or use OIDC, see `supply-chain.md`) |
| Project `.npmrc` (committed) | ONLY non-secret config: registry URL, scope mappings, `ignore-scripts=true` |

### Pattern for project `.npmrc` with private registry

```ini
# Public — committed
@your-org:registry=https://npm.your-corp.com/

# Private auth — read from env var, NOT committed
//npm.your-corp.com/:_authToken=${NPM_TOKEN_PRIVATE}
```

CI exports `NPM_TOKEN_PRIVATE` from secrets; substitution happens at install time. The committed file has only the variable reference.

## Pre-commit hooks for secret detection

```bash
# Install gitleaks pre-commit hook
gitleaks install --pre-commit

# Or via husky + lint-staged
npm install -D husky lint-staged
npx husky init
echo "npx gitleaks protect --staged --redact" > .husky/pre-commit
```

Block commits that introduce secrets. Adds friction for the maintainer; eliminates a class of leaks.

## Cloud-specific secret scans

```bash
# AWS keys in repo
grep -rE "AKIA[A-Z0-9]{16}" . --exclude-dir=node_modules

# Google Cloud service account JSON (look for the marker fields)
grep -rE '"type":\\s*"service_account"' . --include='*.json' --exclude-dir=node_modules

# Azure storage SAS tokens
grep -rE "sv=[0-9]{4}-[0-9]{2}-[0-9]{2}[^&]*sig=" . --exclude-dir=node_modules
```

## What to do when a leak is found

```
1. Treat the secret as compromised. Rotate immediately at the source (AWS, npm, Stripe, etc.).
2. Update CI / deploy / local dev with the new value.
3. Verify the new value works in production before disabling the old.
4. Disable the old credential at the source.
5. Document the incident (date, what leaked, rotation timestamp, verification).
6. Optionally rewrite git history (cosmetic; assume the leak is already public).
7. Add a regression test or hook to prevent the same shape from recurring.
```

Order matters. Rotation first; cleanup last.

## Verification checklist

```bash
# Layer 1 — working tree clean of unignored .env
ls -la .env* 2>/dev/null
git ls-files | grep -E '\\.env' | grep -v 'env\\.example'   # should return nothing

# Layer 2 — no .env adds in history
git log --all --full-history --diff-filter=A --name-only --pretty=format: -- '.env*' | sort -u   # should return nothing

# Layer 3 — gitleaks clean
gitleaks detect --source . --no-git --redact   # exit 0 = clean

# Layer 4 — workflows do not echo secrets
grep -rE '(echo\\s+\\$[A-Z_]+|set -x)' .github/workflows/   # should return nothing

# Layer 5 — built artifact clean (run after build)
grep -rE "(DATABASE_URL|API_SECRET|PRIVATE_KEY|JWT_SECRET)" dist/ build/ 2>/dev/null   # should return nothing

# Layer 6 — docs clean of real-looking secrets
grep -rEn "(sk_live_|AKIA[A-Z0-9]{16}|ghp_[A-Za-z0-9]{36})" README.md docs/ 2>/dev/null   # should return nothing
```

All six clean = secrets-hygiene gate passed.
