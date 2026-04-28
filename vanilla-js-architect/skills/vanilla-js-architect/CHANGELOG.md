# Changelog — vanilla-js-architect

This file is a sibling to `SKILL.md` and is **not** auto-loaded by the Claude Code runtime — keeping it here lets humans browsing the GitHub repo see version history without inflating every skill invocation. Full development notes live in the workspace `docs/vanilla-js-architect.md`.

## 2.2.0 — 2026-04-28

Driven by feedback from the multicarrier shipping backend review (`feedbacks/skill-shipper-2026-04-28.md`) — a Node.js Express project where this skill loaded on an explicit slash-command despite being in its own "When NOT to Use" list, costing ~6.6K tokens for one transferable finding.

- Added **"Skip This Skill If — Hard Gate"** at the very top of `SKILL.md` — 6 programmatic tells (no `exports`/`browser` field, imports `express`/`fastify`/`koa`/`hapi`/`@nestjs`/`next`/`nuxt`/`astro`, no bundler config, runtime is `pm2`/`node` directly, etc). Refuse to engage if 2+ match. Slash-command invocations now hit this gate before paying the load cost.
- Sharpened **Anti-pattern #9b** — added explicit "trace the signal across **every** layer of `await`" tell, not just the leaf. A single missing layer disables cancellation downstream.
- Added **scope qualifier to Anti-pattern #4** — clarifies that "don't mutate consumer-provided options" applies at the public API surface, not to internal middleware (`req.body`, `ctx.state`, `request`) where mutation is idiomatic.
- **Reordered SKILL.md sections** — Top 5 → Anti-Pattern Gallery → Decision Tree → Core Workflow → Reference Guide → Constraints → Output Templates → Review Checklist. Gallery now precedes the decision tree.
- **Removed Recent Changes block** from `SKILL.md` (now in this file). Saves ~200 tokens per invocation.
- Updated MUST DO + Review Checklist with the multi-layer signal-trace rule and the scope-qualified mutation rule.

## 2.1.0 — 2026-04-28

- Promoted Anti-pattern #9 (`AbortSignal`) with three concrete variants instead of a single poll-loop example.
- Added **Anti-pattern #13** — Inconsistent async surface within a single API.
- Added **Core Workflow step 6** — Smoke-test the built artifact after distribution-touching changes.
- Added **Top 5 Rules** lead-in section.
- Shrunk Anti-Pattern Gallery examples (kept `❌` + Tell + 1-line Fix).

## 2.0.0 — 2026-04-27

Full operational rewrite. SKILL.md + 14 references. 5-question decision tree, 12 anti-patterns, self-applicable checklist.

## 1.0.0 — 2026-04-27

Initial version. SKILL.md + 7 generic references. Replaced same day by 2.0.0 when the decision was made to make the skill operational.
