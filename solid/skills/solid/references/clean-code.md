# Clean Code Practices

Operationalized clean code rules. Every guideline has a **mechanical tell** so the rule applies consistently.

## Decision Matrix — Apply Before Writing

| Question | Rule | Mechanical Tell |
|----------|------|-----------------|
| Is this name clear in 6 months without context? | Naming | Vague: `data`, `info`, `util`, `manager`, `handler` |
| Does this method fit on one screen? | Size | > 10 lines = suspect; > 20 = split |
| Can I read this top-to-bottom like a story? | Storytelling | High-level public methods at top; details below |
| Does this comment explain WHY or paraphrase WHAT? | Comments | Comment paraphrases code = delete |
| Does this primitive represent a domain concept? | Value objects | `string` for id/email/money/date = wrap |
| Does this method do one thing? | Compose method | Multiple "and"s in description = extract |

---

## Naming Principles (in priority order)

When naming conflicts, **higher priority wins**.

### 1. Consistency (HIGHEST)

Same concept = same name everywhere. One name per concept.

```typescript
// ❌ — three names for the same operation
class UserService { getUserById(id: string) {} }
class OrderService { fetchClient(id: string) {} }
class ProductService { retrieveCustomer(id: string) {} }

// ✅ — consistent
class UserService { getUser(id: UserId) {} }
class OrderService { getCustomer(id: CustomerId) {} }
class ProductService { getCustomer(id: CustomerId) {} }
```

**Tell:** grep your codebase for synonym pairs (`fetch`/`get`/`retrieve`, `create`/`new`/`make`, `delete`/`remove`/`destroy`). Pick one, refactor others.

### 2. Domain Language

Use the language of the business, not technical jargon.

```typescript
// ❌ — technical
const arr = users.filter(u => u.flag);
const x = process(arr);

// ✅ — domain
const activeMembers = users.filter(member => member.isActive);
const subscriptions = renewExpired(activeMembers);
```

**Tell:** if a domain expert wouldn't recognize the term in your code, you're using jargon.

### 3. Specificity

Avoid vague names. Be specific about what the thing is.

**Banned vague suffixes:**
- `Manager`, `Helper`, `Util`, `Utility`, `Handler`, `Processor`, `Service` (when not in DDD layer)
- `Data`, `Info`, `Object`, `Item` (when describing a domain concept)

```typescript
// ❌ — what does the manager manage? what does it process?
class DataManager { processInfo(data: any) {} }
class UserHelper { getThing(user: User) {} }

// ✅ — specific role
class OrderRepository { save(order: Order) {} }
class UserAuthenticator { authenticate(credentials: Credentials) {} }
```

**Tell:** if you can replace the noun in the class name with `Thing` and the meaning doesn't degrade, the name is too vague.

### 4. Brevity (without losing meaning)

Short when meaning is preserved. Long when needed for clarity.

```typescript
// ❌ — cryptic
const u = getU(id);
const usrLst = getUsrs();
const i = arr.length;

// ❌ — unnecessarily verbose
const listOfActiveUsersInTheCurrentSystem = getActiveUsers();

// ✅ — brief but clear
const user = getUser(id);
const activeUsers = getActiveUsers();
const itemCount = items.length;
```

**Tell:** if pronouncing the name in conversation feels weird, rename. Say the name out loud.

### 5. Searchability

Names should be unique enough to grep across the codebase.

```typescript
// ❌ — too common, can't find usages
const data = fetch();
const e = new Error();

// ✅ — unique and greppable
const userProfile = fetchUserProfile();
const validationError = new ValidationError();
```

**Tell:** if you grep for the name and get >50 hits across the codebase, it's too generic.

### 6. Pronounceability

You should be able to say it in conversation.

```typescript
// ❌
const genymdhms = generate();
const usrCrt = createUser();

// ✅
const timestamp = generate();
const newUser = createUser();
```

**Tell:** rule of thumb — if you'd spell it out letter-by-letter to a colleague, rename.

### 7. Austerity (no filler)

Don't add words that don't carry meaning.

```typescript
// ❌ — `Data`, `Info`, `Object` add nothing
const userData = user;
class UserInfo {}
class OrderObject {}

// ✅ — just the noun
const user = user;
class User {}
class Order {}
```

Common useless suffixes: `Data`, `Info`, `Object`, `Item`, `Element` (in domain code).

### Naming by role

| Code element | Pattern | Examples |
|--------------|---------|----------|
| Class | Noun (PascalCase) | `Order`, `EmailValidator`, `UserRepository` |
| Method | Verb (camelCase) | `calculate`, `save`, `findById` |
| Boolean | `is`/`has`/`can` prefix | `isPremium`, `hasPermission`, `canPublish` |
| Constant | SCREAMING_SNAKE | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Interface (when needed) | Capability noun, no `I` prefix | `EmailService`, `UserRepository` |
| Enum / union | Singular noun | `OrderStatus`, not `OrderStatuses` |

**Anti-pattern: `IUserRepository`**. The `I` prefix is C# / Java legacy noise. In TypeScript, the interface is the natural name; the implementation gets a more specific one (`PostgresUserRepository`).

---

## Object Calisthenics — 9 Rules

Strict in practice; relax with judgment in production. Each rule has a **tell** for when it's violated.

### 1. One Level of Indentation per Method

```typescript
// ❌ — 3 levels deep; cognitive overload
function process(orders: Order[]) {
  for (const order of orders) {
    if (order.isValid()) {
      for (const item of order.items) {
        if (item.inStock) {
          // process...
        }
      }
    }
  }
}

// ✅ — extract methods; each has 1 level
function process(orders: Order[]) {
  orders.filter(o => o.isValid()).forEach(processOrder);
}

function processOrder(order: Order) {
  order.items.filter(i => i.inStock).forEach(processItem);
}
```

**Tell:** any indent level > 1 inside a method body. Extract.

### 2. No `else` Keyword

Use early return, guard clause, or polymorphism.

```typescript
// ❌
function discount(user: User): Percent {
  if (user.isPremium) {
    return Percent.of(20);
  } else {
    return Percent.zero();
  }
}

// ✅ — guard clause
function discount(user: User): Percent {
  if (user.isPremium) return Percent.of(20);
  return Percent.zero();
}
```

**Tell:** any `else` block. Almost always replaceable with early return.

### 3. Wrap All Primitives Representing Domain Concepts

Primitives with meaning beyond their type get value objects.

```typescript
// ❌
function transfer(from: string, to: string, amount: number) {}

// ✅
function transfer(from: AccountId, to: AccountId, amount: Money) {}
```

**Tell:** function parameter typed as `string` or `number` when the value has business meaning (id, email, money, date, status, country, etc.).

**Don't wrap:** indexes, counts of internal collections, boolean flags. Wrap when the type carries domain identity.

### 4. First-Class Collections

A class with a collection should have **no other instance variables**. Wrap the collection.

```typescript
// ❌ — collection mixed with other state
class Order {
  items: OrderItem[] = [];
  customerId: CustomerId;
  total: Money;
}

// ✅ — collection is its own class with relevant behavior
class OrderItems {
  constructor(private readonly items: ReadonlyArray<OrderItem> = []) {}
  add(item: OrderItem): OrderItems { return new OrderItems([...this.items, item]); }
  total(): Money { return this.items.reduce((sum, i) => sum.add(i.subtotal()), Money.zero()); }
  isEmpty(): boolean { return this.items.length === 0; }
  count(): number { return this.items.length; }
}

class Order {
  constructor(
    private readonly items: OrderItems,
    private readonly customer: Customer,
  ) {}
}
```

**Tell:** instance variable typed `Array<X>`/`Map<...>`/`Set<...>` alongside other vars. Extract.

### 5. One Dot per Line (Law of Demeter)

Don't reach through object graphs.

```typescript
// ❌ — train wreck; caller knows too much about Order's internals
const city = order.getCustomer().getAddress().getCity();

// ✅ — ask immediate friend
const city = order.shippingCity();
```

**Tell:** more than one method call chained on object access (`a.getB().getC()`). Extract via Tell-Don't-Ask.

**Exceptions:** fluent builders (intentional chaining), array/string method chains (`arr.filter().map().reduce()`).

### 6. Don't Abbreviate

Abbreviations are usually a sign the class does too much.

```typescript
// ❌
const custRepo = new CustRepo();
const ord = new Ord();
const usrSvc = new UsrSvc();

// ✅
const customerRepository = new CustomerRepository();
const order = new Order();
const userService = new UserService();
```

**Common allowed abbreviations:** `id`, `db`, `i`/`j` (loop counters in tight loops), `req`/`res` (in HTTP middleware idiomatically), `it` (iterator).

**Tell:** if an abbreviation isn't in the universal-tech-vocabulary, write the full word.

### 7. Keep Entities Small

| Entity | Hard limit | Suspect |
|--------|------------|---------|
| Method | 10 lines | > 5 lines with branching |
| Class | 50 lines | > 30 lines with multiple responsibilities |
| File | 200 lines | > 100 lines with multiple classes |

**When to relax:** value objects with many small accessor methods, configuration definitions, generated code. **When to enforce strictly:** business logic, controllers, services.

### 8. No More Than 2 Instance Variables per Class

Forces composition over data dumps.

```typescript
// ❌ — 5 instance variables; multiple concerns
class Order {
  id: OrderId;
  customerId: CustomerId;
  items: OrderItem[];
  total: Money;
  status: OrderStatus;
}

// ✅ — composed of smaller objects
class Order {
  constructor(
    private readonly id: OrderId,
    private readonly state: OrderState,  // groups items, total, status
  ) {}
}

class OrderState {
  constructor(
    private readonly items: OrderItems,
    private readonly status: OrderStatus,
  ) {}
  total(): Money { return this.items.total(); }
}
```

**When to relax:** value objects representing inherently multi-attribute concepts (`Address` with street/city/zip is fine). Use judgment — does the grouping reflect a real concept?

**Tell:** > 4 instance variables. Look for groups that should become their own classes.

### 9. No Getters / Setters / Public Properties

Objects expose **behavior**, not data. "Tell, Don't Ask."

```typescript
// ❌ — caller does the logic
class Account {
  balance: Money;
}
if (account.balance.greaterThan(amount)) {
  account.balance = account.balance.subtract(amount);
}

// ✅ — Account encapsulates the rule
class Account {
  private balance: Money;
  withdraw(amount: Money): WithdrawResult {
    if (this.balance.lessThan(amount)) return WithdrawResult.insufficientFunds();
    this.balance = this.balance.subtract(amount);
    return WithdrawResult.success(this.balance);
  }
}
const result = account.withdraw(amount);
```

**When you genuinely need accessors:** value objects exposing read-only properties via `get`-style methods are fine (`money.cents()`, `money.currency()`). The rule targets mutable state, not immutable views.

**Tell:** every getter call followed by setter call on the same field — should be a single behavior method.

---

## Comments

### When to write comments

**Comments explain WHY, never WHAT or HOW.** Code already says what and how.

```typescript
// ❌ — paraphrases the code; rots when code changes
// Increment counter by 1
counter++;

// Loop through users
for (const user of users) { /* ... */ }

// Check if user is admin
if (user.isAdmin) { /* ... */ }

// ✅ — explains a non-obvious decision
// Compensate for legacy 0-based indexing in upstream API
counter++;

// Process in reverse to handle cascading deletes correctly
for (const user of [...users].reverse()) { /* ... */ }

// Workaround: backend returns "admin" as string instead of boolean (fix tracked in JIRA-1234)
if (user.role === "admin") { /* ... */ }
```

### When NOT to write comments

| Case | Better alternative |
|------|--------------------|
| Explaining a complex condition | Extract to a named method/variable: `if (canPublish())` |
| Section headers ("// VALIDATION") | Extract to private method `validate()` |
| Marking dead code | Delete it. Git remembers. |
| `TODO` without owner/date | Track in issue system instead |
| Commented-out alternatives | Delete. Use git history. |
| API documentation that paraphrases types | Rely on types + JSDoc only when meaningful |

### Allowed comment types

```typescript
// 1. Public API documentation (JSDoc)
/**
 * Charges the customer's primary payment method.
 *
 * @param amount - amount to charge
 * @returns success result with transaction ID, or failure with reason
 * @throws PaymentDeclinedError when the gateway rejects the charge
 * @example
 *   const result = await account.charge(Money.dollars(100));
 */
async charge(amount: Money): Promise<ChargeResult> { /* ... */ }

// 2. Business rule citation
// SOX compliance: audit log retained for 7 years (legal/SOX-2.4)
const RETENTION_YEARS = 7;

// 3. Workaround documentation
// Safari < 15.4 doesn't fire `pagehide` reliably on bfcache restore.
// Fall back to visibilitychange to flush analytics queue.
window.addEventListener("visibilitychange", flushQueue);

// 4. Algorithm explanation when implementation is non-obvious
// Two-pointer technique: O(n) time, O(1) space.
// Faster pointer advances 2 steps per iteration to detect cycles.
function hasCycle(head: Node): boolean { /* ... */ }
```

---

## Formatting

### Vertical spacing rules

```typescript
class OrderProcessor {
  // ✅ Public API at the top — what consumers see first
  process(order: Order): ProcessResult {
    this.validate(order);
    this.calculate(order);
    return this.persist(order);
  }

  // Blank line between concept groups

  // ✅ Supporting methods below, in order of appearance/dependency
  private validate(order: Order): void { /* ... */ }

  private calculate(order: Order): void { /* ... */ }

  private persist(order: Order): ProcessResult { /* ... */ }
}
```

**Rules:**
- Public methods first, private after
- Methods called by `process` appear in the order they're called
- Blank line between methods
- No blank lines inside method bodies (if you need them, the method's too long → extract)

### Horizontal spacing

- Indent: 2 spaces (TypeScript/JavaScript convention) or per project standard
- Max line length: 100-120 characters
- Spaces around binary operators: `a + b`, not `a+b`
- No spaces around unary: `!flag`, `-1`, `++i`

### Storytelling order

Code should read like a story top-to-bottom: high-level intent first, details follow.

```typescript
// ✅ Story order — read like a paragraph
function checkout(cart: Cart, payment: PaymentMethod): CheckoutResult {
  validateCart(cart);
  const total = calculateTotal(cart);
  const charge = processPayment(payment, total);
  if (!charge.success) return CheckoutResult.failed(charge.reason);
  return CheckoutResult.success(persistOrder(cart, total, charge));
}

// Supporting functions follow — in order they're called
function validateCart(cart: Cart): void { /* ... */ }
function calculateTotal(cart: Cart): Money { /* ... */ }
function processPayment(payment: PaymentMethod, total: Money): ChargeResult { /* ... */ }
function persistOrder(cart: Cart, total: Money, charge: ChargeResult): Order { /* ... */ }
```

**Anti-pattern: random method order or alphabetical.** Order matters — it's part of the documentation.

---

## Function / Method Design

### One thing per method (compose method pattern)

A method should do ONE thing at ONE level of abstraction.

```typescript
// ❌ — mixed levels: high-level intent + low-level details inline
function processOrder(order: Order) {
  // Low-level: validation
  if (order.items.length === 0) throw new Error("empty");
  if (order.customer === null) throw new Error("no customer");

  // Low-level: total calculation
  let total = 0;
  for (const item of order.items) {
    total += item.price * item.quantity;
  }

  // High-level: delegate
  this.repo.save(order);
}

// ✅ — single level of abstraction; reads like a story
function processOrder(order: Order) {
  this.validate(order);
  this.calculateTotal(order);
  this.repo.save(order);
}
```

**Tell:** mixed abstraction levels in one method (validation logic + database calls + business calculation in same body).

### Method size by role

| Role | Typical size |
|------|--------------|
| Coordinator (orchestrates calls) | 3-7 lines |
| Domain logic | 5-10 lines |
| Validation | 1-5 lines |
| Calculation | 1-3 lines |
| Getter / accessor | 1 line |

**> 10 lines = inspect for extraction opportunity.** Not always wrong, but always worth a look.

---

## Quick Reference

| Aspect | Rule | Tell |
|--------|------|------|
| Naming consistency | One name per concept | Synonym variants in grep |
| Vague names | Avoid `data`, `manager`, `util` | Replace noun with `Thing` — meaning unchanged |
| Vague booleans | `is`/`has`/`can` prefix | Boolean without prefix |
| Method length | ≤ 10 lines | Body > 10 lines |
| Class length | ≤ 50 lines | > 50 lines |
| Indentation | 1 level per method | Indent > 1 |
| `else` | Avoid via early return | Any `else` block |
| Primitive obsession | Wrap domain primitives | `string` for id/email/money/date |
| Collections | First-class collection class | Array + other vars in same class |
| Demeter | One dot per line | `a.b.c.d` chains |
| Instance variables | ≤ 2 per class | > 2 |
| Parameters | ≤ 3 per function | > 3 → parameter object |
| Comments | WHY only | Comment paraphrases the code |
| Method order | High-level → details | Random order |
| Single responsibility | One reason to change | "and" in description |
