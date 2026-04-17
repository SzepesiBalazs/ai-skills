---
name: php
description: "Build an interactive PHP concept explorer and problem solver in a Vue 3 app. Use when: teaching core PHP concepts, building a PHP challenge/quiz tool, demonstrating closures, OOP, type juggling, match expressions, traits, or enums, creating an in-browser PHP sandbox, solving or explaining PHP interview problems interactively."
argument-hint: "The PHP concept or problem to explore (e.g. type juggling, traits, match expressions, closures)"
---

# Interactive PHP Problem Solver — Core Concepts Explorer

## When to Use

- Building a step-by-step tool for learning and solving PHP concept challenges
- Teaching type juggling, closures, OOP inheritance, traits, interfaces, and PHP 8+ features
- Generating annotated explanations alongside runnable examples for each concept
- Creating a reference for developers moving from other languages to PHP

## Concepts Covered

- **Type juggling & strict types** — loose vs strict comparison, type coercion surprises
- **Arrays** — indexed, associative, multidimensional, array functions pipeline
- **Closures & arrow functions** — `use`, binding, `fn()` syntax
- **OOP** — classes, interfaces, abstract classes, traits, constructor promotion
- **PHP 8+ features** — `match`, named arguments, nullsafe operator `?->`, enums, fibers
- **String functions** — manipulation, formatting, regex
- **Error handling** — exceptions, custom exception hierarchy, `try/catch/finally`
- **Modern patterns** — dependency injection, value objects, fluent interfaces

---

## Problem Structure

Each problem displays:

1. A **prompt** — what the learner must predict or implement
2. A **starter code block** — demonstrates a concept or gotcha
3. **Run / Reset / Reveal** actions
4. An **output panel** — shows what PHP would print
5. An **explanation panel** — plain-language breakdown with the fix or solution

---

## Data Model

Create `src/types/php-explorer.ts`:

```ts
export type ConceptCategory =
  | "type-juggling"
  | "arrays"
  | "closures"
  | "oop"
  | "php8-features"
  | "strings"
  | "error-handling"
  | "patterns";

export type Difficulty = "beginner" | "intermediate" | "advanced";

export interface PhpProblem {
  id: string;
  title: string;
  category: ConceptCategory;
  difficulty: Difficulty;
  /** Short prompt shown above the code block */
  prompt: string;
  /** Starting code shown to the learner */
  starterCode: string;
  /** Expected output lines for auto-check (empty when non-deterministic) */
  expectedOutput: string[];
  /** Plain-language explanation revealed on "Reveal" */
  explanation: string;
  /** Annotated solution shown on "Reveal" */
  solutionCode: string;
  /** Key concept tags */
  tags: string[];
}

export interface OutputLine {
  type: "output" | "error" | "warn";
  value: string;
}

export interface RunResult {
  lines: OutputLine[];
  /** true when all expectedOutput lines match; null when expectedOutput is empty */
  passed: boolean | null;
  errorMessage: string | null;
}
```

**Key conventions:**

- `id` is a kebab-case slug used as the Vue `:key` and route param
- `expectedOutput: []` for problems where output depends on execution environment
- `passed` is `null` when no auto-check is defined — show a neutral badge

---

## Problem Definitions

Create `src/data/php-problems.ts`:

```ts
import type { PhpProblem } from "@/types/php-explorer";

export const problems: PhpProblem[] = [
  // ── Type Juggling ────────────────────────────────────────────────────────
  {
    id: "loose-comparison-trap",
    title: "Loose comparison trap",
    category: "type-juggling",
    difficulty: "intermediate",
    prompt:
      "Predict the output of each comparison. Then rewrite using strict equality.",
    starterCode: `<?php
var_dump(0 == "a");       // ?
var_dump(0 == "");        // ?
var_dump("1" == "01");    // ?
var_dump(100 == "1e2");   // ?
var_dump("" == false);    // ?
var_dump("0" == false);   // ?`,
    expectedOutput: [
      "bool(true)",
      "bool(true)",
      "bool(true)",
      "bool(true)",
      "bool(true)",
      "bool(true)",
    ],
    explanation:
      'PHP\'s `==` coerces both sides to a common type before comparing. A non-numeric string compared to `0` casts the string to `0`. "1e2" is parsed as the float `100`. Use `===` to compare value AND type — no coercion happens.',
    solutionCode: `<?php
// With strict comparison ===
var_dump(0 === "a");    // false — int vs string
var_dump(0 === "");     // false — int vs string
var_dump("1" === "01"); // false — different strings
var_dump(100 === "1e2");// false — int vs string
var_dump("" === false); // false — string vs bool
var_dump("0" === false);// false — string vs bool`,
    tags: ["==", "===", "type coercion", "comparison", "strict_types"],
  },

  {
    id: "strict-types-declaration",
    title: "strict_types and type enforcement",
    category: "type-juggling",
    difficulty: "beginner",
    prompt:
      "What changes when `declare(strict_types=1)` is added? Predict the output with and without it.",
    starterCode: `<?php
// Without strict_types — PHP coerces "3" to int 3
function add(int $a, int $b): int {
    return $a + $b;
}

echo add(2, "3"); // ?`,
    expectedOutput: ["5"],
    explanation:
      'Without `declare(strict_types=1)`, PHP silently coerces "3" to `int 3` — no error. With `declare(strict_types=1)` at the top of the file, passing a string where an `int` is declared throws a `TypeError`. Strict types only apply to the file declaring them, not to called functions in other files.',
    solutionCode: `<?php
declare(strict_types=1);

function add(int $a, int $b): int {
    return $a + $b;
}

// echo add(2, "3"); // TypeError: must be of type int, string given

// Correct:
echo add(2, 3); // 5`,
    tags: ["strict_types", "type hints", "TypeError", "declare"],
  },

  // ── Arrays ───────────────────────────────────────────────────────────────
  {
    id: "array-functions-pipeline",
    title: "Array functions pipeline",
    category: "arrays",
    difficulty: "intermediate",
    prompt:
      "Using only `array_filter`, `array_map`, and `array_reduce`, compute the total price of in-stock items with a 10% discount. No loops.",
    starterCode: `<?php
$inventory = [
    ['name' => 'Widget',      'price' => 25, 'inStock' => true],
    ['name' => 'Gadget',      'price' => 80, 'inStock' => false],
    ['name' => 'Doohickey',   'price' => 40, 'inStock' => true],
    ['name' => 'Thingamajig', 'price' => 15, 'inStock' => true],
];

// Your pipeline here
$total = 0;
echo $total; // 72`,
    expectedOutput: ["72"],
    explanation:
      "Chain: `array_filter` removes out-of-stock items → `array_map` applies the 10% discount → `array_reduce` sums the prices. Note that `array_filter` preserves original keys; `array_values` can re-index if needed. Each function is a pure transformation without mutation.",
    solutionCode: `<?php
$inventory = [
    ['name' => 'Widget',      'price' => 25, 'inStock' => true],
    ['name' => 'Gadget',      'price' => 80, 'inStock' => false],
    ['name' => 'Doohickey',   'price' => 40, 'inStock' => true],
    ['name' => 'Thingamajig', 'price' => 15, 'inStock' => true],
];

$total = array_reduce(
    array_map(
        fn($item) => $item['price'] * 0.9,
        array_filter($inventory, fn($item) => $item['inStock'])
    ),
    fn($carry, $price) => $carry + $price,
    0
);

echo $total; // 72`,
    tags: [
      "array_filter",
      "array_map",
      "array_reduce",
      "functional",
      "pipeline",
    ],
  },

  {
    id: "array-reference-trap",
    title: "Array reference in foreach",
    category: "arrays",
    difficulty: "intermediate",
    prompt: "Predict what is printed after both foreach loops.",
    starterCode: `<?php
$nums = [1, 2, 3, 4];

foreach ($nums as &$val) {
    $val *= 2;
}

foreach ($nums as $val) {
    echo $val . "\n";
}`,
    expectedOutput: ["2", "4", "6", "6"],
    explanation:
      "After the first loop, `$val` is still a reference to the last element of `$nums`. The second loop assigns to `$val` on each iteration — which overwrites `$nums[3]`. The last value printed twice as `6`. Always `unset($val)` after a reference foreach to break the reference.",
    solutionCode: `<?php
$nums = [1, 2, 3, 4];

foreach ($nums as &$val) {
    $val *= 2;
}
unset($val); // Break the reference — always do this!

foreach ($nums as $val) {
    echo $val . "\n"; // 2, 4, 6, 8
}`,
    tags: ["foreach", "reference", "&", "unset", "gotcha"],
  },

  // ── Closures ─────────────────────────────────────────────────────────────
  {
    id: "closure-use-by-value",
    title: "Closure use by value vs reference",
    category: "closures",
    difficulty: "intermediate",
    prompt: "Predict the output of both closures. What is the difference?",
    starterCode: `<?php
$count = 0;

$byValue = function () use ($count) {
    $count++;
    echo $count . "\n";
};

$byRef = function () use (&$count) {
    $count++;
    echo $count . "\n";
};

$byValue();
$byValue();
echo $count . "\n"; // ?

$byRef();
$byRef();
echo $count . "\n"; // ?`,
    expectedOutput: ["1", "1", "0", "1", "2", "2"],
    explanation:
      "`use ($count)` captures a copy of `$count` at closure creation time. The closure has its own copy — the outer variable is unchanged. `use (&$count)` captures a reference — mutations inside the closure affect the outer variable. Use by reference for accumulators or shared state.",
    solutionCode: `<?php
$count = 0;

// By value: captures a copy — outer $count unchanged
$byValue = function () use ($count) {
    $count++;           // modifies the copy
    echo $count . "\n"; // 1 (always 1 — starts from 0 each call)
};

// By reference: captures a reference — outer $count changes
$byRef = function () use (&$count) {
    $count++;           // modifies the original
    echo $count . "\n";
};

$byValue(); // 1
$byValue(); // 1
echo $count . "\n"; // 0 — unchanged

$byRef(); // 1
$byRef(); // 2
echo $count . "\n"; // 2 — modified`,
    tags: ["closure", "use", "by reference", "&", "scope"],
  },

  {
    id: "arrow-function-capture",
    title: "Arrow functions auto-capture",
    category: "closures",
    difficulty: "beginner",
    prompt:
      "Rewrite the closure using PHP 8 arrow function syntax. What does the arrow function capture automatically?",
    starterCode: `<?php
$multiplier = 3;

$multiply = function (int $n) use ($multiplier): int {
    return $n * $multiplier;
};

echo $multiply(5); // 15`,
    expectedOutput: ["15"],
    explanation:
      "Arrow functions (`fn()`) automatically capture variables from the outer scope by value — no `use` keyword needed. They are single-expression and implicitly return the result. They cannot modify outer variables (no reference capture) and cannot span multiple lines.",
    solutionCode: `<?php
$multiplier = 3;

// Arrow function: auto-captures $multiplier by value
$multiply = fn(int $n): int => $n * $multiplier;

echo $multiply(5); // 15

// Equivalent to:
// function (int $n) use ($multiplier): int { return $n * $multiplier; }`,
    tags: ["arrow function", "fn", "capture", "closures", "PHP 7.4"],
  },

  // ── OOP ──────────────────────────────────────────────────────────────────
  {
    id: "trait-conflict-resolution",
    title: "Trait method conflict resolution",
    category: "oop",
    difficulty: "advanced",
    prompt:
      "Two traits both define `greet()`. Predict the error and fix it using `insteadof` and `as`.",
    starterCode: `<?php
trait Hello {
    public function greet(): string {
        return "Hello!";
    }
}

trait Hi {
    public function greet(): string {
        return "Hi!";
    }
}

class Greeter {
    use Hello, Hi;
}

echo (new Greeter())->greet();`,
    expectedOutput: [],
    explanation:
      "When two traits define the same method, PHP throws a fatal error unless you resolve the conflict. Use `insteadof` to specify which trait's method wins, and `as` to alias the losing method under a new name so both remain accessible.",
    solutionCode: `<?php
trait Hello {
    public function greet(): string { return "Hello!"; }
}

trait Hi {
    public function greet(): string { return "Hi!"; }
}

class Greeter {
    use Hello, Hi {
        Hello::greet insteadof Hi;   // Hello wins
        Hi::greet as greetHi;        // Hi is aliased
    }
}

$g = new Greeter();
echo $g->greet();   // Hello!
echo $g->greetHi(); // Hi!`,
    tags: ["trait", "insteadof", "as", "conflict", "mixin"],
  },

  {
    id: "constructor-promotion",
    title: "Constructor property promotion",
    category: "oop",
    difficulty: "beginner",
    prompt:
      "Rewrite the verbose class using PHP 8 constructor property promotion to reduce boilerplate.",
    starterCode: `<?php
class User {
    public int $id;
    public string $name;
    private string $email;

    public function __construct(int $id, string $name, string $email) {
        $this->id    = $id;
        $this->name  = $name;
        $this->email = $email;
    }

    public function getEmail(): string {
        return $this->email;
    }
}

$u = new User(1, 'Alice', 'alice@example.com');
echo $u->name;       // Alice
echo $u->getEmail(); // alice@example.com`,
    expectedOutput: ["Alice", "alice@example.com"],
    explanation:
      "Constructor property promotion (PHP 8.0+) combines property declaration, constructor parameter, and assignment into one. Add a visibility modifier (`public`, `protected`, `private`) to a constructor parameter and PHP handles the rest automatically.",
    solutionCode: `<?php
class User {
    public function __construct(
        public int $id,
        public string $name,
        private string $email,
    ) {}

    public function getEmail(): string {
        return $this->email;
    }
}

$u = new User(1, 'Alice', 'alice@example.com');
echo $u->name;       // Alice
echo $u->getEmail(); // alice@example.com`,
    tags: ["constructor promotion", "PHP 8", "OOP", "boilerplate"],
  },

  {
    id: "interface-vs-abstract",
    title: "Interface vs Abstract class",
    category: "oop",
    difficulty: "intermediate",
    prompt:
      "Design a shape hierarchy. Which requires `interface` and which requires `abstract class`? Implement `Circle` and `Rectangle`.",
    starterCode: `<?php
// Define: Colorable interface (has color, can describe it)
// Define: Shape abstract class (has area() and perimeter())

// class Circle extends Shape implements Colorable { ... }
// class Rectangle extends Shape implements Colorable { ... }

$c = new Circle(5.0, 'red');
echo $c->area();      // 78.54
echo $c->color;       // red`,
    expectedOutput: [],
    explanation:
      "Use an **interface** for a capability contract with no shared state (`Colorable` — any class can be colorable). Use an **abstract class** when you want to share implementation code and enforce a contract (`Shape` — all shapes have area/perimeter logic that may share helpers). A class can implement many interfaces but extend only one abstract class.",
    solutionCode: `<?php
interface Colorable {
    public function describe(): string;
}

abstract class Shape {
    abstract public function area(): float;
    abstract public function perimeter(): float;
}

class Circle extends Shape implements Colorable {
    public function __construct(
        private float $radius,
        public string $color,
    ) {}

    public function area(): float {
        return round(M_PI * $this->radius ** 2, 2);
    }

    public function perimeter(): float {
        return round(2 * M_PI * $this->radius, 2);
    }

    public function describe(): string {
        return "A {$this->color} circle";
    }
}

class Rectangle extends Shape implements Colorable {
    public function __construct(
        private float $width,
        private float $height,
        public string $color,
    ) {}

    public function area(): float { return $this->width * $this->height; }
    public function perimeter(): float { return 2 * ($this->width + $this->height); }
    public function describe(): string { return "A {$this->color} rectangle"; }
}

$c = new Circle(5.0, 'red');
echo $c->area();    // 78.54
echo $c->color;     // red`,
    tags: [
      "interface",
      "abstract",
      "OOP",
      "extends",
      "implements",
      "polymorphism",
    ],
  },

  // ── PHP 8+ Features ──────────────────────────────────────────────────────
  {
    id: "match-expression",
    title: "match vs switch",
    category: "php8-features",
    difficulty: "beginner",
    prompt:
      "Rewrite the switch statement using a `match` expression. What are the key differences?",
    starterCode: `<?php
$status = 2;

switch ($status) {
    case 1:
        $label = 'Active';
        break;
    case 2:
    case 3:
        $label = 'Pending';
        break;
    case 0:
        $label = 'Inactive';
        break;
    default:
        $label = 'Unknown';
}

echo $label; // Pending`,
    expectedOutput: ["Pending"],
    explanation:
      "`match` uses strict (`===`) comparison, returns a value directly, throws `UnhandledMatchError` for unmatched values (no silent fallthrough), and supports comma-separated conditions per arm. It cannot be used when you need fallthrough or side effects per case.",
    solutionCode: `<?php
$status = 2;

$label = match($status) {
    1       => 'Active',
    2, 3    => 'Pending',   // comma = OR condition
    0       => 'Inactive',
    default => 'Unknown',
};

echo $label; // Pending

// Key differences vs switch:
// 1. Strict === comparison (no type coercion)
// 2. Returns a value — assign directly
// 3. No break needed — no fallthrough
// 4. UnhandledMatchError if no arm matches and no default`,
    tags: ["match", "switch", "PHP 8", "strict comparison"],
  },

  {
    id: "nullsafe-operator",
    title: "Nullsafe operator ?->",
    category: "php8-features",
    difficulty: "beginner",
    prompt:
      "Rewrite the nested null-checks using the PHP 8 nullsafe operator `?->`.",
    starterCode: `<?php
class Address {
    public function __construct(public ?string $city = null) {}
}

class User {
    public function __construct(public ?Address $address = null) {}
}

function getCity(?User $user): ?string {
    if ($user === null) return null;
    if ($user->address === null) return null;
    return $user->address->city;
}

$user = new User(new Address('Paris'));
echo getCity($user);  // Paris

$none = null;
echo getCity($none);  // (nothing — null)`,
    expectedOutput: ["Paris"],
    explanation:
      "The nullsafe operator `?->` short-circuits the chain and returns `null` if the left-hand side is `null`, without throwing an error. It eliminates cascading null checks and works with method calls and property access. It cannot be used on the left-hand side of an assignment.",
    solutionCode: `<?php
class Address {
    public function __construct(public ?string $city = null) {}
}

class User {
    public function __construct(public ?Address $address = null) {}
}

function getCity(?User $user): ?string {
    return $user?->address?->city; // returns null at first null in chain
}

$user = new User(new Address('Paris'));
echo getCity($user); // Paris

$none = null;
var_dump(getCity($none)); // NULL`,
    tags: ["nullsafe", "?->", "PHP 8", "null", "chaining"],
  },

  {
    id: "named-arguments",
    title: "Named arguments",
    category: "php8-features",
    difficulty: "beginner",
    prompt:
      "Use named arguments to call `array_slice` and `htmlspecialchars` without memorizing argument order.",
    starterCode: `<?php
$letters = ['a', 'b', 'c', 'd', 'e'];

// Positional — which argument is which?
$slice = array_slice($letters, 1, 3, true);

echo implode(',', $slice); // b,c,d`,
    expectedOutput: ["b,c,d"],
    explanation:
      "Named arguments (PHP 8.0+) let you pass arguments by name in any order, skipping optional parameters without using placeholder values. They work with built-in and user-defined functions and improve readability for functions with many optional parameters.",
    solutionCode: `<?php
$letters = ['a', 'b', 'c', 'd', 'e'];

// Named arguments — self-documenting
$slice = array_slice(
    array: $letters,
    offset: 1,
    length: 3,
    preserve_keys: true,
);

echo implode(',', $slice); // b,c,d

// Also useful for skipping optional params:
$escaped = htmlspecialchars(
    string: '<b>Hello</b>',
    encoding: 'UTF-8',
    // double_encode: true  — skip middle optional
);
echo $escaped; // &lt;b&gt;Hello&lt;/b&gt;`,
    tags: ["named arguments", "PHP 8", "readability"],
  },

  {
    id: "enums",
    title: "Backed enums",
    category: "php8-features",
    difficulty: "intermediate",
    prompt:
      "Model HTTP status codes using a backed enum. Implement a `label()` method and safe creation from an int.",
    starterCode: `<?php
// Define Status enum backed by int
// Values: Ok=200, NotFound=404, ServerError=500

// $status = Status::Ok;
// echo $status->label();  // OK
// echo $status->value;    // 200

// Safe creation from int:
// $s = Status::tryFrom(404);  // Status::NotFound
// $bad = Status::tryFrom(999); // null`,
    expectedOutput: [],
    explanation:
      "Backed enums (PHP 8.1+) associate each case with a scalar value (`int` or `string`). Use `from()` when the value must exist (throws `ValueError` otherwise) and `tryFrom()` for safe nullable creation. Enums can implement interfaces and have methods, but cannot have mutable state.",
    solutionCode: `<?php
enum Status: int {
    case Ok          = 200;
    case NotFound    = 404;
    case ServerError = 500;

    public function label(): string {
        return match($this) {
            Status::Ok          => 'OK',
            Status::NotFound    => 'Not Found',
            Status::ServerError => 'Internal Server Error',
        };
    }
}

$status = Status::Ok;
echo $status->label();  // OK
echo $status->value;    // 200

$s   = Status::tryFrom(404);  // Status::NotFound
$bad = Status::tryFrom(999);  // null

var_dump($s?->label()); // string(9) "Not Found"
var_dump($bad);         // NULL`,
    tags: ["enum", "backed enum", "PHP 8.1", "tryFrom", "match"],
  },

  // ── Strings ──────────────────────────────────────────────────────────────
  {
    id: "string-interpolation",
    title: "String interpolation and heredoc",
    category: "strings",
    difficulty: "beginner",
    prompt:
      "Predict the output of each string literal type. When should you prefer each?",
    starterCode: `<?php
$name = "World";
$price = 9.99;

echo "Hello, $name!\n";
echo 'Hello, $name!\n';
echo "Price: \${price}\n";

$html = <<<EOT
<p>Hello, {$name}!</p>
<p>Price: $price</p>
EOT;

echo $html;`,
    expectedOutput: [],
    explanation:
      "Double-quoted strings (`\"`) interpolate variables and process escape sequences. Single-quoted strings (`'`) are literal — no interpolation, `\\n` is two characters. Heredoc (`<<<EOT`) behaves like double-quoted strings but allows multi-line content without escaping quotes. Nowdoc (`<<<'EOT'`) is the single-quoted equivalent — no interpolation.",
    solutionCode: `<?php
$name  = "World";
$price = 9.99;

echo "Hello, $name!\n";          // Hello, World!  (interpolated + newline)
echo 'Hello, $name!\n';          // Hello, $name!\n  (literal — no interpolation)
echo "Price: \${price}\n";       // Price: ${price}  (\$ escapes $)

// Nowdoc — no interpolation, useful for code samples
$raw = <<<'EOT'
Hello, $name — no interpolation here.
EOT;
echo $raw;

// Heredoc — interpolated multi-line
$html = <<<EOT
<p>Hello, {$name}!</p>
<p>Price: $price</p>
EOT;
echo $html;`,
    tags: ["string", "heredoc", "nowdoc", "interpolation", "single-quote"],
  },

  // ── Error Handling ───────────────────────────────────────────────────────
  {
    id: "exception-hierarchy",
    title: "Custom exception hierarchy",
    category: "error-handling",
    difficulty: "intermediate",
    prompt:
      "Design a custom exception hierarchy for a payment system. Catch specific vs general exceptions in the right order.",
    starterCode: `<?php
// Define: PaymentException (base)
//         InsufficientFundsException extends PaymentException
//         CardDeclinedException extends PaymentException

function charge(float $amount, float $balance): void {
    if ($balance < $amount) {
        throw new InsufficientFundsException("Need $amount, have $balance");
    }
}

try {
    charge(100.0, 50.0);
} catch (/* specific */ $e) {
    echo "Funds: " . $e->getMessage();
} catch (/* general */ $e) {
    echo "Payment failed: " . $e->getMessage();
}`,
    expectedOutput: [],
    explanation:
      "Catch blocks are evaluated top-to-bottom — more specific exceptions must come before more general ones. Catching `PaymentException` before `InsufficientFundsException` would absorb all payment errors, making the specific catch unreachable. PHP 8.0+ union catches (`catch (A|B $e)`) let you handle multiple exception types in one block without hierarchy.",
    solutionCode: `<?php
class PaymentException extends RuntimeException {}
class InsufficientFundsException extends PaymentException {}
class CardDeclinedException extends PaymentException {}

function charge(float $amount, float $balance): void {
    if ($balance < $amount) {
        throw new InsufficientFundsException(
            "Need $$amount, have $$balance"
        );
    }
}

try {
    charge(100.0, 50.0);
} catch (InsufficientFundsException $e) {
    // Most specific — caught first
    echo "Funds: " . $e->getMessage();
} catch (PaymentException $e) {
    // General payment error
    echo "Payment failed: " . $e->getMessage();
} catch (\Throwable $e) {
    // Catch-all for unexpected errors
    echo "Unexpected: " . $e->getMessage();
} finally {
    echo "\nTransaction complete."; // always runs
}`,
    tags: [
      "exception",
      "hierarchy",
      "try/catch",
      "finally",
      "RuntimeException",
    ],
  },

  // ── Patterns ─────────────────────────────────────────────────────────────
  {
    id: "fluent-builder",
    title: "Fluent builder pattern",
    category: "patterns",
    difficulty: "intermediate",
    prompt:
      "Implement a `QueryBuilder` with a fluent interface. Each method should return `$this` to allow chaining.",
    starterCode: `<?php
// $sql = (new QueryBuilder())
//     ->select('id', 'name')
//     ->from('users')
//     ->where('active = 1')
//     ->orderBy('name')
//     ->limit(10)
//     ->build();
//
// echo $sql;
// SELECT id, name FROM users WHERE active = 1 ORDER BY name LIMIT 10`,
    expectedOutput: [],
    explanation:
      "The fluent interface pattern returns `$this` from each mutating method, enabling a readable method chain. Each method accumulates state. `build()` is the terminal method that assembles and returns the final result without returning `$this`. This is not a replacement for a parameterized query — always use prepared statements with real databases to prevent SQL injection.",
    solutionCode: `<?php
class QueryBuilder {
    private array $columns  = ['*'];
    private string $table   = '';
    private array $wheres   = [];
    private ?string $order  = null;
    private ?int $limitVal  = null;

    public function select(string ...$columns): static {
        $this->columns = $columns;
        return $this;
    }

    public function from(string $table): static {
        $this->table = $table;
        return $this;
    }

    public function where(string $condition): static {
        $this->wheres[] = $condition;
        return $this;
    }

    public function orderBy(string $column): static {
        $this->order = $column;
        return $this;
    }

    public function limit(int $n): static {
        $this->limitVal = $n;
        return $this;
    }

    public function build(): string {
        $sql = 'SELECT ' . implode(', ', $this->columns)
             . ' FROM ' . $this->table;

        if ($this->wheres) {
            $sql .= ' WHERE ' . implode(' AND ', $this->wheres);
        }
        if ($this->order !== null) {
            $sql .= ' ORDER BY ' . $this->order;
        }
        if ($this->limitVal !== null) {
            $sql .= ' LIMIT ' . $this->limitVal;
        }
        return $sql;
    }
}

$sql = (new QueryBuilder())
    ->select('id', 'name')
    ->from('users')
    ->where('active = 1')
    ->orderBy('name')
    ->limit(10)
    ->build();

echo $sql;
// SELECT id, name FROM users WHERE active = 1 ORDER BY name LIMIT 10`,
    tags: [
      "fluent interface",
      "builder",
      "method chaining",
      "pattern",
      "static return",
    ],
  },
];
```

---

## Execution Model

PHP runs server-side — a browser-based PHP explorer simulates output by:

1. **Pre-computed outputs** — `expectedOutput` stores what PHP would print; the UI displays it as though run live
2. **Inline annotations** — comments in `solutionCode` show output inline: `// Paris`
3. **No actual PHP execution in-browser** — unlike JavaScript, PHP cannot run in a browser sandbox; the "Run" action shows `expectedOutput` or triggers a backend PHP runner if available

### Optional: Backend PHP Runner

If a PHP runner endpoint is available (e.g. `/api/run-php`):

```ts
async function runPhp(code: string): Promise<OutputLine[]> {
  const response = await fetch("/api/run-php", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code }),
  });
  const data = await response.json();
  return data.output; // [{ type: 'output', value: '...' }]
}
```

**Security:** A real PHP runner endpoint MUST:

- Execute code inside a sandboxed container (Docker with no network, read-only filesystem)
- Set execution time limits (`max_execution_time`)
- Disable dangerous functions: `exec`, `shell_exec`, `system`, `passthru`, `file_put_contents`
- Never run as root

---

## Explanation Logic

Each problem exposes:

1. **What** — what the code actually does (vs what learners expect)
2. **Why** — the PHP rule or runtime behavior causing it
3. **Fix** — the idiomatic PHP solution
4. **Annotations** — inline comments in `solutionCode` pointing to the key lines

### Reveal Flow

1. User clicks **Reveal** → `ExplanationPanel` appears below the output
2. Code editor becomes read-only, filled with `solutionCode`
3. Output panel pre-fills with `expectedOutput` (or re-runs if backend available)
4. **Try Again** resets to `starterCode` and hides explanation

---

## Difficulty Badges

| Value          | Color  | Use for                                                                  |
| -------------- | ------ | ------------------------------------------------------------------------ |
| `beginner`     | green  | Single concept, no surprising behavior                                   |
| `intermediate` | yellow | Two interacting features, common gotchas (`&` in foreach, type juggling) |
| `advanced`     | red    | Requires deep PHP internals knowledge (trait conflict, refcounting)      |

---

## Examples

### Type juggling gotcha

```php
<?php
// Loose comparison with 0 and strings
var_dump(0 == "php");  // true in PHP 7, false in PHP 8!
// PHP 8 changed: non-numeric strings compared to 0 now return false
// Always use === to avoid version-dependent surprises
```

### PHP 8.0 vs PHP 7 breaking change note

> `0 == "php"` returns `true` in PHP 7 (string cast to 0) but `false` in PHP 8 (0 cast to string "0"). Problems covering this should note the PHP version when behavior differs.

---

## Adding a New Problem

1. Add a `PhpProblem` entry to `src/data/php-problems.ts`
2. Set `category` to an existing `ConceptCategory` (or extend the union type)
3. Write `starterCode` that shows the gotcha or exercise — not the answer
4. `explanation`: state **what** happens, **why**, then **the fix**
5. `expectedOutput`: exact strings PHP `echo`/`print`/`var_dump` would produce synchronously; use `[]` when output is environment-dependent
6. `solutionCode` must be a **complete, self-contained runnable PHP file** starting with `<?php` and including all declarations needed to produce `expectedOutput`
7. Note PHP version requirements in `tags` when a feature requires PHP 7.4+ or PHP 8.0+
