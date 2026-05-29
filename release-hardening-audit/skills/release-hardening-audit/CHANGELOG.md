# Changelog

All notable changes to the `release-hardening-audit` skill.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] — 2026-04-29

First feedback-driven release. Driven by `feedbacks/whorl-2026-04-29.md` (browser SDK + AWS Lambda umbrella audit, two passes). Two HIGH-severity findings (gallery dismissal bias on Express-coupled cards; Hard Gate refusing legitimate monorepo umbrellas) plus two MEDIUM-severity gallery gaps (build-determinism, CDN SRI) addressed.

### Changed

- **Hard Gate Tell #1 softened for monorepo / umbrella shapes.** Previously: "No `package.json` at the repo root → refuse." Now: "No `package.json` *anywhere* in the tree → refuse." A new **Monorepo / umbrella handling** subsection documents the per-subproject audit discipline: enumerate subproject `package.json`s, run Phase 1 per subproject, walk the gallery against each surface model, tag findings with subproject prefix (`client:B1`, `identify:H4`), merge in Phase 3, apply per-subproject in Phase 5. Workflows at the repo root are still audited as a single CI surface.

- **Anti-Pattern Gallery cards #11, #12, #13, #14, and #18 gain a Surface adaptation block** — explicit AWS Lambda + API Gateway HTTP API v2 / Cloudflare Workers (Hono) / Next.js (App Router middleware + API routes) / standalone Node `http.createServer` analogs for each Express idiom (helmet → response-headers map; CORS → gateway/middleware allowlist; cookies → `Set-Cookie` flag matrix; body limit → length check pre-`JSON.parse`; trust proxy → trustable client-IP source per runtime). Each adaptation has a runtime-specific Tell and Fix. Cards #15, #16, #17 (password hashing, JWT verify, `Math.random` for tokens) are flagged as runtime-agnostic — primitives are JavaScript / Node language-level, no adaptation needed.

- **Surface model extended.** `framework` enum gains `'lambda'` / `'workers'` / `'node-http'` (in addition to existing `'express'` / `'fastify'` / `'koa'` / `'next'` / `'none'`). New boolean `distributed_via_cdn` drives card #20 applicability.

- **Phase 1 (Discover) extended with two new probes:** Step 3 also looks for `template.yml/yaml` (SAM → Lambda), `serverless.yml` (Serverless Framework → typically Lambda), `wrangler.toml/json` (Cloudflare Workers). New step 8 greps `.github/workflows/` + `scripts/` + Makefile + `package.json` for CDN deploy commands (`aws s3 sync`, `gh-pages`, `wrangler ... deploy/publish`, `vercel deploy`, `cloudfront`, `netlify deploy`) → sets `distributed_via_cdn`.

- **Phase 2 surface→entries table updated:** new row for `framework: 'lambda' | 'workers' | 'next' | 'node-http'` reuses cards #11-#14, #18 with the Surface adaptation blocks. New conditional row "+ #20 SRI" appears when `distributed_via_cdn: true`.

### Added

- **Anti-pattern #19 — Banner / build artifact embeds non-deterministic timestamp** (NICE-TO-HAVE). Tell: greps for `new Date()` / `Date.now()` / `Build YYYY-` in bundler config files, OR `md5sum` differs across two clean rebuilds of the same source. Fix: parameterize via `pkg.version` only; accept a `BUILD_DATE` env var emitted by CI for traceability without per-machine drift. Verifies via reproducible-build assertion.

- **Anti-pattern #20 — CDN-distributed bundle published without SRI guidance** (NICE-TO-HAVE). Tell: `dist/*.min.js` ships AND a CDN deploy step exists AND no `<script integrity="sha384-..." crossorigin="anonymous">` snippet in docs / SHA-384 hash emission at build / SRI hashes published with the bundle. Fix: emit SHA-384 per bundle (`openssl dgst -sha384 -binary | openssl base64 -A`) at postbuild; document the consumer `<script integrity="..." crossorigin>` snippet; set `Cross-Origin-Resource-Policy: cross-origin` on the CDN object so browsers actually run the integrity check.

### Rationale

The Whorl audit (release-hardening-audit `0.2.0`, two passes on a JavaScript monorepo combining a Rollup-built browser SDK with an AWS Lambda backend) surfaced one shared root cause across the two HIGH-severity findings: **the skill assumed canonical Express-on-Node shape**. Cards #11-#18 used Express idioms in their literal Tells/Fixes, which biased the auditor toward dismissing them on a Lambda surface ("this is `app.use(helmet())`, doesn't apply") rather than adapting them ("the analog on Lambda is `response.headers = {...}`, does it apply?"). The fix is structural: each card now carries explicit per-runtime analogs, forcing the audit walk to verify rather than dismiss. The Hard Gate softening fixes a related bias — assumption that the audit target has a single root manifest. Both changes generalize the skill from "Express-on-Node hardening audit" to "JS/TS/Node hardening audit, runtime-aware".

The MEDIUM-severity additions (#19, #20) cover gaps the Whorl auditor surfaced unprompted: build determinism (banner build-date breaks reproducible-build invariant on CDN-distributed bundles) and Subresource Integrity (CDN-distributed browser bundles without SRI guidance leave consumers exposed to silent CDN compromise).

## [0.2.0] — 2026-04-29

Workflow contract change: **interactive remediation flow**. The skill now pauses after analysis, presents findings as a decision table, waits for explicit user confirmation, applies only confirmed in-repo fixes, and reports affected files separately.

### Changed

- **Audit Workflow expanded from 3 phases to 6:**
  - Phase 1: Discover (unchanged)
  - Phase 2: Audit (unchanged)
  - Phase 3: **Present** — emits Artifact A (glanceable decision table) and Artifact B (detailed cards) — replaces the old monolithic markdown report
  - Phase 4: **Confirm** — pauses with explicit prompt menu (`all` / `block` / `block+harden` / `<ids>` / `none`); waits for user response
  - Phase 5: **Apply** — executes only confirmed in-repo fixes via `Edit` / `Write` / `Bash`; runs verification command after each; captures pass/fail
  - Phase 6: **Report changes applied** — Changes-applied table + Files-modified list with per-finding attribution + Remaining-user-actions list + full verification re-run block

- **Per-finding metadata extended** with three new required fields:
  - **Effort estimate** (XS / S / M / L) with mechanical thresholds (XS <5min, S <30min, M <2h, L >2h)
  - **Auto-fix capability** (✓ Yes / ⚠ Partial / ✗ No) with explicit semantics — ✓ = Claude applies via single repo state change; ⚠ = Claude applies in-repo portion + user does external action; ✗ = external action only, Claude cannot apply
  - **External setup required** — out-of-repo step (npm OIDC dashboard config, Sentry account, secret rotation at provider, branch protection settings) or "None"

- **Decision factors callout** added to detailed cards (Artifact B): effort, auto-fix capability, external setup, reversibility, blast radius, recommendation

- **Constraints (MUST DO / MUST NOT DO)** updated to enforce the interactive contract:
  - MUST DO: present Phase 3 before Phase 5; pause at Phase 4; capture verification result per fix; produce per-finding file-modification attribution in Phase 6
  - MUST NOT DO: apply any fix before user confirmation (even BLOCK); auto-execute external actions; mark a finding `✓ Applied` if verification failed; retry failing verification by re-applying; collapse Phases 3 and 5

- **Review Checklist** restructured into 5 sub-categories matching the 6-phase workflow (discovery+audit, presentation, confirmation, application, reporting); 12 new items covering the interactive flow

- **Output Template section** simplified — single source of truth now lives in the Workflow section under Phase 3 and Phase 6, no duplication

- **Frontmatter `description`** extended to advertise the interactive flow so Claude correctly anticipates the conversation shape on invocation; `scope` changed from `audit-and-report` to `audit-confirm-apply-report`; `output-format` from `markdown-report` to `decision-table-then-applied-changes`

### Rationale

User feedback during the 0.1.0 design session: the skill should not silently apply fixes after analysis. The original 3-phase model (Discover → Audit → Report) ended at "produce a markdown report" — the user has to manually transcribe findings into action. The 6-phase model adds the remediation loop while preserving the report-only path (when user says `none` in Phase 4, Phase 6 is skipped and the Phase 3 report stands).

The decision table in Phase 3 is the load-bearing artifact: it surfaces effort and auto-fix capability up front, letting the user triage without reading every card. Cards (Artifact B) carry the depth needed to commit to a fix; the table carries the scan needed to prioritize.

### Migration notes

Anyone using the 0.1.0 contract programmatically (e.g., parsing the markdown report) needs to adapt to the new shape: there is no longer a single end-of-audit report. Phase 3 is the analysis output; Phase 6 (when present) is the application output. The two are emitted in separate turns of the conversation, with the user's Phase 4 response between them.

## [0.1.0] — 2026-04-29

Initial draft. Pre-shipping release; not registered as production-quality until real-world feedback validates the gallery entries against actual JS/TS/Node release pipelines.

### Added

- **SKILL.md** with Hard Skip Gate (5 programmatic tells), Top 5 Rules, 18-entry Anti-Pattern Gallery, 3-level severity model (BLOCK / HARDEN / NICE-TO-HAVE), 3-phase Audit Workflow (Discover → Audit → Report), Reference Guide, MUST DO / MUST NOT DO Constraints, Output Template, Review Checklist
- **6 reference files**:
  - `references/build-artifacts.md` — sourcemap policy, env-var inlining (DefinePlugin / Vite define / esbuild define semantics), dotenv leakage, console scrubbing, path leakage configs per bundler, minification config, bundle inspection commands
  - `references/distribution-config.md` — CSP recipes (nonce / hashes / strict-dynamic), SRI, security headers (helmet defaults), CORS allowlist patterns, cookie flag matrix, body-parser limits, `trust proxy` configuration
  - `references/supply-chain.md` — OIDC trusted publishing, lockfile hygiene, postinstall script audit, CVE scanning thresholds (npm audit / osv-scanner), SBOM (CycloneDX / syft), license audit, `engines.node`, `files` field audit, typosquatting / dependency confusion
  - `references/secrets-hygiene.md` — `.env` policy, `.gitignore` patterns, gitleaks / trufflehog history scan, `.npmrc` token handling, CI secret leakage prevention, pre-commit hooks, built-artifact secret scan
  - `references/cicd-hardening.md` — Actions permissions matrix, `pull_request_target` deep-dive, OIDC for cloud auth (AWS / GCP / Azure), action pinning to commit SHA, workflow injection patterns, branch protection
  - `references/tools-and-checks.md` — operational matrix of `gitleaks`, `osv-scanner`, `npm audit`, `retire.js`, `semgrep`, `publint`, `arethetypeswrong`, `source-map-explorer`, `eslint-plugin-security`, `audit-ci`; install + run + threshold config; CI integration patterns; first-audit runbook

### Scope decisions

- **JS / TS / Node.js only.** No Python / Go / Rust / Docker-base coverage. Tightened from initial multi-ecosystem proposal to keep token cost manageable and the gallery sharp.
- **Audits the shipped artifact and surface, not changed code.** Code-level vulnerability scanning belongs to `/security-review` (the official Anthropic skill).
- **18 gallery entries** prioritized for "humanly hard to spot" patterns: sourcemap leakage variants (paths inside `.map`), build-time env-var inlining via wildcard `define`, `dotenv` in client bundle, `package.json` `files` drift, transitive postinstall, OIDC vs `NPM_TOKEN`, `.d.ts` internal leak, Express defaults (helmet / CORS / cookies / body limit / trust proxy / crypto / JWT).
- **Severity model is release-blocking semantics**, not CVSS. BLOCK = do-not-ship, HARDEN = ship-and-fix, NICE-TO-HAVE = track. Rationale documented in SKILL.md.

### Pre-shipping caveats

- No real-world feedback yet — the gallery is theoretically grounded but not validated against repeated production audits.
- References are at "operational stub" depth — enough commands to run an audit, but expansion expected as feedback identifies gaps.
- No anti-pattern entries yet for: monorepo-specific concerns (workspace inheritance of unsafe configs), Docker build-stage secret leakage in node_modules layers, Vercel / Netlify / Cloudflare Pages env-var visibility model.
- Pinned at `0.1.0` until at least 3 real-world audits validate the workflow end-to-end. Will bump to `1.0.0` once shipping-quality criteria in workspace `CLAUDE.md` are fully met.
