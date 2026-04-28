# When NOT to Apply (the discipline override)

The most important reference in this skill. SOLID + TDD + clean code applied to the wrong context is **harmful**. This file is the operational filter that decides when the rules apply, when they relax, and when they don't apply at all.

## The Core Insight

**Discipline scales with longevity and stakes.**

- Code that lives 2 hours → minimal discipline
- Code that lives 2 weeks → some discipline
- Code that lives 2 years in production → full discipline

Applying full SOLID/TDD to throwaway code wastes time. Applying none to long-lived production code creates the unmaintainable nightmares that motivated SOLID in the first place.

---

## Decision Tree — Which Discipline Level?

```
Will this code live > 1 month and be maintained by others?
├── NO ↓
│   Is it pure exploration / spike / proof of concept?
│   ├── YES → DISCIPLINE LEVEL: Minimal (just make it work)
│   └── NO  ↓
│       Is it a one-off script (data migration, build script)?
│       ├── YES → DISCIPLINE LEVEL: Low (clear names + comments only)
│       └── NO  ↓
│           Is it glue code (5-line wiring of two libs)?
│           ├── YES → DISCIPLINE LEVEL: Low
│           └── NO  → continue
└── YES ↓
    Is it production code with real users?
    ├── YES → DISCIPLINE LEVEL: Full (SOLID + TDD + everything)
    └── NO  ↓
        Is it MVP under explicit "we'll rewrite" agreement?
        ├── YES → DISCIPLINE LEVEL: Medium (with documented debt)
        └── NO  → DISCIPLINE LEVEL: Full
```

---

## Discipline Levels

### Level 0 — None (true throwaway)

**Examples:** REPL exploration, ad-hoc query, one-time data fix you'll run once.

**What you skip:**
- TDD
- SOLID
- Naming polish
- Refactoring
- Documentation

**What still applies:**
- Don't lose user data (backups before destructive operations)
- Don't commit secrets
- Don't introduce security holes that survive past the script

**Lifetime:** hours.

```typescript
// throwaway data fix script — just make it work, then delete
const users = JSON.parse(fs.readFileSync("users.json", "utf-8"));
users.forEach(u => { u.active = true; });
fs.writeFileSync("users-fixed.json", JSON.stringify(users));
```

### Level 1 — Low (one-off scripts, glue code)

**Examples:** build script, deployment helper, data migration, scheduled job that runs forever but is 20 lines.

**What you do:**
- Clear variable names
- Top-level comment saying what it does + when it runs
- Basic error handling so it fails loudly, not silently
- One e2e smoke test (actually run it on test data)

**What you skip:**
- Unit tests (test by running)
- Class hierarchies / abstractions
- Value objects (use primitives)
- Refactoring beyond clarity

**Lifetime:** weeks-months, but rarely modified.

```typescript
// migrate-users-to-v2.ts
// Run once during the v2 migration. Reads users.json, transforms to new schema,
// writes users-v2.json. Idempotent — safe to re-run.

import { readFileSync, writeFileSync } from "node:fs";

function migrate() {
  const raw = readFileSync("users.json", "utf-8");
  const usersV1 = JSON.parse(raw) as Array<{ id: string; name: string; active: boolean }>;

  const usersV2 = usersV1.map(u => ({
    id: u.id,
    fullName: u.name,
    status: u.active ? "active" : "inactive",
    createdAt: new Date().toISOString(),
  }));

  writeFileSync("users-v2.json", JSON.stringify(usersV2, null, 2));
  console.log(`Migrated ${usersV2.length} users`);
}

try {
  migrate();
} catch (err) {
  console.error("Migration failed:", err);
  process.exit(1);
}
```

### Level 2 — Medium (MVP, prototype that might become real)

**Examples:** MVP feature with explicit "we'll rewrite" agreement, time-pressured proof-of-concept that might ship.

**What you do:**
- Some tests for happy paths
- Reasonable structure (not God class, not 200-line method)
- Document the debt loudly: README section, TODO with issue link

**What you skip:**
- 100% test coverage
- Premature abstractions
- Multiple implementations of interfaces
- Polish

**Lifetime:** weeks to months. **Critical:** track the debt or it becomes Level 3 by accident.

### Level 3 — Full (production code)

**Examples:** Anything users depend on, anything other developers will modify, anything you can't lose.

**Apply everything in this skill.**

---

## Specific Contexts and Their Discipline Levels

### Spikes / Exploration

**Goal:** learn the shape of a problem.

**Approach:**
- No tests
- No abstractions — straight code
- One file is fine
- Throw it away when you've learned

**Anti-pattern:** spike code "graduating" to production. The whole point is throwaway. If you find a spike worth keeping, **rewrite it properly** with TDD from scratch. Don't just merge the spike with some polish.

```typescript
// spike: can we use library X for our use case?
// 50 lines of throwaway code to test feasibility
import { X } from "x-library";
const result = X.doStuff(/* hardcoded test inputs */);
console.log(result);
```

### Throwaway Scripts

**Goal:** one-time task that runs N times and is done.

**Approach:** Level 1 discipline. Clear names, error logging, smoke test on test data.

**Don't apply:** classes, dependency injection, value objects, design patterns. The cost exceeds the benefit.

### Glue Code (3-10 lines wiring libraries)

```typescript
// Glue code — no domain logic, just wiring
import { createClient } from "redis";
import { Cache } from "./cache";

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();
export const cache = new Cache(redis);
```

**Approach:** clear names. No abstractions. No tests for the glue itself (test the components separately).

### Configuration Files (disguised as code)

```typescript
// config/eslint.config.js — declarative configuration
module.exports = {
  rules: {
    "no-unused-vars": "error",
    "complexity": ["error", 10],
    /* ... */
  },
};
```

**Approach:** none of the SOLID rules apply. This is data, not behavior.

### MVPs Under Time Pressure

**The trap:** "We'll rewrite this later" — usually a lie.

**Discipline rules for MVPs:**
1. **Document the debt explicitly.** Comment in the file: "MVP — will be rewritten before launch (JIRA-123)."
2. **Track the rewrite as a real ticket** with a deadline.
3. **Apply Level 2 discipline:** happy-path tests, reasonable structure, no God classes.
4. **Don't skip tests entirely.** Even Level 2 has SOME tests. Otherwise you can't verify the rewrite preserves behavior.

If the MVP ships and survives without rewrite for 6 months, it's now production code. **Treat it as such immediately** — start adding the missing tests, refactor incrementally per the legacy code playbook (`legacy-code.md`).

### Feature Flags / A/B Test Code

Both branches of an A/B test are temporary by definition. The discipline question:
- Will the winning variant become permanent?
- If YES → apply full discipline to the winner
- The losing variant is throwaway — minimal discipline, delete cleanly

**Anti-pattern:** A/B test branches that never get cleaned up. Each becomes Level 3 by accident.

### Generated Code

Code generated by tools (protobuf, GraphQL codegen, OpenAPI codegen).

**Approach:** none of the discipline rules apply to the generated files themselves. **Don't refactor them.** The generator is the source of truth — modify the schema or templates, regenerate.

The code that USES the generated code follows full discipline.

### Test Code

Test code is production code in disguise. It must be maintained.

**Apply:**
- Naming clarity (concrete examples)
- AAA structure
- Test builders for shared setup
- No copy-paste — extract helpers (after Rule of Three)

**Skip / relax:**
- TDD on tests themselves (you're not testing the test framework)
- Some patterns (singleton in test fixtures is OK)

### Legacy Code You Don't Own

Code from another team or vendor that you import.

**Approach:**
- Don't refactor it (you'll need to rebase forever)
- Wrap it in your own interface (Adapter pattern)
- Test only via the wrapper

---

## Override Triggers — When Discipline Level Changes

### Promote (more discipline)

| Trigger | New level |
|---------|-----------|
| Spike code being shipped to production | Level 0 → 3 (rewrite from scratch) |
| MVP surviving 6 months | Level 2 → 3 (start treating as production) |
| Throwaway script being scheduled to run regularly | Level 1 → 2 (it's now production) |
| Glue code growing beyond 50 lines with logic | Level 1 → 3 (it's now real code) |

### Demote (less discipline)

| Trigger | New level |
|---------|-----------|
| Production code being deleted next week | Level 3 → 1 (don't refactor on the way out) |
| Feature flag's losing variant chosen | Level 3 → 0 (delete cleanly) |
| Code being replaced via Strangler Fig | Level 3 → 2 (legacy mode) |

---

## Anti-Patterns of Misapplication

### Over-engineering for hypothetical futures

**Symptom:** interface with one implementation; abstraction added "in case we need it"; configuration for things never configured.

```typescript
// ❌ — premature abstraction in an MVP
interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  save(user: User): Promise<void>;
  /* ...8 more methods none used yet */
}

class PostgresUserRepository implements UserRepository {
  /* only one impl ever exists */
}

class UserService {
  constructor(private readonly repo: UserRepository) {}
  /* ... */
}
```

For an MVP, just write the class direct:

```typescript
class UserRepository {
  findById(id: UserId): Promise<User | null> { /* ... */ }
  save(user: User): Promise<void> { /* ... */ }
}
```

Add the interface when you NEED it (test fake or second implementation).

### TDD on throwaway code

**Symptom:** spike with full test suite. Tests delete with the code anyway.

**Fix:** spike has no tests. Rewrite if shipping.

### SOLID applied to glue code

**Symptom:** 3-line wiring of two libraries split across 4 files with dependency injection container.

**Fix:** put it inline. Glue is glue.

### Refactoring code about to be deleted

**Symptom:** PR refactoring a feature scheduled for removal next sprint.

**Fix:** don't. Wasted effort.

### "Just make it work" forever

**Symptom:** "MVP" code from 2 years ago, never rewritten, now drowning in debt.

**Fix:** stop pretending it's still an MVP. Apply legacy code playbook (`legacy-code.md`). Refactor incrementally per hot spot.

---

## The Pragmatic Filter

Before applying any rule, ask:

1. **Will this code outlive my interest in it?**
2. **Will someone else have to maintain it?**
3. **Will real users depend on it?**

If YES to any → full discipline.
If NO to all three → minimal discipline.

This is the honest test. Most "engineers waste time on perfection" complaints come from misapplied discipline (Level 3 effort on Level 0 code). Most "this codebase is unmaintainable" complaints come from missing discipline (Level 0 effort on Level 3 code).

---

## How to Document the Discipline Level

For team clarity, mark the level in the file or commit:

```typescript
// MVP — Level 2 discipline
// Will be rewritten before launch. Tracked in JIRA-1234.
// Don't add abstractions. Don't add features. Just make it work.
```

```bash
# git commit message
chore: spike for new auth flow (Level 0 — to be deleted)
```

```markdown
## In the README
This service is currently in MVP phase. We've made deliberate trade-offs:
- Limited test coverage (happy path only)
- No abstraction over the database
- Logic in services that should be in entities

Tracked in [REWRITE-FOR-LAUNCH](link). Do not propagate these patterns.
```

---

## Quick Reference

| Code lifetime | Discipline | What you do | What you skip |
|---------------|-----------|-------------|---------------|
| Hours | Level 0 | Make it work | Tests, abstractions, polish |
| Weeks (one-off) | Level 1 | Clear names, error handling | Unit tests, design patterns |
| Months (MVP) | Level 2 | Happy path tests, basic structure | Full coverage, premature abstraction |
| Years (production) | Level 3 | Full SOLID + TDD + everything | Nothing |

### Red flags (misapplied discipline)

- Spike with full test suite
- 5-line glue with dependency injection container
- Throwaway script with class hierarchies
- MVP with 100% test coverage and no shipped features

### Red flags (missing discipline)

- "MVP" from 2 years ago in production
- Spike that "graduated" to production without rewrite
- Throwaway script now running in cron forever
- Glue code grown to 500 lines with business logic

### The honest test

```
Will real users depend on this code in 6 months?
├── YES → full discipline
└── NO  → minimal discipline + plan to delete
```

If you're not sure, **assume it'll outlive you and apply full discipline.** Most code does.
