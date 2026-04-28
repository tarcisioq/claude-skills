# Managing Complexity

Operational rules for keeping codebases simple. Includes **metrics** with concrete thresholds — not vibes.

## Decision Matrix — Complexity Triage

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Small change touches 10+ files | Change Amplification | Consolidate responsibilities |
| Reading code requires holding 5+ classes in head | Cognitive Load | Improve naming, encapsulation |
| Change in A breaks unrelated B | Unknown Unknowns | Reduce hidden dependencies, global state |
| Method has nesting depth > 3 | Cognitive Complexity | Extract / early returns |
| Function with cyclomatic complexity > 10 | Branch explosion | Replace with polymorphism / strategy |
| Class with > 50 lines + > 2 responsibilities | God Class | Split per responsibility (SRP) |

---

## Two Types of Complexity

### Essential complexity

Inherent to the problem domain. **Cannot be removed**, only managed.

- Business rules (tax calculations, regulatory compliance)
- Domain invariants (Order can't be empty)
- User requirements (multi-currency, multi-tenant)

### Accidental complexity

Introduced by our solutions. **Should be minimized.**

- Poor abstractions (`UserManagerFactoryProvider`)
- Unnecessary indirection (callbacks calling callbacks)
- Framework ceremony (boilerplate)
- Tooling overhead
- "Just in case" abstractions

**Goal:** make essential complexity expressed as clearly as possible. Eliminate accidental complexity ruthlessly.

```typescript
// ❌ — over-engineered for hypothetical needs
abstract class UserServiceFactoryProvider<T extends UserConfig> {
  protected abstract createConfig(): T;
  static getInstance<T>(): UserServiceFactoryProvider<T> { /* ... */ }
}

// ✅ — addresses the actual essential complexity
class UserService {
  getUser(id: UserId): Promise<User | null> { /* ... */ }
}
```

---

## Complexity Metrics — Numerical Thresholds

These have **mechanical thresholds**. Use them in linters / CI / pre-commit checks.

### Cyclomatic Complexity (CC)

Measures the number of linearly independent paths through code. Roughly: count the branches.

| CC | Verdict |
|----|---------|
| 1-4 | Simple, low risk |
| 5-7 | Moderate, manageable |
| 8-10 | Complex, refactor candidate |
| 11+ | Hard to test, hard to understand. **Refactor required.** |

```typescript
// CC = 4 (1 base + 3 branches: if, else if, else)
function shippingCost(country: string, total: number): number {
  if (country === "US") return total > 50 ? 0 : 5;
  else if (country === "BR") return 15;
  else return 25;
}

// CC = 1 — polymorphism removes branches
function shippingCost(method: ShippingMethod, total: Money): Money {
  return method.cost(total);
}
```

**Tools:** `eslint-plugin-complexity`, ESLint built-in `complexity` rule.

```json
// .eslintrc.json
{ "rules": { "complexity": ["error", 10] } }
```

### Cognitive Complexity

Like CC, but weighs **nesting depth** more heavily — closer to how humans perceive difficulty.

| Cognitive | Verdict |
|-----------|---------|
| 0-5 | Simple |
| 6-10 | Manageable |
| 11-15 | Complex |
| 16+ | Refactor required |

```typescript
// Cognitive complexity ~11 — nested loops + conditions
function findValidOrders(orders: Order[]): Order[] {
  const result = [];
  for (const order of orders) {           // +1
    if (order.isActive) {                 // +2 (nested)
      for (const item of order.items) {   // +3 (nested)
        if (item.inStock) {               // +4 (nested deeper)
          if (item.price > 0) {           // +5 (nested deeper)
            result.push(order);
            break;
          }
        }
      }
    }
  }
  return result;
}

// Cognitive complexity ~3 — flat with early returns + extracted predicates
function findValidOrders(orders: Order[]): Order[] {
  return orders.filter(isValid);
}

function isValid(order: Order): boolean {
  if (!order.isActive) return false;
  return order.items.some(item => item.inStock && item.price > 0);
}
```

**Tool:** SonarQube, `eslint-plugin-sonarjs`.

### Nesting Depth

Levels of indentation inside a function.

| Depth | Verdict |
|-------|---------|
| 1 | Ideal (object calisthenics) |
| 2 | Acceptable |
| 3 | Suspect — extract |
| 4+ | Refactor required |

```typescript
// ❌ — depth 4
function process(orders: Order[]) {
  for (const order of orders) {
    if (order.isValid()) {
      for (const item of order.items) {
        if (item.inStock) {        // ← depth 4
          item.process();
        }
      }
    }
  }
}

// ✅ — depth 1, multiple methods
function process(orders: Order[]) {
  orders.filter(o => o.isValid()).forEach(processItems);
}
function processItems(order: Order) {
  order.items.filter(i => i.inStock).forEach(i => i.process());
}
```

### Method / Class / File Size

| Element | Hard limit | Suspect |
|---------|-----------|---------|
| Method | 10 lines | > 5 with branching |
| Class | 50 lines | > 30 with mixed concerns |
| File | 200 lines | > 100 with multiple classes |
| Constructor params | 3 | > 2 if not value object |
| Instance variables | 2 | > 2 in non-value-object |

**Tools:** ESLint `max-lines-per-function`, `max-lines`, `max-params`.

### Halstead Metrics (for advanced analysis)

Counts unique operators / operands. Useful for:
- **Volume** — how much information the reader must process
- **Difficulty** — how prone to errors
- **Effort** — estimated implementation time

Useful when picking refactoring targets in legacy code. Most teams don't need it day-to-day.

---

## Detecting Complexity

### 1. Change Amplification

**Symptom:** "To add a phone number field to User, I edited 15 files."

**Cause:** scattered responsibilities, leaky abstractions, missing encapsulation.

**Test:** trace a common change. Count files modified. > 5 = signal.

**Fix:** consolidate the change site. Apply DIP — abstract the boundary so changes localize.

### 2. Cognitive Load

**Symptom:** "I need to read 10 classes to understand what one of them does."

**Cause:** tight coupling, hidden dependencies, unclear naming, too many concepts in one place.

**Test:** ask a colleague to explain a class without context. If they need to ask about 5+ other classes, load is too high.

**Fix:**
- Improve naming so external behavior is obvious
- Encapsulate (hide internals via private)
- Extract value objects so types tell the story
- Compose method pattern: high-level → details

### 3. Unknown Unknowns

**Symptom:** "I changed X and something completely unrelated broke."

**Cause:** global state, hidden dependencies, side effects, implicit contracts.

**Test:** can you predict every effect of changing a method? If no, hidden coupling exists.

**Fix:**
- Eliminate globals (singletons, static state, global mutables)
- Make dependencies explicit (constructor injection)
- Document side effects in JSDoc
- Treat method calls as command-query separation: a function either DOES something (returns void, has side effects) or RETURNS something (no side effects)

---

## XP Values for Fighting Complexity

From Extreme Programming. Apply when making design decisions.

### 1. Communication

Code communicates clearly. Names, structure, tests all carry intent.

### 2. Simplicity

Simplest thing that works. Add complexity only when forced.

### 3. Feedback

Fast feedback loops catch complexity early. TDD, CI, code review, monitoring.

### 4. Courage

Refactor aggressively. Don't let smells accumulate.

### 5. Respect

Respect future readers (including yourself in 6 months). Write for humans first.

---

## KISS — Keep It Simple

**Decision tree:**

```
Is this the simplest solution that works?
├── YES → Ship it.
└── NO  ↓
    Why am I adding complexity?
    ├── Real, current requirement → keep
    ├── "We might need it later"   → REMOVE (YAGNI)
    ├── "More flexible / generic" → REMOVE if not used by 3+ scenarios
    └── "Cleaner design"           → consider — does it actually reduce essential complexity?
```

```typescript
// ❌ — over-engineered for one current use
class UserServiceFactoryProvider {
  private static instance: UserServiceFactoryProvider;
  static getInstance(): UserServiceFactoryProvider { /* ... */ }
  createFactory(): UserServiceFactory { /* ... */ }
}
class UserServiceFactory {
  create(): UserService { /* ... */ }
}

// ✅ — KISS
class UserService {
  getUser(id: UserId): Promise<User | null> { /* ... */ }
}
```

---

## YAGNI — You Aren't Gonna Need It

**Don't build features until they're needed by a real, current requirement.**

### Tells

- "We might need this someday"
- "It would be nice to have"
- "For future extensibility"
- "Just in case"
- Optional method parameters never set
- Configuration toggles never flipped
- Abstractions with one implementation

### The cost of YAGNI violations

1. **Development time** — building unused features
2. **Maintenance burden** — code that must be kept working
3. **Cognitive load** — more to understand
4. **Wrong abstraction** — guessing future needs incorrectly (often wrong)
5. **Testing burden** — paths that have to be covered

```typescript
// ❌ — fields "in case we need them"
class User {
  middleName?: string;
  secondaryEmail?: string;
  faxNumber?: string;
  linkedinProfile?: string;
  twitterHandle?: string;
  addressLineThree?: string;
}

// ✅ — only what's needed NOW
class User {
  constructor(
    private readonly id: UserId,
    private email: Email,
    private name: Name,
  ) {}
}
// Add fields when concrete requirement arrives.
```

---

## DRY — Don't Repeat Yourself (with the Rule of Three)

> "Every piece of knowledge should have a single, unambiguous representation."

**BUT:** apply only after the **third** occurrence. Wrong abstraction is worse than duplication.

```
Duplication #1 → leave it
Duplication #2 → note it, leave it
Duplication #3 → NOW extract — abstraction has 3 examples to fit
```

### Why wait?

With 1 or 2 examples, you don't yet see what's *essential* vs *incidental*. Extracting at #1 locks in your guess.

```typescript
// First — just write it
function processUserOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
}

// Second — note similarity, leave it
function processGuestOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
  emailGuestReceipt(o);
}

// Third — NOW you see the pattern
function processCorporateOrder(o: Order) {
  validate(o); calculateTotal(o); save(o);
  applyBulkDiscount(o);
}

// Extract with confidence — 3 examples to validate the abstraction
function processOrder(o: Order, finalize: (o: Order) => void) {
  validate(o); calculateTotal(o); save(o);
  finalize(o);
}
```

### The DRY misapplication

DRY is about KNOWLEDGE, not code. Two pieces of code that look identical but represent different concepts should NOT be DRY'd.

```typescript
// ❌ — these LOOK the same but represent different concepts
const taxRate = 0.07;
const commissionRate = 0.07;
// DRYing into one constant `RATE = 0.07` couples unrelated business rules.

// ✅ — keep them separate; they CAN diverge.
const TAX_RATE = 0.07;
const COMMISSION_RATE = 0.07;
```

---

## Separation of Concerns

> "Each module should address a single concern."

### Concerns to separate

- **Business logic** vs **infrastructure** (DB / HTTP / FS)
- **What** (policy) vs **how** (mechanism)
- **Input** vs **processing** vs **output**
- **Data** vs **behavior** (sometimes — see anemic-vs-rich-model debate)
- **Stable** vs **volatile** (stable interfaces, volatile implementations)

### Example

```typescript
// ❌ — concerns mixed
class OrderProcessor {
  process(order: Order) {
    // VALIDATION
    if (!order.items.length) throw new Error("empty");

    // CALCULATION
    let total = 0;
    for (const item of order.items) total += item.price * item.quantity;

    // PERSISTENCE
    const db = new Database();
    db.query(`INSERT INTO orders ...`);

    // NOTIFICATION
    new EmailClient().send(order.customer.email, "Confirmed");
  }
}

// ✅ — concerns separated, dependencies injected
class OrderProcessor {
  constructor(
    private readonly validator: OrderValidator,
    private readonly calculator: OrderCalculator,
    private readonly repo: OrderRepository,
    private readonly notifier: OrderNotifier,
  ) {}

  process(order: Order): ProcessResult {
    this.validator.validate(order);
    const total = this.calculator.calculate(order);
    const saved = this.repo.save(order);
    this.notifier.notifyConfirmation(saved);
    return ProcessResult.success(saved);
  }
}
```

---

## Code Hotspots & Churn Analysis

Identify high-priority refactor targets via **commit history + complexity**.

### Hotspot detection

```bash
# Files with most commits in last 12 months
git log --since="12 months ago" --name-only --pretty=format: \
  | sort | uniq -c | sort -rn | head -20
```

### Churn × Complexity = Risk

A file with high CC AND high churn = **highest risk**. Refactor priority order:

```
1. High churn + high complexity  → URGENT — actively decaying
2. Low churn  + high complexity → SCHEDULE — works but fragile
3. High churn + low complexity  → MONITOR — well-managed change
4. Low churn  + low complexity  → IGNORE — leave alone
```

**Tools:**
- `code-maat` (Adam Tornhill's tool from "Your Code as a Crime Scene")
- `git-quick-stats`
- Custom scripts combining `git log` + `cloc` + complexity tools

### Output example

```
File                          Commits  Complexity  Risk
order/OrderProcessor.ts       45       28          ⚠ URGENT
user/UserService.ts           12       30          ⚠ SCHEDULE
billing/StripeAdapter.ts      40       8           OK
domain/Money.ts               15       4           OK
```

The first one has been changing constantly AND is complex → next refactor target.

---

## Technical Debt Management

### Types of debt

1. **Deliberate** — conscious tradeoff for speed (documented, tracked)
2. **Accidental** — mistakes, incomplete knowledge (discovered later)
3. **Bit rot** — code degrades as ecosystem moves (deps, language, runtime)

### Boy Scout Rule

> "Leave the code better than you found it."

Every commit:
- Improve one name
- Extract one method
- Add one missing test
- Delete one dead piece

Small, continuous improvements compound.

### When to pay down debt

| Situation | Action |
|-----------|--------|
| Debt is in your path right now | Refactor while you're there (Boy Scout) |
| Debt blocks new feature | Pay it down BEFORE feature work |
| Debt causes recurring bugs | Schedule dedicated session |
| Hot debt (frequently changed area) | Prioritize per hotspot analysis |
| Cold debt (rarely touched) | Leave it; focus elsewhere |

### When NOT to refactor

- Code with no test coverage (add tests first — see `legacy-code.md`)
- Code being deleted soon (waste of effort)
- Code outside your team's ownership (without coordination)
- Friday afternoon before deploy (don't ship untested refactor)

---

## The Four Elements of Simple Design

From XP. In **priority order**:

1. **Runs all the tests** — correctness first
2. **Expresses intent** — clear names, story structure
3. **No duplication** — DRY (after Rule of Three)
4. **Minimal** — fewest classes/methods possible

If all four hold, the design is simple enough.

```typescript
// ❌ — fails #2 (intent unclear) and #4 (over-engineered)
class XYZManager {
  do(d: any): any { /* ... */ }
}

// ✅ — passes all four
class OrderProcessor {
  process(order: Order): ProcessResult { /* ... */ }
}
```

---

## Quick Reference — Numerical Thresholds

| Metric | Hard limit | Suspect |
|--------|-----------|---------|
| Cyclomatic complexity | 10 | > 7 |
| Cognitive complexity | 15 | > 10 |
| Nesting depth | 2 | > 1 |
| Method lines | 10 | > 5 with branching |
| Class lines | 50 | > 30 |
| File lines | 200 | > 100 |
| Constructor params | 3 | > 2 (non-VO) |
| Instance variables | 2 | > 2 (non-VO) |
| Public methods | 5 | > 3 (non-VO) |
| Method calls in chain (Demeter) | 1 | > 1 |

**Note:** thresholds are heuristics, not laws. Use judgment. Value objects can have many small accessors and stay simple. Domain logic should be ruthlessly minimized.

---

## Quick Reference — Decision Cheatsheet

| Situation | Apply |
|-----------|-------|
| New code | KISS first, YAGNI everywhere |
| Duplication | Wait for Rule of Three |
| "We might need it" | NO. Add when needed. |
| Method > 10 lines | Extract |
| > 3 params | Parameter object |
| Nesting > 2 | Extract or early return |
| Refactor planning | Hotspot analysis (churn × complexity) |
| Existing tests | Boy Scout — improve while passing |
| No tests | Add characterization tests first |
| In rush, can't refactor | Document as TODO with issue link |
