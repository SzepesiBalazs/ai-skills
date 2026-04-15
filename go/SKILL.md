---
name: go
description: "Build an interactive Go language concept explorer and problem solver in a Vue 3 app. Use when: teaching core Go concepts, building a Go challenge/quiz tool, demonstrating goroutines, channels, interfaces, pointers, error handling, slices, maps, defer, or the Go type system, creating an in-browser Go concept sandbox, solving or explaining Go interview problems interactively."
argument-hint: "The Go concept or problem to explore (e.g. goroutines, interfaces, error handling)"
---

# Interactive Go Problem Solver — Core Concepts Explorer

## When to Use

- Building a step-by-step tool for learning and solving Go concept challenges
- Teaching goroutines, channels, interfaces, pointers, slices, maps, defer, or error handling
- Creating a Go concept sandbox with editable code and annotated explanations
- Generating runnable examples and exercises for Go interview preparation

## Concepts Covered

| Category              | Topics                                                             |
| --------------------- | ------------------------------------------------------------------ |
| Types & Variables     | zero values, short declaration `:=`, `var`, typed consts           |
| Functions             | multiple return values, named returns, variadic, first-class funcs |
| Pointers              | `*T`, `&`, pointer receivers vs value receivers                    |
| Structs & Interfaces  | embedding, implicit interface satisfaction, empty interface        |
| Error Handling        | `error` interface, `errors.New`, `fmt.Errorf %w`, sentinel errors  |
| Slices & Maps         | backing arrays, append, copy, nil vs empty, map delete             |
| Goroutines & Channels | `go`, unbuffered/buffered channels, `select`, `sync.WaitGroup`     |
| Defer, Panic, Recover | defer ordering, recover pattern, when to use panic                 |
| Generics (Go 1.18+)   | type parameters, constraints, `comparable`, generic functions      |
| Testing               | `testing.T`, table-driven tests, `t.Run`, benchmarks               |

## Problem Structure

Each problem has:

- **Prompt** — what to predict, fix, or implement
- **Starter code** — demonstrates the concept or gotcha
- **Expected output** — exact strings logged to stdout (empty `[]` for concurrent/timing-dependent problems)
- **Explanation** — plain-language description: what happens → why → the fix
- **Solution code** — complete, self-contained, runnable program

## Data Model

Create `src/types/go-explorer.ts`:

```ts
export type GoCategory =
  | "types-variables"
  | "functions"
  | "pointers"
  | "structs-interfaces"
  | "error-handling"
  | "slices-maps"
  | "goroutines-channels"
  | "defer-panic-recover"
  | "generics"
  | "testing";

export type Difficulty = "beginner" | "intermediate" | "advanced";

export interface GoProblem {
  id: string;
  title: string;
  category: GoCategory;
  difficulty: Difficulty;
  prompt: string;
  starterCode: string;
  expectedOutput: string[];
  explanation: string;
  solutionCode: string;
  tags: string[];
}

export interface OutputLine {
  type: "log" | "error" | "warn";
  value: string;
}

export interface RunResult {
  lines: OutputLine[];
  /** null when expectedOutput is [] */
  passed: boolean | null;
  errorMessage: string | null;
}
```

## Problem Definitions

Create `src/data/go-problems.ts`:

```ts
import type { GoProblem } from "@/types/go-explorer";

export const problems: GoProblem[] = [
  // ── Types & Variables ─────────────────────────────────────────────────────
  {
    id: "zero-values",
    title: "Zero values",
    category: "types-variables",
    difficulty: "beginner",
    prompt:
      "What does this program print? Explain why Go does not require explicit initialization.",
    starterCode: `package main

import "fmt"

func main() {
    var i int
    var f float64
    var b bool
    var s string
    fmt.Println(i, f, b, s == "")
}`,
    expectedOutput: ["0 0 false true"],
    explanation:
      'In Go every variable is initialized to its zero value when declared without an explicit value. `int` → 0, `float64` → 0, `bool` → false, `string` → "" (empty). This eliminates an entire class of undefined-behavior bugs common in C/C++.',
    solutionCode: `package main

import "fmt"

func main() {
    var i int
    var f float64
    var b bool
    var s string
    fmt.Println(i, f, b, s == "") // 0 0 false true
}`,
    tags: ["zero value", "var", "types"],
  },
  {
    id: "short-declaration",
    title: "Short variable declaration := vs var",
    category: "types-variables",
    difficulty: "beginner",
    prompt:
      "Fix the compilation error and explain when to prefer `:=` over `var`.",
    starterCode: `package main

import "fmt"

func main() {
    x := 10
    x := 20   // compile error — why?
    fmt.Println(x)
}`,
    expectedOutput: ["20"],
    explanation:
      "`:=` declares AND assigns. Redeclaring an already-declared variable in the same scope with `:=` is a compile error. Use `=` for reassignment. `:=` is preferred inside functions for brevity; `var` is used at package level or when you need an explicit type.",
    solutionCode: `package main

import "fmt"

func main() {
    x := 10
    x = 20   // reassign, not redeclare
    fmt.Println(x) // 20
}`,
    tags: [":=", "var", "declaration", "scope"],
  },

  // ── Functions ─────────────────────────────────────────────────────────────
  {
    id: "multiple-return",
    title: "Multiple return values",
    category: "functions",
    difficulty: "beginner",
    prompt:
      "Implement `divide(a, b float64) (float64, error)` that returns an error when dividing by zero. Print the result or the error.",
    starterCode: `package main

import (
    "errors"
    "fmt"
)

func divide(a, b float64) (float64, error) {
    // your code here
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println(result)
    }

    _, err = divide(5, 0)
    if err != nil {
        fmt.Println("Error:", err)
    }
}`,
    expectedOutput: ["5", "Error: division by zero"],
    explanation:
      "Go functions can return multiple values. The idiomatic Go pattern for fallible operations is `(result, error)`. Callers must check the error before using the result. The blank identifier `_` discards unwanted return values.",
    solutionCode: `package main

import (
    "errors"
    "fmt"
)

func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println(result) // 5
    }

    _, err = divide(5, 0)
    if err != nil {
        fmt.Println("Error:", err) // Error: division by zero
    }
}`,
    tags: ["multiple return", "error", "nil", "blank identifier"],
  },

  // ── Pointers ──────────────────────────────────────────────────────────────
  {
    id: "pointer-receiver",
    title: "Value receiver vs pointer receiver",
    category: "pointers",
    difficulty: "intermediate",
    prompt:
      "Why does `increment` not modify `counter`? Fix it using a pointer receiver.",
    starterCode: `package main

import "fmt"

type Counter struct{ value int }

func (c Counter) increment() {
    c.value++
}

func main() {
    c := Counter{}
    c.increment()
    c.increment()
    fmt.Println(c.value) // want 2, got?
}`,
    expectedOutput: ["2"],
    explanation:
      "A value receiver receives a copy of the struct — mutations are local and discarded. A pointer receiver `(c *Counter)` receives the address of the original struct, so changes persist. Use pointer receivers when a method must mutate state or when the struct is large (avoids copying).",
    solutionCode: `package main

import "fmt"

type Counter struct{ value int }

func (c *Counter) increment() {
    c.value++
}

func main() {
    c := Counter{}
    c.increment()
    c.increment()
    fmt.Println(c.value) // 2
}`,
    tags: ["pointer", "receiver", "struct", "method"],
  },

  // ── Structs & Interfaces ──────────────────────────────────────────────────
  {
    id: "implicit-interface",
    title: "Implicit interface satisfaction",
    category: "structs-interfaces",
    difficulty: "intermediate",
    prompt:
      "Add a `Dog` type that satisfies the `Animal` interface without using the `implements` keyword. Print the speech of each animal via a shared function.",
    starterCode: `package main

import "fmt"

type Animal interface {
    Speak() string
}

type Cat struct{ Name string }

func (c Cat) Speak() string {
    return c.Name + " says meow"
}

// Add Dog here

func makeNoise(a Animal) {
    fmt.Println(a.Speak())
}

func main() {
    makeNoise(Cat{Name: "Whiskers"})
    // makeNoise(Dog{Name: "Rex"})
}`,
    expectedOutput: ["Whiskers says meow", "Rex says woof"],
    explanation:
      "Go interfaces are satisfied implicitly — any type with matching methods automatically satisfies the interface. No `implements` declaration is needed. This enables duck typing while keeping static type safety. The `Animal` interface requires only a `Speak() string` method.",
    solutionCode: `package main

import "fmt"

type Animal interface {
    Speak() string
}

type Cat struct{ Name string }
func (c Cat) Speak() string { return c.Name + " says meow" }

type Dog struct{ Name string }
func (d Dog) Speak() string { return d.Name + " says woof" }

func makeNoise(a Animal) {
    fmt.Println(a.Speak())
}

func main() {
    makeNoise(Cat{Name: "Whiskers"})
    makeNoise(Dog{Name: "Rex"})
}`,
    tags: ["interface", "implicit", "duck typing", "struct"],
  },

  // ── Error Handling ────────────────────────────────────────────────────────
  {
    id: "error-wrapping",
    title: "Error wrapping with %w",
    category: "error-handling",
    difficulty: "intermediate",
    prompt:
      "Wrap the original error with context using `fmt.Errorf`. Then unwrap it with `errors.Is` to check for the sentinel error.",
    starterCode: `package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

func findUser(id int) error {
    return ErrNotFound // imagine a DB lookup
}

func getUser(id int) error {
    err := findUser(id)
    if err != nil {
        // wrap with context — your code here
        return err
    }
    return nil
}

func main() {
    err := getUser(42)
    fmt.Println(err)
    fmt.Println(errors.Is(err, ErrNotFound))
}`,
    expectedOutput: ["getUser 42: not found", "true"],
    explanation:
      '`fmt.Errorf("...: %w", err)` wraps the original error, preserving it in the chain. `errors.Is` unwraps the chain recursively to check for a target sentinel error. This lets callers distinguish error types without exposing internal error strings. Use `%v` when you want to log without preserving the chain.',
    solutionCode: `package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("not found")

func findUser(id int) error {
    return ErrNotFound
}

func getUser(id int) error {
    err := findUser(id)
    if err != nil {
        return fmt.Errorf("getUser %d: %w", id, err)
    }
    return nil
}

func main() {
    err := getUser(42)
    fmt.Println(err)                       // getUser 42: not found
    fmt.Println(errors.Is(err, ErrNotFound)) // true
}`,
    tags: ["error", "fmt.Errorf", "%w", "errors.Is", "sentinel", "wrapping"],
  },

  // ── Slices & Maps ─────────────────────────────────────────────────────────
  {
    id: "slice-append-gotcha",
    title: "Slice append and shared backing array",
    category: "slices-maps",
    difficulty: "advanced",
    prompt:
      "Predict the output and explain why modifying `b` also changes `a`.",
    starterCode: `package main

import "fmt"

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := a[:3]
    b = append(b, 99)
    fmt.Println(a)
    fmt.Println(b)
}`,
    expectedOutput: ["[1 2 3 99 5]", "[1 2 3 99]"],
    explanation:
      "A slice is a header (pointer, length, capacity) over a backing array. `b := a[:3]` creates a new header sharing the same array. `append(b, 99)` has room within the existing capacity (len=3, cap=5), so it writes 99 at index 3 of the shared array — mutating `a[3]`. Use `b := append([]int(nil), a[:3]...)` or `copy` to get an independent slice.",
    solutionCode: `package main

import "fmt"

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := a[:3]
    b = append(b, 99)
    fmt.Println(a) // [1 2 3 99 5]
    fmt.Println(b) // [1 2 3 99]

    // Safe independent copy:
    // b := make([]int, 3)
    // copy(b, a[:3])
}`,
    tags: ["slice", "append", "backing array", "copy", "capacity"],
  },
  {
    id: "map-nil-panic",
    title: "Writing to a nil map panics",
    category: "slices-maps",
    difficulty: "beginner",
    prompt:
      "Why does this panic? Fix it. Also show how to safely check for key existence.",
    starterCode: `package main

import "fmt"

func main() {
    var m map[string]int
    m["key"] = 1   // panic?
    fmt.Println(m["key"])
    fmt.Println(m["missing"])
}`,
    expectedOutput: ["1", "0"],
    explanation:
      "A `var m map[string]int` declaration sets m to nil. Reading from a nil map returns the zero value safely, but writing panics. Always initialize with `make(map[string]int)` or a map literal `map[string]int{}`. Use the two-value form `v, ok := m[key]` to distinguish a missing key from a zero value.",
    solutionCode: `package main

import "fmt"

func main() {
    m := make(map[string]int)
    m["key"] = 1
    fmt.Println(m["key"])     // 1
    fmt.Println(m["missing"]) // 0

    // Check existence:
    // v, ok := m["missing"]
    // fmt.Println(v, ok) // 0 false
}`,
    tags: ["map", "nil", "make", "zero value", "ok idiom"],
  },

  // ── Goroutines & Channels ─────────────────────────────────────────────────
  {
    id: "goroutine-waitgroup",
    title: "WaitGroup for goroutine synchronization",
    category: "goroutines-channels",
    difficulty: "intermediate",
    prompt:
      "The program exits before goroutines finish. Fix it using `sync.WaitGroup`.",
    starterCode: `package main

import (
    "fmt"
    "sync"
)

func worker(id int) {
    fmt.Printf("worker %d done\\n", id)
}

func main() {
    for i := 1; i <= 3; i++ {
        go worker(i)
    }
    // main returns before goroutines run
}`,
    expectedOutput: [],
    explanation:
      "`go f()` starts a goroutine concurrently, but `main` continues immediately and may exit before the goroutines run. `sync.WaitGroup` solves this: call `wg.Add(1)` before starting each goroutine, `wg.Done()` when it finishes (via `defer`), and `wg.Wait()` in main to block until all are done. For concurrent output, order is non-deterministic — `expectedOutput` is left empty.",
    solutionCode: `package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("worker %d done\\n", id)
}

func main() {
    var wg sync.WaitGroup
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }
    wg.Wait()
}`,
    tags: ["goroutine", "WaitGroup", "sync", "concurrency"],
  },
  {
    id: "channel-pipeline",
    title: "Channel pipeline: producer → consumer",
    category: "goroutines-channels",
    difficulty: "intermediate",
    prompt:
      "Implement a producer that sends numbers 1–5 on a channel and a consumer that prints each value squared.",
    starterCode: `package main

import "fmt"

func produce(ch chan<- int) {
    // send 1-5 then close
}

func consume(ch <-chan int) {
    // print each value squared
}

func main() {
    ch := make(chan int)
    go produce(ch)
    consume(ch)
}`,
    expectedOutput: ["1", "4", "9", "16", "25"],
    explanation:
      "Directional channel types `chan<- int` (send-only) and `<-chan int` (receive-only) document intent and enforce safety at compile time. Closing the channel (`close(ch)`) signals no more values — `range ch` stops cleanly when the channel is closed. Always close from the sender, never the receiver.",
    solutionCode: `package main

import "fmt"

func produce(ch chan<- int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch)
}

func consume(ch <-chan int) {
    for v := range ch {
        fmt.Println(v * v)
    }
}

func main() {
    ch := make(chan int)
    go produce(ch)
    consume(ch)
}`,
    tags: ["channel", "pipeline", "range", "close", "directional channel"],
  },

  // ── Defer, Panic, Recover ─────────────────────────────────────────────────
  {
    id: "defer-order",
    title: "Defer execution order",
    category: "defer-panic-recover",
    difficulty: "beginner",
    prompt: "Predict the output. Explain the LIFO ordering of deferred calls.",
    starterCode: `package main

import "fmt"

func main() {
    fmt.Println("start")
    defer fmt.Println("first defer")
    defer fmt.Println("second defer")
    defer fmt.Println("third defer")
    fmt.Println("end")
}`,
    expectedOutput: [
      "start",
      "end",
      "third defer",
      "second defer",
      "first defer",
    ],
    explanation:
      "`defer` pushes a function call onto a stack. When the surrounding function returns, deferred calls execute in Last-In-First-Out (LIFO) order — like unwinding. Common uses: closing files/connections, unlocking mutexes, and ensuring cleanup regardless of early returns or panics.",
    solutionCode: `package main

import "fmt"

func main() {
    fmt.Println("start")
    defer fmt.Println("first defer")
    defer fmt.Println("second defer")
    defer fmt.Println("third defer")
    fmt.Println("end")
    // Output: start → end → third defer → second defer → first defer
}`,
    tags: ["defer", "LIFO", "stack", "cleanup"],
  },
  {
    id: "recover-panic",
    title: "Recovering from a panic",
    category: "defer-panic-recover",
    difficulty: "advanced",
    prompt:
      "Use `recover` inside a deferred function to catch the panic and print the recovered message instead of crashing.",
    starterCode: `package main

import "fmt"

func riskyOp() {
    panic("something went wrong")
}

func safeOp() {
    // add defer + recover here
    riskyOp()
}

func main() {
    safeOp()
    fmt.Println("program continues")
}`,
    expectedOutput: ["recovered: something went wrong", "program continues"],
    explanation:
      "`recover()` stops a panicking goroutine and returns the panic value — but only when called directly inside a `defer`d function. Outside of `defer`, `recover` returns nil. Use this pattern to build resilient servers or middleware that must not crash on a single bad request. Do not use `recover` as general error handling — only catch panics at clear boundaries.",
    solutionCode: `package main

import "fmt"

func riskyOp() {
    panic("something went wrong")
}

func safeOp() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered:", r)
        }
    }()
    riskyOp()
}

func main() {
    safeOp()
    fmt.Println("program continues")
}`,
    tags: ["recover", "panic", "defer", "error handling"],
  },

  // ── Generics ──────────────────────────────────────────────────────────────
  {
    id: "generic-filter",
    title: "Generic Filter function (Go 1.18+)",
    category: "generics",
    difficulty: "intermediate",
    prompt:
      "Implement a generic `Filter[T any](slice []T, predicate func(T) bool) []T` function. Use it to filter integers and strings.",
    starterCode: `package main

import "fmt"

func Filter[T any](slice []T, predicate func(T) bool) []T {
    // your code here
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6}
    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println(evens)

    words := []string{"go", "rust", "python", "java"}
    short := Filter(words, func(s string) bool { return len(s) <= 3 })
    fmt.Println(short)
}`,
    expectedOutput: ["[2 4 6]", "[go]"],
    explanation:
      "Go 1.18 introduced type parameters. `[T any]` declares a type parameter `T` constrained to any type. The compiler infers `T` from the argument types at call sites — no explicit `Filter[int](...)` needed. Generics eliminate the need for `interface{}` + type assertions for reusable algorithms.",
    solutionCode: `package main

import "fmt"

func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0, len(slice))
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6}
    evens := Filter(nums, func(n int) bool { return n%2 == 0 })
    fmt.Println(evens) // [2 4 6]

    words := []string{"go", "rust", "python", "java"}
    short := Filter(words, func(s string) bool { return len(s) <= 3 })
    fmt.Println(short) // [go]
}`,
    tags: ["generics", "type parameter", "any", "constraint", "filter"],
  },
];
```

## Execution Model

Go problems run server-side via the [Go Playground API](https://go.dev/play/) or a local Go binary. In a Vue app without a backend, two strategies are available:

### Strategy A — Go Playground API

```ts
async function runGoCode(code: string): Promise<RunResult> {
  const response = await fetch("https://go.dev/_/compile", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ version: "2", body: code }),
  });
  const data = await response.json();
  // data.Events[].Message contains stdout/stderr
  const lines = (data.Events ?? []).map((e: { Message: string }) => ({
    type: e.Kind === "stderr" ? "error" : "log",
    value: e.Message.trimEnd(),
  }));
  return {
    lines,
    passed: null, // check after receiving output
    errorMessage: data.Errors || null,
  };
}
```

### Strategy B — Static explanation mode (no execution)

For fully offline use, display starter code as read-only with syntax highlighting. The "Run" button shows the expected output directly from the problem definition. A "Reveal" button displays the solution code and explanation. This keeps the UX interactive without needing a backend.

**Recommended approach:** Use Strategy B as the default with a toggle to "Run in Playground" (Strategy A) for problems where execution matters.

## Explanation Logic

Each problem explanation follows this structure:

1. **What happens** — state the observed (surprising) behavior
2. **Why it happens** — explain the underlying Go mechanism (spec rule, runtime behavior, type system)
3. **The fix** — show the idiomatic Go solution and any alternatives

### Difficulty Badges

| Value          | Color  | Use for                                                                   |
| -------------- | ------ | ------------------------------------------------------------------------- |
| `beginner`     | green  | Single-concept problems, straightforward syntax                           |
| `intermediate` | yellow | Two interacting mechanisms (e.g. slice + backing array, interface + nil)  |
| `advanced`     | red    | Requires mental model of the runtime (goroutine scheduler, memory layout) |

## Reveal Flow

1. User clicks **Reveal** → `ExplanationPanel` appears below the output
2. Code editor switches to read-only, pre-filled with `solutionCode`
3. Expected output is displayed automatically from the problem definition
4. A "Try again" button resets to `starterCode` and hides the explanation

## Key Conventions

### Problem Authoring Rules

| Rule                                                                         | Reason                                                        |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `starterCode` must be a complete, valid Go program (`package main`, imports) | Prevents confusing compile errors unrelated to the concept    |
| `expectedOutput` lines are exact `fmt.Println` output strings                | Used for auto-check; must exclude trailing newlines           |
| Set `expectedOutput: []` for goroutine/channel problems                      | Output order is non-deterministic                             |
| `solutionCode` must be fully self-contained                                  | Running it must produce `expectedOutput` without modification |
| Explanation: what → why → fix                                                | Mirrors the learning progression                              |

### Adding a New Problem

1. Add a `GoProblem` entry to `src/data/go-problems.ts`
2. Set `category` to an existing `GoCategory` (or extend the union type)
3. Write `starterCode` that demonstrates the gotcha — not the fix
4. Provide exact `expectedOutput` string matches for synchronous-output problems
5. Write `explanation` in plain language, not jargon
6. Ensure `solutionCode` is a complete, runnable, self-contained program

### Go-Specific Patterns to Teach

```
Idiomatic Go checklist for every solution:
- Error handling: if err != nil { return ..., err } — never ignore errors
- defer for cleanup: defer f.Close() immediately after open
- Accept interfaces, return structs
- Short variable names: i, n, err, ok — not index, number, error, okay
- Table-driven tests: []struct{ input, expected }{ ... }
- Avoid global mutable state; pass dependencies as function arguments
```

## Examples

### Example: Interface nil trap (advanced)

**Prompt:** Why does `isNil` return `false` even though `err` appears to be nil?

```go
package main

import "fmt"

type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func getError(fail bool) error {
    var err *MyError
    if fail {
        err = &MyError{"oops"}
    }
    return err // returns interface holding (*MyError, nil) — not nil interface!
}

func main() {
    err := getError(false)
    fmt.Println(err == nil) // false — why?
}
```

**Explanation:** An interface value is only nil when both its type and value are nil. Returning a typed nil pointer `(*MyError)(nil)` as an `error` interface creates an interface with a non-nil type pointer — so `err == nil` is false. Fix: return `nil` directly (`return nil`), not a typed nil.

---

### Example: Goroutine closure capture (intermediate)

**Prompt:** The program prints three identical values. Fix it.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(i) // prints 3, 3, 3
        }()
    }
    wg.Wait()
}
```

**Fix:** Pass `i` as a parameter to the goroutine closure — `go func(n int) { ... }(i)` — to capture the value at each iteration rather than the shared loop variable.
