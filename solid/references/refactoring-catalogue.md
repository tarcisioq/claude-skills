# Refactoring Catalogue

Mechanical refactor recipes — Fowler-style. Each recipe has **numbered steps** so the refactor is reproducible without "feel". Tests must stay green throughout.

## Universal Rules (apply to every recipe)

1. **No refactor without tests.** If untested, add characterization tests first (`legacy-code.md`).
2. **One refactor at a time.** Don't combine multiple recipes in one commit.
3. **Run tests after each step.** If red, undo or take a smaller step.
4. **Commit frequently.** Each green state = a commit. Easy to revert.
5. **Don't change behavior during a refactor.** Behavior changes are separate commits.

---

## Index

| Recipe | When to use |
|--------|-------------|
| [Extract Method](#extract-method) | Long Method, mixed abstraction levels |
| [Inline Method](#inline-method) | Method body is clearer than the name |
| [Extract Variable](#extract-variable) | Complex expression, unclear meaning |
| [Inline Variable](#inline-variable) | Variable adds noise |
| [Rename](#rename) | Name doesn't convey intent |
| [Replace Conditional with Polymorphism](#replace-conditional-with-polymorphism) | Switch / if-else chain on type |
| [Replace Primitive with Value Object](#replace-primitive-with-value-object) | Primitive Obsession |
| [Introduce Parameter Object](#introduce-parameter-object) | Long Parameter List |
| [Extract Class](#extract-class) | Large Class with mixed responsibilities |
| [Inline Class](#inline-class) | Lazy Class adds no value |
| [Move Method](#move-method) | Feature Envy |
| [Replace Inheritance with Composition](#replace-inheritance-with-composition) | LSP violation; inheritance for code reuse |
| [Hide Delegate](#hide-delegate) | Train Wreck (Demeter violation) |
| [Replace Conditional with Guard Clauses](#replace-conditional-with-guard-clauses) | Nested conditions, else after early return |
| [Replace Magic Number with Named Constant](#replace-magic-number-with-named-constant) | Magic numbers / strings |
| [Decompose Conditional](#decompose-conditional) | Complex condition |
| [Consolidate Duplicate Conditional Fragments](#consolidate-duplicate-conditional-fragments) | Same code at end of all branches |
| [Replace Type Code with Class](#replace-type-code-with-class) | String / int "types" with no validation |
| [Encapsulate Field](#encapsulate-field) | Public field used externally |
| [Remove Setting Method](#remove-setting-method) | Setter on something that should be immutable |

---

## Extract Method

**When:** method > 10 lines, mixed abstraction levels, or section needs a comment to explain.

**Steps:**

1. Identify the cohesive block of lines to extract.
2. Create a new method with an intention-revealing name.
3. Identify variables used by the block:
   - Variables only used in the block → make them local in the new method
   - Variables modified by the block → return them (or pass mutable container)
   - Variables read by the block → pass as parameters
4. Copy the block into the new method.
5. Replace the original block with a call to the new method.
6. **Run tests.** If green, commit.
7. Repeat for next cohesive block.

**Example:**

```typescript
// Before
function processOrder(order: Order) {
  // Validate
  if (!order.items.length) throw new Error("empty");
  if (!order.customer) throw new Error("no customer");

  // Calculate total
  let total = 0;
  for (const item of order.items) {
    total += item.price * item.quantity;
  }

  // Save
  db.orders.insert({ ...order, total });

  // Notify
  emailService.send(order.customer.email, "Order confirmed");
}

// After Step 1: extract validate
function processOrder(order: Order) {
  validate(order);

  let total = 0;
  for (const item of order.items) {
    total += item.price * item.quantity;
  }

  db.orders.insert({ ...order, total });
  emailService.send(order.customer.email, "Order confirmed");
}

function validate(order: Order): void {
  if (!order.items.length) throw new Error("empty");
  if (!order.customer) throw new Error("no customer");
}

// After Step 2: extract calculateTotal
function processOrder(order: Order) {
  validate(order);
  const total = calculateTotal(order);
  db.orders.insert({ ...order, total });
  emailService.send(order.customer.email, "Order confirmed");
}

function calculateTotal(order: Order): number {
  return order.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// Continue until all blocks extracted...
```

---

## Inline Method

**When:** the method body is as clear as the method name, OR the method's body is no longer doing what the name says.

**Steps:**

1. Verify the method isn't polymorphic (don't inline overrides).
2. Find all callers.
3. Replace each call with the method's body.
4. Delete the method.
5. **Run tests.**

```typescript
// Before
function getRating(driver: Driver): number {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(driver: Driver): boolean {
  return driver.numberOfLateDeliveries > 5;
}

// After: inline (the helper isn't adding much value)
function getRating(driver: Driver): number {
  return driver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

---

## Extract Variable

**When:** complex expression makes a line hard to read.

**Steps:**

1. Pick the sub-expression you want to name.
2. Declare a `const` (or `final`) variable with an intention-revealing name.
3. Replace the sub-expression with the variable name.
4. **Run tests.**

```typescript
// Before
if ((order.itemCount * order.itemPrice + order.shipping - order.discount) > 1000) {
  applyDiscountTier();
}

// After
const orderTotal = order.itemCount * order.itemPrice + order.shipping - order.discount;
if (orderTotal > 1000) {
  applyDiscountTier();
}
```

---

## Inline Variable

**When:** variable name adds no clarity beyond the expression itself.

**Steps:**

1. Find the variable's only use.
2. Replace the variable with its expression.
3. Delete the declaration.
4. **Run tests.**

```typescript
// Before
const basePrice = order.basePrice;
return basePrice > 1000;

// After
return order.basePrice > 1000;
```

---

## Rename

**When:** name doesn't convey intent, or context has shifted.

**Steps:**

1. Use IDE rename (most IDEs handle this safely).
2. If no IDE: find all references manually, replace each.
3. **Run tests.**
4. Update documentation/comments referencing the old name.

```typescript
// Before
function calc(o: Order): number { /* ... */ }

// After
function calculateTotalWithTax(o: Order): Money { /* ... */ }
```

**Critical:** rename does NOT change behavior. If tests break, you actually changed something — undo and try again.

---

## Replace Conditional with Polymorphism

**When:** switch / if-else chain on a `type` / `kind` / `category` field, especially if duplicated across the codebase.

**Steps:**

1. Identify all switches/if-chains on the same type field.
2. Create an interface (or abstract class) representing the abstraction.
3. For each variant in the switch:
   - Create a class implementing the interface
   - Move the corresponding branch's logic into the class
4. Replace the type field with an instance of the abstraction.
5. Delete the switch — call the polymorphic method instead.
6. **Run tests after each branch migration.**

```typescript
// Before
type Shape = { type: "circle"; radius: number }
           | { type: "square"; side: number };

function area(shape: Shape): number {
  switch (shape.type) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.side ** 2;
  }
}
function perimeter(shape: Shape): number {
  switch (shape.type) {
    case "circle": return 2 * Math.PI * shape.radius;
    case "square": return 4 * shape.side;
  }
}

// Step 1: define interface
interface Shape {
  area(): number;
  perimeter(): number;
}

// Step 2: implement per variant
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

// Step 3: callers use polymorphism
function area(shape: Shape): number { return shape.area(); }
function perimeter(shape: Shape): number { return shape.perimeter(); }
```

---

## Replace Primitive with Value Object

**When:** primitive (`string`, `number`, `boolean`) used for a domain concept across the codebase.

**Steps:**

1. Identify all callsites using the primitive.
2. Create the value object class with validation in constructor.
3. Add factory method or constructor that takes the primitive.
4. Update one callsite at a time:
   - Wrap the primitive at construction
   - Update the type signatures
   - Update consumers to use methods on the value object
5. **Run tests after each callsite.**
6. When all callsites use the value object, the primitive type disappears from public APIs.

```typescript
// Before — Email as string everywhere
function createUser(email: string, name: string) {
  if (!email.includes("@")) throw new Error("invalid email");
  /* ... */
}

function changeEmail(user: User, email: string) {
  if (!email.includes("@")) throw new Error("invalid email");
  user.email = email;
}

// Step 1: introduce Email value object
class Email {
  constructor(private readonly value: string) {
    if (!Email.isValid(value)) throw new InvalidEmail(value);
  }
  static isValid(s: string): boolean { return /.@./.test(s); }
  toString(): string { return this.value; }
  equals(other: Email): boolean { return this.value === other.value; }
}

// Step 2: migrate one callsite
function createUser(email: Email, name: string) { /* validation already happened */ }

// Step 3: migrate next callsite
function changeEmail(user: User, email: Email) { user.email = email; }

// Eventually: every email is an Email object. The string primitive disappears.
```

---

## Introduce Parameter Object

**When:** function with > 3 parameters, or same group of parameters appears together repeatedly.

**Steps:**

1. Identify the parameter group.
2. Create a class (often a value object) wrapping the parameters.
3. Update one function at a time to take the new object.
4. Update callers to construct the object.
5. **Run tests after each function.**
6. Look for opportunities to add behavior to the new class.

```typescript
// Before
function createOrder(
  customerId: string,
  shippingStreet: string,
  shippingCity: string,
  shippingZip: string,
  shippingCountry: string,
  paymentToken: string,
  amount: number,
  currency: string,
) { /* ... */ }

// Step 1: extract Address
class Address {
  constructor(
    readonly street: string,
    readonly city: string,
    readonly zip: string,
    readonly country: string,
  ) {}
}

// Step 2: extract Money
class Money {
  constructor(
    readonly amount: number,
    readonly currency: string,
  ) {}
}

// Step 3: refactored signature
function createOrder(
  customerId: CustomerId,
  shipping: Address,
  payment: PaymentToken,
  amount: Money,
) { /* ... */ }
```

---

## Extract Class

**When:** Large Class — > 50 lines, > 2 instance vars, methods span multiple verb domains.

**Steps:**

1. Identify a cohesive group of fields + methods that belong together.
2. Create a new class.
3. Move the fields to the new class.
4. Move the methods to the new class. Update parameter lists.
5. Update the original class to delegate via a field of the new class.
6. **Run tests after each move.**
7. Repeat if more groups exist.

```typescript
// Before
class User {
  name: string;
  email: string;
  passwordHash: string;
  loginAttempts: number;
  lockedUntil: Date | null;

  login(password: string): boolean { /* checks passwordHash, increments loginAttempts */ }
  resetPassword(): void { /* ... */ }
  changeEmail(newEmail: string): void { /* ... */ }
  changeName(newName: string): void { /* ... */ }
}

// Step 1: extract Authentication fields/methods
class UserCredentials {
  constructor(
    private passwordHash: string,
    private loginAttempts: number = 0,
    private lockedUntil: Date | null = null,
  ) {}

  verify(password: string): boolean { /* ... */ }
  resetPassword(): void { /* ... */ }
  isLocked(): boolean { /* ... */ }
}

class User {
  constructor(
    private id: UserId,
    private email: Email,
    private name: Name,
    private credentials: UserCredentials,
  ) {}

  login(password: string): boolean {
    if (this.credentials.isLocked()) return false;
    return this.credentials.verify(password);
  }
}
```

---

## Inline Class

**When:** Lazy Class — adds no value beyond delegating.

**Steps:**

1. Find all uses of the class to inline.
2. Move its fields and methods to a parent / consuming class.
3. Update callers.
4. Delete the original class.
5. **Run tests.**

```typescript
// Before
class TelephoneNumber {
  private number: string;
  getNumber(): string { return this.number; }
}

class User {
  private phone: TelephoneNumber;
  getPhoneNumber(): string { return this.phone.getNumber(); }
}

// After: inline TelephoneNumber (it added no value)
class User {
  private phone: string;
  getPhoneNumber(): string { return this.phone; }
}
```

**Caveat:** if the class will grow in scope, inlining is premature. Inline only when the class has stayed thin for a while.

---

## Move Method

**When:** Feature Envy — method uses another class's data more than its own.

**Steps:**

1. Identify the target class (the one whose data the method uses).
2. Copy the method to the target class.
3. Update parameters: the original `this` becomes a parameter.
4. Update the original class to delegate or remove it.
5. Update callers.
6. **Run tests.**

```typescript
// Before — Order envies Customer
class Order {
  calculateShipping(customer: Customer): Money {
    if (customer.country.equals(Country.US)) {
      if (customer.state.equals(State.CA)) return Money.dollars(10);
      return Money.dollars(15);
    }
    return Money.dollars(25);
  }
}

// After — Customer knows its own shipping cost
class Customer {
  shippingCostFor(orderTotal: Money): Money {
    if (this.country.equals(Country.US)) {
      if (this.state.equals(State.CA)) return Money.dollars(10);
      return Money.dollars(15);
    }
    return Money.dollars(25);
  }
}

class Order {
  calculateShipping(): Money {
    return this.customer.shippingCostFor(this.total());
  }
}
```

---

## Replace Inheritance with Composition

**When:** LSP violation, "Refused Bequest" smell, or inheritance used for code reuse.

**Steps:**

1. Identify the parent class methods/fields used by the subclass.
2. Convert the subclass: instead of extending, hold an instance of the parent (or a strategy).
3. Update the subclass methods to delegate.
4. Update callers — they no longer rely on the inheritance.
5. **Run tests.**

```typescript
// Before — inheritance for code reuse
class List {
  add(item: any): void { /* ... */ }
  remove(item: any): void { /* ... */ }
  contains(item: any): boolean { /* ... */ }
}

class Set extends List {
  add(item: any): void {
    if (!this.contains(item)) super.add(item);
  }
}

// After — composition
class Set {
  private list = new List();

  add(item: any): void {
    if (!this.list.contains(item)) this.list.add(item);
  }
  contains(item: any): boolean { return this.list.contains(item); }
  remove(item: any): void { this.list.remove(item); }
}
```

---

## Hide Delegate

**When:** Train Wreck — `a.b().c().d()` chains expose internal structure.

**Steps:**

1. Identify the chain.
2. Create a delegating method on the first object.
3. Replace the chain with the new method.
4. **Run tests.**
5. If the original method on the intermediate is no longer needed externally, mark it private.

```typescript
// Before
const city = order.getCustomer().getAddress().getCity();

// Step 1: add delegate on Order
class Order {
  shippingCity(): City {
    return this.customer.shippingAddress().city();
  }
}

// Step 2: replace chain at callsites
const city = order.shippingCity();

// Step 3: if `getCustomer()` is no longer used externally, mark private
class Order {
  // private (was public)
  private getCustomer(): Customer { /* ... */ }
}
```

---

## Replace Conditional with Guard Clauses

**When:** nested `if` with single early-return path; `else` after early return.

**Steps:**

1. Identify the conditional structure.
2. Convert each "skip" condition to an early return at the top of the method.
3. Remove the corresponding `else`.
4. **Run tests.**

```typescript
// Before
function payAmount(employee: Employee): number {
  let result;
  if (employee.isSeparated) {
    result = 0;
  } else {
    if (employee.isRetired) {
      result = retiredAmount();
    } else {
      result = normalAmount();
    }
  }
  return result;
}

// After
function payAmount(employee: Employee): number {
  if (employee.isSeparated) return 0;
  if (employee.isRetired) return retiredAmount();
  return normalAmount();
}
```

---

## Replace Magic Number with Named Constant

**When:** numeric/string literal with no clear meaning, or used in multiple places.

**Steps:**

1. Identify the magic value.
2. Declare a constant with an intention-revealing name (`SCREAMING_SNAKE_CASE`).
3. Replace each occurrence.
4. **Run tests.**

```typescript
// Before
if (user.age >= 21) { allowAlcohol(); }
if (order.weight > 50) { applyHeavyShippingFee(); }

// After
const LEGAL_DRINKING_AGE_US = 21;
const HEAVY_SHIPPING_THRESHOLD_KG = 50;

if (user.age >= LEGAL_DRINKING_AGE_US) { allowAlcohol(); }
if (order.weight > HEAVY_SHIPPING_THRESHOLD_KG) { applyHeavyShippingFee(); }
```

---

## Decompose Conditional

**When:** complex `if` condition that's hard to read.

**Steps:**

1. Extract the condition into a method or variable with a meaningful name.
2. Optionally extract the then-block and else-block into methods.
3. **Run tests.**

```typescript
// Before
if (date.before(SUMMER_START) || date.after(SUMMER_END)) {
  charge = quantity * winterRate + winterServiceCharge;
} else {
  charge = quantity * summerRate;
}

// After
function notSummer(date: Date): boolean {
  return date.before(SUMMER_START) || date.after(SUMMER_END);
}
function winterCharge(quantity: number): Money { /* ... */ }
function summerCharge(quantity: number): Money { /* ... */ }

if (notSummer(date)) {
  charge = winterCharge(quantity);
} else {
  charge = summerCharge(quantity);
}
```

---

## Consolidate Duplicate Conditional Fragments

**When:** the same code appears in all branches of an `if`.

**Steps:**

1. Identify the duplicated code.
2. Move it outside the `if` (before, after, or both).
3. **Run tests.**

```typescript
// Before
if (isSpecialDeal) {
  total = price * 0.95;
  send();
} else {
  total = price * 0.98;
  send();
}

// After
total = isSpecialDeal ? price * 0.95 : price * 0.98;
send();
```

---

## Replace Type Code with Class

**When:** "type" represented as a `string` or `int` with no validation.

**Steps:**

1. Create a class to represent the type.
2. Define each valid value as a constant of the class.
3. Replace the primitive with the class everywhere.
4. **Run tests after each migration.**

```typescript
// Before — type code as string, no validation, no behavior
class Order {
  status: string; // "pending", "paid", "shipped" — but nothing prevents typos
}
order.status = "Paid";  // bug — case-sensitive

// After — proper enum-like class
class OrderStatus {
  static readonly PENDING = new OrderStatus("pending");
  static readonly PAID = new OrderStatus("paid");
  static readonly SHIPPED = new OrderStatus("shipped");

  private constructor(private readonly value: string) {}

  equals(other: OrderStatus): boolean { return this.value === other.value; }
  toString(): string { return this.value; }
}

class Order {
  status: OrderStatus;
}
order.status = OrderStatus.PAID; // typo-proof, behavior-extensible
```

In TypeScript you can also use a tagged union or string literal type, but a class lets you add behavior (`.canTransitionTo()`, `.isFinal()`).

---

## Encapsulate Field

**When:** public field accessed directly from outside.

**Steps:**

1. Add a getter (and setter if needed).
2. Update all external accesses to use the getter/setter.
3. Mark the field private.
4. **Run tests.**

```typescript
// Before
class Person {
  public name: string;
}
p.name = "Alice";

// After
class Person {
  private _name: string;
  get name(): string { return this._name; }
  changeName(newName: Name): void { this._name = newName.toString(); }
}
```

**Note:** prefer methods that express behavior (`changeName`) over generic setters. Setters are anemic.

---

## Remove Setting Method

**When:** an entity exposes a setter for something that should be set only at construction.

**Steps:**

1. Verify the field is set only at construction.
2. Make the field `readonly` (or equivalent).
3. Delete the setter.
4. Update callers to set via constructor.
5. **Run tests.**

```typescript
// Before
class User {
  private id: string;
  setId(id: string) { this.id = id; }
}

// After
class User {
  constructor(private readonly id: UserId) {}
  // setter gone
}
```

---

## When to Stop Refactoring

Diminishing returns set in. Stop when:

- The smell that triggered the refactor is gone
- Tests pass
- Code is reasonably clear
- Further refactoring would be speculative (no current need)

**Don't refactor until perfect.** Ship when the smell is addressed.

---

## Quick Reference

| Smell | Recipe |
|-------|--------|
| Long Method | Extract Method |
| Mixed abstraction levels | Extract Method (per level) |
| Switch on type | Replace Conditional with Polymorphism |
| Primitive Obsession | Replace Primitive with Value Object |
| Long Parameter List | Introduce Parameter Object |
| Large Class | Extract Class |
| Lazy Class | Inline Class |
| Feature Envy | Move Method |
| LSP violation | Replace Inheritance with Composition |
| Train Wreck | Hide Delegate |
| `else` after early return | Replace Conditional with Guard Clauses |
| Magic number | Replace Magic Number with Named Constant |
| Complex condition | Decompose Conditional |
| Duplicate in branches | Consolidate Duplicate Conditional Fragments |
| String/int type code | Replace Type Code with Class |
| Public field | Encapsulate Field |
| Mutable identity | Remove Setting Method |
