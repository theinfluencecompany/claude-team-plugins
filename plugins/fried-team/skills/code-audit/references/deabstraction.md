# Deabstraction: Removing Unnecessary Complexity

Reference for auditing code against unnecessary abstraction, indirection, and intermediaries. Use when evaluating §2.4 (Abstraction Quality) findings.

---

## Core Principles

### The Wrong Abstraction (Sandi Metz)

> "Duplication is far cheaper than the wrong abstraction."

A wrong abstraction accrues compounding costs as developers bend it for cases it was never designed to handle. The fix: inline back into callers, remove dead branches, re-extract only what is truly shared.

**The destructive cycle:**
1. Programmer A sees duplication, extracts an abstraction.
2. New requirement arrives that *almost* fits.
3. Programmer B adds a parameter and conditional.
4. Repeat. The abstraction becomes incomprehensible.
5. Everyone is afraid to touch it. Development slows.

**Diagnostic:** Has this abstraction accumulated boolean flags, type switches, or "mode" parameters? If yes — it is the wrong abstraction.

### AHA Programming — Avoid Hasty Abstractions (Kent C. Dodds)

Neither DRY nor WET should be applied dogmatically. Tolerate duplication until the commonalities "scream at you for abstraction." Optimize for change first, not for the elimination of duplication.

```
DRY (Never Repeat) <-----> AHA (Abstract When It Screams) <-----> WET (Repeat Freely)
```

### Rule of Three (Don Roberts / Martin Fowler)

Two instances of similar code do not justify abstraction. Wait for the THIRD occurrence. The third gives enough information to understand what truly varies and what stays the same.

### YAGNI — You Aren't Gonna Need It (Kent Beck)

Never build on speculation. Only implement what is required by a current, concrete need. Before adding any capability: "Do I need this RIGHT NOW for a real use case?"

---

## Removal Patterns

### Inline Function

When a function's body is as clear as its name, or when indirection has become pointless — inline it.

```typescript
// BEFORE — trivial delegation chain
function getRating(driver: Driver) {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(driver: Driver) {
  return driver.numberOfLateDeliveries > 5;
}

// AFTER — direct
function getRating(driver: Driver) {
  return driver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

**Apply when:** Function body is a single expression. Name adds no explanatory value beyond what the code says. Excessive indirection (A calls B calls C, where B/C do nothing).

### Inline Class

If a class is not doing enough to justify its existence, fold it into its consumer.

**Apply when:** Class is a thin data holder with no behavior, or has been gutted by previous refactorings. Also useful as prep: inline two classes into one, then re-extract along better boundaries.

### Kill Your Darlings

When you feel proud of a clever design, stop and ask: would a new team member understand this in 5 minutes? Is there a boring, obvious alternative?

```typescript
// BEFORE — "elegant" monad pipeline for... checking if a user is active
const isActive = compose(chain(validateSession), map(extractToken), fromNullable)(user);

// AFTER — boring and obvious
const isActive = user != null && user.session && !user.session.expired;
```

---

## Structural Simplification

### Flat Over Nested

Each level of nesting is a cognitive tax. Flatten via guard clauses and early returns.

```typescript
// BEFORE — nested pyramid
function process(data: Data) {
  if (data != null) {
    if (data.isValid()) {
      if (data.hasPermission()) {
        return doWork(data);
      } else { throw new PermissionError(); }
    } else { throw new ValidationError(); }
  } else { throw new ValueError("No data"); }
}

// AFTER — flat guard clauses
function process(data: Data) {
  if (data == null) throw new ValueError("No data");
  if (!data.isValid()) throw new ValidationError();
  if (!data.hasPermission()) throw new PermissionError();
  return doWork(data);
}
```

### Direct Style Over Wrapper Chains

Write code in natural sequential style. Let the runtime handle async/concurrency. Avoid monadic wrappers, callback chains, and continuation-passing when they add ceremony without benefit.

```typescript
// BEFORE — promise chain
getUser(id)
  .then(user => getOrders(user.id))
  .then(orders => getItems(orders[0].id))
  .then(items => render(items))
  .catch(handleError);

// AFTER — direct style
const user = await getUser(id);
const orders = await getOrders(user.id);
const items = await getItems(orders[0].id);
render(items);
```

### Composition Over Deep Inheritance

Favor composing from small, focused components over deep inheritance trees. Inheritance creates rigid coupling; composition creates flexible assemblies.

**Refactoring recipe:**
1. Create a field and place the superclass object in it.
2. Delegate methods to the field.
3. Remove the inheritance relationship.

---

## Code Organization

### Locality of Behavior (Carson Gross / htmx)

> "The behaviour of a unit of code should be as obvious as possible by looking only at that unit of code."

Reading one file should reveal what that code does — not require chasing references across the codebase. If understanding a single behavior requires reading 3+ files, the locality is broken.

### Colocation (Kent C. Dodds)

Place code as close to where it is relevant as possible. Things that change together should live together. Code + tests + styles + types for a feature belong in one directory, not scattered by file type.

### Pit of Success (Rico Mariani)

Design APIs and systems so the easiest path IS the correct path. If the API naturally guides to right usage, defensive middleware to prevent wrong usage becomes unnecessary.

---

## Unnecessary Middleware & Wrapper Layers

Pass-through layers that add no transformation, no validation, and no error handling are pure overhead.

**Diagnostic questions:**
- Does this middleware/wrapper transform the request or response?
- Does it handle errors differently from the layers above or below?
- If removed, would any test fail?
- Is this wrapper hiding a network boundary that developers need to be aware of?

**Keep when:** Authentication, rate limiting, request validation, circuit breaking, observability — genuine cross-cutting value.

**Remove when:** It exists because "that's the pattern," not because it does anything.

---

## Decision Framework

### When to Remove an Abstraction

```
[ ] Does this abstraction solve a CURRENT, CONCRETE problem?
    If not → remove (YAGNI)

[ ] Can you explain what this layer DOES (not what it IS)?
    If not → pure indirection

[ ] Does removing it cause any test to fail?
    If not → it may not be doing anything

[ ] Has it accumulated boolean flags or mode parameters?
    If yes → Wrong Abstraction (Sandi Metz)

[ ] Do you need to read 3+ files to understand one behavior?
    If yes → LoB violation, consider inlining

[ ] Is this class/function just calling another class/function?
    If yes → Inline Function / Inline Class

[ ] Was this built because "we might need it" or "that's the pattern"?
    If yes → Architecture Astronaut / Premature Abstraction
```

### When to KEEP an Abstraction

```
[ ] Represents a genuine domain concept
[ ] Multiple callers use it the SAME way (no flags/modes)
[ ] Removing it would duplicate complex logic
[ ] Enforces an invariant callers would otherwise violate
[ ] Hides complexity callers genuinely don't need
[ ] Provides a seam for testing otherwise impossible
[ ] Has been stable (few changes) over a meaningful period
```

### The Deabstraction Sequence

When an abstraction is identified for removal:

1. **Identify all callers** of the abstraction.
2. **Inline** into each caller (Inline Function / Inline Class).
3. **Remove dead code** — each inlined copy has branches only other callers used.
4. **Simplify each caller** independently — optimize for its specific context.
5. **Look for new, better abstractions** — with the old one gone, real patterns may emerge.
6. **Apply Rule of Three** — only re-extract when you see the same pattern three times.

---

## Warning Signs (Architecture Astronaut — Joel Spolsky)

> "When you go too far up, abstraction-wise, you run out of oxygen."

- The architecture diagram has more boxes than features.
- A new developer needs a week to understand the "framework" before writing business logic.
- The system can theoretically handle requirements it will never face.
- Class names include: "Manager," "Handler," "Processor," "Engine," "Framework," "Platform," "Bus."

**Counter-test:** "What concrete problem does this solve TODAY?"
