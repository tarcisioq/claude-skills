# CI/CD Hardening

Audit of GitHub Actions workflows, the most common CI/CD surface for JS/TS/Node projects. Patterns generalize to GitLab CI, CircleCI, etc., but commands target GitHub.

## Permissions matrix

GitHub Actions has a per-token permission model. Get this right and most CI risks fall away.

### Default permission disaster

Repos created before 2023-02-02 default to **`GITHUB_TOKEN` write-all** unless explicitly tightened. Newer repos default to read-all. Always check the actual setting.

```bash
# Check the repo-level default (requires gh CLI)
gh api repos/<owner>/<repo>/actions/permissions/workflow | jq '.default_workflow_permissions'
# Should: "read"
```

If `"write"`, fix at: `Settings → Actions → General → Workflow permissions → "Read repository contents and packages permissions"`.

### Workflow-level — explicit permissions block

```yaml
# ❌ — no permissions block, inherits default (write-all on legacy repos)
on: [push]

# ✅ — explicit, least-privilege
permissions:
  contents: read

on: [push]

jobs:
  release:
    permissions:
      contents: write          # for git tag creation
      id-token: write          # for OIDC (npm publish, AWS, etc.)
      packages: write          # for ghcr.io
    runs-on: ubuntu-latest
```

### Required vs optional permissions table

| Action / step needs | Permission |
|---|---|
| `actions/checkout` (read-only) | `contents: read` |
| Push commits / tags | `contents: write` |
| `npm publish --provenance` (OIDC) | `id-token: write` + `contents: read` |
| `gh release create` | `contents: write` |
| Docker push to ghcr.io | `packages: write` |
| Comment on PRs | `pull-requests: write` |
| Add labels / reviews | `pull-requests: write` |
| Update issue status | `issues: write` |

### Audit

```bash
# Workflows missing top-level permissions
for f in .github/workflows/*.yml; do
  if ! grep -q '^permissions:' "$f"; then
    # Also check that no job declares permissions individually
    if ! grep -E '^\\s{4,}permissions:' "$f" >/dev/null; then
      echo "NO PERMISSIONS BLOCK: $f"
    fi
  fi
done
```

## `pull_request_target` — the privilege escalation trap

`pull_request_target` triggers on PRs but runs in the **base repository's context** with the base repo's secrets. Combined with checkout of the PR head, an attacker submits a PR that runs malicious code with full secret access.

### The classic attack

```yaml
# ❌ — vulnerable
on:
  pull_request_target:
    types: [opened, synchronize]
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}   # ← attacker code
      - run: npm ci                                        # ← runs attacker scripts
      - run: npm test                                      # ← runs attacker test code
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}             # ← exfiltrates here
```

### Safe patterns

| Need | Pattern |
|---|---|
| CI on PRs from forks (no secrets needed) | Use `pull_request` trigger; runs in PR context, no secrets, isolated |
| Bot label / triage / size check | `pull_request_target` is fine, but DO NOT check out `head.sha`; only operate on event metadata |
| CI on PRs with secrets (e.g. e2e against staging) | Require approval label gate: `if: contains(github.event.pull_request.labels.*.name, 'safe-to-test')` AND have a maintainer add the label per-PR |

### Audit

```bash
grep -lE 'pull_request_target' .github/workflows/*.yml \
  | xargs grep -lE 'ref:.*github\\.event\\.pull_request\\.(head\\.sha|head\\.ref)' 2>/dev/null
# Each result is BLOCK
```

## OIDC for cloud auth — eliminate long-lived keys

Long-lived AWS / GCP / Azure keys in GitHub secrets are an exfiltration target. OIDC replaces them with short-lived tokens scoped to the workflow.

### AWS

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      aws-region: us-east-1
  - run: aws s3 sync ./dist s3://my-bucket
```

Configure on AWS side: IAM Identity Provider for `token.actions.githubusercontent.com`, role with trust policy allowing your `repo:owner/repo:ref:refs/heads/main` (or environment).

### GCP

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service_account: deployer@project.iam.gserviceaccount.com
```

### Azure

```yaml
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}        # not a secret — public OIDC client ID
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### Audit

```bash
# Long-lived AWS keys still in use
grep -rE 'AWS_ACCESS_KEY_ID|AWS_SECRET_ACCESS_KEY' .github/workflows/

# GCP service account JSON in secrets
grep -rE 'GOOGLE_APPLICATION_CREDENTIALS|service_account_key' .github/workflows/

# Azure client secret (vs federated)
grep -rE 'AZURE_CLIENT_SECRET' .github/workflows/
```

Each match is HARDEN — migrate to OIDC.

## Action pinning

Tags and branches are mutable. A pinned action by SHA cannot be hijacked by an attacker who compromises the action's repo.

```yaml
# ❌ — tag, mutable
- uses: actions/checkout@v4

# ⚠️ — major tag, somewhat safer (Dependabot updates) but still mutable
- uses: actions/checkout@v4

# ✅ — pinned to SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
```

For first-party actions (`actions/*`, `github/*`) tag pinning is generally OK — the org has access controls, audit logs, and provenance. For third-party actions, pin to SHA.

### Audit

```bash
# Third-party actions not pinned to SHA
grep -hE '^\\s+- uses: ' .github/workflows/*.yml \
  | grep -vE '(actions/|github/|google-github-actions/|aws-actions/|azure/)' \
  | grep -vE '@[0-9a-f]{40}\\b'
```

Add Dependabot to keep SHAs current:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Workflow injection — untrusted input as shell

User-controlled fields (PR title, branch name, issue body, commit message) interpolated into `run:` shell scripts execute as code.

```yaml
# ❌ — PR title injected as shell
- run: echo "Title: ${{ github.event.pull_request.title }}"
# PR titled `"; rm -rf $HOME #` runs the rm command
```

### Safe pattern

```yaml
# ✅ — pass via env var; shell sees a string, not a code substitution
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "Title: $PR_TITLE"
```

### Audit

```bash
# Inline ${{ ... }} in shell scripts — high-risk surfaces
grep -rE 'run:.*\\$\\{\\{\\s*github\\.event\\.(pull_request|issue|comment)\\.' .github/workflows/

# Same for branch / ref names with arbitrary characters
grep -rE 'run:.*\\$\\{\\{\\s*github\\.(head_ref|ref_name)\\b' .github/workflows/
```

Each match is HARDEN.

## Self-hosted runner risks

Self-hosted runners on public repos are dangerous: any PR can run code on the runner, with whatever the runner machine has access to (build cache, ssh keys, network reach).

### Mitigations

- Self-hosted runners ONLY for private repos. For public repos, use GitHub-hosted (or ephemeral self-hosted via Actions Runner Controller).
- If self-hosted on public is unavoidable, scope at organization level with PR approval required (`actions: { runners: { allowedRepos: [...] } }`).
- Use ephemeral runners (one-shot, destroyed after job).

### Audit

```bash
# Workflows targeting self-hosted runners
grep -rE 'runs-on:.*(self-hosted|labels:|group:)' .github/workflows/
```

For each: verify the repo is private OR ephemeral runner config OR PR-approval gating exists.

## Branch protection essentials

Audited via `gh`:

```bash
gh api repos/<owner>/<repo>/branches/main/protection 2>/dev/null | jq '{
  required_reviewers: .required_pull_request_reviews.required_approving_review_count,
  dismiss_stale: .required_pull_request_reviews.dismiss_stale_reviews,
  require_signed_commits: .required_signatures.enabled,
  require_status_checks: .required_status_checks.strict,
  required_checks: .required_status_checks.contexts,
  allow_force_push: .allow_force_pushes.enabled,
  allow_deletions: .allow_deletions.enabled
}'
```

### Recommended values

| Setting | Value |
|---|---|
| Required reviewers | ≥1 (≥2 for high-impact repos) |
| Dismiss stale reviews on push | true |
| Required status checks | strict + list each CI job |
| Require signed commits | true (HARDEN) |
| Allow force push | false |
| Allow deletions | false |
| Restrict who can push | repo admins + release bot only |

## Dependabot / Renovate config

Keep dep CVE patches arriving without manual chasing.

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      production-deps:
        dependency-type: "production"
      dev-deps:
        dependency-type: "development"
        update-types: ["patch", "minor"]
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## Verification commands

```bash
# All workflows have explicit permissions
for f in .github/workflows/*.yml; do
  grep -q '^permissions:' "$f" || grep -qE '^\\s{4,}permissions:' "$f" || echo "MISSING: $f"
done

# No pull_request_target + untrusted checkout
grep -lE 'pull_request_target' .github/workflows/*.yml | xargs grep -lE 'ref:.*github\\.event\\.pull_request\\.(head\\.sha|head\\.ref)' 2>/dev/null

# No long-lived cloud keys
grep -rE 'AWS_ACCESS_KEY_ID|AZURE_CLIENT_SECRET|GOOGLE_APPLICATION_CREDENTIALS' .github/workflows/

# Third-party actions pinned to SHA
grep -hE '^\\s+- uses: ' .github/workflows/*.yml \
  | grep -vE '(actions/|github/|google-github-actions/|aws-actions/|azure/)' \
  | grep -vE '@[0-9a-f]{40}\\b'

# No shell injection from event payloads
grep -rE 'run:.*\\$\\{\\{\\s*github\\.event\\.' .github/workflows/

# Branch protection on main
gh api repos/<owner>/<repo>/branches/main/protection >/dev/null && echo OK
```

All six clean = CI/CD-hardening gate passed.
