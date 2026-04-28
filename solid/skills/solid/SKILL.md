---
name: solid
description: Use when writing, refactoring, reviewing, or designing code in any object-oriented language. Operationalizes SOLID principles, TDD discipline, clean code practices, and software design through mechanical rules — not abstract principles. Examples in TypeScript but rules apply to any OO language (Java, C#, PHP, Python, etc). Not for one-off scripts, throwaway prototypes, or trivial glue code (see references/when-not-to-apply.md).
license: MIT
metadata:
  author: https://github.com/tarcisioq
  version: "2.1.0"
  domain: software-engineering
  triggers: SOLID, TDD, clean code, refactoring, code review, architecture, design patterns, code smells, value objects, single responsibility, dependency injection
  role: senior-engineer
  scope: design-implementation-review
  output-format: code
---

# SOLID — Engineering Discipline

Operational reference for writing senior-level code: SOLID principles, TDD, clean code, refactoring, design patterns. **Mechanical rules over abstract principles** — every guideline has a concrete tell so the output is consistent across sessions.

## How to Use This Skill

This skill is structured for **lookup**, not linear reading.

1. Read this `SKILL.md` end-to-end once at the start of a task
2. Apply the **Core Workflow**
3. Hit a specific decision → load the matching **reference file** (table below)
4. Before declaring done → run the **Review Checklist**

If you're writing code without having loaded the relevant reference, stop and load it. Generic OO knowledge from training data produces plausible-looking-but-inconsistent code — that's the exact problem this skill fixes.

## When to Use This Skill

- Writing any non-trivial production code (features, fixes, services)
- Refactoring existing code
- Designing classes, modules, or architecture
- Reviewing code (yours or others')
- Implementing new behavior in a tested codebase
- Debugging that requires understanding design

## When NOT to Use (mandatory check)

Applying full SOLID/TDD discipline to the wrong context wastes time and adds harmful complexity. Skip this skill when:

- **Throwaway scripts** — bash replacements, one-off data migrations, build scripts
- **Spikes / exploratory prototypes** — code you'll throw away after learning
- **MVPs in time pressure** with explicit "we'll rewrite this" agreement
- **Glue code** with no logic (5 lines wiring two libraries together)
- **Configuration files** disguised as code

See `references/when-not-to-apply.md` for the operational decision tree.

## Top 5 Rules — Read This First

These are the rules that fire on almost every real review. If you're short on time, anchor here before the Decision Trees and Anti-Pattern Gallery below.

| # | Rule | Tell to detect |
|---|------|----------------|
| 1 | **Methods ≤ 10 lines** | Method body > 10 lines OR name needs "and" to describe — apply Compose Method (Anti-pattern #8) |
| 2 | **Classes ≤ 50 lines, ≤ 2 ivars of *mutable runtime state*** | Constructor-injected collaborators do **NOT** count toward the 2-ivar limit (see Q2 below). > 50 lines or > 2 stateful ivars → split per concern (Anti-pattern #11) |
| 3 | **Wrap domain primitives in value objects** | Any `string`/`number`/`Date` parameter representing email, money, id, age, country, status — wrap (Anti-pattern #1) |
| 4 | **No `else` after early return** | Any `else` block after an `if` that returns — collapse to guard clauses (Anti-pattern #9) |
| 5 | **Throw typed errors with stable `code`; never sometimes-throw-sometimes-return** | Strings, `new Error(msg)` without code, or mixed throw/return for the same failure — define a typed exception (Q4) |

After these, the rest of the document covers edge cases, less-frequent smells, and references for deep dives.

## Recent Changes

- **2.1.0** (2026-04-28) — Reformulated the "≤ 2 instance variables" rule to exempt constructor-injected collaborators (only mutable runtime state counts). Added this Top 5 Rules section. Shrunk Anti-Pattern Gallery examples (kept ❌ + 1-line fix; full ✅ rewrites moved to references). Added this changelog block.
- **2.0.0** (2026-04-27) — Full operational rewrite: 7 decision trees, 14 anti-patterns, 14 references, self-applicable checklist. Replaces former `clean-code-discipline` and `tdd-workflow` plans.

Full history: `docs/solid.md` (workspace dev doc).

## Core Workflow

### 1. Clarify the requirement (before any code)

- Write the **acceptance criteria** in plain language. If you can't, you don't understand it yet.
- Identify the **domain concepts**. These become value objects + entities.
- Identify the **boundaries** — what's "in" the system, what's external?

### 2. Decide TDD or not (decision tree below)

Most production code → TDD. Spike code → no TDD. See Q1 in Decision Trees.

### 3. Write the simplest test that fails (RED)

- Concrete example, not abstract: `"applies 20% discount to premium users"` not `"can apply discount"`
- Tests behavior, not implementation
- Arrange-Act-Assert structure

### 4. Write the simplest code to pass (GREEN)

- Hardcode if needed (Fake It). Let next tests force generalization.
- No abstraction yet. No "what if we need…". Just pass.

### 5. Refactor (REFACTOR — design happens here)

- Look at duplication. Apply Rule of Three (extract on 3rd, not 1st).
- Apply size limits (see Constraints below).
- Run review checklist.

### 6. Repeat — RED → GREEN → REFACTOR

When done, run the **full Review Checklist** at the end of this file.

## Decision Trees — Resolve Before Writing Code

### Q1: TDD or just write code?

```
Is this throwaway / spike / one-off?
├── YES → No TDD. Write minimal code. (See when-not-to-apply.md)
└── NO  ↓
    Does the codebase have working test infrastructure?
    ├── NO  → First task: set up testing. Then return here.
    └── YES ↓
        Is this fixing a bug?
        ├── YES → Write failing test that REPRODUCES the bug. Then fix.
        └── NO  → Write failing test for the new behavior. Then implement.
```

**TDD is non-negotiable for non-trivial production code.** Skip only by the rules above.

### Q2: Split this class? When?

A class needs splitting when **any** of these fire:

| Tell (mechanical) | Action |
|-------------------|--------|
| `> 50 lines` | Split — likely doing too much |
| `> 2 instance variables of mutable runtime state` (constructor-injected collaborators don't count) | Split — multiple concerns of state mixed |
| `> 5 public methods` | Split — interface too wide |
| Method names span 2+ verb domains (e.g. `save*` + `validate*` + `format*`) | Split per domain |
| You describe it with "and" ("Order **and** invoice generation **and** persistence") | Split per "and" |
| 2+ stakeholders would request changes to different parts | Split per stakeholder |

**Don't split when:** value object (immutable, behaves as a single concept), Result-type wrapper, true aggregate root.

### Q3: Compose or inherit?

```
Default: COMPOSE.

Inherit ONLY when ALL three are true:
1. True "is-a" relationship (a Dog IS-A Animal — survives the substitution test)
2. Framework requires it (extending Error, EventTarget, Component base class)
3. Hierarchy ≤ 1 level deep

Otherwise: composition (inject behavior via constructor, use Strategy/Decorator).
```

If you find yourself adding `instanceof` checks or `if (parent.X) { ... } else { childOnly() }`, the inheritance is wrong — refactor to composition.

### Q4: How to signal failure?

| Failure characteristic | Use |
|------------------------|-----|
| Programmer error (bad args, misuse, contract violation) | Throw typed exception (`throw new InvalidEmail()`) |
| Expected runtime failure (network, validation that consumer routinely handles) | Return `Result` type (`{ ok: boolean, value? error? }`) |
| Operation may legitimately have no result | Return `null` (NOT `undefined`, NOT a sentinel like `-1`) |
| Async operation failure | Reject Promise with typed error |

**Never** throw strings. **Never** sometimes-throw-sometimes-return-error. **Never** use `undefined` for documented absence.

### Q5: Refactor now or later?

```
Are tests covering the code I'm about to refactor?
├── NO  → STOP. Add characterization tests first. (legacy-code.md)
└── YES ↓
    Is the refactor in my path right now (the code I'm working on)?
    ├── YES → Refactor as Boy Scout improvement
    └── NO  ↓
        Is the smell blocking new features or causing bugs?
        ├── YES → Schedule dedicated session
        └── NO  → Note it, leave it. YAGNI.
```

### Q6: Use a design pattern?

```
Did I recognize the problem? (have I seen this exact shape 2+ times before?)
├── NO  → Don't force a pattern. Write straight code.
└── YES ↓
    Does the pattern make this SIMPLER?
    ├── NO  → Don't use it. Pattern is overhead, not virtue.
    └── YES ↓
        Will the team understand it without explanation?
        ├── NO  → Use only with comment + reference link
        └── YES → Apply pattern
```

**Anti-pattern: Pattern-first design.** Patterns emerge from refactoring, not blueprint.

### Q7: Mock, fake, stub, or real?

| Need | Use |
|------|-----|
| Verify a method was called with specific args | **Mock** (`expect(spy).toHaveBeenCalledWith(...)`) |
| Provide canned return values | **Stub** (returns fixed data) |
| Substitute a slow/external dependency with a working in-memory version | **Fake** (`InMemoryUserRepo`) |
| Test pure domain logic | **Real objects** — no doubles |
| Pass a dependency that's never used | **Dummy** (`{} as Logger`) |

**Default: real objects + fakes for I/O.** Reach for mocks only when verifying interaction is the actual contract (e.g. "ensure email was sent").

## Anti-Patterns Gallery

These look correct but aren't. Training-data drift points at them — actively reject.

### 1. Primitive Obsession

```typescript
// ❌ Raw primitives for domain concepts — invalid states unrepresentable
function createUser(email: string, age: number, country: string) {
  if (!email.includes('@')) throw new Error("invalid email");
  if (age < 0) throw new Error("invalid age");
  // Validation scattered, easy to forget at every callsite
}
```

**Tell to detect:** any function param with type `string` or `number` representing a domain concept (id, email, money, date, country, status). Wrap.
**Fix:** introduce a value object that validates at construction (`class Email { constructor(value) { /* validate */ } }`); push validation to the type boundary so callers can't pass invalid state. See `references/object-design.md`.

### 2. Boolean parameters

```typescript
// ❌ — caller has no clue what `true` means without reading the impl
user.save(true);

// ❌ — multiple booleans is a combinatorial explosion of meaning
user.save(true, false, true);
```

**Tell:** boolean parameter on any public method.
**Fix:** split into named methods (`user.saveAndNotify()`, `user.saveQuietly()`) when the call-sites diverge; use a named options object (`user.save({ notify: true })`) when most flags share a code path.

### 3. Switch on type

```typescript
// ❌ — adding a new shape requires editing every switch
function getArea(shape: Shape): number {
  if (shape.type === "circle") return Math.PI * shape.radius ** 2;
  if (shape.type === "square") return shape.side ** 2;
  if (shape.type === "triangle") return 0.5 * shape.base * shape.height;
}
```

**Tell:** `if/else` chain or `switch` on a `type` / `kind` / `discriminator` field.
**Fix:** Replace Conditional with Polymorphism (`refactoring-catalogue.md`) — define an interface (`area(): number`), one class per variant; adding a variant = adding a class, no edits to existing code.

### 4. Anemic domain model

```typescript
// ❌ — User is a data bag; logic lives in service. Tell, don't ask violated.
class User {
  email: string;
  isPremium: boolean;
  balance: number;
}
class UserService {
  applyDiscount(user: User, amount: number) {
    if (user.isPremium) user.balance -= amount * 0.8;
    else user.balance -= amount;
  }
}
```

**Tell:** classes with only getters/setters; logic in `*Service` that mutates other classes' state.
**Fix:** move behavior onto the entity that owns the data (`user.withdraw(amount)` returning a `Result`); the service then orchestrates instead of manipulating internals.

### 5. Premature abstraction

```typescript
// ❌ — interface with one implementation, "for future flexibility"
interface UserRepository { save(u: User): Promise<void>; }
class PostgresUserRepository implements UserRepository { /* only impl */ }
```

**Tell:** abstraction (interface/abstract class) with only ONE implementation, where you don't have a test double or different runtime variant.
**Fix:** use the concrete class directly; introduce the interface when a 2nd impl arrives (Rule of Three) or when a test double is genuinely needed.
**Exception:** DDD architectures legitimately keep the interface in the domain layer with the concrete impl in infrastructure (DIP at architecture level). That's intentional, not premature.

### 6. Speculative generality

```typescript
// ❌ — methods/options "in case we need them"
class PaymentProcessor {
  process(amount: Money) { /* ... */ }
  rollback() { throw new Error("Not implemented"); }
  audit() { throw new Error("Not implemented"); }
  scheduleRecurring() { throw new Error("Not implemented"); }
}
```

**Tell:** unused parameters, `throw new Error("Not implemented")`, options never set.
**Fix:** delete unused methods, params, and options; reintroduce only when a real use-case arrives. YAGNI > flexibility budget.

### 7. Train wreck (Law of Demeter violation)

```typescript
// ❌ — chain of dots. Caller knows too much about object graph.
order.getCustomer().getAddress().getCity().getState().toUpperCase();
```

**Tell:** more than one `.` per expression chain on object accesses.
**Fix:** ask the immediate friend (`order.getShippingState()`); push the traversal *into* the object that owns the data, so callers depend on one layer instead of the whole graph.

### 8. Method longer than 10 lines

```typescript
// ❌ — `processOrder` does 5 things. Hard to test, hard to name.
function processOrder(order: Order) {
  // Validate (5 lines)
  // Calculate total (8 lines)
  // Apply discounts (6 lines)
  // Persist (4 lines)
  // Notify (3 lines)
}
```

**Tell:** method body > 10 lines OR method name needs "and" to describe.
**Fix:** apply Compose Method (`refactoring-catalogue.md`) — extract each section into a named helper so the parent reads as a high-level recipe (`validateOrder` → `calculateTotal` → `applyDiscounts` → ...).

### 9. Else after early return

```typescript
// ❌ — `else` is dead weight; the `if` already returned
function getDiscount(user: User): Percent {
  if (user.isPremium) {
    return Percent.of(20);
  } else {
    return Percent.of(0);
  }
}
```

**Tell:** any `else` block. Almost always a missed early return.
**Fix:** flatten with guard clauses (`if (user.isPremium) return Percent.of(20); return Percent.of(0);`) — drops one indentation level and removes dead syntax.

### 10. Mocking value objects

```typescript
// ❌ — mocking a Money instance hides bugs in Money itself
const mockMoney = { toCents: jest.fn().mockReturnValue(100) };
order.add(mockMoney);
```

**Tell:** mocking anything that's pure data + pure logic (Money, Email, DateRange).
**Fix:** use the real value object (`order.add(Money.dollars(1))`); reserve doubles for infrastructure boundaries (DB, network, filesystem).

### 11. God object

```typescript
// ❌ — one class knows everything
class User {
  // Authentication
  login() { } logout() { } resetPassword() {}
  // Preferences
  setTheme() { } setLanguage() {}
  // Notifications
  sendEmail() { } sendSMS() {}
  // Billing
  charge() { } refund() {}
}
```

**Tell:** > 5 instance variables of mutable state (injected collaborators don't count — see Q2) OR class with method names spanning 3+ verb domains.
**Fix:** split per stakeholder/verb domain (`User` keeps identity + behavior; carve out `AuthService`, `UserPreferences`, `NotificationService`, `BillingService`). Each piece passes the "and" test alone.

### 12. Comments explaining what

```typescript
// ❌ — comment paraphrases the code; rots when code changes
// Increment the user's age by one
user.age = user.age + 1;

// Check if user is older than 18
if (user.age > 18) { /* ... */ }
```

**Tell:** comment that paraphrases the next line.
**Fix:** delete what-comments; rename identifiers if the code isn't self-explanatory. Keep a comment only when it captures *why* — a business rule, a workaround, a non-obvious constraint (`// Legal: drinking age here is 21, not 18`).

### 13. Singleton hiding dependencies

```typescript
// ❌ — caller has hidden dependency on global state; impossible to test in isolation
class OrderService {
  process(order: Order) {
    Logger.getInstance().info("processing");
    Database.getInstance().save(order);
  }
}
```

**Tell:** `getInstance()` calls or static method calls reaching out to "shared" services.
**Fix:** make every dependency explicit through the constructor (`constructor(private logger: Logger, private db: Database)`) so tests can substitute fakes and the dependency graph is visible at construction.

### 14. Long parameter list

```typescript
// ❌ — > 3 params; ordering is fragile; meaning hidden
function createOrder(
  customerId: string,
  items: Item[],
  shippingAddress: Address,
  billingAddress: Address,
  paymentMethod: string,
  discountCode: string,
  notes: string
) { }
```

**Tell:** > 3 parameters in any function/method.
**Fix:** Introduce Parameter Object (`refactoring-catalogue.md`) — group related params into a typed input class (`CreateOrderInput`); name the groupings (`addresses: { shipping, billing }`, `options: { discountCode, notes }`) so meaning isn't carried by position.

## Reference Guide

Load detailed guidance based on what you're about to do.

| Topic | Reference | Load when you see… |
|-------|-----------|---------------------|
| SOLID principles | `references/solid-principles.md` | Designing classes, deciding to split, applying DIP, designing interfaces |
| TDD | `references/tdd.md` | Writing tests, deciding test order, choosing London vs Detroit, mock vs fake |
| Clean code | `references/clean-code.md` | Naming, formatting, structure, object calisthenics |
| Code smells | `references/code-smells.md` | Reviewing existing code, deciding "is this bad?", finding refactor targets |
| Refactoring catalogue | `references/refactoring-catalogue.md` | Need step-by-step recipe: Extract Method, Replace Conditional with Polymorphism, Introduce Parameter Object, etc. |
| Object design | `references/object-design.md` | Choosing class stereotype, value object vs entity, designing aggregates |
| Cohesion + coupling | `references/cohesion-and-coupling.md` | Measuring coupling, deciding module boundaries, instability metric |
| Architecture | `references/architecture.md` | Layering, dependency rule, hexagonal/clean arch, feature vs layer organization |
| Design patterns | `references/design-patterns.md` | Recognizing a pattern fit, deciding NOT to apply, anti-pattern alternatives |
| Testing | `references/testing.md` | Test types, AAA, doubles, builders, contract tests, testing per layer |
| Complexity | `references/complexity.md` | Cyclomatic/cognitive complexity, KISS/YAGNI/DRY tradeoffs, accidental vs essential |
| Legacy code | `references/legacy-code.md` | No tests + need to change, characterization tests, seams, sprout method/class |
| Code review | `references/code-review.md` | Reviewing PR, what to flag, comment patterns, self-review checklist |
| When NOT to apply | `references/when-not-to-apply.md` | Spike, prototype, MVP, throwaway script — decision tree |

## Constraints

### MUST DO
- Write a failing test before production code (unless under "When NOT to Use")
- Wrap primitives that represent domain concepts in value objects
- Keep methods ≤ 10 lines, classes ≤ 50 lines, files ≤ 200 lines
- Keep **mutable runtime-state** instance variables ≤ 2 per class (constructor-injected collaborators don't count — see Q2). If you have > 2 fields of *state*, compose them into a single value/structure object.
- Keep parameter list ≤ 3; use parameter object beyond that
- One level of indentation per method (early return for guards)
- Apply Rule of Three before extracting duplication
- Inject dependencies via constructor (no `getInstance()`, no static imports of services)
- Use polymorphism instead of `switch`/`if-else` chains on type discriminators
- Run the Review Checklist before declaring done

### MUST NOT DO
- Write production code without a failing test (in production code paths)
- Use raw primitives for domain concepts (id, email, money, date, status, etc.)
- Throw strings or untyped errors — use typed exception classes
- Use `else` when early return works
- Comment what the code does — only why
- Mock value objects (mock infrastructure, not domain values)
- Add abstractions before the third concrete need (Rule of Three)
- Use boolean parameters on public methods (split into named methods or options object)
- Reach across object graphs (`a.b.c.d` — Law of Demeter)
- Use inheritance for code reuse — compose instead

## Review Checklist (run before declaring done)

Apply this list to every change. If any item is "no" or "not sure", fix before declaring complete.

### Tests
- [ ] Every new behavior has a failing-first test
- [ ] Tests use Arrange-Act-Assert structure
- [ ] Test names are concrete examples, not abstract ("returns 5 when adding 2 + 3", not "can add")
- [ ] Tests verify behavior, not implementation (no testing private fields)
- [ ] No mocks for value objects; doubles only for infrastructure boundaries

### Naming
- [ ] Same concept = same name everywhere (no `getUser` + `fetchClient` for the same thing)
- [ ] Domain language, not technical jargon
- [ ] No vague names: `data`, `info`, `manager`, `handler`, `processor`, `utils`
- [ ] Booleans prefixed with `is`/`has`/`can`
- [ ] Methods are verbs; classes are nouns

### Structure
- [ ] No method > 10 lines
- [ ] No class > 50 lines
- [ ] No method with > 1 level of indentation (early returns)
- [ ] No class with > 2 instance variables of *mutable runtime state* (injected collaborators don't count — compose state into a value/structure object if more)
- [ ] No function with > 3 parameters (parameter object)
- [ ] No `else` blocks (early returns or polymorphism)

### Design
- [ ] Single responsibility per class (passes the "and" test)
- [ ] Domain concepts wrapped in value objects (no raw primitives)
- [ ] Logic lives with data (Tell, Don't Ask — no anemic models)
- [ ] Dependencies injected, not instantiated internally
- [ ] No `switch`/`if-else` on type discriminator (use polymorphism)
- [ ] Composition preferred over inheritance
- [ ] No cross-object reaches (Law of Demeter — `a.b.c` is suspicious)

### Complexity
- [ ] No abstractions added "for future flexibility" (YAGNI)
- [ ] No duplication extracted before 3rd occurrence (Rule of Three)
- [ ] Public API documented with WHY, not WHAT
- [ ] No commented-out code (delete it; git remembers)

### When-not-to-apply override
- [ ] Confirmed this code is NOT a throwaway / spike / glue (if it is, the rules above relax — see `when-not-to-apply.md`)

## Behavioral Principles (memorize)

These are the deep rules. Reference them when explaining decisions.

- **Tell, Don't Ask** — Command objects, don't query and decide. The object that has the data has the behavior.
- **Design by Contract** — Every method has preconditions, postconditions, invariants. Document them via types or asserts.
- **Hollywood Principle** — "Don't call us, we'll call you." (Inversion of Control / Dependency Injection)
- **Law of Demeter** — Only talk to immediate friends. One dot per line.
- **Rule of Three** — Don't extract until the third occurrence. Wrong abstraction is worse than duplication.
- **YAGNI** — You aren't gonna need it. Don't build for hypothetical futures.
- **KISS** — Simplest thing that works. Add complexity only when forced by a real requirement.
- **Boy Scout Rule** — Leave the code better than you found it. Small improvements every visit.

## Output Templates

When implementing a feature, deliver:

1. **Failing test** that describes the new behavior (RED)
2. **Minimum production code** that passes the test (GREEN)
3. **Refactored code** with smells eliminated (REFACTOR)
4. **Value objects** for any domain primitives introduced
5. **Public methods documented** with JSDoc/equivalent (`@param`, `@returns`, `@throws`, `@example`)
6. **No dead code** — everything you wrote is reachable from a test or public API
