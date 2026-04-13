---
name: javascript
description: "Build an interactive JavaScript concept explorer and problem solver in a Vue 3 app. Use when: teaching core JS concepts, building a JS challenge/quiz tool, demonstrating closures, prototypes, async/await, the event loop, scope, or the this keyword, creating an in-browser JS sandbox, solving or explaining JavaScript interview problems interactively."
argument-hint: "The JS concept or problem to explore (e.g. closures, async/await, prototype chain)"
---

# Interactive JavaScript Problem Solver — Core Concepts Explorer

## When to Use

- Building a step-by-step tool for learning and solving JavaScript concept challenges
- Teaching closures, scope, `this`, prototype chains, async/await, promises, or the event loop
- Creating an in-browser JS sandbox that runs user code and captures output
- Generating annotated explanations alongside runnable examples for each concept

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│  JsExplorerView (page-level view)                                        │
│                                                                          │
│  ┌─────────────────────┐  ┌──────────────────────────────────────────┐  │
│  │  ConceptSidebar     │  │  ProblemWorkspace                        │  │
│  │                     │  │                                          │  │
│  │  Scope & Closures   │  │  ┌──────────────────────────────────┐   │  │
│  │  > Closure trap  ◀──│──│  │ ProblemStatement                 │   │  │
│  │    Hoisting          │  │  │ "What logs to the console?"     │   │  │
│  │    let vs var        │  │  │                                  │   │  │
│  │                     │  │  │ ┌──────────────────────────────┐ │   │  │
│  │  this & Prototypes  │  │  │ │ CodeEditor (textarea)        │ │   │  │
│  │    this in methods  │  │  │ │ for (var i = 0; i < 3; i++) { │ │   │  │
│  │    new & classes    │  │  │ │   setTimeout(() => {         │ │   │  │
│  │    prototype chain  │  │  │ │     console.log(i)           │ │   │  │
│  │                     │  │  │ │   }, 0)                      │ │   │  │
│  │  Async / Promises   │  │  │ │ }                            │ │   │  │
│  │    Promise chain    │  │  │ └──────────────────────────────┘ │   │  │
│  │    async/await      │  │  │                                  │   │  │
│  │    error handling   │  │  │  [▶ Run]  [Reset]  [Reveal]      │   │  │
│  │                     │  │  └──────────────────────────────────┘   │  │
│  │  Event Loop         │  │                                          │  │
│  │    call stack       │  │  ┌──────────────────────────────────┐   │  │
│  │    microtasks       │  │  │ OutputConsole                    │   │  │
│  │    macrotasks       │  │  │ > 3                              │   │  │
│  │                     │  │  │ > 3                              │   │  │
│  │  Destructuring &    │  │  │ > 3                              │   │  │
│  │  Spread / Rest      │  │  └──────────────────────────────────┘   │  │
│  │                     │  │                                          │  │
│  │  Iterators &        │  │  ┌──────────────────────────────────┐   │  │
│  │  Generators         │  │  │ ExplanationPanel                 │   │  │
│  │                     │  │  │ var is function-scoped. By the   │   │  │
│  │  Functional JS      │  │  │ time the callbacks fire, i is 3. │   │  │
│  │    map/filter/      │  │  │ Fix: use let (block-scoped) or   │   │  │
│  │    reduce           │  │  │ an IIFE to capture i per loop.   │   │  │
│  └─────────────────────┘  │  └──────────────────────────────────┘   │  │
│                            └──────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

Six pieces:

1. **ConceptSidebar** — groups problems by concept category; clicking a problem loads it into the workspace
2. **ProblemStatement** — displays the challenge title, prompt, and difficulty badge
3. **CodeEditor** — editable `<textarea>` pre-filled with the starter code for the active problem
4. **OutputConsole** — captures and displays output produced by the user's code run in a sandboxed `Function`
5. **ExplanationPanel** — shown after "Reveal"; explains the concept in plain language with an annotated fix
6. **useJsRunner composable** — sandboxes code execution, intercepts `console.log`, captures runtime errors, and returns structured output lines

## Data Model

Create `src/types/js-explorer.ts`:

```ts
export type ConceptCategory =
  | "scope-closures"
  | "this-prototypes"
  | "async-promises"
  | "event-loop"
  | "destructuring-spread"
  | "iterators-generators"
  | "functional";

export type Difficulty = "beginner" | "intermediate" | "advanced";

export interface JsProblem {
  id: string;
  title: string;
  category: ConceptCategory;
  difficulty: Difficulty;
  /** Short prompt shown above the editor */
  prompt: string;
  /** Starting code in the editor */
  starterCode: string;
  /** Expected output lines (used to auto-check the user's run result) */
  expectedOutput: string[];
  /** Plain-language explanation shown on Reveal */
  explanation: string;
  /** Annotated solution code shown on Reveal */
  solutionCode: string;
  /** Key concept tags for filtering */
  tags: string[];
}

export interface OutputLine {
  type: "log" | "error" | "warn";
  value: string;
}

export interface RunResult {
  lines: OutputLine[];
  /** true if every expectedOutput line matches in order */
  passed: boolean | null;
  errorMessage: string | null;
}
```

**Key conventions:**

- `id` is a kebab-case slug — used as the route param and Vue `:key`
- `expectedOutput` is `[]` when no auto-check is possible (e.g. async or timing-dependent problems)
- `RunResult.passed` is `null` when no expected output is defined — show a neutral result badge

## Problem Definitions

Create `src/data/js-problems.ts`:

```ts
import type { JsProblem } from "@/types/js-explorer";

export const problems: JsProblem[] = [
  // ── Scope & Closures ─────────────────────────────────────────────────────
  {
    id: "closure-loop-var",
    title: "Closure + var in a loop",
    category: "scope-closures",
    difficulty: "intermediate",
    prompt:
      "What does this code print? Why? Fix it to print 0, 1, 2 using two different approaches.",
    starterCode: `for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}`,
    expectedOutput: [],
    explanation:
      "`var` is function-scoped, so all three callbacks share the same `i`. By the time the event loop fires them, the loop has finished and `i` is 3. Fix with `let` (block-scoped) or an IIFE that captures `i` per iteration.",
    solutionCode: `// Fix 1 — let (block-scoped, new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}

// Fix 2 — IIFE (captures i immediately)
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i);
}`,
    tags: ["var", "let", "closure", "setTimeout", "loop"],
  },
  {
    id: "closure-counter",
    title: "Private counter with closure",
    category: "scope-closures",
    difficulty: "beginner",
    prompt:
      "Implement `makeCounter()` that returns an object with `increment`, `decrement`, and `value` methods. The internal count must not be directly accessible.",
    starterCode: `function makeCounter() {
  // your code here
}

const counter = makeCounter();
counter.increment();
counter.increment();
counter.decrement();
console.log(counter.value()); // 1`,
    expectedOutput: ["1"],
    explanation:
      "Closures let inner functions access variables from their enclosing scope even after the outer function has returned. The `count` variable lives in `makeCounter`'s scope and is only reachable through the returned methods.",
    solutionCode: `function makeCounter() {
  let count = 0;
  return {
    increment() { count++; },
    decrement() { count--; },
    value() { return count; },
  };
}

const counter = makeCounter();
counter.increment();
counter.increment();
counter.decrement();
console.log(counter.value()); // 1`,
    tags: ["closure", "encapsulation", "factory function"],
  },

  // ── this & Prototypes ────────────────────────────────────────────────────
  {
    id: "this-method-loss",
    title: "Lost this in a callback",
    category: "this-prototypes",
    difficulty: "intermediate",
    prompt:
      "Why does `greet` log `undefined` instead of the name? Fix it three ways: bind, arrow function, and arrow method.",
    starterCode: `const user = {
  name: "Alice",
  greet: function () {
    setTimeout(function () {
      console.log("Hello,", this.name);
    }, 0);
  },
};
user.greet();`,
    expectedOutput: [],
    explanation:
      "A regular function passed as a callback loses its original `this` — it defaults to `undefined` in strict mode or the global object. Arrow functions capture `this` lexically from their enclosing scope, solving the problem.",
    solutionCode: `const user = {
  name: "Alice",
  // Fix: arrow function captures lexical this
  greet: function () {
    setTimeout(() => {
      console.log("Hello,", this.name); // "Hello, Alice"
    }, 0);
  },
};
user.greet();`,
    tags: ["this", "bind", "arrow function", "callback"],
  },
  {
    id: "prototype-chain",
    title: "Prototype chain lookup",
    category: "this-prototypes",
    difficulty: "intermediate",
    prompt:
      "Predict the output. Then add a `speak` method to `Animal.prototype` so every animal can speak without duplicating the function.",
    starterCode: `function Animal(name) {
  this.name = name;
}

const dog = new Animal("Rex");
console.log(dog.hasOwnProperty("name")); // ?
console.log(dog.hasOwnProperty("toString")); // ?
console.log(dog instanceof Animal); // ?`,
    expectedOutput: ["true", "false", "true"],
    explanation:
      "`hasOwnProperty` returns true only for properties set directly on the instance. `toString` lives on `Object.prototype`, not on `dog` itself — it's found via the prototype chain. Methods added to `Constructor.prototype` are shared across all instances without copying.",
    solutionCode: `function Animal(name) {
  this.name = name;
}

const dog = new Animal("Rex");
console.log(dog.hasOwnProperty("name"));
console.log(dog.hasOwnProperty("toString"));
console.log(dog instanceof Animal);

// Bonus: shared method via prototype
Animal.prototype.speak = function () {
  console.log(this.name + " says hi");
};
// dog.speak(); // Rex says hi`,
    tags: ["prototype", "new", "instanceof", "hasOwnProperty"],
  },

  // ── Async / Promises ─────────────────────────────────────────────────────
  {
    id: "promise-chain-order",
    title: "Promise chain execution order",
    category: "async-promises",
    difficulty: "intermediate",
    prompt: "Predict the exact order of the console.log calls.",
    starterCode: `console.log("A");

Promise.resolve()
  .then(() => console.log("B"))
  .then(() => console.log("C"));

console.log("D");`,
    expectedOutput: [],
    explanation:
      "Synchronous code runs first (A, D). Promise `.then` callbacks are microtasks — they run after the current synchronous task completes but before any macrotasks (like setTimeout). Each `.then` schedules the next one, so B then C.",
    solutionCode: `console.log("A"); // 1st — synchronous

Promise.resolve()
  .then(() => console.log("B")) // 3rd — microtask, runs after sync
  .then(() => console.log("C")); // 4th — microtask, runs after B

console.log("D"); // 2nd — synchronous

// Output order: A → D → B → C`,
    tags: ["promise", "microtask", "event loop", "then"],
  },
  {
    id: "async-await-error",
    title: "Handling errors in async/await",
    category: "async-promises",
    difficulty: "beginner",
    prompt:
      "Rewrite the promise chain using async/await and add proper error handling.",
    starterCode: `function fetchUser(id) {
  return id > 0
    ? Promise.resolve({ id, name: "Alice" })
    : Promise.reject(new Error("Invalid ID"));
}

// Rewrite this using async/await:
fetchUser(-1)
  .then((user) => console.log(user.name))
  .catch((err) => console.log("Error:", err.message));`,
    expectedOutput: [],
    explanation:
      "`await` unwraps a resolved Promise. For rejected Promises, wrap `await` in a `try/catch`. This gives synchronous-looking control flow while remaining non-blocking.",
    solutionCode: `function fetchUser(id) {
  return id > 0
    ? Promise.resolve({ id, name: "Alice" })
    : Promise.reject(new Error("Invalid ID"));
}

async function run() {
  try {
    const user = await fetchUser(-1);
    console.log(user.name);
  } catch (err) {
    console.log("Error:", err.message);
  }
}
run();`,
    tags: ["async", "await", "try/catch", "promise", "error handling"],
  },

  // ── Event Loop ───────────────────────────────────────────────────────────
  {
    id: "event-loop-order",
    title: "Microtasks vs macrotasks",
    category: "event-loop",
    difficulty: "advanced",
    prompt: "Predict the exact output order and explain why.",
    starterCode: `setTimeout(() => console.log("timeout"), 0);
Promise.resolve().then(() => console.log("promise"));
queueMicrotask(() => console.log("microtask"));
console.log("sync");`,
    expectedOutput: [],
    explanation:
      "Synchronous code runs first. After the call stack empties, the microtask queue drains completely (Promise callbacks, queueMicrotask) before any macrotask (setTimeout callback) is picked up. Promise `.then` and `queueMicrotask` are both microtasks and run in registration order.",
    solutionCode: `setTimeout(() => console.log("timeout"), 0);    // macrotask — fires last
Promise.resolve().then(() => console.log("promise")); // microtask — 2nd
queueMicrotask(() => console.log("microtask"));        // microtask — 3rd
console.log("sync");                                   // synchronous — 1st

// Output order: sync → promise → microtask → timeout`,
    tags: ["event loop", "microtask", "macrotask", "setTimeout", "promise"],
  },

  // ── Destructuring & Spread ───────────────────────────────────────────────
  {
    id: "destructuring-defaults",
    title: "Destructuring with defaults and rename",
    category: "destructuring-spread",
    difficulty: "beginner",
    prompt:
      "Destructure `config` to get `host` (default `'localhost'`), `port` (default `3000`), and rename `secure` to `isHttps`.",
    starterCode: `const config = { port: 8080, secure: true };

// Your destructuring here:
// const { ... } = config;

console.log(host);    // localhost
console.log(port);    // 8080
console.log(isHttps); // true`,
    expectedOutput: ["localhost", "8080", "true"],
    solutionCode: `const config = { port: 8080, secure: true };
const { host = "localhost", port = 3000, secure: isHttps = false } = config;
console.log(host);
console.log(port);
console.log(isHttps);`,
    explanation:
      "Destructuring syntax: `{ key: newName = defaultValue }`. Defaults only apply when the value is `undefined`. The rename and default can be combined in one declaration.",
    tags: ["destructuring", "default values", "rename", "object"],
  },

  // ── Iterators & Generators ───────────────────────────────────────────────
  {
    id: "generator-range",
    title: "Range generator",
    category: "iterators-generators",
    difficulty: "intermediate",
    prompt:
      "Implement a `range(start, end, step)` generator that yields values lazily without building an array.",
    starterCode: `function* range(start, end, step = 1) {
  // your code here
}

for (const n of range(0, 10, 2)) {
  console.log(n); // 0 2 4 6 8
}`,
    expectedOutput: ["0", "2", "4", "6", "8"],
    explanation:
      "A generator function uses `yield` to pause execution and return a value on demand. The `for...of` loop calls `.next()` behind the scenes — no intermediate array is created, making this memory-efficient for large ranges.",
    solutionCode: `function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}

for (const n of range(0, 10, 2)) {
  console.log(n);
}`,
    tags: ["generator", "yield", "iterator", "for...of", "lazy evaluation"],
  },

  // ── Functional JS ────────────────────────────────────────────────────────
  {
    id: "functional-pipeline",
    title: "Compose a data pipeline",
    category: "functional",
    difficulty: "intermediate",
    prompt:
      "Using only `map`, `filter`, and `reduce`, compute the total price of in-stock items after a 10% discount. No loops allowed.",
    starterCode: `const inventory = [
  { name: "Widget", price: 25, inStock: true },
  { name: "Gadget", price: 80, inStock: false },
  { name: "Doohickey", price: 40, inStock: true },
  { name: "Thingamajig", price: 15, inStock: true },
];

// Your pipeline here
const total = 0;
console.log(total); // 72`,
    expectedOutput: ["72"],
    explanation:
      "Chain: filter out out-of-stock items → map to discounted prices → reduce to sum. Each step is a pure transformation — no mutation, no side effects. Reading it top-to-bottom describes what the code does, not how.",
    solutionCode: `const inventory = [
  { name: "Widget", price: 25, inStock: true },
  { name: "Gadget", price: 80, inStock: false },
  { name: "Doohickey", price: 40, inStock: true },
  { name: "Thingamajig", price: 15, inStock: true },
];

const total = inventory
  .filter((item) => item.inStock)
  .map((item) => item.price * 0.9)
  .reduce((sum, price) => sum + price, 0);
console.log(total);`,
    tags: [
      "map",
      "filter",
      "reduce",
      "functional",
      "pipeline",
      "pure function",
    ],
  },
];
```

## useJsRunner Composable

Create `src/composables/useJsRunner.ts`:

```ts
import { ref } from "vue";
import type { OutputLine, RunResult, JsProblem } from "@/types/js-explorer";

export function useJsRunner(problem: JsProblem) {
  const output = ref<OutputLine[]>([]);
  const result = ref<RunResult | null>(null);

  function run(code: string): void {
    const lines: OutputLine[] = [];

    // Intercept console methods — never eval untrusted external input in production
    const sandboxConsole = {
      log: (...args: unknown[]) =>
        lines.push({ type: "log", value: args.map(String).join(" ") }),
      warn: (...args: unknown[]) =>
        lines.push({ type: "warn", value: args.map(String).join(" ") }),
      error: (...args: unknown[]) =>
        lines.push({ type: "error", value: args.map(String).join(" ") }),
    };

    let errorMessage: string | null = null;

    try {
      // Sandboxed with new Function — scope is isolated; no access to module internals
      // Only expose the sandboxed console, not the real one
      const fn = new Function("console", code);
      fn(sandboxConsole);
    } catch (err) {
      errorMessage = err instanceof Error ? err.message : String(err);
      lines.push({ type: "error", value: errorMessage });
    }

    const passed =
      problem.expectedOutput.length > 0
        ? problem.expectedOutput.every(
            (expected, i) => lines[i]?.value === expected,
          )
        : null;

    output.value = lines;
    result.value = { lines, passed, errorMessage };
  }

  function reset(): void {
    output.value = [];
    result.value = null;
  }

  return { output, result, run, reset };
}
```

**Security note:** `new Function(code)` executes in strict isolation from module scope. Never inject user-controlled values into the outer string — only pass the code string written in the editor.

## Component: CodeEditor.vue

```vue
<script setup lang="ts">
import { ref, watch } from "vue";

const props = defineProps<{ modelValue: string; disabled?: boolean }>();
const emit = defineEmits<{ "update:modelValue": [value: string] }>();

const localCode = ref(props.modelValue);
watch(
  () => props.modelValue,
  (v) => (localCode.value = v),
);
</script>

<template>
  <textarea
    v-model="localCode"
    :disabled="disabled"
    class="code-editor"
    spellcheck="false"
    autocomplete="off"
    autocorrect="off"
    @input="emit('update:modelValue', localCode)"
  />
</template>
```

## Component: OutputConsole.vue

```vue
<script setup lang="ts">
import type { OutputLine } from "@/types/js-explorer";
defineProps<{ lines: OutputLine[]; errorMessage?: string | null }>();
</script>

<template>
  <div
    class="output-console"
    role="log"
    aria-label="Console output"
    aria-live="polite"
  >
    <p v-if="lines.length === 0" class="output-empty">
      Run your code to see output.
    </p>
    <p
      v-for="(line, i) in lines"
      :key="i"
      :class="['output-line', `output-line--${line.type}`]"
    >
      <span aria-hidden="true">&gt; </span>{{ line.value }}
    </p>
  </div>
</template>
```

## Key Conventions

### Code Execution Safety

| Rule                                                  | Reason                                                    |
| ----------------------------------------------------- | --------------------------------------------------------- |
| Use `new Function("console", code)`                   | Isolates scope; real `window`/`document` not accessible   |
| Never inject variables into the code string           | Prevents code injection through problem data              |
| No `eval()`                                           | Less safe — walks the caller's scope chain                |
| No `setTimeout` interception needed for sync problems | Async problems display a note that timing output may vary |

### Async Problem Handling

Problems in `async-promises` and `event-loop` categories run natively — `setTimeout`, `Promise`, and `queueMicrotask` are real browser APIs available inside `new Function`. **All async problems must set `expectedOutput: []`** — the synchronous `passed` check runs before callbacks fire and cannot capture async output. The output console will still display async output once callbacks fire (Vue tracks array mutations reactively), and the explanation panel guides the user through the expected order.

### Difficulty Badges

| Value          | Color  | Use for                                                                  |
| -------------- | ------ | ------------------------------------------------------------------------ |
| `beginner`     | green  | Single-concept problems, no tricky ordering                              |
| `intermediate` | yellow | Two interacting concepts, common gotchas                                 |
| `advanced`     | red    | Requires mental model of the runtime (event loop, prototype chain depth) |

### Reveal Flow

1. User clicks **Reveal** → `ExplanationPanel` appears below the console
2. Code editor switches to read-only and is pre-filled with `solutionCode`
3. Output console reruns automatically with the solution so user sees the correct result
4. A "Try again" button resets to `starterCode` and hides the explanation

### Adding a New Problem

1. Add a `JsProblem` entry to `src/data/js-problems.ts`
2. Set `category` to an existing `ConceptCategory` (or extend the union type)
3. Provide `starterCode` that demonstrates the gotcha, not the solution
4. Write `explanation` in plain language — first state _what_ happens, then _why_, then the fix
5. `expectedOutput` lines must be exact string matches to what `console.log` produces synchronously; use `[]` for async problems
6. `solutionCode` must be a **complete, self-contained runnable program** — include all variable declarations, helper functions, and `console.log` calls needed to produce `expectedOutput`. Never write partial snippets that reference undefined variables or functions.
