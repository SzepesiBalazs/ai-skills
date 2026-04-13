---
name: typescript
description: "Build an interactive TypeScript concept explorer with a live type error playground in a Vue 3 app. Use when: teaching TypeScript type system concepts, building a TS challenge tool, demonstrating generics, utility types, discriminated unions, mapped types, type guards, conditional types, or the satisfies operator, creating an in-browser TypeScript sandbox that transpiles code and shows type diagnostics alongside runtime output, solving TypeScript interview problems interactively."
argument-hint: "The TypeScript concept or problem to explore (e.g. generics, utility types, discriminated unions, type guards)"
---

# Interactive TypeScript Explorer — Type System & Error Playground

## When to Use

- Building a step-by-step tool for learning TypeScript type system challenges
- Teaching generics, utility types, discriminated unions, mapped types, or type guards
- Creating an in-browser TypeScript sandbox that transpiles code, shows compiler diagnostics, and runs the output
- Building a dedicated **Type Error Playground** where users learn to read and fix real TS diagnostics
- Generating annotated explanations for TypeScript interview problems

## Architecture Overview

The app has two top-level modes toggled in the sidebar: **Concept Problems** and **Type Error Playground**.

```
┌───────────────────────────────────────────────────────────────────────────┐
│  TsExplorerView (page-level view)                                         │
│                                                                           │
│  ┌──────────────────────┐  ┌───────────────────────────────────────────┐ │
│  │  ConceptSidebar      │  │  [mode: concept]  ProblemWorkspace        │ │
│  │                      │  │                                           │ │
│  │  ● Concept Problems  │  │  ┌─────────────────────────────────────┐  │ │
│  │  ○ Type Error        │  │  │ ProblemStatement                    │  │ │
│  │    Playground        │  │  │ "What is the inferred type of x?"   │  │ │
│  │  ─────────────────── │  │  │                                     │  │ │
│  │  Type System         │  │  │ ┌─────────────────────────────────┐ │  │ │
│  │    Type inference    │  │  │ │ CodeEditor (textarea)           │ │  │ │
│  │    Union narrowing   │  │  │ │ function identity<T>(x: T): T { │ │  │ │
│  │    Literal types     │  │  │ │   return x;                     │ │  │ │
│  │                      │  │  │ │ }                               │ │  │ │
│  │  Generics            │  │  │ └─────────────────────────────────┘ │  │ │
│  │    Constraints       │  │  │                                     │  │ │
│  │    Conditional types │  │  │  [▶ Run]  [Reset]  [Reveal]         │  │ │
│  │    infer keyword     │  │  └─────────────────────────────────────┘  │ │
│  │                      │  │                                           │ │
│  │  Utility Types       │  │  ┌─────────────────────────────────────┐  │ │
│  │    Partial / Required│  │  │ OutputConsole                       │  │ │
│  │    Pick / Omit       │  │  │ ◆ TYPE  TS2322 — Type 'string' is   │  │ │
│  │    ReturnType        │  │  │         not assignable to 'number'  │  │ │
│  │    Awaited           │  │  │ > Output: 42                        │  │ │
│  │                      │  │  └─────────────────────────────────────┘  │ │
│  │  Discriminated       │  │                                           │ │
│  │  Unions              │  │  ┌─────────────────────────────────────┐  │ │
│  │    Exhaustive switch │  │  │ ExplanationPanel                    │  │ │
│  │    never type        │  │  │ ReturnType<T> extracts the return   │  │ │
│  │                      │  │  │ type of a function type T using     │  │ │
│  │  Mapped Types &      │  │  │ conditional types with `infer`.     │  │ │
│  │  Template Literals   │  │  └─────────────────────────────────────┘  │ │
│  │                      │  └───────────────────────────────────────────┘ │
│  │  Type Guards         │                                                │
│  │    typeof / in       │  ┌───────────────────────────────────────────┐ │
│  │    Custom predicates │  │  [mode: playground]  TypeErrorPlayground  │ │
│  │    satisfies         │  │                                           │ │
│  └──────────────────────┘  │  ┌──────────────────┐ ┌───────────────┐  │ │
│                             │  │ ErrorScenarioList│ │ ErrorWorkspace│  │ │
│                             │  │                  │ │               │  │ │
│                             │  │ TS2322 Mismatch  │ │ CodeEditor    │  │ │
│                             │  │ TS2345 Argument  │ │ (editable)    │  │ │
│                             │  │ TS2339 No prop   │ │ ──────────── │  │ │
│                             │  │ TS7006 Implicit  │ │ [▶ Check Fix] │  │ │
│                             │  │  any             │ │ [Reset]       │  │ │
│                             │  │ TS2366 Missing   │ │ [Reveal]      │  │ │
│                             │  │  return          │ │ ──────────── │  │ │
│                             │  │                  │ │ CheckResult  │  │ │
│                             │  │                  │ │ DiagList     │  │ │
│                             │  └──────────────────┘ └───────────────┘  │ │
│                             └───────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

Nine pieces total:

**Concept Problems mode (6):**

1. **ConceptSidebar** — groups problems by category; clicking loads the problem; toggle switches to playground mode
2. **ProblemStatement** — challenge title, prompt, and difficulty badge
3. **CodeEditor** — editable `<textarea>` pre-filled with starter TypeScript code
4. **OutputConsole** — shows both any pre-defined type diagnostic notes AND the transpiled runtime output
5. **ExplanationPanel** — shown after "Reveal"; plain-language explanation with annotated fix
6. **useTsRunner composable** — transpiles TypeScript with `ts.transpileModule`, runs the JS output with `new Function`, returns runtime output + transpile errors

**Type Error Playground mode (3):**

7. **TypeErrorPlayground** — top-level wrapper; owns the selected scenario
8. **ErrorWorkspace** — shows `CodeEditor` (editable, pre-filled with `brokenCode`), `[▶ Check Fix]` / `[Reset]` / `[Reveal]` buttons, a live `DiagnosticList` updated by the checker after each check, a pass/fail badge, and the lesson + fixed code revealed on demand
9. **ErrorScenarioList** — scrollable list of error scenarios grouped by TS error code

## Data Model

Create `src/types/ts-explorer.ts`:

```ts
export type ConceptCategory =
  | "type-system"
  | "generics"
  | "utility-types"
  | "discriminated-unions"
  | "mapped-template"
  | "type-guards";

export type Difficulty = "beginner" | "intermediate" | "advanced";

export interface TsProblem {
  id: string;
  title: string;
  category: ConceptCategory;
  difficulty: Difficulty;
  /** Short prompt shown above the editor */
  prompt: string;
  /** Starting TypeScript code in the editor */
  starterCode: string;
  /** Expected runtime output lines after transpilation */
  expectedOutput: string[];
  /** Plain-language explanation shown on Reveal */
  explanation: string;
  /** Annotated solution code shown on Reveal */
  solutionCode: string;
  /** Key concept tags for filtering */
  tags: string[];
}

export interface OutputLine {
  kind: "log" | "error" | "warn" | "type-error";
  value: string;
}

export interface RunResult {
  lines: OutputLine[];
  transpileError: string | null;
  /** true if every expectedOutput line matches in order; null if no check defined */
  passed: boolean | null;
}

// ── Type Error Playground ─────────────────────────────────────────────────

export interface TsDiagnostic {
  /** TS error code, e.g. 2322 */
  code: number;
  /** Plain-text message as the compiler would emit it */
  message: string;
  /** 1-based line number the squiggly should appear on */
  line: number;
}

export interface ErrorScenario {
  id: string;
  /** Primary error code this scenario teaches */
  errorCode: number;
  title: string;
  /** Short challenge prompt shown above the editor — tells the user what to fix */
  prompt: string;
  /** TypeScript code with intentional type errors — pre-fills the editor */
  brokenCode: string;
  /** Pre-computed diagnostics shown before the user makes any edits */
  diagnostics: TsDiagnostic[];
  /** Corrected code shown on Reveal */
  fixedCode: string;
  /**
   * Plain-language explanation: what caused the error and the mental model
   * needed to understand and prevent it
   */
  lesson: string;

  /**
   * CheckResult — computed live by useErrorChecker after the user runs
   * [▶ Check Fix]. Not part of scenario data — lives in composable state.
   */
}
```

**Key conventions:**

- `TsProblem.starterCode` contains valid TypeScript; runtime errors (not type errors) are the learning target in Concept Problems
- `ErrorScenario.diagnostics` are the initial diagnostics shown before any editing — they match what `tsc` would emit on `brokenCode`
- After the user edits and clicks `[▶ Check Fix]`, `useErrorChecker` runs a real type-check and replaces the displayed diagnostics with live results
- `TsDiagnostic.line` drives CSS-based squiggly underline highlighting in the editor

## Problem Definitions

Create `src/data/ts-problems.ts`:

```ts
import type { TsProblem } from "@/types/ts-explorer";

export const problems: TsProblem[] = [
  // ── Type System ──────────────────────────────────────────────────────────
  {
    id: "type-inference-gotcha",
    title: "Widened inference vs. literal types",
    category: "type-system",
    difficulty: "intermediate",
    prompt:
      "TypeScript infers `direction` as `string`, not `'left' | 'right'`. Why? Fix it so `move` only accepts the two literal values without widening.",
    starterCode: `function move(dir: "left" | "right") {
  console.log("Moving:", dir);
}

const direction = "left"; // inferred as string
move(direction);           // TS error: Argument of type 'string' is not assignable to parameter of type '"left" | "right"'

// Fix: use a const assertion or type annotation
`,
    expectedOutput: ["Moving: left"],
    explanation:
      "When you write `const direction = 'left'`, TypeScript widens the type to `string` by default. To keep the narrow literal type, use `as const` (`const direction = 'left' as const`) or annotate explicitly (`const direction: 'left' = 'left'`). `as const` is preferred because it scales to objects and arrays.",
    solutionCode: `function move(dir: "left" | "right") {
  console.log("Moving:", dir);
}

const direction = "left" as const; // type is 'left', not string
move(direction); // OK`,
    tags: ["literal types", "as const", "type widening", "inference"],
  },
  {
    id: "union-narrowing",
    title: "Narrowing a string | number union",
    category: "type-system",
    difficulty: "beginner",
    prompt:
      "Implement `formatValue` so it logs the value padded to 6 chars if it's a string, or multiplied by 10 if it's a number. Use a type guard to narrow the union.",
    starterCode: `function formatValue(value: string | number): void {
  // your code here
}

formatValue("hi");   // "hi    "
formatValue(7);      // 70`,
    expectedOutput: ["hi    ", "70"],
    explanation:
      "`typeof value === 'string'` is a type guard — inside the `if` branch TypeScript narrows `value` to `string`, giving access to `.padEnd`. In the `else` branch it's narrowed to `number`. This is control-flow based narrowing: no cast needed.",
    solutionCode: `function formatValue(value: string | number): void {
  if (typeof value === "string") {
    console.log(value.padEnd(6));
  } else {
    console.log(value * 10);
  }
}

formatValue("hi");
formatValue(7);`,
    tags: ["typeof", "narrowing", "union", "type guard", "control flow"],
  },

  // ── Generics ─────────────────────────────────────────────────────────────
  {
    id: "generic-identity",
    title: "Generic identity — preserving the literal type",
    category: "generics",
    difficulty: "beginner",
    prompt:
      "Write a generic `identity<T>` function. Then call it twice: once letting TS infer `T`, once passing `T` explicitly. Log the output. Notice how inference preserves the literal type.",
    starterCode: `function identity<T>(value: T): T {
  return value;
}

const a = identity(42);           // T inferred as 42 (literal)
const b = identity<string>("hi"); // T explicitly string
console.log(a);
console.log(b);`,
    expectedOutput: ["42", "hi"],
    explanation:
      "Generics let you write one function that works for any type while preserving type information. When you pass `42`, TypeScript infers `T = 42` (the literal). When you write `identity<string>('hi')`, you widen to `string`. The key insight: generics tie the return type to the input type without losing precision.",
    solutionCode: `function identity<T>(value: T): T {
  return value;
}

const a = identity(42);
const b = identity<string>("hi");
console.log(a);
console.log(b);`,
    tags: ["generics", "type inference", "explicit type parameter"],
  },
  {
    id: "generic-constraint",
    title: "Generic constraint with extends",
    category: "generics",
    difficulty: "intermediate",
    prompt:
      "Write a generic `getLength<T extends { length: number }>` function that returns the length of anything with a length property. Call it with a string and an array.",
    starterCode: `function getLength<T>(thing: T): number {
  return thing.length; // TS error: Property 'length' does not exist on type 'T'
}

console.log(getLength("hello")); // 5
console.log(getLength([1, 2, 3])); // 3`,
    expectedOutput: ["5", "3"],
    explanation:
      "Without a constraint, TypeScript can't assume `T` has a `length` property. Adding `T extends { length: number }` tells the compiler: 'T can be any type, as long as it has a numeric `length`'. The constraint is structural — any object with a `length: number` satisfies it.",
    solutionCode: `function getLength<T extends { length: number }>(thing: T): number {
  return thing.length;
}

console.log(getLength("hello")); // 5
console.log(getLength([1, 2, 3])); // 3`,
    tags: ["generics", "extends", "constraint", "structural typing"],
  },
  {
    id: "conditional-infer",
    title: "Extracting a return type with infer",
    category: "generics",
    difficulty: "advanced",
    prompt:
      "Implement `MyReturnType<T>` — a re-implementation of TypeScript's built-in `ReturnType` utility — using conditional types and `infer`. Then extract the return type of `getUser`.",
    starterCode: `type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { id: 1, name: "Alice" };
}

type User = MyReturnType<typeof getUser>;
// User should be: { id: number; name: string }

const user: User = { id: 2, name: "Bob" };
console.log(user.name);`,
    expectedOutput: ["Bob"],
    explanation:
      "`T extends (...args: any[]) => infer R` checks whether `T` is a function type, and if so, captures its return type in the local variable `R` using `infer`. `infer` only works inside conditional types. This is exactly how `ReturnType<T>` is defined in TypeScript's lib. `never` is the fallback for non-function types.",
    solutionCode: `type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { id: 1, name: "Alice" };
}

type User = MyReturnType<typeof getUser>;
const user: User = { id: 2, name: "Bob" };
console.log(user.name);`,
    tags: ["infer", "conditional types", "ReturnType", "advanced generics"],
  },

  // ── Utility Types ─────────────────────────────────────────────────────────
  {
    id: "partial-required",
    title: "Partial updates with Partial and Required",
    category: "utility-types",
    difficulty: "beginner",
    prompt:
      "Implement `updateSettings` that takes the current settings and a partial patch object, and returns the merged result fully typed. Then call it and log the merged host.",
    starterCode: `interface AppSettings {
  host: string;
  port: number;
  debug: boolean;
}

function updateSettings(
  current: AppSettings,
  patch: Partial<AppSettings>
): AppSettings {
  return { ...current, ...patch };
}

const defaults: AppSettings = { host: "localhost", port: 3000, debug: false };
const updated = updateSettings(defaults, { port: 8080, debug: true });
console.log(updated.host);
console.log(updated.port);`,
    expectedOutput: ["localhost", "8080"],
    explanation:
      "`Partial<T>` makes every property optional by mapping over all keys with `?`. This is the standard pattern for update/patch functions — the caller only provides what changed. `Required<T>` is the inverse: removes all `?` modifiers. Both are built-in mapped types in TypeScript's lib.",
    solutionCode: `interface AppSettings {
  host: string;
  port: number;
  debug: boolean;
}

function updateSettings(
  current: AppSettings,
  patch: Partial<AppSettings>
): AppSettings {
  return { ...current, ...patch };
}

const defaults: AppSettings = { host: "localhost", port: 3000, debug: false };
const updated = updateSettings(defaults, { port: 8080, debug: true });
console.log(updated.host);
console.log(updated.port);`,
    tags: ["Partial", "Required", "utility types", "patch pattern"],
  },
  {
    id: "pick-omit",
    title: "Shape types with Pick and Omit",
    category: "utility-types",
    difficulty: "intermediate",
    prompt:
      "Given the `User` type, create `PublicUser` using `Omit` to remove sensitive fields, and `UserPreview` using `Pick` for just the id and name. Log both.",
    starterCode: `interface User {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
}

type PublicUser = Omit<User, "passwordHash" | "email">;
type UserPreview = Pick<User, "id" | "name">;

const user: User = { id: 1, name: "Alice", email: "a@b.com", passwordHash: "abc123" };

const pub: PublicUser = { id: user.id, name: user.name };
const preview: UserPreview = { id: user.id, name: user.name };

console.log(pub.name);
console.log(preview.id);`,
    expectedOutput: ["Alice", "1"],
    explanation:
      "`Pick<T, K>` keeps only the listed keys; `Omit<T, K>` keeps everything except them. Both construct new object types structurally — no runtime behaviour, compile-time only. Use `Omit` when you want to remove sensitive or internals-only fields before passing data across a boundary (e.g. to the UI layer).",
    solutionCode: `interface User {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
}

type PublicUser = Omit<User, "passwordHash" | "email">;
type UserPreview = Pick<User, "id" | "name">;

const user: User = { id: 1, name: "Alice", email: "a@b.com", passwordHash: "abc123" };
const pub: PublicUser = { id: user.id, name: user.name };
const preview: UserPreview = { id: user.id, name: user.name };
console.log(pub.name);
console.log(preview.id);`,
    tags: ["Pick", "Omit", "utility types", "structural typing"],
  },
  {
    id: "record-type",
    title: "Typed look-up maps with Record",
    category: "utility-types",
    difficulty: "beginner",
    prompt:
      "Build a `statusLabels` map from HTTP status codes to display strings using `Record`. Ensure both keys and values are strongly typed.",
    starterCode: `type HttpStatus = 200 | 404 | 500;

const statusLabels: Record<HttpStatus, string> = {
  200: "OK",
  404: "Not Found",
  500: "Internal Server Error",
};

console.log(statusLabels[200]);
console.log(statusLabels[404]);`,
    expectedOutput: ["OK", "Not Found"],
    explanation:
      "`Record<K, V>` is a shorthand for `{ [P in K]: V }`. When `K` is a union (like `200 | 404 | 500`), TypeScript enforces that all three keys are present — missing one is a compile error. This makes `Record` a great pattern for exhaustive look-up tables.",
    solutionCode: `type HttpStatus = 200 | 404 | 500;

const statusLabels: Record<HttpStatus, string> = {
  200: "OK",
  404: "Not Found",
  500: "Internal Server Error",
};
console.log(statusLabels[200]);
console.log(statusLabels[404]);`,
    tags: ["Record", "utility types", "mapped type", "exhaustive"],
  },

  // ── Discriminated Unions ──────────────────────────────────────────────────
  {
    id: "discriminated-union-basic",
    title: "Discriminated union for a result type",
    category: "discriminated-unions",
    difficulty: "intermediate",
    prompt:
      "Define a `Result<T>` discriminated union with `{ ok: true; value: T }` and `{ ok: false; error: string }`. Write `unwrap` that logs the value or error. TypeScript must narrow automatically — no casts.",
    starterCode: `type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function unwrap<T>(result: Result<T>): void {
  if (result.ok) {
    console.log("Value:", result.value);
  } else {
    console.log("Error:", result.error);
  }
}

unwrap({ ok: true, value: 42 });
unwrap({ ok: false, error: "Not found" });`,
    expectedOutput: ["Value: 42", "Error: Not found"],
    explanation:
      "The `ok` field is the discriminant — a literal type that uniquely identifies each branch. When TypeScript sees `result.ok === true`, it narrows the union to only the first branch, making `result.value` available without a cast. This pattern is safer than throwing exceptions for expected failures.",
    solutionCode: `type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function unwrap<T>(result: Result<T>): void {
  if (result.ok) {
    console.log("Value:", result.value);
  } else {
    console.log("Error:", result.error);
  }
}

unwrap({ ok: true, value: 42 });
unwrap({ ok: false, error: "Not found" });`,
    tags: [
      "discriminated union",
      "narrowing",
      "result type",
      "pattern matching",
    ],
  },
  {
    id: "exhaustive-never",
    title: "Exhaustive checking with never",
    category: "discriminated-unions",
    difficulty: "advanced",
    prompt:
      "Add a `'pending'` state to `Status` and update `describe` so TypeScript enforces that every branch is handled. Use a `never` assertion to make new unhandled variants a compile error.",
    starterCode: `type Status = "active" | "inactive"; // add 'pending' here

function describe(s: Status): void {
  switch (s) {
    case "active":
      console.log("Active");
      break;
    case "inactive":
      console.log("Inactive");
      break;
    default: {
      const _exhaustive: never = s; // compile error if a case is missing
      throw new Error("Unhandled status: " + _exhaustive);
    }
  }
}

describe("active");
describe("inactive");`,
    expectedOutput: ["Active", "Inactive"],
    explanation:
      "Assigning `s` to a variable typed `never` in the default branch makes TypeScript check that all variants are handled. If you add `'pending'` but forget a case, the assignment `const _: never = s` becomes a compile error because `s` would have type `'pending'` there, not `never`. This is exhaustive checking.",
    solutionCode: `type Status = "active" | "inactive" | "pending";

function describe(s: Status): void {
  switch (s) {
    case "active":   console.log("Active");   break;
    case "inactive": console.log("Inactive"); break;
    case "pending":  console.log("Pending");  break;
    default: {
      const _exhaustive: never = s;
      throw new Error("Unhandled status: " + _exhaustive);
    }
  }
}

describe("active");
describe("inactive");`,
    tags: ["never", "exhaustive", "switch", "discriminated union"],
  },

  // ── Mapped Types & Template Literals ──────────────────────────────────────
  {
    id: "mapped-readonly",
    title: "Building DeepReadonly with mapped types",
    category: "mapped-template",
    difficulty: "advanced",
    prompt:
      "The built-in `Readonly<T>` is shallow — nested objects stay mutable. Implement `DeepReadonly<T>` that recursively makes all nested properties readonly.",
    starterCode: `type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

interface Config {
  server: { host: string; port: number };
  debug: boolean;
}

const cfg: DeepReadonly<Config> = {
  server: { host: "localhost", port: 3000 },
  debug: false,
};

// cfg.server.host = "prod"; // TS error — nested readonly works
console.log(cfg.server.host);
console.log(cfg.debug);`,
    expectedOutput: ["localhost", "false"],
    explanation:
      "Mapped types iterate over `keyof T` using `[K in keyof T]`. The `readonly` modifier is added to each key. The conditional `T[K] extends object ? DeepReadonly<T[K]> : T[K]` recurses into nested objects. Functions and primitive values pass through unchanged. This pattern is the foundation of many Vue and Redux immutability utilities.",
    solutionCode: `type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

interface Config {
  server: { host: string; port: number };
  debug: boolean;
}

const cfg: DeepReadonly<Config> = {
  server: { host: "localhost", port: 3000 },
  debug: false,
};

console.log(cfg.server.host);
console.log(cfg.debug);`,
    tags: ["mapped types", "readonly", "recursive", "immutability"],
  },
  {
    id: "template-literal-events",
    title: "Template literal event name types",
    category: "mapped-template",
    difficulty: "intermediate",
    prompt:
      "Use a template literal type to derive strongly-typed event handler names from a union of event names. Then implement `on` so the handler name is always `'on' + Capitalize<EventName>`.",
    starterCode: `type EventName = "click" | "focus" | "blur";
type HandlerName = \`on\${Capitalize<EventName>}\`;
// HandlerName = 'onClick' | 'onFocus' | 'onBlur'

type EventMap = {
  [K in EventName as \`on\${Capitalize<K>}\`]: () => void;
};

const handlers: EventMap = {
  onClick: () => console.log("clicked"),
  onFocus: () => console.log("focused"),
  onBlur:  () => console.log("blurred"),
};

handlers.onClick();
handlers.onFocus();`,
    expectedOutput: ["clicked", "focused"],
    explanation:
      "Template literal types combine string literal unions with string manipulation intrinsics (`Capitalize`, `Uppercase`, `Lowercase`, `Uncapitalize`). `[K in EventName as \`on\${Capitalize<K>}\`]` remaps the key name during the mapped type iteration — a feature called 'key remapping'. This is the pattern Vue and many event emitters use to derive typed handler names at compile time.",
    solutionCode: `type EventName = "click" | "focus" | "blur";

type EventMap = {
  [K in EventName as \`on\${Capitalize<K>}\`]: () => void;
};

const handlers: EventMap = {
  onClick: () => console.log("clicked"),
  onFocus: () => console.log("focused"),
  onBlur:  () => console.log("blurred"),
};

handlers.onClick();
handlers.onFocus();`,
    tags: [
      "template literal types",
      "Capitalize",
      "key remapping",
      "mapped types",
    ],
  },

  // ── Type Guards ───────────────────────────────────────────────────────────
  {
    id: "custom-type-predicate",
    title: "Custom type predicate function",
    category: "type-guards",
    difficulty: "intermediate",
    prompt:
      "Write `isError(value: unknown): value is Error` so TypeScript narrows the type inside `if (isError(v))` blocks. Test it with an Error and a plain string.",
    starterCode: `function isError(value: unknown): value is Error {
  return value instanceof Error;
}

function handle(value: unknown): void {
  if (isError(value)) {
    console.log("Error:", value.message); // .message available — narrowed to Error
  } else {
    console.log("Value:", String(value));
  }
}

handle(new Error("oops"));
handle("hello");`,
    expectedOutput: ["Error: oops", "Value: hello"],
    explanation:
      "A function returning `value is SomeType` is a type predicate. When TypeScript sees the call in an `if` condition, it narrows the argument to `SomeType` inside the true branch. This is more powerful than inline `typeof`/`instanceof` checks because it centralises the guard logic and works with complex, multi-property checks.",
    solutionCode: `function isError(value: unknown): value is Error {
  return value instanceof Error;
}

function handle(value: unknown): void {
  if (isError(value)) {
    console.log("Error:", value.message);
  } else {
    console.log("Value:", String(value));
  }
}

handle(new Error("oops"));
handle("hello");`,
    tags: ["type predicate", "is", "instanceof", "unknown", "narrowing"],
  },
  {
    id: "satisfies-operator",
    title: "satisfies for inference without widening",
    category: "type-guards",
    difficulty: "intermediate",
    prompt:
      "Use `satisfies` to validate that `palette` matches `Record<string, string | string[]>` while preserving the narrow inferred type of each value so `.toUpperCase()` and `.join` are accessible without casts.",
    starterCode: `type Palette = Record<string, string | string[]>;

const palette = {
  red:   "#ff0000",
  green: ["#00ff00", "#33ff33"],
  blue:  "#0000ff",
} satisfies Palette;

// These work because satisfies preserves the narrow inferred types:
console.log(palette.red.toUpperCase());
console.log(palette.green.join(", "));`,
    expectedOutput: ["#FF0000", "#00ff00, #33ff33"],
    explanation:
      "`satisfies` (TS 4.9+) validates that an expression matches a type without widening the inferred type of the variable. If you used `: Palette` instead, `palette.red` would be `string | string[]` and `.toUpperCase()` would be a type error. With `satisfies`, the constraint is checked but the narrow type (`string` for `red`, `string[]` for `green`) is preserved.",
    solutionCode: `type Palette = Record<string, string | string[]>;

const palette = {
  red:   "#ff0000",
  green: ["#00ff00", "#33ff33"],
  blue:  "#0000ff",
} satisfies Palette;

console.log(palette.red.toUpperCase());
console.log(palette.green.join(", "));`,
    tags: ["satisfies", "type widening", "inference", "TS 4.9"],
  },
];
```

## Error Scenario Definitions

Create `src/data/ts-error-scenarios.ts`:

```ts
import type { ErrorScenario } from "@/types/ts-explorer";

export const errorScenarios: ErrorScenario[] = [
  // ── TS2322 ────────────────────────────────────────────────────────────────
  {
    id: "ts2322-basic",
    errorCode: 2322,
    title: "Assigning a string to a number",
    prompt:
      "Fix the type error: assign a value of the correct type to `count`.",
    brokenCode: `let count: number = 0;
count = "five"; // TS2322`,
    diagnostics: [
      {
        code: 2322,
        message: "Type 'string' is not assignable to type 'number'.",
        line: 2,
      },
    ],
    fixedCode: `let count: number = 0;
count = 5; // OK — a real number`,
    lesson:
      "TS2322 fires whenever you try to assign a value to a variable or property whose declared type doesn't include the type of the value. The fix is either to change the value's type (cast or convert) or to widen the declared type (e.g. `number | string`).",
  },
  {
    id: "ts2322-object-shape",
    errorCode: 2322,
    title: "Object missing a required property",
    prompt: "Fix the type error: make `p` satisfy the `Point` interface.",
    brokenCode: `interface Point {
  x: number;
  y: number;
}

const p: Point = { x: 10 }; // TS2322 — y is missing`,
    diagnostics: [
      {
        code: 2322,
        message:
          "Type '{ x: number; }' is not assignable to type 'Point'. Property 'y' is missing in type '{ x: number; }' but required in type 'Point'.",
        line: 6,
      },
    ],
    fixedCode: `const p: Point = { x: 10, y: 0 }; // OK`,
    lesson:
      "When assigning an object literal to a typed variable, TypeScript requires all non-optional properties to be present. To make a property optional, add `?` to its declaration (`y?: number`). Otherwise, provide the missing property in the literal.",
  },

  // ── TS2345 ────────────────────────────────────────────────────────────────
  {
    id: "ts2345-argument",
    errorCode: 2345,
    title: "Passing the wrong type to a function",
    prompt: "Fix the type error: call `double` with the correct argument type.",
    brokenCode: `function double(n: number): number {
  return n * 2;
}

const result = double("4"); // TS2345`,
    diagnostics: [
      {
        code: 2345,
        message:
          "Argument of type 'string' is not assignable to parameter of type 'number'.",
        line: 5,
      },
    ],
    fixedCode: `const result = double(4);        // OK — pass a number
// or convert: double(Number("4"))`,
    lesson:
      "TS2345 is the argument-passing version of TS2322. The call site passes a type the function's parameter signature doesn't accept. Fix by converting the value at the call site (`Number(str)`, `parseInt`) or by widening the parameter type if that makes sense for the API.",
  },

  // ── TS2339 ────────────────────────────────────────────────────────────────
  {
    id: "ts2339-no-property",
    errorCode: 2339,
    title: "Accessing a property that doesn't exist on the type",
    prompt:
      "Fix the type error: make `year` accessible on `car` without using `any`.",
    brokenCode: `interface Car {
  make: string;
  model: string;
}

const car: Car = { make: "Toyota", model: "Corolla" };
console.log(car.year); // TS2339`,
    diagnostics: [
      {
        code: 2339,
        message: "Property 'year' does not exist on type 'Car'.",
        line: 7,
      },
    ],
    fixedCode: `interface Car {
  make: string;
  model: string;
  year?: number; // add the property to the type
}
console.log(car.year); // OK (possibly undefined)`,
    lesson:
      "TS2339 means you're reading a property that TypeScript doesn't know exists on the type. Either add the property to the interface/type, or narrow to a subtype where the property exists, or use optional chaining: `car?.year`. Never use a cast (`(car as any).year`) just to silence this error — it hides real mistakes.",
  },
  {
    id: "ts2339-union-access",
    errorCode: 2339,
    title: "Accessing a property only present on one union member",
    prompt:
      "Fix the type error: access `radius` safely by narrowing the union before using it.",
    brokenCode: `type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };

function area(s: Shape): number {
  return Math.PI * s.radius ** 2; // TS2339 — radius only on circle
}`,
    diagnostics: [
      {
        code: 2339,
        message:
          "Property 'radius' does not exist on type 'Shape'. Property 'radius' does not exist on type '{ kind: \"rect\"; width: number; height: number; }'.",
        line: 6,
      },
    ],
    fixedCode: `function area(s: Shape): number {
  if (s.kind === "circle") {
    return Math.PI * s.radius ** 2; // narrowed to circle branch
  }
  return s.width * s.height;
}`,
    lesson:
      "When a property only exists on some members of a union, TypeScript won't let you access it before narrowing. Use the discriminant (`kind`) to narrow to the specific branch, then the property is safely accessible. This is the primary motivation for discriminated unions.",
  },

  // ── TS7006 ────────────────────────────────────────────────────────────────
  {
    id: "ts7006-implicit-any",
    errorCode: 7006,
    title: "Parameter implicitly has an 'any' type",
    prompt:
      "Fix the type error: give every parameter an explicit type annotation.",
    brokenCode: `// tsconfig: "noImplicitAny": true

function greet(name) { // TS7006
  return "Hello, " + name;
}`,
    diagnostics: [
      {
        code: 7006,
        message: "Parameter 'name' implicitly has an 'any' type.",
        line: 3,
      },
    ],
    fixedCode: `function greet(name: string): string {
  return "Hello, " + name;
}`,
    lesson:
      "With `noImplicitAny: true` in `tsconfig.json` (which strict mode enables), TypeScript requires every parameter to have an explicit type or a type it can infer from a default value. `any` is the escape hatch that defeats type checking — TypeScript warns when you accidentally opt into it via omission rather than intent.",
  },

  // ── TS2366 ────────────────────────────────────────────────────────────────
  {
    id: "ts2366-missing-return",
    errorCode: 2366,
    title: "Function lacks ending return statement",
    prompt:
      "Fix the type error: ensure every code path in `divide` returns a `number` (or change the return type).",
    brokenCode: `function divide(a: number, b: number): number {
  if (b !== 0) {
    return a / b;
  }
  // TS2366 — no return for the b === 0 case
}`,
    diagnostics: [
      {
        code: 2366,
        message:
          "Function lacks ending return statement and return type does not include 'undefined'.",
        line: 1,
      },
    ],
    fixedCode: `function divide(a: number, b: number): number {
  if (b !== 0) {
    return a / b;
  }
  throw new Error("Division by zero");
  // Alternative: return NaN; or change return type to number | undefined
}`,
    lesson:
      "TypeScript requires that all code paths in a function return the declared type. A missing `else` or `default` branch means some executions fall off the end without returning anything (implicitly `undefined`). Fix by covering all paths: add a final `return`, `throw`, or change the return type to include `undefined`.",
  },

  // ── TS2769 ────────────────────────────────────────────────────────────────
  {
    id: "ts2769-overload-mismatch",
    errorCode: 2769,
    title: "No overload matches this call",
    prompt:
      "Fix the type error: call `Array.from` with an argument that matches one of its overload signatures.",
    brokenCode: `// Array.from is overloaded — one form takes an iterable,
// another takes { length } + mapFn

const arr = Array.from({ length: 3 }, (_, i) => i * 2);
// Fine ^ but this:
const bad = Array.from(42); // TS2769 — 42 is not iterable or array-like`,
    diagnostics: [
      {
        code: 2769,
        message:
          "No overload matches this call. Overload 1 of 2, '(arrayLike: ArrayLike<unknown>): unknown[]', gave the following error: Argument of type 'number' is not assignable to parameter of type 'ArrayLike<unknown>'. Overload 2 of 2, '(iterable: Iterable<unknown> | ArrayLike<unknown>): unknown[]', gave the following error: Argument of type 'number' is not assignable to parameter of type 'Iterable<unknown> | ArrayLike<unknown>'.",
        line: 6,
      },
    ],
    fixedCode: `const arr = Array.from({ length: 3 }, (_, i) => i * 2); // [0, 2, 4]
// or:
const arr2 = Array.from([1, 2, 3]);       // from iterable
const arr3 = Array.from("hello");         // from string (iterable)`,
    lesson:
      "TS2769 means none of a function's overloads accept the types you provided. TypeScript lists every overload attempt and why each failed. Read from the bottom up — the last attempted overload is usually the most relevant one. The fix is to match one of the declared signatures.",
  },
];
```

## useTsRunner Composable

Create `src/composables/useTsRunner.ts`:

```ts
import { ref } from "vue";
import type { OutputLine, RunResult, TsProblem } from "@/types/ts-explorer";
// TypeScript compiler is a standard npm dependency in any TS project
import ts from "typescript";

export function useTsRunner(problem: TsProblem) {
  const output = ref<OutputLine[]>([]);
  const result = ref<RunResult | null>(null);

  function run(tsCode: string): void {
    const lines: OutputLine[] = [];
    let transpileError: string | null = null;

    // Step 1 — Transpile TypeScript → JavaScript
    let jsCode: string;
    try {
      const transpileOutput = ts.transpileModule(tsCode, {
        compilerOptions: {
          target: ts.ScriptTarget.ES2020,
          module: ts.ModuleKind.None,
          strict: true,
        },
        reportDiagnostics: true,
      });

      // Surface syntax/transpile-time errors (not full type errors — those need
      // a full program; transpileModule gives syntax errors only)
      for (const diag of transpileOutput.diagnostics ?? []) {
        const msg = ts.flattenDiagnosticMessageText(diag.messageText, "\n");
        lines.push({ kind: "type-error", value: `TS${diag.code} — ${msg}` });
      }

      jsCode = transpileOutput.outputText;
    } catch (err) {
      transpileError = err instanceof Error ? err.message : String(err);
      lines.push({ kind: "error", value: transpileError });
      output.value = lines;
      result.value = { lines, transpileError, passed: null };
      return;
    }

    // Step 2 — Run the transpiled JS in a sandboxed scope
    const sandboxConsole = {
      log: (...args: unknown[]) =>
        lines.push({ kind: "log", value: args.map(String).join(" ") }),
      warn: (...args: unknown[]) =>
        lines.push({ kind: "warn", value: args.map(String).join(" ") }),
      error: (...args: unknown[]) =>
        lines.push({ kind: "error", value: args.map(String).join(" ") }),
    };

    try {
      // Scope-isolated; only sandboxed console is passed in
      const fn = new Function("console", jsCode);
      fn(sandboxConsole);
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      transpileError = msg;
      lines.push({ kind: "error", value: msg });
    }

    const runtimeLines = lines.filter((l) => l.kind === "log");
    const passed =
      problem.expectedOutput.length > 0
        ? problem.expectedOutput.every(
            (expected, i) => runtimeLines[i]?.value === expected,
          )
        : null;

    output.value = lines;
    result.value = { lines, transpileError, passed };
  }

  function reset(): void {
    output.value = [];
    result.value = null;
  }

  return { output, result, run, reset };
}
```

**Security note:** `new Function(jsCode)` has no access to the module's scope. The TypeScript source is transpiled server-side only in this sandbox — no user input is ever injected into the transpile call or the code string.

## useErrorChecker Composable

Install the virtual filesystem package first:

```bash
npm install @typescript/vfs
```

Create `src/composables/useErrorChecker.ts`:

```ts
import { ref, watch } from "vue";
import ts from "typescript";
import * as tsvfs from "@typescript/vfs";
import type { TsDiagnostic, ErrorScenario } from "@/types/ts-explorer";

// Singleton lib map — loaded once from CDN on first check, then reused
let libMapPromise: Promise<Map<string, string>> | null = null;

function getLibMap(): Promise<Map<string, string>> {
  if (!libMapPromise) {
    libMapPromise = tsvfs.createDefaultMapFromCDN(
      { target: ts.ScriptTarget.ES2020 },
      ts.version,
      true,
      ts,
    );
  }
  return libMapPromise;
}

export function useErrorChecker(scenario: ErrorScenario) {
  const userCode = ref(scenario.brokenCode);
  // Start with the pre-computed diagnostics so the editor shows squiggles immediately
  const liveDiagnostics = ref<TsDiagnostic[]>([...scenario.diagnostics]);
  const checking = ref(false);
  const passed = ref<boolean | null>(null);
  const revealed = ref(false);

  // Reset state when the scenario changes
  watch(
    () => scenario.id,
    () => {
      userCode.value = scenario.brokenCode;
      liveDiagnostics.value = [...scenario.diagnostics];
      passed.value = null;
      revealed.value = false;
    },
  );

  async function checkFix(): Promise<void> {
    checking.value = true;
    passed.value = null;

    try {
      const libMap = await getLibMap();
      const projectMap = new Map(libMap);
      projectMap.set("input.ts", userCode.value);

      const system = tsvfs.createSystem(projectMap);
      const { compilerHost } = tsvfs.createVirtualCompilerHost(
        system,
        { strict: true, noEmit: true, target: ts.ScriptTarget.ES2020 },
        ts,
      );

      const program = ts.createProgram(
        ["input.ts"],
        { strict: true, noEmit: true, target: ts.ScriptTarget.ES2020 },
        compilerHost,
      );

      const semanticDiags = program.getSemanticDiagnostics();
      const results: TsDiagnostic[] = semanticDiags
        .filter((d) => d.file?.fileName === "input.ts")
        .map((d) => ({
          code: d.code,
          message: ts.flattenDiagnosticMessageText(d.messageText, "\n"),
          line:
            d.file && d.start !== undefined
              ? d.file.getLineAndCharacterOfPosition(d.start).line + 1
              : 0,
        }));

      liveDiagnostics.value = results;
      passed.value = results.length === 0;
    } catch (err) {
      // Checker failure — surface as a diagnostic rather than crashing
      liveDiagnostics.value = [
        {
          code: 0,
          message: err instanceof Error ? err.message : String(err),
          line: 0,
        },
      ];
      passed.value = false;
    } finally {
      checking.value = false;
    }
  }

  function reset(): void {
    userCode.value = scenario.brokenCode;
    liveDiagnostics.value = [...scenario.diagnostics];
    passed.value = null;
    revealed.value = false;
  }

  return {
    userCode,
    liveDiagnostics,
    checking,
    passed,
    revealed,
    checkFix,
    reset,
  };
}
```

**How it works:**

- `getLibMap()` fetches TypeScript standard lib `.d.ts` files from unpkg CDN the first time it is called, then caches the promise so subsequent calls are instant
- `tsvfs.createVirtualCompilerHost` gives the TS compiler a fully in-memory filesystem — no Node.js `fs` needed
- `program.getSemanticDiagnostics()` runs real type checking (same as `tsc`) including TS2322, TS2345, TS2339, TS7006, etc.
- `liveDiagnostics` starts with `scenario.diagnostics` (instant, no network required) and is replaced with real results after the first `checkFix()` call

## Component: OutputConsole.vue

```vue
<script setup lang="ts">
import type { OutputLine } from "@/types/ts-explorer";
defineProps<{ lines: OutputLine[]; transpileError?: string | null }>();
</script>

<template>
  <div class="output-console" role="log" aria-label="Output" aria-live="polite">
    <p v-if="lines.length === 0" class="output-empty">
      Run your code to see output.
    </p>
    <p
      v-for="(line, i) in lines"
      :key="i"
      :class="['output-line', `output-line--${line.kind}`]"
    >
      <span v-if="line.kind === 'type-error'" aria-label="Type error"
        >◆ TYPE
      </span>
      <span v-else aria-hidden="true">&gt; </span>
      {{ line.value }}
    </p>
  </div>
</template>
```

## Component: TypeErrorPlayground.vue

The main container for the error playground mode:

```vue
<script setup lang="ts">
import { ref, computed } from "vue";
import { errorScenarios } from "@/data/ts-error-scenarios";
import type { ErrorScenario } from "@/types/ts-explorer";
import ErrorScenarioList from "./ErrorScenarioList.vue";
import ErrorWorkspace from "./ErrorWorkspace.vue";

const selectedId = ref<string>(errorScenarios[0].id);

const selected = computed<ErrorScenario>(
  () => errorScenarios.find((s) => s.id === selectedId.value)!,
);
</script>

<template>
  <div class="type-error-playground">
    <ErrorScenarioList
      :scenarios="errorScenarios"
      :activeId="selectedId"
      @select="selectedId = $event"
    />
    <ErrorWorkspace :scenario="selected" />
  </div>
</template>
```

## Component: ErrorWorkspace.vue

```vue
<script setup lang="ts">
import { useErrorChecker } from "@/composables/useErrorChecker";
import type { ErrorScenario } from "@/types/ts-explorer";
import CodeEditor from "./CodeEditor.vue";

const props = defineProps<{ scenario: ErrorScenario }>();

const {
  userCode,
  liveDiagnostics,
  checking,
  passed,
  revealed,
  checkFix,
  reset,
} = useErrorChecker(props.scenario);
</script>

<template>
  <div class="error-workspace">
    <!-- Challenge prompt -->
    <p class="error-prompt">{{ scenario.prompt }}</p>

    <!-- Editable code editor — user types their fix here -->
    <CodeEditor v-model="userCode" :disabled="checking" />

    <!-- Action buttons -->
    <div class="error-actions">
      <button @click="checkFix" :disabled="checking" class="btn-check">
        {{ checking ? "Checking…" : "▶ Check Fix" }}
      </button>
      <button @click="reset" :disabled="checking" class="btn-reset">
        Reset
      </button>
      <button @click="revealed = !revealed" class="btn-reveal">
        {{ revealed ? "Hide solution" : "Reveal" }}
      </button>
    </div>

    <!-- Pass / fail badge — shown after first check -->
    <div
      v-if="passed !== null"
      :class="[
        'check-result',
        passed ? 'check-result--pass' : 'check-result--fail',
      ]"
    >
      <span v-if="passed">✓ All type errors fixed!</span>
      <span v-else
        >✗ {{ liveDiagnostics.length }} error{{
          liveDiagnostics.length !== 1 ? "s" : ""
        }}
        remaining</span
      >
    </div>

    <!-- Live diagnostics — initial pre-computed, replaced by real check results -->
    <div class="diagnostics-section">
      <h3>TypeScript Diagnostics</h3>
      <p v-if="liveDiagnostics.length === 0" class="diagnostics-empty">
        No type errors — looks good!
      </p>
      <ul
        v-else
        class="diagnostic-list"
        role="list"
        aria-label="Compiler errors"
      >
        <li
          v-for="(diag, i) in liveDiagnostics"
          :key="i"
          class="diagnostic-item"
        >
          <span v-if="diag.code" class="diagnostic-code"
            >error TS{{ diag.code }}</span
          >
          <span class="diagnostic-message">{{ diag.message }}</span>
          <span v-if="diag.line" class="diagnostic-line"
            >Line {{ diag.line }}</span
          >
        </li>
      </ul>
    </div>

    <!-- Solution revealed on demand -->
    <template v-if="revealed">
      <pre
        class="code-block code-block--fixed"
      ><code>{{ scenario.fixedCode }}</code></pre>
      <div class="lesson-text">{{ scenario.lesson }}</div>
    </template>
  </div>
</template>
```

## Key Conventions

### Transpilation vs. Type Checking

| Mechanism                                  | What it catches                         | Used in                              |
| ------------------------------------------ | --------------------------------------- | ------------------------------------ |
| `ts.transpileModule` + `reportDiagnostics` | Syntax errors, some invalid syntax      | `useTsRunner` runtime run            |
| `ErrorScenario.diagnostics` (pre-computed) | Full type errors — shown before editing | ErrorWorkspace initial render        |
| `@typescript/vfs` + `ts.createProgram`     | Full semantic type errors (real `tsc`)  | `useErrorChecker` on Check Fix click |
| `new Function(transpiledJs)`               | Runtime errors after transpilation      | `useTsRunner` runtime run            |

`@typescript/vfs` creates a fully in-memory TypeScript environment in the browser. `createDefaultMapFromCDN` fetches the TS standard lib `.d.ts` files from CDN on first use (~1–2 s), then caches them for all subsequent checks.

### Error Playground Flow

1. User selects a scenario from `ErrorScenarioList` — `brokenCode` pre-fills the editor, `scenario.diagnostics` show immediately (no network wait)
2. User edits the broken code directly in the `CodeEditor`
3. User clicks **▶ Check Fix** → `useErrorChecker.checkFix()` runs a real `ts.createProgram` type check via `@typescript/vfs`
4. `liveDiagnostics` is replaced with the real results; pass/fail badge appears
   - Zero errors → green `✓ All type errors fixed!` badge
   - Remaining errors → red badge + updated diagnostic list with real line numbers
5. User can click **Reveal** at any time to see `fixedCode` and `lesson` without affecting check state
6. **Reset** restores `brokenCode` and the original `scenario.diagnostics`

### CSS Squiggly Underlines

Apply a red wavy underline to `.code-line--error` lines using pure CSS (no JS DOM manipulation):

```css
.code-line--error {
  text-decoration: underline wavy #ef4444;
  text-underline-offset: 3px;
}
```

### Difficulty Badges

| Value          | Color  | Use for                                                                       |
| -------------- | ------ | ----------------------------------------------------------------------------- |
| `beginner`     | green  | Single utility type or basic narrowing                                        |
| `intermediate` | yellow | Two interacting type features, common gotchas like type widening              |
| `advanced`     | red    | Requires understanding of distributive conditional types, `infer`, or `never` |

### Reveal Flow (Concept Problems)

1. User clicks **Reveal** → `ExplanationPanel` appears
2. Code editor switches to read-only, pre-filled with `solutionCode`
3. `useTsRunner.run(solutionCode)` fires automatically — user sees the passing output
4. A "Try again" button resets to `starterCode` and hides the explanation

### Adding a New Concept Problem

1. Add a `TsProblem` entry to `src/data/ts-problems.ts`
2. Set `category` to an existing `ConceptCategory` (or extend the union)
3. Write `starterCode` that compiles and runs but demonstrates the TS concept gap
4. `expectedOutput` must match what `console.log` produces after transpilation
5. Write `explanation` in three parts: _what_ TypeScript does, _why_, and _how to fix or leverage it_
6. `solutionCode` must be a **complete, self-contained runnable program** — include all type declarations, interface definitions, helper functions, variable declarations, and `console.log` calls needed to produce `expectedOutput`. Never write partial type-only snippets.

### Adding a New Error Scenario

1. Add an `ErrorScenario` entry to `src/data/ts-error-scenarios.ts`
2. Set `errorCode` to the primary TS diagnostic code (e.g. `2322`)
3. Write a concise `prompt` — one sentence telling the user what to fix (e.g. `"Fix the type error: ..."`)
4. Write `brokenCode` with exactly the error you want to teach — keep it minimal, max ~10 lines
5. Set `diagnostics[].line` to the 1-based line number; copy `message` verbatim from a real `tsc --strict` run
6. Write `fixedCode` as the minimal correct version of `brokenCode` (same structure, removes only the error)
7. Write `lesson` in three parts: the compiler rule, the mental model for why it fires, and the fix pattern
