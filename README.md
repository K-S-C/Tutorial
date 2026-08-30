# The K Programming Language — Full Tutorial & Reference

K is a small, self-contained, general-purpose language: variables,
closures, classes, error handling, and matrix math are all **built-in
syntax**, not a library you import. Source compiles to bytecode
(`compiler.rs`) and runs on a stack-based VM (`vm.rs`) — one binary, no
interpreter to install, no package manager.

```k
class Dog {
    fn init(name) { self.name = name; }
    fn speak() { return f"{self.name} says Woof!"; }
}
let d = new Dog("Rex");
print(d.speak());

let weights = [[0.8, -0.1], [0.4, 0.9]];
let inputs  = [[1.5, 0.2]];
print(relu(inputs @ weights));   // matrix multiply is a language operator, '@'
```

---

## Table of contents

1. [Getting started (CLI, REPL, GUI)](#1-getting-started-cli-repl-gui)
2. [Lexical basics — comments, semicolons, tokens](#2-lexical-basics)
3. [Variables: `let` and `const`](#3-variables-let-and-const)
4. [Data types](#4-data-types)
5. [Operators](#5-operators)
6. [String interpolation](#6-string-interpolation)
7. [Lists and dicts](#7-lists-and-dicts)
8. [Indexing (lists, strings, dicts)](#8-indexing)
9. [Control flow](#9-control-flow)
10. [Functions](#10-functions)
11. [Closures](#11-closures)
12. [Classes and inheritance](#12-classes-and-inheritance)
13. [Error handling: try / catch / throw](#13-error-handling-try--catch--throw)
14. [`import`](#14-import)
15. [Built-in functions (complete list)](#15-built-in-functions-complete-list)
16. [Built-in methods (list / dict / string)](#16-built-in-methods-list--dict--string)
17. [Matrix math](#17-matrix-math)
18. [Operator precedence table](#18-operator-precedence-table)
19. [Full worked examples](#19-full-worked-examples)
20. [Gotchas and things that surprise people](#20-gotchas-and-things-that-surprise-people)
21. [Quick syntax cheat-sheet](#21-quick-syntax-cheat-sheet)

---

## 1. Getting started (CLI, REPL, GUI)

The compiled binary is called `k`. It supports:

| Command | What it does |
|---|---|
| `k script.k` | Lex → parse → compile → run the file, print output |
| `k` (no args) | Start the interactive REPL |
| `k repl` | Same as above, explicit |
| `k gui` | Launch the built-in graphical IDE (dark/light theme, F5 to run, Ctrl+S save, Ctrl+O open, Ctrl+N new) |
| `k -h` / `--help` | Print usage |
| `k -v` / `--version` | Print version |

**REPL details:** one VM instance stays alive for the whole session, so
variables/functions/classes defined on one line are visible on the
next. It supports arrow-key history, Ctrl+R search, and automatically
keeps reading additional lines while you have an unclosed `{`, `(`, or
`[` (shown with a `...` prompt). Special commands (must be typed at the
start of a fresh line):

- `:help` — show REPL help
- `:load <file.k>` — load and execute a `.k` file into the current session
- `:vars` — list all current global variables and their values
- `:clear` — clear the screen
- `:exit` or `:q` (or Ctrl+D) — quit

```
$ k repl
K Language REPL v0.1.0 — :help for commands, Ctrl+D or :exit to quit
k> let x = 40;
k> print(x + 2);
42
k> :vars
  x = 40
k> :exit
Goodbye.
```

Every stage (lexer, parser, compiler) returns a `Result` — a bad script
never crashes the process, it prints `Lex error: ...`, `Parse error:
...`, or `Compile error: ...` and stops.

---

## 2. Lexical basics

- **Statements end in `;`.** The parser is lenient about a missing
  trailing `;` at the very end of a block (`match_tok` just doesn't
  find one and moves on), but always terminate statements with `;` —
  it's the reliable, idiomatic style used throughout every example.
- **Comments:**
  - Line comments: `// like this, to end of line`
  - Block comments: `/* like this, can span lines */` — not nestable
    (an unterminated block comment is a lex error).
- **Identifiers:** start with a letter or `_`, followed by
  letters/digits/`_`. K is case-sensitive.
- **Numbers:** `42` is an `Int` (`i64`). `3.14` is a `Float` (`f64`). A
  literal is only recognized as a float if there's at least one digit
  *after* the dot (so `1.` alone would not lex as a float digit
  sequence the way you might expect — always write `1.0`).
- **Strings:** double-quoted only, `"like this"`. Supported escapes:
  `\n`, `\t`, `\r`, `\"`, `\\` (any other escaped character is passed
  through literally). Single-quoted strings are **not** supported.
- **Reserved keywords** (cannot be used as identifiers):
  ```
  let const fn return if elif else while for in break continue
  true false null class new try catch throw and or not import
  ```

---

## 3. Variables: `let` and `const`

```k
let name = "K";
const version = 1;
```

- `let` declares a mutable binding.
- `const` declares a binding that is *intended* to be constant.

> **Important — verified from the compiler source:** `let` and `const`
> currently compile through the exact same code path and produce
> identical bytecode. There is **no runtime enforcement** that
> prevents reassigning a `const` — it behaves exactly like `let`. Treat
> `const` as a documentation/intent signal for readers of your code,
> not as a language guarantee.

**Optional type annotations** are accepted after the name for
documentation purposes only — the parser consumes and discards them,
they do no type checking:

```k
let count: int = 0;      // ": int" is parsed and ignored
fn add(a: int, b: int) -> int {   // param types and "-> int" return type: also ignored
    return a + b;
}
```

Reassignment uses `=`, `+=`, `-=`, `*=`, `/=` (there is no `%=` or
`**=`):

```k
let total = 0;
total += 5;   // total = total + 5
total -= 1;
total *= 2;
total /= 3;
```

Variables declared at the top level of a script are **globals**;
variables declared with `let`/`const` inside a function body, loop, or
block are **locals**, resolved by the compiler to fixed stack slots
(fast — no hash-map lookup at runtime).

---

## 4. Data types

| Type | Example literal | Rust representation |
|---|---|---|
| `int` | `42`, `-7` | `i64` |
| `float` | `3.14`, `-0.5` | `f64` |
| `str` | `"hello"` | `Rc<String>` |
| `bool` | `true`, `false` | `bool` |
| `null` | `null` | unit |
| `list` | `[1, 2, 3]` | `Rc<RefCell<Vec<Value>>>` (reference type!) |
| `dict` | `{"a": 1, "b": 2}` | `Rc<RefCell<HashMap<String, Value>>>` (reference type!) |
| `func` | a closure created by `fn` or a function expression | — |
| `method` | a bound method (`instance.method`) | — |
| `builtin` | a native function like `print` | — |
| `class` | created with `class` | — |
| `instance` | created with `new ClassName(...)` | — |

`type(value)` returns the type name above as a string, e.g.
`type(42)` → `"int"`, `type([1,2])` → `"list"`.

**Lists and dicts are reference types.** Assigning a list/dict to
another variable, or passing it into a function, shares the *same*
underlying storage — mutating it through one name is visible through
the other:

```k
let a = [1, 2, 3];
let b = a;
b.append(4);
print(a);   // [1, 2, 3, 4]  -- a and b are the same list
```

### Truthiness

Used by `if`, `while`, `and`/`or`, and unary `!`/`not`:

| Value | Truthy? |
|---|---|
| `false`, `null` | falsy |
| `0` (int) | falsy |
| `0.0` (float) | falsy |
| `""` (empty string) | falsy |
| `[]` (empty list) | falsy |
| `{}` (empty dict) | falsy |
| anything else (including negative numbers, non-empty containers, functions, instances) | truthy |

### Equality (`==` / `!=`)

- `Int`/`Float` compare across types numerically (`1 == 1.0` is `true`).
- `Str == Str` compares text.
- `List == List` compares length and element-wise equality
  (deep/structural, recursive).
- **Dicts, instances, functions, and classes are never equal to
  anything via `==`** (not even to themselves as separate values) —
  the VM's equality function has no case for them and falls through to
  `false`. Don't rely on `==` to compare dicts or instances.

---

## 5. Operators

### Arithmetic

| Op | Meaning | Notes |
|---|---|---|
| `+` | add / concatenate | see below |
| `-` | subtract | `int - int` stays `int`; otherwise `float` |
| `*` | multiply | `int * int` stays `int`; otherwise `float` |
| `/` | divide | **always returns a `float`**, even `4 / 2` → `2.0`. Dividing by zero throws a catchable runtime error `"division by zero"`. |
| `%` | modulo | `int % int` (nonzero divisor) stays `int`; otherwise `float` |
| `**` | power | **always returns a `float`**; right-associative (`2 ** 3 ** 2` = `2 ** (3 ** 2)` = `512.0`) |
| unary `-` | negate | **always returns a `float`** — `-5` as a standalone unary expression evaluates to `-5.0`, not the int literal. (`let x = -5;` still gives you `-5.0`, not int `-5`. Only the bare literal `5` used positively is an `Int`.) |
| `@` | matrix multiply | see [§17 Matrix math](#17-matrix-math) |

**`+` overloads by type**, checked in this order:
1. If either operand is a `str`, the *other* operand is converted to
   its display string and concatenated: `"n=" + 5` → `"n=5"`,
   `5 + "!"` → `"5!"`.
2. If both operands are `list`, they're concatenated into a new list:
   `[1,2] + [3]` → `[1, 2, 3]`.
3. Otherwise, numeric addition (`int+int` stays `int`; mixed/float
   promotes to `float`).

### Comparison

`==  !=  <  >  <=  >=`

- Numbers compare numerically.
- Strings compare lexicographically (`"apple" < "banana"` → `true`).
- Mixing incompatible types (e.g. comparing a list with `<`) is a
  runtime error.

### Logical

- `and` / `or` (keyword form) or `&&` / `||` (symbol form) — both are
  accepted and mean the same thing. These are **short-circuiting**:
  the right side is only evaluated if needed.
- `not` (keyword) or `!` (symbol) for logical negation. Both produce a
  `bool` regardless of the input's type (uses the truthiness table
  above).
- There is **no bitwise `&` or `|`** — a bare `&` or `|` not followed
  by its doubled partner is a lex error ("did you mean `&&`?").

### Assignment

`=` `+=` `-=` `*=` `/=` — valid assignment targets are a plain
identifier (`x = 1`), an index (`list[0] = 1`, `dict["k"] = 1`), or a
field (`obj.field = 1`). Every assignment expression also evaluates to
the assigned value, so `print(x = 5)` prints `5`.

---

## 6. String interpolation

Prefix a string literal with `f` to enable `{expr}` interpolation —
any full K expression can go inside the braces:

```k
let name = "World";
let n = 3;
print(f"Hello, {name}! {n + 1} greetings.");
// Hello, World! 4 greetings.
```

- Escapes (`\n`, `\t`, `\"`, `\\`) work the same as in normal strings.
- Braces can nest (e.g. for a dict/list literal inside the
  interpolation) — the lexer tracks brace depth so `{ {1:2}["1"] }`-style
  expressions parse correctly.
- Regular (non-`f`) strings do **not** interpolate — `"{name}"` prints
  literally as `{name}`.

---

## 7. Lists and dicts

### List literals

```k
let empty = [];
let nums = [1, 2, 3];
let mixed = [1, "two", 3.0, true, null];
let nested = [[1, 2], [3, 4]];   // used for matrices, see §17
```

### Dict literals

Keys can be any expression, but only string keys actually work for
lookup — non-string keys are converted to their display string when
the dict is built (`OpCode::BuildDict` does `to_display` on any
non-`Str` key), so prefer literal string keys:

```k
let d = {"a": 1, "b": 2, "c": 3};
print(d["a"]);        // 1
d["d"] = 4;            // add/replace a key
print(d.keys());       // ["a", "b", "c", "d"]  (order not guaranteed — backed by a HashMap)
```

> Dict key order is **not preserved** — it's backed by a Rust
> `HashMap`, so don't rely on insertion order when iterating `keys()`
> or `for k in dict`.

---

## 8. Indexing

Lists, strings, and dicts all support `target[index]`:

```k
let l = [10, 20, 30];
print(l[0]);     // 10
print(l[-1]);     // 30  -- negative indices count from the end
l[1] = 99;         // list index assignment

let s = "hello";
print(s[0]);      // "h"  -- indexing a string returns a 1-character string
print(s[-1]);      // "o"

let d = {"x": 1};
print(d["x"]);    // 1
```

- **Negative indexing is supported** for lists and strings (`-1` is
  the last element), but out-of-range indices (positive or negative)
  raise a catchable runtime error, e.g. `"list index out of range"`.
- **Strings are immutable** — there is no `s[0] = "H"` form (only
  `List` and `Dict` support index *assignment*; assigning into a
  string index is a runtime error: "invalid index assignment target").
- Dict indexing requires a string key; a missing key raises
  `"key '<k>' not found"`.

---

## 9. Control flow

### `if` / `elif` / `else`

```k
if score >= 90 {
    print("A");
} elif score >= 80 {
    print("B");
} else if score >= 70 {   // 'else if' also works, same as a chained 'elif'
    print("C");
} else {
    print("F");
}
```

Note: **conditions are not parenthesized** and bodies always require
braces `{ }` — there is no one-line, brace-less `if`.

### `while`

```k
let i = 0;
while i < 3 {
    print("while loop:", i);
    i += 1;
}
```

### `for ... in`

Iterates over a list, a string (character by character), or a dict
(its keys):

```k
for n in range(1, 6) { print(n); }     // 1 2 3 4 5
for ch in "abc" { print(ch); }          // "a" "b" "c"
for key in {"x": 1, "y": 2} { print(key); }   // iterates dict keys (order not guaranteed)
```

### `break` / `continue`

Work exactly as you'd expect, including inside nested loops (each
`break`/`continue` applies to its own innermost enclosing loop).

```k
for i in range(0, 10) {
    if i == 3 { continue; }
    if i == 6 { break; }
    print(i);
}
// 0 1 2 4 5
```

---

## 10. Functions

```k
fn greet(who, greeting = "Hello") {
    return f"{greeting}, {who}!";
}
print(greet("world"));            // Hello, world!
print(greet("K", "Welcome"));     // Welcome, K!
```

- Declared with `fn name(params) { ... }`.
- **Default parameter values** are supported (`greeting = "Hello"`
  above) — the default expression is evaluated fresh at call time if
  the argument is omitted.
- `return expr;` returns a value; a bare `return;` (or falling off the
  end of the function body) returns `null`.
- Functions support full **recursion**, including through
  nested/locally-declared functions.
- Calling with too few arguments does **not** error — missing
  positional args (with no default) are simply bound to `null`; extra
  arguments beyond the declared parameters are silently ignored (the
  VM only reads up to `arity` args when building the call frame).

### Anonymous functions / function expressions

`fn(params) { ... }` used as an expression, without a name, creates a
closure value you can store or pass around:

```k
let square = fn(x) { return x * x; };
print(square(5));   // 25

let add = fn(a, b) { return a + b; };
print(add(2, 3));   // 5
```

---

## 11. Closures

K functions are real closures: a nested function that references a
variable from an enclosing function keeps a live link to that
variable (captured by reference through a shared cell), not a
snapshot copy.

```k
fn make_counter() {
    let count = 0;
    fn increment() {
        count += 1;
        return count;
    }
    return increment;
}

let counter1 = make_counter();
let counter2 = make_counter();
print(counter1());   // 1
print(counter1());   // 2
print(counter2());   // 1  -- independent state, not shared with counter1
```

Under the hood: a plain local variable lives directly on the VM stack
(cheap). Only variables actually *captured* by a nested closure get
boxed into a shared, reference-counted cell — so closures cost extra
only where you actually use them. This works through multiple levels
of nesting (verified in the source comments up to 3-level nested
closures).

---

## 12. Classes and inheritance

```k
class Shape {
    fn init(name) { self.name = name; }
    fn area() { return 0; }
    fn describe() { return f"{self.name} has area {self.area()}"; }
}

class Circle(Shape) {
    fn init(radius) {
        self.name = "Circle";
        self.radius = radius;
    }
    fn area() { return 3.14159 * self.radius ** 2; }
}

let c = new Circle(3);
print(c.describe());   // Circle has area 28.27431
```

- `class Name { ... }` declares a class; a class body may **only**
  contain methods (`fn ...`) — no top-level fields, no class-level
  constants.
- `class Name(Parent) { ... }` declares single inheritance from
  `Parent`. Method lookup walks up the parent chain if a method isn't
  found on the instance's own class (dynamic dispatch works: calling
  `self.area()` from `describe()` in `Shape` correctly calls
  `Circle.area()` on a `Circle` instance).
- `fn init(...)` is the constructor, called automatically by `new
  ClassName(args)`. If a class has no `init`, `new` just creates an
  empty instance.
- **`self` is implicit** inside every method — do **not** declare it
  as a parameter. It's automatically bound to slot 0 of every method
  call.
- Fields are created dynamically the first time you assign
  `self.field = value` — there's no upfront field declaration; assign
  whatever fields you need in `init` (or later).
- `new ClassName(args)` instantiates: allocates the instance, runs
  `init` (if defined) with those args, and returns the instance.

---

## 13. Error handling: try / catch / throw

Errors in K are ordinary catchable values — a runtime error (like
division by zero, a missing key, an out-of-range index) never crashes
the process; it unwinds to the nearest enclosing `try`/`catch`,
propagating cleanly across nested function calls.

```k
fn safe_divide(a, b) {
    try {
        return a / b;
    } catch e {
        print("error:", e);
        return null;
    }
}

print(safe_divide(10, 2));   // 5.0
print(safe_divide(10, 0));   // error: division by zero  -> then prints null
print("program kept running");
```

- `try { ... } catch <name> { ... }` — the catch variable name is
  optional; if omitted it defaults to `err` (`catch { ... }` is the
  same as `catch err { ... }`).
- `throw expr;` raises `expr` as the error value — it can be any K
  value, not just a string (`throw "bad input";`, `throw 42;`, `throw
  {"code": 404};` are all valid).
- An error thrown with no enclosing `try`/`catch` propagates all the
  way out and the program prints `Uncaught error: <value>` and stops.
- `try`/`catch` correctly unwinds across nested function calls — a
  `throw` deep inside several levels of function calls is caught by
  the nearest active `try` up the call chain, not just the immediate
  caller.

---

## 14. `import`

```k
import "lib.k";
```

`import "<path>"` reads the named file **at compile time**, tokenizes
and parses it, and splices its statements directly into the current
compile unit (like a textual include) — it is not a module system with
namespaces; everything imported lands directly in the current scope.
The path is relative to wherever the `k` process is run from.

---

## 15. Built-in functions (complete list)

These are always available as globals (no import needed) — this is
the **exact and complete list** implemented by the VM, nothing more:

| Function | Signature | Behavior |
|---|---|---|
| `print(...)` | `print(a, b, c, ...)` | Prints all args, space-separated, followed by a newline. Returns `null`. |
| `len(x)` | list / dict / str | Number of elements / keys / characters. |
| `str(x)` | any | Converts to its display string. |
| `int(x)` | int / float / bool / str | Converts to `int`. Truncates floats. Parses numeric strings (trims whitespace); a non-numeric string errors. |
| `float(x)` | any number-like | Converts to `float`. |
| `bool(x)` | any | Converts using the truthiness table (§4). |
| `type(x)` | any | Returns the type name as a string (`"int"`, `"list"`, etc). |
| `range(end)` / `range(start, end)` / `range(start, end, step)` | ints | Returns a `list` of ints. `step` cannot be `0`. Works with negative steps for descending ranges. |
| `abs(x)` | number | Absolute value, **always returned as a `float`**. |
| `min(...)` | `min(a, b, ...)` or `min(list)` | Minimum, as a `float`. |
| `max(...)` | `max(a, b, ...)` or `max(list)` | Maximum, as a `float`. |
| `sum(list)` | list of numbers | Sum, as a `float`. |
| `sorted(list)` | list of numbers | Returns a **new** sorted list (ascending); does not mutate the input. |
| `round(x)` | number | Rounds to nearest integer, returned as `int`. |
| `input()` | — | Present but currently always returns an empty string `""` (not wired to real stdin reading). |
| `relu(x)` | number or (nested) list | Elementwise `max(x, 0)`. |
| `sigmoid(x)` | number or (nested) list | Elementwise `1 / (1 + e^-x)`. |
| `tanh(x)` | number or (nested) list | Elementwise hyperbolic tangent. |
| `softmax(list)` | flat list of numbers | Numerically-stable softmax (subtracts the max before exponentiating). |
| `transpose(matrix)` | list of lists | Matrix transpose. |
| `flatten(x)` | (nested) list | Recursively flattens any depth of nested lists into one flat list. |

Everything else (e.g. `print`, `min`, `max`) is exactly what's shown
above — there's no `math.sqrt`, no file I/O, no `random`, no `map`,
`filter`, or `reduce` builtin. If you need behavior beyond this list,
you write it yourself in K.

---

## 16. Built-in methods (list / dict / string)

Methods are called with dot syntax: `value.method(args)`.

### List methods

| Method | Effect |
|---|---|
| `.append(x)` / `.push(x)` | Adds `x` to the end (mutates in place). Returns `null`. |
| `.pop()` | Removes and returns the last element (mutates in place). Returns `null` if the list is empty. |
| `.sort()` | Sorts numerically ascending, **in place**. Returns `null`. |
| `.reverse()` | Reverses **in place**. Returns `null`. |
| `.contains(x)` | Returns `bool` — whether `x` is present (uses the same deep equality as `==`). |

### Dict methods

| Method | Effect |
|---|---|
| `.keys()` | Returns a `list` of keys (order not guaranteed). |
| `.values()` | Returns a `list` of values (order not guaranteed). |
| `.get(key)` / `.get(key, default)` | Returns the value for `key`, or `default` (or `null` if no default given) when missing. Does **not** error on a missing key (unlike `dict[key]`). |
| `.remove(key)` | Removes `key` and returns its value, or `null` if absent. |

### String methods

Strings are immutable — every string method returns a **new** string
(or other value) rather than mutating.

| Method | Effect |
|---|---|
| `.upper()` | Uppercase copy. |
| `.lower()` | Lowercase copy. |
| `.trim()` | Copy with leading/trailing whitespace removed. |
| `.split(sep)` | Splits on `sep` (defaults to `" "` if omitted), returns a `list` of strings. |
| `.replace(old, new)` | Replaces all occurrences of `old` with `new`. |
| `.contains(sub)` | `bool` — substring test. |
| `.startsWith(sub)` | `bool`. |
| `.endsWith(sub)` | `bool`. |

```k
let s = "  Hello World  ";
print(s.trim().lower());              // "hello world"
print(s.trim().split(" "));           // ["Hello", "World"]
print("banana".replace("a", "o"));    // "bonono"
print("K lang".startsWith("K"));      // true
```

---

## 17. Matrix math

Matrices are just **lists of lists of numbers** — there's no separate
matrix type. `@` is a real binary operator (parses at the same
precedence level as `*`, `/`, `%`) that performs matrix multiplication:

```k
let inputs  = [[1.5, 0.2]];
let weights = [[0.8, -0.1], [0.4, 0.9]];

let hidden = inputs @ weights;
print("hidden:", hidden);                    // [[1.28, 0.03]]
print("activated:", relu(hidden));
print("transposed weights:", transpose(weights));
print("softmax:", softmax([2.0, 1.0, 0.1]));
```

- `a @ b` requires both operands to be non-empty "matrices" (lists of
  lists of numbers) with compatible inner dimensions
  (`a`'s row length must equal `b`'s row count) — otherwise it's a
  runtime error ("matrix dimension mismatch" or "requires two
  matrices").
- `transpose(m)` swaps rows/columns.
- `relu`, `sigmoid`, `tanh` apply elementwise and work on scalars,
  flat lists, or nested (matrix) lists uniformly — they recurse into
  nested lists automatically.
- `softmax(list)` expects a flat list of numbers, not a matrix.
- `flatten(m)` collapses any depth of nested lists into one flat list.
- All matrix/elementwise results come back as `float` values.

---

## 18. Operator precedence table

From loosest to tightest binding (matches the parser's recursive-descent
chain exactly):

| Level | Operators | Associativity |
|---|---|---|
| 1 (loosest) | `=` `+=` `-=` `*=` `/=` (assignment) | right |
| 2 | `or` / `\|\|` | left |
| 3 | `and` / `&&` | left |
| 4 | `==` `!=` | left |
| 5 | `<` `>` `<=` `>=` | left |
| 6 | `+` `-` | left |
| 7 | `*` `/` `%` `@` | left |
| 8 | `**` (power) | **right** |
| 9 | unary `-`, `!` / `not` | — |
| 10 (tightest) | call `()`, index `[]`, field `.` | left |

```k
print(2 + 3 * 4 ** 2);   // 2 + 3*16 = 50
print(-2 ** 2);           // unary binds looser than **: -(2**2) = -4.0
```

---

## 19. Full worked examples

These are the actual example scripts shipped in `examples/`, annotated.

### `basics.k` — variables, control flow, functions, interpolation

```k
// K basics: variables, control flow, functions, string interpolation.

let name = "K";
const version = 1;
print(f"Hello from {name} v{version}!");

fn greet(who, greeting = "Hello") {
    return f"{greeting}, {who}!";
}
print(greet("world"));
print(greet("K", "Welcome"));

let total = 0;
for n in range(1, 6) {
    total += n;
}
print(f"1..5 sums to {total}");

let i = 0;
while i < 3 {
    print("while loop:", i);
    i += 1;
}
```

**Output:**
```
Hello from K v1!
Hello, world!
Welcome, K!
1..5 sums to 15
while loop: 0
while loop: 1
while loop: 2
```

### `classes.k` — inheritance and implicit `self`

```k
// Classes with inheritance. 'self' is implicit inside a method --
// do not declare it as a parameter.

class Shape {
    fn init(name) { self.name = name; }
    fn area() { return 0; }
    fn describe() { return f"{self.name} has area {self.area()}"; }
}

class Circle(Shape) {
    fn init(radius) {
        self.name = "Circle";
        self.radius = radius;
    }
    fn area() { return 3.14159 * self.radius ** 2; }
}

let c = new Circle(3);
print(c.describe());
```

**Output:** `Circle has area 28.27431`

### `errors.k` — catchable errors

```k
// Errors are catchable values, never a crash.

fn safe_divide(a, b) {
    try {
        return a / b;
    } catch e {
        print("error:", e);
        return null;
    }
}

print(safe_divide(10, 2));
print(safe_divide(10, 0));
print("program kept running");
```

**Output:**
```
5.0
error: division by zero
null
program kept running
```

### `matrix_math.k` — `@` as a language operator

```k
// Matrix math is built-in syntax ('@'), not a library import.

let inputs = [[1.5, 0.2]];
let weights = [[0.8, -0.1], [0.4, 0.9]];

let hidden = inputs @ weights;
print("hidden:", hidden);
print("activated:", relu(hidden));
print("transposed weights:", transpose(weights));
print("softmax:", softmax([2.0, 1.0, 0.1]));
```

**Output:**
```
hidden: [[1.28, 0.03]]
activated: [[1.28, 0.03]]
transposed weights: [[0.8, 0.4], [-0.1, 0.9]]
softmax: [0.710949502625004, 0.2595601109490743, 0.02949038642592176]
```

### A bonus combined example (closures + classes + errors + lists)

```k
class Stack {
    fn init() { self.items = []; }
    fn push(x) { self.items.append(x); }
    fn pop() {
        if len(self.items) == 0 {
            throw "pop from empty stack";
        }
        return self.items.pop();
    }
    fn size() { return len(self.items); }
}

let s = new Stack();
s.push(1);
s.push(2);
s.push(3);
print(f"stack size: {s.size()}");
print(s.pop());   // 3

try {
    s.pop(); s.pop();
    s.pop();       // this one throws: stack is now empty
} catch e {
    print(f"caught: {e}");
}

fn make_adder(n) {
    fn adder(x) { return x + n; }
    return adder;
}
let add5 = make_adder(5);
print(add5(10));   // 15
```

---

## 20. Gotchas and things that surprise people

- **`const` doesn't actually stop reassignment.** It compiles
  identically to `let` — it's a naming convention only.
- **Type annotations (`: int`, `-> int`) are parsed and thrown away.**
  K has no static type checking; annotate for readability only.
- **`/`, `**`, and unary `-` always produce a `float`**, even on two
  ints. `4 / 2` is `2.0`, not `2`. `-5` as an expression is `-5.0`.
  Only `+`, `-` (binary), `*`, and `%` preserve `int` when both
  operands are `int`.
- **Lists and dicts are shared references, not copies**, when
  assigned or passed to functions — mutating one name mutates every
  other name pointing at the same list/dict.
- **`==` never returns `true` for two dicts or two instances**, even
  if they hold identical data — the equality function has no case for
  those types.
- **Dict/keys iteration order is not guaranteed** (backed by a hash
  map, not an ordered map).
- **Extra function arguments are silently ignored; missing ones become
  `null`** — there's no arity-mismatch error at call time.
- **`self` must never be declared as a parameter** in a method — it's
  implicitly bound to slot 0 automatically.
- **Class bodies may only contain `fn` methods** — no fields, no
  nested classes, no class-level constants.
- **String indices/negative indices work, but strings are
  immutable** — you cannot assign into `s[i]`.
- **`import` is a compile-time textual splice**, not a namespaced
  module system — everything imported lands directly in your current
  scope, so name collisions are your responsibility.
- **`input()` is a stub** — it always returns `""`, it does not
  actually read from stdin in the current implementation.
- **No bitwise `&`/`|`** — only the doubled `&&`/`||` (logical) forms
  exist; a lone `&` or `|` is a lex error.
- **No `elif`-less one-liner `if`** — braces are always required, and
  there's no ternary `? :` operator.

---

## 21. Quick syntax cheat-sheet

```k
// variables
let x = 1;
const PI = 3.14159;

// types
let i = 1; let f = 1.5; let s = "hi"; let b = true; let n = null;
let l = [1, 2, 3];
let d = {"key": "value"};

// string interpolation
print(f"x is {x}, doubled is {x * 2}");

// control flow
if x > 0 { }
elif x == 0 { }
else { }

while x < 10 { x += 1; }

for item in [1, 2, 3] { print(item); }
for i in range(0, 10, 2) { print(i); }   // start, end, step

// functions
fn add(a, b = 0) { return a + b; }
let square = fn(x) { return x * x; };

// closures
fn counter() {
    let n = 0;
    fn inc() { n += 1; return n; }
    return inc;
}

// classes
class Animal {
    fn init(name) { self.name = name; }
    fn speak() { return "..."; }
}
class Dog(Animal) {
    fn speak() { return f"{self.name} says Woof"; }
}
let d = new Dog("Rex");

// errors
try {
    throw "oops";
} catch e {
    print(e);
}

// matrices
let m = [[1, 2], [3, 4]] @ [[5, 6], [7, 8]];

// import (compile-time textual include)
import "helpers.k";
```
