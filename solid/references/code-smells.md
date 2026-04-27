# Code Smells & Anti-Patterns

Operational catalog of code smells. Each smell has:

1. **Mechanical tell** — observable trigger (line count, syntactic pattern, structural shape)
2. **Why it matters** — what problem it predicts
3. **Refactor recipe** — name + when to use (full step-by-step in `refactoring-catalogue.md`)
4. **Code example** — `❌` smell vs `✅` fixed

## Smell Detection Decision Matrix

| Tell observed | Likely smell | Refactor |
|---------------|--------------|----------|
| Method > 10 lines | Long Method | Extract Method |
| Class > 50 lines | Large Class / God Class | Extract Class |
| Function with > 3 params | Long Parameter List | Introduce Parameter Object |
| `if/else` chain on type field | Switch on Type | Replace Conditional with Polymorphism |
| `string` / `number` for domain concept | Primitive Obsession | Replace Primitive with Value Object |
| Same group of params in many places | Data Clumps | Extract Class |
| Method uses other class's data more than its own | Feature Envy | Move Method |
| `a.b.c.d` chains | Train Wreck (Demeter) | Hide Delegate |
| `instanceof` checks in callers | LSP violation | Replace Inheritance with Composition |
| Empty methods / `throw "not implemented"` | Refused Bequest / Speculative | Inline / delete |
| Comment paraphrases code | Self-explanatory failure | Rename / Extract Method |
| Class with only getters/setters | Anemic Domain Model | Move behavior to entity |
| `else` block | Missing early return | Replace with Guard Clause |
| `boolean` parameter | Flag Argument | Split into named methods |

---

## The Five Categories

### 1. Bloaters — code grown too large

| Smell | Tell | Refactor |
|-------|------|----------|
| Long Method | > 10 lines | Extract Method |
| Large Class | > 50 lines / > 2 instance vars | Extract Class |
| Long Parameter List | > 3 params | Introduce Parameter Object |
| Data Clumps | Same params appear together repeatedly | Extract Class |
| Primitive Obsession | Domain concept as `string`/`number` | Replace with Value Object |

### 2. OO Abusers — misuse of OO mechanisms

| Smell | Tell | Refactor |
|-------|------|----------|
| Switch on Type | `if`-chain or `switch` on `type`/`kind` field | Replace Conditional with Polymorphism |
| Refused Bequest | Subclass overrides parent with empty/throw | Replace Inheritance with Composition |
| Parallel Inheritance | New subclass requires another parallel subclass | Merge Hierarchies |
| Alternative Classes Different Interfaces | Same concept, different APIs | Rename + Extract Superclass |

### 3. Change Preventers — make change hard

| Smell | Tell | Refactor |
|-------|------|----------|
| Divergent Change | One class changes for many reasons | Extract Class (per reason) |
| Shotgun Surgery | One change touches many classes | Move Method/Field together |
| Parallel Inheritance | (see above) | Merge Hierarchies |

### 4. Dispensables — code to remove

| Smell | Tell | Refactor |
|-------|------|----------|
| Comments | Comment paraphrases the next line | Delete + Rename |
| Duplicate Code | Same lines in 3+ places (Rule of Three) | Extract Method/Class |
| Dead Code | Unreachable / never called | Delete |
| Speculative Generality | Unused parameters, abstract base for one impl | Inline / Delete |
| Lazy Class | Class adds no value beyond its single method | Inline Class |

### 5. Couplers — excessive coupling

| Smell | Tell | Refactor |
|-------|------|----------|
| Feature Envy | Method calls > 2 methods on another object | Move Method |
| Inappropriate Intimacy | Class accesses another's private state | Move Method / Extract Class |
| Message Chains | `a.b.c.d` access chains | Hide Delegate |
| Middle Man | Class only delegates to another | Inline Class |

---

## Top Smells — Detailed

Each entry: tell → why → recipe → example.

### Long Method

**Tell:** method body > 10 lines, OR method name needs "and" to describe.

**Why:** mixed abstraction levels; hard to test isolated parts; hard to name precisely.

**Recipe:** Extract Method per cohesive block.

```typescript
// ❌ — 25 lines, four concerns mixed
function processOrder(order: Order) {
  // Validate
  if (!order.items.length) throw new Error("empty");
  if (!order.customer) throw new Error("no customer");
  if (order.total > 10_000) throw new Error("too large");

  // Calculate
  let total = 0;
  for (const item of order.items) {
    total += item.price * item.quantity;
    if (item.discount) total -= item.discount;
  }

  // Apply tax
  const taxRate = getTaxRate(order.customer.state);
  total = total * (1 + taxRate);

  // Save
  db.orders.insert({ ...order, total });

  // Notify
  emailService.send(order.customer.email, "Order confirmed");
}

// ✅ — composed of single-concern methods; reads like a story
function processOrder(order: Order) {
  validateOrder(order);
  const total = calculateTotalWithTax(order);
  saveOrder(order, total);
  notifyCustomer(order);
}
```

### Large Class / God Class

**Tell:** class > 50 lines OR > 2 instance variables OR methods span 2+ verb domains.

**Why:** multiple responsibilities; hard to test; hard to change without breaking unrelated features.

**Recipe:** Extract Class per verb domain.

```typescript
// ❌ — User has identity + auth + preferences + notifications + billing
class User {
  name: string;
  email: string;
  passwordHash: string;
  theme: string;
  language: string;

  login(password: string) {}
  logout() {}
  resetPassword() {}

  setTheme(t: string) {}
  setLanguage(l: string) {}

  sendEmail(msg: string) {}
  sendSMS(msg: string) {}

  charge(amount: number) {}
  refund(txId: string) {}
}

// ✅ — split per verb domain; User keeps only identity
class User {
  constructor(
    private readonly id: UserId,
    private email: Email,
  ) {}
}

class AuthService {
  login(credentials: Credentials): AuthResult { /* ... */ }
  logout(session: Session): void { /* ... */ }
  resetPassword(email: Email): void { /* ... */ }
}

class UserPreferences {
  setTheme(theme: Theme): void { /* ... */ }
  setLanguage(lang: Language): void { /* ... */ }
}

class NotificationService {
  send(channel: Channel, msg: Message): void { /* ... */ }
}

class BillingService {
  charge(user: UserId, amount: Money): ChargeResult { /* ... */ }
  refund(transactionId: TransactionId): RefundResult { /* ... */ }
}
```

### Long Parameter List

**Tell:** function with > 3 parameters.

**Why:** parameter ordering becomes fragile; meaning hidden by position; adding new params breaks every call site.

**Recipe:** Introduce Parameter Object.

```typescript
// ❌
function createOrder(
  customerId: string,
  shippingAddress: Address,
  billingAddress: Address,
  paymentMethod: string,
  discountCode: string | null,
  notes: string,
  rushDelivery: boolean,
) { /* ... */ }

// Caller
createOrder("cust-1", addr1, addr2, "stripe", null, "", false); // unreadable

// ✅
class CreateOrderInput {
  constructor(
    readonly customer: Customer,
    readonly addresses: { shipping: Address; billing: Address },
    readonly payment: PaymentMethod,
    readonly options: { discount?: DiscountCode; notes?: string; rush?: boolean },
  ) {}
}

function createOrder(input: CreateOrderInput) { /* ... */ }
```

### Primitive Obsession

**Tell:** `string`, `number`, `boolean` for domain concepts (id, email, money, date, status, country, etc.).

**Why:** validation scattered across callsites; invalid states representable; type system can't help.

**Recipe:** Replace Primitive with Value Object.

```typescript
// ❌ — primitives everywhere; validation scattered
function transfer(from: string, to: string, amount: number, currency: string) {
  if (!from.match(/^acc_/)) throw new Error("invalid from");
  if (!to.match(/^acc_/)) throw new Error("invalid to");
  if (amount <= 0) throw new Error("invalid amount");
  if (!["USD", "EUR", "BRL"].includes(currency)) throw new Error("invalid currency");
  // ... actual transfer logic, mixed with re-validation
}

// ✅ — invalid states impossible by construction
class AccountId {
  constructor(private readonly value: string) {
    if (!value.match(/^acc_[a-z0-9]+$/)) throw new InvalidAccountId(value);
  }
  toString(): string { return this.value; }
  equals(other: AccountId): boolean { return this.value === other.value; }
}

class Money {
  constructor(
    private readonly amountCents: number,
    private readonly currency: Currency,
  ) {
    if (amountCents < 0 || !Number.isInteger(amountCents)) throw new InvalidAmount();
  }
}

class Currency {
  private static readonly ALLOWED = ["USD", "EUR", "BRL"] as const;
  constructor(private readonly code: string) {
    if (!Currency.ALLOWED.includes(code as any)) throw new InvalidCurrency(code);
  }
}

function transfer(from: AccountId, to: AccountId, amount: Money) {
  // Validation already happened. Pure business logic here.
}
```

### Switch on Type

**Tell:** `if/else` chain or `switch` discriminating on a `type`/`kind`/`category` field.

**Why:** OCP violation — adding a new variant requires editing every switch. Often duplicated across the codebase.

**Recipe:** Replace Conditional with Polymorphism.

```typescript
// ❌ — every new shape requires editing all switches
type Shape = { type: "circle"; radius: number }
           | { type: "square"; side: number }
           | { type: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.type) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
    case "triangle": return 0.5 * shape.base * shape.height;
  }
}

function perimeter(shape: Shape): number {
  switch (shape.type) {                          // duplicate switch!
    case "circle": return 2 * Math.PI * shape.radius;
    case "square": return 4 * shape.side;
    case "triangle": return /* approx */ 0;
  }
}

// ✅ — polymorphism; new shape = new class, no edits to existing
interface Shape {
  area(): number;
  perimeter(): number;
}

class Circle implements Shape {
  constructor(private readonly radius: number) {}
  area(): number { return Math.PI * this.radius ** 2; }
  perimeter(): number { return 2 * Math.PI * this.radius; }
}

class Square implements Shape {
  constructor(private readonly side: number) {}
  area(): number { return this.side ** 2; }
  perimeter(): number { return 4 * this.side; }
}

class Triangle implements Shape {
  constructor(private readonly base: number, private readonly height: number) {}
  area(): number { return 0.5 * this.base * this.height; }
  perimeter(): number { /* approx */ return 0; }
}
```

**Exception:** truly fixed enumerations (HTTP methods, days of week) where you'll never add variants. Polymorphism is overkill — a switch is fine.

### Feature Envy

**Tell:** method makes more calls on another object than on its own; reads >2 fields of another class.

**Why:** the method belongs in the OTHER class. Indicates wrong responsibility allocation.

**Recipe:** Move Method.

```typescript
// ❌ — Order keeps poking Customer's data
class Order {
  calculateShipping(customer: Customer): Money {
    if (customer.country === "US") {
      if (customer.state === "CA") return Money.dollars(10);
      if (customer.state === "NY") return Money.dollars(12);
      return Money.dollars(15);
    }
    if (customer.country === "BR") return Money.dollars(25);
    return Money.dollars(40);
  }
}

// ✅ — Customer knows its own shipping cost
class Customer {
  shippingCostFor(orderTotal: Money): Money {
    if (this.address.country.equals(Country.US)) {
      if (this.address.state.equals(State.CA)) return Money.dollars(10);
      if (this.address.state.equals(State.NY)) return Money.dollars(12);
      return Money.dollars(15);
    }
    if (this.address.country.equals(Country.BR)) return Money.dollars(25);
    return Money.dollars(40);
  }
}

class Order {
  calculateShipping(): Money {
    return this.customer.shippingCostFor(this.total());
  }
}
```

### Data Clumps

**Tell:** the same group of variables (3+) appears together in multiple places (params, fields, etc.).

**Why:** the group is a missing concept that wants to be a class.

**Recipe:** Extract Class.

```typescript
// ❌ — street/city/zip/country always appear together
function createUser(name: string, street: string, city: string, zip: string, country: string) {}
function shipPackage(orderId: string, street: string, city: string, zip: string, country: string) {}
function validateAddress(street: string, city: string, zip: string, country: string) {}

// ✅ — Address is the missing concept
class Address {
  constructor(
    readonly street: Street,
    readonly city: City,
    readonly zip: Zip,
    readonly country: Country,
  ) {}
  isValid(): boolean { /* ... */ return true; }
}

function createUser(name: Name, address: Address) {}
function shipPackage(orderId: OrderId, to: Address) {}
```

### Inappropriate Intimacy

**Tell:** class A reaches into class B's internal data; A's tests break when B's internals change.

**Why:** breaks encapsulation; couples internals; "Tell, Don't Ask" violation.

**Recipe:** Move Method or Extract Class. Make B do the work.

```typescript
// ❌ — Order pokes Inventory's private structure
class Order {
  process(inventory: Inventory) {
    for (const item of this.items) {
      const stock = inventory.stockLevels[item.sku];
      if (stock.quantity < item.quantity) throw new Error("out of stock");
      inventory.stockLevels[item.sku].quantity -= item.quantity;
    }
  }
}

// ✅ — Tell, don't ask. Inventory manages its own state.
class Inventory {
  reserve(items: ReadonlyArray<OrderItem>): ReserveResult {
    for (const item of items) {
      if (!this.canReserve(item)) return ReserveResult.outOfStock(item);
    }
    this.deductStock(items);
    return ReserveResult.success();
  }
}

class Order {
  process(inventory: Inventory): ProcessResult {
    const result = inventory.reserve(this.items);
    return result.isSuccess() ? ProcessResult.ok() : ProcessResult.failed(result.failedItem());
  }
}
```

### Speculative Generality

**Tell:** unused parameters; methods that throw "not implemented"; abstract bases with one impl; flags for hypothetical features.

**Why:** YAGNI — costs maintenance and cognitive load now for benefits that never come.

**Recipe:** Inline / Delete.

```typescript
// ❌ — built for hypothetical needs
interface PaymentProcessor {
  process(amount: Money): void;
  rollback(): void;            // never called
  audit(): AuditReport;        // never called
  scheduleRecurring(): void;   // never called
  generateReport(): Report;    // never called
}

class StripeProcessor implements PaymentProcessor {
  process(amount: Money): void { /* real */ }
  rollback() { throw new Error("not implemented"); }
  audit() { throw new Error("not implemented"); }
  scheduleRecurring() { throw new Error("not implemented"); }
  generateReport() { throw new Error("not implemented"); }
}

// ✅ — only what's used now
class StripeProcessor {
  process(amount: Money): void { /* ... */ }
}
// Add interface + other methods when SECOND implementation exists.
```

### Comments

**Tell:** comment paraphrases the next line(s); section headers ("// validation"); commented-out code.

**Why:** comments rot when code changes; signal that names aren't expressive enough.

**Recipe:** Rename + Extract Method (sections become methods).

```typescript
// ❌
function check(user: User): boolean {
  // Check if user is older than 18
  if (user.age >= 18) {
    // Check if user has verified email
    if (user.emailVerified) {
      // Check if user is not banned
      return !user.banned;
    }
  }
  return false;
}

// ✅
function canAccessAdultContent(user: User): boolean {
  if (!user.isAdult()) return false;
  if (!user.hasVerifiedEmail()) return false;
  if (user.isBanned()) return false;
  return true;
}
```

### Anemic Domain Model

**Tell:** entities are data bags (only getters/setters); business logic in `*Service` classes that mutate entity state.

**Why:** "Tell, Don't Ask" violation; logic scattered; rules duplicated; hard to enforce invariants.

**Recipe:** Move Behavior to Entity.

```typescript
// ❌ — User is a data bag; logic in service
class User {
  email: string;
  isPremium: boolean;
  balance: number;
  loyaltyPoints: number;
}

class UserService {
  applyDiscount(user: User, amount: number) {
    if (user.isPremium) {
      user.balance -= amount * 0.8;
      user.loyaltyPoints += Math.floor(amount * 0.1);
    } else {
      user.balance -= amount;
      user.loyaltyPoints += Math.floor(amount * 0.05);
    }
  }
}

// ✅ — User has behavior; service orchestrates, doesn't manipulate
class User {
  constructor(
    private balance: Money,
    private loyaltyPoints: LoyaltyPoints,
    private readonly tier: UserTier,
  ) {}

  withdraw(amount: Money): WithdrawResult {
    const final = this.tier.applyDiscount(amount);
    if (this.balance.lessThan(final)) return WithdrawResult.insufficientFunds();
    this.balance = this.balance.subtract(final);
    this.loyaltyPoints = this.loyaltyPoints.add(this.tier.pointsEarned(amount));
    return WithdrawResult.success();
  }
}
```

### Train Wreck (Demeter Violation)

**Tell:** chained method calls / property accesses on object graphs (`a.b().c().d()`).

**Why:** caller knows too much about object structure; rename in `B.c` ripples through every caller.

**Recipe:** Hide Delegate.

```typescript
// ❌
const city = order.getCustomer().getAddress().getCity();

// ✅ — Order exposes the result; Customer/Address internals stay hidden
const city = order.shippingCity();

class Order {
  shippingCity(): City {
    return this.customer.shippingAddress().city();
  }
}
```

**Exception:** fluent builders, native collection method chains (`.filter().map()`).

---

## Refactoring Workflow When You Find a Smell

```
1. Confirm tests cover the affected code
   ├── NO  → Add characterization tests first (legacy-code.md)
   └── YES → continue
2. Apply the refactor recipe (one step at a time)
3. Run tests after EACH step
   ├── FAIL → undo, try smaller step
   └── PASS → commit, continue
4. Stop when smell is gone OR you hit diminishing returns
5. Don't combine refactors in a single commit
```

Multiple smells in one PR? Address the **biggest blocker** first; leave others as TODO with a tracking issue.

---

## What's NOT a Smell (false alarms)

| Looks like a smell | But isn't, when… |
|--------------------|------------------|
| Long method | Single linear list of operations with no branching (e.g. SQL builder, mapper) |
| Large class | Value object with many small accessor methods |
| Duplication | < 3 occurrences (Rule of Three) |
| Switch | On a truly fixed enumeration (HTTP method, weekday) |
| Comment | Explains business rule, workaround, or non-obvious algorithm |
| Long parameter list | Constructor of a value object with many attributes |
| `else` | Inside a clear two-branch logical construct (sometimes clearer) |
| Primitive | Truly is a primitive (loop counter, internal index) |

**Rule:** smells are heuristics, not rules. Confirm the underlying problem exists before refactoring.

---

## Quick Reference

| Smell | Tell | Refactor recipe |
|-------|------|-----------------|
| Long Method | > 10 lines | Extract Method |
| Large Class | > 50 lines / > 2 vars | Extract Class |
| Long Parameter List | > 3 params | Introduce Parameter Object |
| Primitive Obsession | `string`/`number` for domain | Replace Primitive with Value Object |
| Data Clumps | Same params group | Extract Class |
| Switch on Type | `if`-chain on type field | Replace Conditional with Polymorphism |
| Feature Envy | Uses other class's data | Move Method |
| Inappropriate Intimacy | Reaches into private state | Move Method / Extract Class |
| Train Wreck | `a.b.c.d` chains | Hide Delegate |
| Speculative Generality | Unused abstractions | Inline / Delete |
| Anemic Domain | Logic in service, data in entity | Move Behavior to Entity |
| Comments | Paraphrase code | Rename / Extract Method |
| `else` | Any `else` block | Replace with Guard Clause |
| God Class | > 5 vars / 3+ verb domains | Extract Class per domain |
| Boolean param | `boolean` arg in public method | Split into named methods |
