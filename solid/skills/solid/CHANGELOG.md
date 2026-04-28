# Changelog — solid

This file is a sibling to `SKILL.md` and is **not** auto-loaded by the Claude Code runtime — keeping it here lets humans browsing the GitHub repo see version history without inflating every skill invocation. Full development notes live in the workspace `docs/solid.md`.

## 2.2.0 — 2026-04-28

Driven by feedback from the multicarrier shipping backend review (`feedbacks/skill-shipper-2026-04-28.md`).

- Added **Decision Tree Q8** — cancellation propagation across every layer of `await`. Language-agnostic (covers `AbortSignal`/`CancellationToken`/`context.Context`).
- Added **Anti-pattern #15** — Mutation across an `await` boundary (hidden temporal coupling).
- Added **exemption to Anti-pattern #3** for closed protocol-level enums (HTTP verbs, file extensions, MIME types) — table dispatch is fine; the polymorphism would only shuffle names.
- **Reordered SKILL.md sections** — Top 5 → Anti-Pattern Gallery → Decision Trees → Workflow → Reference Guide → Constraints → Behavioral Principles → Output Templates → Review Checklist. Highest-leverage content (gallery) now precedes trees.
- **Gated Output Templates** behind a "skip if reviewing" prelude — RED-GREEN-REFACTOR templates apply to implementation, not audit.
- **Removed Recent Changes block** from `SKILL.md` (now in this file). Saves ~200 tokens per invocation.
- Updated MUST DO + MUST NOT DO + Review Checklist with cancellation rule and mutation-across-await rule.

## 2.1.0 — 2026-04-28

- Reformulated the "≤ 2 instance variables" rule to apply only to *mutable runtime state* — constructor-injected collaborators don't count. Updated Q2, MUST DO, Review Checklist, and Anti-pattern #11 accordingly.
- Added **Top 5 Rules** lead-in section so short sessions can anchor on the highest-firing rules.
- Shrunk Anti-Pattern Gallery examples (kept `❌` + Tell + 1-line Fix; full `✅` rewrites moved to references).

## 2.0.0 — 2026-04-27

Full operational rewrite. SKILL.md + 14 references. Replaces the planned `clean-code-discipline` and `tdd-workflow` skills — single skill covers SOLID + TDD + clean code + refactoring + architecture + patterns + review + legacy + when-not-to-apply.

## 1.0.0 — 2026-04-13

Original version obtained from the internet. Replaced by 2.0.0 with operational rewrite.
