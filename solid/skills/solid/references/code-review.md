# Code Review

Operational guide for reviewing PRs (yours and others'). Mechanical checklist + comment patterns + what to skip.

## Decision Matrix — What to Review

| Aspect | Reviewer's job? |
|--------|-----------------|
| Code style (indentation, semicolons, quotes) | **No** — that's the linter |
| Correctness, edge cases | **Yes** |
| Design (SOLID, smells, structure) | **Yes** |
| Test coverage and quality | **Yes** |
| Naming clarity | **Yes** |
| Performance (when measurably critical) | **Yes** — with measurement evidence |
| Architectural fit | **Yes** for non-trivial changes |
| Personal preference style | **No** — review the code, not the author |

---

## Self-Review (BEFORE opening the PR)

Run through this list yourself first. Ship a PR you'd approve.

### Self-review checklist

- [ ] Diff opens cleanly — no committed `node_modules`, `.env`, or build artifacts
- [ ] All tests pass locally
- [ ] No `console.log`, debug code, or commented-out code
- [ ] No TODOs without an issue link or owner
- [ ] Variable / function names are clear without context
- [ ] PR title summarizes the change in one line
- [ ] PR description explains WHY (the WHAT is in the diff)
- [ ] Diff size < ~400 lines (split if larger)
- [ ] Linked issue / ticket if one exists
- [ ] Screenshots or recording if UI change
- [ ] Migration steps documented if breaking

### PR description template

```markdown
## What
One-paragraph summary of the change.

## Why
The motivation (bug, feature, refactor, perf). Link to issue if applicable.

## How
Key design decisions or non-obvious approaches. Skip if straightforward.

## Testing
What I did to verify (test types added, manual checks).

## Notes for reviewer
Anything specific to look at, follow-up TODOs, etc.

## Screenshots / Recording
(For UI changes)
```

---

## Reviewing Others' PRs

### Pre-review

1. **Read the PR description first.** Understand intent before judging.
2. **Run the diff in your head.** What's the actual change?
3. **Look at file count.** > 20 files = ask for split. Big PRs hide bugs.

### Review pass order

```
1. Architecture / structure: does this change fit the codebase?
2. Public API: signatures, contracts, breaking changes
3. Tests: coverage, quality, test names
4. Implementation: smells, complexity, edge cases
5. Naming and readability
6. Style nits (only if linter doesn't catch — most should be automated)
```

Don't flag style if a linter could. Don't flag preference. Flag substance.

---

## What to Look For (operational checklist)

### Correctness

- [ ] Edge cases handled? (empty, null, max size, negative, concurrent)
- [ ] Error paths covered? (network failure, validation, timeout)
- [ ] Off-by-one? Boundaries (`<` vs `<=`)?
- [ ] Race conditions in async code?
- [ ] Resource leaks? (handles, timers, listeners not cleaned up)

### Tests

- [ ] Tests exist for new behavior
- [ ] Tests use AAA structure
- [ ] Test names are concrete examples, not abstract
- [ ] No mocks for value objects (mock infrastructure only)
- [ ] No testing of private methods
- [ ] Tests fail when they should (try mentally inverting expected → does test still pass? red flag)
- [ ] No flaky tests (depend on timing, randomness, network)

### Design (SOLID + smells)

- [ ] No God class (class > 50 lines doing many things)
- [ ] No Long Method (> 10 lines)
- [ ] No Long Parameter List (> 3)
- [ ] No Primitive Obsession (raw `string`/`number` for domain concepts)
- [ ] No Switch on Type (use polymorphism)
- [ ] No Feature Envy (method using another class's data more than its own)
- [ ] No Train Wreck (`a.b.c.d` chains)
- [ ] No Speculative Generality (unused abstractions)
- [ ] No Anemic Domain (data bag + service mutating it)
- [ ] Dependencies injected, not instantiated (`new ConcreteX()` in domain code)
- [ ] No `else` after early return
- [ ] No magic numbers / strings (named constants or value objects)

### Naming

- [ ] Class is a noun
- [ ] Method is a verb (or `is`/`has`/`can` for boolean)
- [ ] No vague names (`data`, `info`, `manager`, `util`)
- [ ] Domain language, not technical jargon
- [ ] Consistent — same concept, same name throughout

### Performance (only when measured)

- [ ] No N+1 queries in loops
- [ ] No O(n²) where O(n) is feasible
- [ ] No unnecessary re-renders / re-computations (if measurable)

**Don't review performance based on hunch.** Demand measurement before flagging "this might be slow."

### Security

- [ ] User input validated (especially before DB / shell / file ops)
- [ ] No secrets in code or logs
- [ ] No raw HTML interpolation (XSS)
- [ ] SQL parameterized (no string concat)
- [ ] Auth check on every protected endpoint
- [ ] CSRF protection where needed
- [ ] Rate limiting on public endpoints

---

## Comment Patterns

### Conventional comments (https://conventionalcomments.org/)

Use prefix labels so intent is clear.

| Label | Meaning |
|-------|---------|
| `nit:` | Minor, non-blocking suggestion |
| `suggestion:` | Suggests a change; up to author |
| `issue:` | Real problem; requires fix before merge |
| `question:` | Asking for clarification |
| `praise:` | Pointing out something well-done |
| `chore:` | Routine task (e.g. update changelog) |
| `thought:` | Thinking out loud, no action required |
| `polish:` | Cosmetic improvement, non-blocking |

### Examples

```
nit: Could rename `data` to `userProfile` for clarity.

suggestion: Consider extracting this conditional into a named predicate
            (`isEligibleForDiscount(user)`) — would make the intent clearer.

issue: This will throw if `user.email` is null. Need a null check or
       value object that prevents construction with null.

question: Why is the timeout 5000ms? Is that based on observed p99,
          or is it arbitrary?

praise: This builder pattern made the test setup really readable.

issue (blocking): The auth middleware was removed from this route.
                  Was that intentional? Looks like a security regression.
```

### Tone

- **Critique the code, not the author.** "This method has high complexity" not "you wrote complex code."
- **Suggest, don't dictate.** "Consider X" leaves room for discussion. "Change to X" closes it.
- **Explain WHY.** "Use polymorphism here because adding a new payment type currently requires editing 3 files."
- **Praise good work.** It costs you nothing and improves morale.

### Conflict resolution

If you and the author disagree:

1. Ask "what's your reasoning?" — you might be missing context
2. If still disagree, state the trade-off you see
3. If still stuck, defer to the more senior or more domain-knowledgeable
4. If still stuck, escalate (tech lead, architecture decision, ADR)
5. Don't block PRs over preferences. Reserve blocking for real issues (correctness, security, big design problems).

---

## Common Smells to Catch in Review

### Hidden global state

```typescript
// Smell: function depends on something not in its signature
function processOrder(order: Order) {
  const config = ConfigSingleton.getInstance();  // hidden dep
  // ...
}
```

**Comment:**
```
issue: ConfigSingleton makes this function untestable in isolation.
       Consider injecting via parameter or constructor.
```

### Catching and swallowing errors

```typescript
try {
  await sendEmail(user);
} catch (err) {
  // silently ignored
}
```

**Comment:**
```
issue: Catch-and-swallow loses the error. Either:
       - log it (`logger.error("send email failed", { err })`)
       - rethrow as a typed error
       - return a Result indicating failure
       Silent failure is the worst option.
```

### Async without await / unhandled promise

```typescript
function process() {
  doAsyncThing();  // returns Promise but caller doesn't await
}
```

**Comment:**
```
issue: `doAsyncThing()` returns a Promise that's not awaited.
       Either `await` it or attach `.catch()` for error handling.
       Unhandled rejection will crash on Node.
```

### N+1 query

```typescript
const users = await getUsers();
for (const user of users) {
  user.orders = await getOrdersForUser(user.id);  // N+1
}
```

**Comment:**
```
issue: N+1 queries here — one query per user.
       Use a join or a batch query like `getOrdersForUsers(users.map(u => u.id))`.
```

### Mutating input

```typescript
function addTax(order: Order, rate: number) {
  order.total = order.total * (1 + rate);  // mutates caller's order
  return order;
}
```

**Comment:**
```
issue: Mutates the caller's `order`. This violates immutability and
       leads to surprising bugs. Return a new value object instead.
```

### Test paraphrases implementation

```typescript
it("should set isActive to true", () => {
  user.activate();
  expect(user.isActive).toBe(true);
});
```

**Comment:**
```
suggestion: Test name describes implementation, not behavior.
            Consider: "after activation, the user can log in"
            Test the behavior, not the property assignment.
```

---

## What NOT to Flag

| Don't flag | Reason |
|------------|--------|
| Indentation, semicolons, quotes | Linter's job |
| Variable name preference (e.g. `idx` vs `i`) | Personal preference |
| File organization unless harmful | Author's call |
| Performance "could be" issues without measurement | Speculation |
| Patterns the codebase doesn't use yet | Don't impose new convention in one PR |
| Test framework preference | Use what the project uses |
| "This isn't how I'd write it" | Not the bar |

If you find yourself flagging style heavy, the project needs a stricter linter — not more reviewer effort.

---

## PR Size Guidelines

| Lines changed | Verdict |
|---------------|---------|
| < 50 | Easy, 5-min review |
| 50-200 | Normal review (15-30 min) |
| 200-400 | Long review (30-60 min); request structure |
| 400-1000 | Hard to review well; ask for split |
| > 1000 | **Reject — request splitting** |

**Why split:** large PRs hide bugs because reviewers skim. Multiple small PRs get better review.

### How to split

- One refactor per PR (separate from feature changes)
- One concept per PR ("add Order entity" + "add OrderRepository" + "add OrderController" = 3 PRs)
- Code generation / mass renames in their own PR (clearly labeled "no logic change")

---

## When to Approve vs Request Changes

```
Block (request changes) when:
- Correctness issue
- Security issue
- Tests missing or bad
- Design issue that would be expensive to fix later
- Breaking change without docs/migration
- Public API change without RFC/discussion

Approve with comments when:
- Suggestions only (`nit:` or `suggestion:`)
- Code is correct and tested
- Concerns are minor or stylistic

Approve cleanly when:
- Clean change, well-tested, fits architecture
```

---

## Author Etiquette (when receiving review)

- **Don't take it personally.** Code review = the code, not you.
- **Ask for clarification** if a comment is unclear.
- **Acknowledge each comment** — fix, push back with reasoning, or "good point, will address in follow-up".
- **Push back when you disagree.** Reviewers can be wrong; convince with evidence.
- **Don't mass-resolve comments without addressing them.**
- **Keep PRs reviewable.** Small, focused, with clear description.

---

## Code Review Anti-Patterns

| Anti-pattern | Symptom | Fix |
|--------------|---------|-----|
| Rubber-stamp approval | "LGTM" within 2 minutes of huge PR | Demand time; split PR |
| Bikeshedding | 30 comments on style; 0 on logic | Linter + focus on substance |
| Personal attacks | Comments target the author | Critique the code |
| Block on preference | "I wouldn't do it this way" without reason | Block only on real issues |
| Pile-on after first comment | Multiple reviewers raising same issue | One reviewer per issue; resolve before next |
| Review for the resume | Lots of comments to look thorough | Quality > quantity |
| Ignoring the why | Comment changes A → B without explaining | Always explain trade-off |

---

## Quick Reference

### Reviewer checklist (5 min skim → 30 min deep dive)

1. **Skim** — file count, diff size, PR description
2. **Architecture** — does this fit?
3. **Public API** — contracts, breaking changes
4. **Tests** — coverage, quality
5. **Implementation** — smells, edge cases
6. **Naming** — clarity
7. **Style nits** — only if linter doesn't

### Comment labels (intent makes review easier)

```
nit: minor, non-blocking
suggestion: try this; up to you
issue: must fix before merge
question: clarify
praise: nice work here
```

### Block (request changes) when

- Correctness / security / data loss risk
- Tests missing or actively bad
- Design problem that would be expensive to fix later

### Don't block on

- Personal style
- "I'd write it differently" without trade-off
- Speculation without measurement

### PR size

- Aim < 400 lines
- Split when > 400
- Reject > 1000
