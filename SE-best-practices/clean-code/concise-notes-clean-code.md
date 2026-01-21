## 1. Clean Code — Core Foundations

* Clean code minimizes **cost of change over time**, not just readability.
* Code is read **far more often** than it is written.
* Clean code enables:

  * Safe modification
  * Predictable behavior
  * Localized impact of changes
* Dirty code increases:

  * Regression risk
  * Debugging time
  * Team fear of change
* Clean code is **easy to delete**.
* Cleanliness ≠ cleverness.

---

## 2. Naming Conventions (Highest ROI Skill)

### General Rules

* Names must reveal **intent**, not implementation.
* Prefer clarity over brevity.
* Length of name ∝ scope of usage.
* Avoid abbreviations unless universally understood.
* Avoid encoding type or context redundantly.

### Variables

* Use nouns that reflect domain meaning.
* Avoid generic names (`data`, `value`, `temp`).
* Boolean names must read like predicates:

  * `isActive`, `hasPermission`, `canRetry`

### Methods

* Use verbs.
* Method names should answer **what**, not **how**.
* Avoid ambiguous names (`process`, `handle`, `doWork`).

### Classes

* Represent **single responsibility / concept**.
* Avoid god objects and utility dumping grounds.

---

## 3. Writing Clean Functions (Most Important)

### Core Principles

* A function must do **one thing**.
* Maintain **one level of abstraction** per function.
* Function behavior should be **predictable**.

### Size & Structure

* Smaller is better, but **side-effect free** matters more than size.
* Avoid deeply nested logic.
* Prefer early returns over nested conditionals.

### Arguments

* 0–2 arguments ideal.
* 3 arguments → warning sign.
* 4+ arguments → design smell.
* Replace long parameter lists with parameter objects.

### Side Effects

* Avoid mutating external state.
* Avoid hidden behavior.
* Avoid modifying arguments.

### Red Flags

* Boolean parameters.
* Output parameters.
* Catching exceptions inside business logic.
* Mixed responsibilities.

---

## 4. Comments (Last Resort)

### Golden Rule

* Comments compensate for **unclear code**.
* Code should be **self-documenting**.

### Valid Uses

* Legal / license headers.
* Public API contracts.
* Business reasoning (why, not how).
* Non-obvious algorithms.

### Invalid Uses

* Explaining obvious code.
* Commenting bad naming.
* TODOs that replace design fixes.

### Strategy

* Eliminate comments by refactoring.
* Move intent into method and variable names.
* Tests age better than comments.

---

## 5. Code Formatting & Indentation

* Formatting improves **visual parsing speed**.
* Humans should not format code manually.
* Use:

  * IDE auto-formatter
  * One linter
  * CI enforcement
* Formatting debates indicate tooling failure.
* Consistency > personal preference.

---

## 6. Code Smells (Early Decay Signals)

### High-Impact Smells

* Long methods → mixed responsibilities.
* Large classes → god objects.
* Duplicate code → inconsistent behavior risk.
* Magic numbers / strings → missing domain language.
* Feature envy → misplaced responsibility.

### Senior Insight

* Smells are **signals**, not rules.
* Understand root cause before refactoring.
* Blind refactoring causes regressions.

---

## 7. Refactoring (Danger Zone)

### Core Rule

* Never refactor without **behavior protection (tests)**.

### Safe Refactoring

* Rename variables/methods.
* Extract methods/classes.
* Move code without changing behavior.
* Replace magic values with constants.

### Dangerous Refactoring

* Changing logic and structure together.
* Large refactors without tests.
* Opportunistic “cleanup” changes.

### Workflow

1. Lock behavior with tests.
2. Refactor in small steps.
3. Run tests after every change.
4. Commit frequently.

---

## 8. Error Handling in Clean Code

### Principles

* Exceptions are part of your API.
* Never catch exceptions you cannot handle.
* Preserve original context.
* Fail fast, not silently.

### Best Practices

* Use specific exception types.
* Provide meaningful, actionable messages.
* Avoid generic `Exception` or `RuntimeException`.
* Validate inputs at system boundaries.

### Anti-Patterns

* Swallowing exceptions.
* Logging and continuing blindly.
* Catching exceptions too early.

---

## 9. Dependency Management (Architecture Begins Here)

### Core Principle

* High-level modules must not depend on low-level details.

### Problems with Tight Coupling

* Hard to test.
* Hard to replace.
* Hard to evolve.
* Changes ripple across system.

### Solutions

* Dependency inversion.
* Constructor injection.
* Interface-based design.
* Clear module boundaries.

### Outcome

* Replaceable components.
* Scalable teams.
* Safer refactoring.

---

## 10. TDD (Test-Driven Design, Not Testing)

### Correct Mindset

* TDD is a **design tool**, not a coverage tool.
* Tests guide clean APIs and responsibilities.

### Red → Green → Refactor

* Red: clarify intent.
* Green: minimal correctness.
* Refactor: improve design.

### Signals from Tests

* Hard-to-write tests → bad design.
* Excessive mocking → tight coupling.

---

## 11. Senior Synthesis (Architect View)

* Clean code manages **risk**, not aesthetics.
* Naming, functions, and dependencies define system health.
* Refactoring skill defines seniority.
* Architecture emerges from disciplined clean code.
* The best systems are:

  * Boring
  * Predictable
  * Calm under change

---

### Key Senior Rule (Final)

> If code cannot be changed safely, it is already broken.
