---
name: tdd-principles
description: "Learn and apply Test-Driven Development (TDD) principles through guided explanation and coding exercises. Use when: teaching or learning TDD, understanding the Red-Green-Refactor cycle, practicing writing tests before code, improving code design through testing, doing TDD kata exercises, explaining why TDD leads to better design."
argument-hint: "The feature, function, or concept to implement using TDD (e.g. shopping cart, string calculator, password validator)"
---

# Test-Driven Development — Principles & Practice

## When to Use

- Teaching or learning the TDD workflow step by step
- Practicing writing a failing test before writing any production code
- Demonstrating how tests drive API design and code structure
- Doing a TDD kata to build fluency with the Red-Green-Refactor loop
- Explaining why TDD reduces bugs and improves maintainability

---

## Concepts Covered

1. **The Three Laws of TDD** — the strict rules that govern the cycle
2. **Red-Green-Refactor** — the fundamental loop
3. **Test anatomy** — Arrange / Act / Assert (AAA)
4. **One assertion per test** — why narrow tests are easier to debug
5. **Fake it till you make it** — the role of the simplest passing implementation
6. **Triangulation** — adding a second test case to force a general solution
7. **Refactoring under a green suite** — how to improve design safely
8. **Test doubles** — stubs, spies, and mocks and when to reach for each

---

## The Three Laws of TDD

| Law       | Rule                                                                                               |
| --------- | -------------------------------------------------------------------------------------------------- |
| **Law 1** | You may not write production code until you have written a failing unit test.                      |
| **Law 2** | You may not write more of a unit test than is sufficient to fail (including compile failures).     |
| **Law 3** | You may not write more production code than is sufficient to make the currently failing test pass. |

These laws keep the feedback loop tight — typically under 60 seconds per cycle.

---

## Red-Green-Refactor Cycle

```
        ┌──────────────────────────────────────┐
        │                                      │
        ▼                                      │
   ┌─────────┐      Write the           ┌──────────────┐
   │   RED   │ ──── smallest test ────▶ │   Failing    │
   │         │      that describes      │   test suite │
   └─────────┘      new behavior        └──────────────┘
        │
        │ Write the minimum code
        │ to make the test pass
        ▼
   ┌─────────┐                          ┌──────────────┐
   │  GREEN  │ ──────────────────────▶  │   Passing    │
   │         │                          │   test suite │
   └─────────┘                          └──────────────┘
        │
        │ Improve structure — no new
        │ behavior. Tests stay green.
        ▼
   ┌─────────┐                          ┌──────────────┐
   │REFACTOR │ ──────────────────────▶  │   Still      │
   │         │                          │   passing    │
   └─────────┘                          └──────────────┘
        │
        └──────────────── repeat ───────────────────────▶
```

### Phase-by-phase rules

**Red phase**

- Write _one_ test that captures the next small piece of behavior.
- The test must fail — if it passes immediately, it is not testing new behavior.
- Failing _to compile_ counts as red.

**Green phase**

- Write the **minimum** code that makes the test pass — even if it looks silly (hardcoding a return value is fine at this stage).
- Do not add code that is not required by a currently failing test.

**Refactor phase**

- Clean up duplication, extract helpers, rename for clarity.
- Run the test suite after every change. If anything goes red, undo.
- Stop refactoring when the code expresses intent cleanly.

---

## Test Anatomy — Arrange / Act / Assert

Every well-formed test follows three sections:

```ts
it("returns the sum of two positive numbers", () => {
  // Arrange — set up inputs and collaborators
  const calculator = new Calculator();

  // Act — call the unit under test
  const result = calculator.add(3, 4);

  // Assert — verify the outcome
  expect(result).toBe(7);
});
```

| Section     | Purpose                                           |
| ----------- | ------------------------------------------------- |
| **Arrange** | Create the system under test (SUT) and its inputs |
| **Act**     | Call exactly one method / function                |
| **Assert**  | Make exactly one logical claim about the outcome  |

**One assertion per test** is a guideline, not a law — but multiple assertions on the same concept in one test are acceptable; multiple _concepts_ in one test are not.

---

## Fake It Till You Make It

When you first write a failing test, the simplest code that passes is often a hardcoded constant:

```ts
// Test
it("returns 0 for an empty string", () => {
  expect(add("")).toBe(0);
});

// Simplest green implementation
function add(s: string): number {
  return 0; // fake — hardcoded
}
```

This is not cheating — it makes the test pass, which is the goal of the green phase. The next test forces the fake away.

---

## Triangulation

Write a second test that cannot pass with the current hardcoded value. This forces you to generalise:

```ts
it("returns the number itself for a single-number string", () => {
  expect(add("5")).toBe(5); // 0 fails here — must parse now
});
```

Now the fake `return 0` breaks. You are forced to write real parsing logic. Triangulation is the technique of adding test cases until the only solution is the correct general one.

---

## Test Doubles

| Type     | What it does                          | When to use                                     |
| -------- | ------------------------------------- | ----------------------------------------------- |
| **Stub** | Returns a fixed value                 | Replace a slow or external dependency (DB, API) |
| **Spy**  | Records calls, lets real logic run    | Verify a side effect happened                   |
| **Mock** | Pre-programmed with expectations      | Strict interaction testing                      |
| **Fake** | Working but simplified implementation | In-memory DB, in-process file system            |

Use the **simplest double** that satisfies the test. Prefer stubs and fakes; reach for mocks only when interaction matters.

---

## Problem Structure — TDD Kata

Each exercise below follows this format:

1. **Goal** — what the finished function does
2. **Starting point** — an empty function signature and an empty test file
3. **Steps** — ordered list of tests to write, one at a time
4. **Rules** — constraints that prevent cheating the cycle

You work through steps sequentially. For each step:

- Write the test (Red)
- Write the minimum code (Green)
- Refactor if needed
- Move to the next step

---

## Kata 1 — String Calculator

> Classic TDD kata by Roy Osherove. Excellent for beginners.

### Goal

Implement `add(input: string): number` that sums numbers inside a string.

### Starting Point

```ts
// src/stringCalculator.ts
export function add(input: string): number {
  throw new Error("not implemented");
}
```

```ts
// src/stringCalculator.test.ts
import { describe, it, expect } from "vitest";
import { add } from "./stringCalculator";

describe("String Calculator", () => {
  // tests go here — one at a time
});
```

### Steps (write one test per step — do not read ahead)

**Step 1 — Empty string returns 0**

```ts
it("returns 0 for an empty string", () => {
  expect(add("")).toBe(0);
});
```

**Step 2 — Single number returns that number**

```ts
it("returns the number for a single-value string", () => {
  expect(add("1")).toBe(1);
});
```

**Step 3 — Two comma-separated numbers**

```ts
it("returns the sum of two comma-separated numbers", () => {
  expect(add("1,2")).toBe(3);
});
```

**Step 4 — Handle newlines as delimiters (in addition to commas)**

```ts
it("handles newlines as delimiters", () => {
  expect(add("1\n2,3")).toBe(6);
});
```

**Step 5 — Custom single-character delimiter**

The input may start with `//[delimiter]\n` to declare a custom delimiter:

```ts
it("supports a custom delimiter declared in the header", () => {
  expect(add("//;\n1;2")).toBe(3);
});
```

**Step 6 — Negative numbers throw**

```ts
it("throws for negative numbers, listing all negatives in the message", () => {
  expect(() => add("1,-2,3,-4")).toThrow("negatives not allowed: -2, -4");
});
```

**Step 7 — Numbers > 1000 are ignored**

```ts
it("ignores numbers greater than 1000", () => {
  expect(add("2,1001")).toBe(2);
  expect(add("2,1000")).toBe(1002); // 1000 is NOT ignored
});
```

### Reference Solution

<details>
<summary>Only read after completing all steps</summary>

```ts
export function add(input: string): number {
  if (input === "") return 0;

  let delimiter = /,|\n/;
  let body = input;

  if (input.startsWith("//")) {
    const headerEnd = input.indexOf("\n");
    const declared = input.slice(2, headerEnd);
    delimiter = new RegExp(escapeRegExp(declared));
    body = input.slice(headerEnd + 1);
  }

  const numbers = body.split(delimiter).map(Number);

  const negatives = numbers.filter((n) => n < 0);
  if (negatives.length > 0) {
    throw new Error(`negatives not allowed: ${negatives.join(", ")}`);
  }

  return numbers.filter((n) => n <= 1000).reduce((sum, n) => sum + n, 0);
}

function escapeRegExp(s: string): string {
  return s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}
```

</details>

---

## Kata 2 — Password Validator

> Good for practising triangulation and boundary conditions.

### Goal

Implement `validatePassword(password: string): ValidationResult`:

```ts
interface ValidationResult {
  valid: boolean;
  errors: string[];
}
```

### Rules

| Rule              | Condition                     |
| ----------------- | ----------------------------- |
| Minimum length    | At least 8 characters         |
| Uppercase         | At least one uppercase letter |
| Lowercase         | At least one lowercase letter |
| Digit             | At least one digit            |
| Special character | At least one of `!@#$%^&*`    |

### Steps

**Step 1 — Empty password is invalid**

```ts
it("rejects an empty password", () => {
  const result = validatePassword("");
  expect(result.valid).toBe(false);
});
```

**Step 2 — Too short**

```ts
it("rejects a password shorter than 8 characters", () => {
  const result = validatePassword("Ab1!");
  expect(result.valid).toBe(false);
  expect(result.errors).toContain("at least 8 characters required");
});
```

**Step 3 — Missing uppercase**

```ts
it("rejects a password with no uppercase letter", () => {
  const result = validatePassword("abcdef1!");
  expect(result.errors).toContain("at least one uppercase letter required");
});
```

**Step 4 — Missing digit**

```ts
it("rejects a password with no digit", () => {
  const result = validatePassword("Abcdefgh!");
  expect(result.errors).toContain("at least one digit required");
});
```

**Step 5 — Missing special character**

```ts
it("rejects a password with no special character", () => {
  const result = validatePassword("Abcdefg1");
  expect(result.errors).toContain(
    "at least one special character required (!@#$%^&*)",
  );
});
```

**Step 6 — Valid password**

```ts
it("accepts a password that satisfies all rules", () => {
  const result = validatePassword("Abcdef1!");
  expect(result.valid).toBe(true);
  expect(result.errors).toHaveLength(0);
});
```

**Step 7 — All errors reported at once**

```ts
it("reports all broken rules simultaneously", () => {
  const result = validatePassword("abc");
  expect(result.errors.length).toBeGreaterThanOrEqual(4);
});
```

### Reference Solution

<details>
<summary>Only read after completing all steps</summary>

```ts
export interface ValidationResult {
  valid: boolean;
  errors: string[];
}

const RULES: Array<{ message: string; test: (p: string) => boolean }> = [
  { message: "at least 8 characters required", test: (p) => p.length >= 8 },
  {
    message: "at least one uppercase letter required",
    test: (p) => /[A-Z]/.test(p),
  },
  {
    message: "at least one lowercase letter required",
    test: (p) => /[a-z]/.test(p),
  },
  { message: "at least one digit required", test: (p) => /\d/.test(p) },
  {
    message: "at least one special character required (!@#$%^&*)",
    test: (p) => /[!@#$%^&*]/.test(p),
  },
];

export function validatePassword(password: string): ValidationResult {
  const errors = RULES.filter((r) => !r.test(password)).map((r) => r.message);
  return { valid: errors.length === 0, errors };
}
```

</details>

---

## Execution Model

Tests run with **Vitest** (or Jest — patterns are identical):

```bash
# Run in watch mode — re-runs on every file save
npx vitest --watch

# Run once
npx vitest run
```

The test runner output:

```
✓ returns 0 for an empty string                   (1ms)
✓ returns the number for a single-value string    (0ms)
× returns the sum of two comma-separated numbers  ← you are here
```

Work on exactly one failing test at a time. Do not move to the next step until the current test is green.

---

## Explanation Logic — How to Evaluate Your TDD Practice

| Signal                                               | What it means                                                                            |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| You wrote the implementation first and then the test | You are not doing TDD — tests become documentation, not design drivers                   |
| You wrote multiple tests before any code             | You have skipped the red phase loop — slow down                                          |
| The test is hard to write                            | The API design is wrong — let the test tell you what to change                           |
| You need a mock for every test                       | The code has too many dependencies — consider simplifying                                |
| Refactoring broke tests                              | The tests are coupled to implementation details — prefer testing behavior, not internals |
| You cannot think of a next test                      | You are done — ship it                                                                   |

---

## Key Rules Summary

- **Test first, always** — no production code without a red test
- **One failing test at a time** — resist the urge to write more
- **Make it pass, then make it right** — green before refactor
- **Tests are first-class code** — apply the same clean-code standards
- **Delete tests that no longer add value** — a passing test that duplicates another is noise
- **Never mock what you own** — only stub external systems (HTTP, DB, file system)
