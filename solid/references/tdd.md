# Test-Driven Development

TDD operational reference. Goes beyond Red-Green-Refactor — covers the actual decisions you face: London vs Detroit, double-loop, mock vs fake, test naming, when to NOT TDD.

## Decision Tree — TDD Right Now?

```
Is this throwaway / spike / one-off script?
├── YES → No TDD. Ship procedural code, throw away if needed.
└── NO  ↓
    Does test infrastructure exist in this codebase?
    ├── NO  → First task: bootstrap minimal test runner. Then return.
    └── YES ↓
        Am I fixing a bug?
        ├── YES → Write failing test that REPRODUCES the bug. Then fix. Test stays.
        └── NO  ↓
            Is the behavior trivial (< 3 lines, no branches, no I/O)?
            ├── YES → Optional. Inline test if framework allows.
            └── NO  → TDD. Red-Green-Refactor.
```

## The Core Loop

```
RED ──→ GREEN ──→ REFACTOR ──→ RED → ...
 ↑                                   │
 └───────────────────────────────────┘
```

### RED — write a failing test

- Concrete example, not abstract: `"applies 20% discount to premium users"` not `"can apply discount"`
- Domain language, not technical: `"recognizes 'racecar' as palindrome"` not `"sets isPalindrome to true"`
- One behavior per test
- Test must FAIL for the right reason (compilation failure counts as failing)

```typescript
// ✅ Concrete + domain language
it("applies 20% discount to premium users", () => {
  // ...
});

// ❌ Abstract, technical
it("can apply discount", () => { /* ... */ });
it("should set the discount property", () => { /* ... */ });
```

### GREEN — simplest code that passes

Two strategies:

**Fake It** (preferred when learning the shape):
```typescript
function discountFor(user: User): Percent {
  return Percent.of(20); // hardcoded — let next test force generalization
}
```

**Obvious Implementation** (when solution is trivial):
```typescript
function discountFor(user: User): Percent {
  return user.isPremium ? Percent.of(20) : Percent.zero();
}
```

**Triangulation:** add a second test that forces the variation:
```typescript
it("applies 0% discount to standard users", () => { /* ... */ });
// Now you can't hardcode 20 anymore — must generalize
```

### REFACTOR — design happens here

This is where SOLID applies. Look at:

- Duplication (apply Rule of Three)
- Method size (> 10 lines → extract)
- Naming clarity
- Conditional complexity (replace with polymorphism)
- Primitive obsession (extract value object)

**Tests must stay green throughout refactor.** If a refactor breaks tests, you broke behavior — undo and approach differently.

## The Three Laws of TDD (Bob Martin)

1. **No production code** without a failing test
2. **No more test code** than is sufficient to fail (compilation failure counts)
3. **No more production code** than is sufficient to pass the failing test

These laws guarantee that **every line of production code was first justified by a test**. If you can comment out a line and tests still pass, the line was either dead or undertested.

## London (Mockist) vs Detroit (Classic) Schools

There are two TDD styles. They produce different code. **Choose intentionally.**

### Detroit / Classic / Chicago

- **Test with real objects.** Mock only at infrastructure boundaries (DB, HTTP, filesystem).
- **Test outside-in or inside-out** — usually inside-out (build small, compose).
- **Tests are coupled to behavior, not interaction.**
- **Refactoring rarely breaks tests.**

```typescript
// Detroit: real Money, real DiscountPolicy, only the email gateway is faked
const order = new Order();
order.add(new OrderItem(Money.dollars(100)));
const total = order.calculateTotal(new PremiumDiscount());
expect(total.equals(Money.dollars(80))).toBe(true);
```

### London / Mockist

- **Mock all collaborators.** Test only the unit under test.
- **Outside-in design** — start with the highest-level test, mock everything below, drill down.
- **Tests verify interactions** (which methods called, with what args).
- **Refactoring often breaks tests** because they're coupled to call patterns.

```typescript
// London: every collaborator is mocked
const policy = mock<DiscountPolicy>();
policy.apply.mockReturnValue(Money.dollars(80));

const order = new Order(policy); // policy is injected
order.calculateTotal();

expect(policy.apply).toHaveBeenCalledWith(Money.dollars(100));
```

### Decision tree — which school?

```
Is this domain code (entities, value objects, pure logic)?
├── YES → Detroit. Use real objects. Tests will be fast and stable.
└── NO  ↓
    Does the unit orchestrate multiple collaborators with interaction protocol that matters?
    ├── YES → London. Mock collaborators; verify interactions.
    └── NO  ↓
        Is the unit at an architecture boundary (controller, gateway adapter)?
        ├── YES → London. The whole point is the protocol.
        └── NO  → Detroit. Tests stay closer to behavior.
```

**Default: Detroit for domain, London for application/orchestration layers.**

## Outside-In vs Inside-Out

Two ways to start a feature.

### Inside-Out (build up)

Start with smallest pieces, compose into bigger features.

```
1. Test + implement Money         (value object)
2. Test + implement OrderItem     (value object)
3. Test + implement Order         (entity using above)
4. Test + implement OrderService  (use case orchestrating)
```

**Good for:** domain logic, when you understand the model. Risk: building parts you don't end up needing.

### Outside-In (drive in)

Start with end-to-end test (acceptance test). Drill down to units.

```
1. Acceptance test: "user can place order via API"  ← FAIL (nothing exists)
2. Stub OrderController to return canned response   ← acceptance still fails (returns wrong thing)
3. Add OrderService with mocked repository          ← unit-level TDD inside
4. Implement repository                             ← drill all the way down
5. Acceptance test passes                           ← all unit tests still passing
```

**Good for:** features where the boundary contract is the priority (APIs, UI flows). Drives toward real customer value.

### Double-Loop TDD (the ATDD pattern)

```
┌─ Outer loop (acceptance test) ─────────────────┐
│   RED                                          │
│   │                                            │
│   ↓                                            │
│  ┌─ Inner loop (unit tests) ─┐                │
│  │ RED → GREEN → REFACTOR    │  ← repeat until │
│  │                           │     outer test  │
│  │ RED → GREEN → REFACTOR    │     can pass    │
│  └───────────────────────────┘                │
│   │                                            │
│   ↓                                            │
│   GREEN (acceptance)                           │
└────────────────────────────────────────────────┘
```

The outer loop is one acceptance test that stays RED until you've built enough units to satisfy it. The inner loop is normal Red-Green-Refactor. **This is the canonical TDD pattern for new features.**

## The Rule of Three (when to extract duplication)

```
Duplication #1 → leave it alone
Duplication #2 → note it, leave it (premature DRY is dangerous)
Duplication #3 → NOW extract (the abstraction now has 3 examples to fit)
```

**Why:** wrong abstraction is worse than duplication. With only 1 or 2 examples, you don't yet know which parts are essential vs incidental. Wait for 3.

```typescript
// Use #1 — leave it
function processUserOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
}

// Use #2 — note (TODO maybe), leave
function processGuestOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
  emailGuestReceipt(o);
}

// Use #3 — NOW extract; the abstraction has 3 examples to fit
function processBusinessOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
  applyBulkDiscount(o);
}

// Refactored
function processOrder(o: Order, finalize: (o: Order) => void) {
  validate(o); calculateTotal(o); save(o);
  finalize(o);
}
```

## Triangulation

Each new test "sculpts" the implementation toward generality.

```typescript
// Test 1
it("returns 0 for negative price", () => {
  expect(price(-1)).toBe(0);
});
// Simplest impl: function price(_x) { return 0; }

// Test 2 — forces generalization
it("returns price for positive number", () => {
  expect(price(10)).toBe(10);
});
// Now: function price(x) { return x < 0 ? 0 : x; }

// Test 3 — forces another dimension
it("rounds to 2 decimals", () => {
  expect(price(10.123)).toBe(10.12);
});
```

Each test removes a "degree of freedom" until the implementation handles all cases.

## Transformation Priority Premise

When going from RED to GREEN, prefer simpler transformations:

| Priority | Transformation | Example |
|----------|----------------|---------|
| 1 | `{}` → `nil` | empty body → return null |
| 2 | `nil` → constant | return null → return 0 |
| 3 | constant → variable | return 0 → return x |
| 4 | unconditional → conditional | return x → if (..) return x else return y |
| 5 | scalar → collection | x → [x] |
| 6 | statement → recursion | direct → recursive call |
| 7 | value → mutated value | immutable → mutation |

Lower priority = simpler transformation. **Avoid jumping to a higher-priority transformation until forced** — it's evidence you're solving more than the test asks.

## Arrange-Act-Assert (AAA)

Every test follows this shape:

```typescript
it("applies 20% discount to premium users", () => {
  // ARRANGE — set up world
  const user = aUser().premium().build();
  const cart = new Cart(user);
  cart.add(Money.dollars(100));

  // ACT — execute the behavior under test (one action)
  const total = cart.calculateTotal();

  // ASSERT — verify outcome (preferably one)
  expect(total.equals(Money.dollars(80))).toBe(true);
});
```

**Rules:**
- One ACT per test (multiple assertions are fine if all about the same act)
- Empty lines separating the three sections
- ARRANGE often the longest; ACT one line; ASSERT one or few

### Writing AAA backwards

When stuck, write reverse:
1. **ASSERT first** — what outcome do I want to verify?
2. **ACT** — what action produces that outcome?
3. **ARRANGE** — what setup does the act need?

This forces you to clarify the behavior before the implementation.

## Test Naming Patterns

```typescript
// Pattern 1: when_X_then_Y
it("when adding 2 + 3, returns 5", () => {});

// Pattern 2: should + behavior + context
it("should apply 20% discount when user is premium", () => {});

// Pattern 3: subject_verb_object (BDD-style describe blocks)
describe("Cart", () => {
  describe("when user is premium", () => {
    it("applies 20% discount to total", () => {});
    it("includes free shipping", () => {});
  });
});

// Pattern 4: domain example
it("recognizes 'racecar' as a palindrome", () => {});
it("rejects empty string as palindrome", () => {});
```

**Anti-patterns:**

```typescript
// ❌ — implementation-focused
it("should set isPalindrome to true", () => {});

// ❌ — abstract; what does "work" mean?
it("should work", () => {});

// ❌ — vague conditions
it("handles edge case", () => {});

// ❌ — repeating the function name without context
it("addUser test 1", () => {});
```

## Test Doubles — Decision Tree

| Need | Use | Example |
|------|-----|---------|
| Object passed but never used | **Dummy** | `{} as Logger` |
| Returns canned values | **Stub** | `repo.findById = () => Promise.resolve(user)` |
| Working in-memory implementation | **Fake** | `class InMemoryUserRepo implements UserRepo` |
| Records calls for verification | **Spy** | `jest.spyOn(repo, "save")` |
| Verifies interaction (call count, args) | **Mock** | `expect(spy).toHaveBeenCalledWith(...)` |

### When to use which (decision tree)

```
Am I testing a value object or pure logic?
├── YES → Use real objects. No doubles.
└── NO  ↓
    Am I crossing an infrastructure boundary (DB, HTTP, FS, time)?
    ├── YES ↓
    │   Will I write multiple tests against this dependency?
    │   ├── YES → Build a Fake (InMemoryX) — reusable across tests
    │   └── NO  → Stub for this single test
    └── NO  ↓
        Is the interaction itself the contract under test (e.g. "must call email service")?
        ├── YES → Mock — verify the interaction
        └── NO  → Use real object
```

**Strong default: build Fakes.** Fakes are real implementations with a simpler backing (in-memory). They give you confidence (real code paths) and speed (no I/O).

```typescript
// ✅ Fake — full UserRepo contract, in-memory
class InMemoryUserRepo implements UserRepo {
  private users = new Map<string, User>();
  async save(user: User): Promise<void> { this.users.set(user.id.toString(), user); }
  async findById(id: UserId): Promise<User | null> { return this.users.get(id.toString()) ?? null; }
}

// Use Fake in tests instead of stubs/mocks
const repo = new InMemoryUserRepo();
const service = new UserService(repo);
```

### Mock and Fake — the famous pitfalls

- **Mocking value objects** → mock infrastructure, never `Money` or `Email`
- **Mocking what you don't own** → wrap third-party libraries in your own interface, mock that
- **Over-mocking** → if a test has 6 mocks, the unit-under-test is doing too much
- **Mocking concrete classes** → forces tight coupling; mock interfaces

## Test Builders

Make complex object creation in tests cheap.

```typescript
// ✅ Builder pattern — defaults + selective overrides
class UserBuilder {
  private email = Email.of("default@example.com");
  private age = Age.of(30);
  private premium = false;
  private name = "Test User";

  withEmail(e: string): this { this.email = Email.of(e); return this; }
  withAge(a: number): this { this.age = Age.of(a); return this; }
  premiumUser(): this { this.premium = true; return this; }

  build(): User {
    return new User(this.email, this.age, this.name, this.premium);
  }
}

// Convenient factory function
function aUser(): UserBuilder { return new UserBuilder(); }

// Usage
const user = aUser().premiumUser().withAge(25).build();
const guest = aUser().build();
const admin = aUser().withEmail("admin@x.com").build();
```

**Why:** as objects evolve, only the builder needs updates — not 100 test files.

## Property-Based Testing

Beyond example-based tests. Generate random inputs against properties.

```typescript
import fc from "fast-check";

it("addition is commutative", () => {
  fc.assert(fc.property(
    fc.integer(), fc.integer(),
    (a, b) => add(a, b) === add(b, a)
  ));
});

it("Money.add never produces negative when both positive", () => {
  fc.assert(fc.property(
    fc.nat({ max: 1_000_000 }), fc.nat({ max: 1_000_000 }),
    (a, b) => Money.cents(a).add(Money.cents(b)).cents() >= 0
  ));
});
```

**When to use:** mathematical properties, invariants, round-trip serialization, parsing edge cases. Catches inputs you didn't think of.

**When NOT:** business rules tied to specific examples (more than one valid answer per input).

## Mutation Testing

Verify your tests actually catch bugs by introducing them deliberately.

```bash
# Stryker mutator — TypeScript/JavaScript
npx stryker run
```

Stryker mutates your code (changes `>` to `>=`, removes `!`, replaces `+` with `-`, etc.) and runs the test suite. **A mutant that survives = a test that didn't catch the change** — your test coverage is weaker than you thought.

**Score interpretation:**
- > 80% → solid suite
- 60-80% → meaningful gaps
- < 60% → tests primarily verify happy path; many edge cases uncovered

Run on critical domain code, not the whole codebase.

## When NOT to TDD

| Context | Reason |
|---------|--------|
| Spike / exploration | You're learning the shape; tests would lock you in |
| Throwaway scripts | Cost > benefit |
| UI tweaking (visual) | Visual feedback IS the test loop |
| Truly trivial: getters, simple wiring | Test would just paraphrase the code |
| Glue code (3 lines wiring two libs) | Nothing to verify beyond integration |
| Performance experiments | Tests measure behavior, not perf |

**For everything else: TDD.** Especially for: business rules, validation, state transitions, financial calculations, parsing, anything with branching logic.

## Common TDD Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Writing code before test | "I'll add a test later" | Stop. Revert. Test first. |
| Writing too much test | Test asserts 5 things at once | Split into multiple tests |
| Writing too much code | Implementation handles cases the test doesn't cover | Delete the extra code; add tests if cases matter |
| Skipping refactor | "It works, ship it" | Always refactor before next RED — design happens here |
| Testing implementation | Test breaks on rename / extract | Test behavior via public API |
| Abstract test names | `"works"`, `"handles edge case"` | Concrete examples |
| Premature extraction | DRY at duplication 1 | Wait for Rule of Three |
| Mocking value objects | Mock for `Money` | Use real value object |
| Testing private methods | Brittle | Test only via public API |
| Slow tests | Suite > 30s for unit layer | Find I/O leaks; replace with fakes |

## Quick Reference

| Concern | Pattern |
|---------|---------|
| Test name | Concrete + domain language: `"applies 20% discount to premium users"` |
| Structure | Arrange-Act-Assert with blank lines separating |
| Order | RED → GREEN → REFACTOR (never skip the third) |
| Generalization | Triangulate — add tests that force variation |
| Duplication | Rule of Three — extract on 3rd, not 1st |
| Domain logic | Detroit (real objects) |
| Orchestration / boundaries | London (mock collaborators) |
| New feature | Double-loop: outer acceptance + inner unit cycle |
| Infrastructure dep | Build a Fake; reuse across tests |
| Test data setup | Builder pattern with `aUser().premium().build()` |
| Coverage of cases | Property-based for math/invariants; example-based for business rules |
| Test quality | Mutation testing — find mutants that survive |
| Don't TDD when | Spike, throwaway, trivial getters, visual UI tweak |
