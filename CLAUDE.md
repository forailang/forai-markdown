# forai Language Reference

## What is forai

forai is a programming language compiled to WebAssembly, designed so that AI can write correct programs on the first try.

### Core Principles

1. **One way to do things.** There are no style debates. No `if/else` — use `match`. No semicolons. No optional braces. If two people write the same logic, they produce the same code.

2. **Tests are not optional.** Every public function must have a `test` block. Every public function, type, and enum must have a `doc` block. The compiler rejects code that lacks them. This means every program is documented and tested by construction — not by convention.

3. **Errors are values.** There are no exceptions. Functions that can fail return `!T` (Result). Functions that may have no value return `?T` (Optional). The `?` operator propagates errors. Pattern matching (`ok`/`err`, `some`/`none`) handles them explicitly.

4. **Minimal surface area.** The entire language fits in this document. There are ~40 keywords, one looping construct (`each`), one branching construct (`match`), and a small standard library. Less to learn means fewer mistakes.

## Types

### Primitives

| Type   | Description                 |
|--------|-----------------------------|
| `i64`  | 64-bit signed integer       |
| `i32`  | 32-bit signed integer       |
| `f64`  | 64-bit float                |
| `f32`  | 32-bit float                |
| `bool` | `true` or `false`           |
| `str`  | UTF-8 string                |
| `byte` | Single byte                 |

### Composite Types

| Syntax   | Description              | Example                              |
|----------|--------------------------|--------------------------------------|
| `[T]`    | Array                    | `[1, 2, 3]`                         |
| `{K:V}`  | Map                      | `{1: 100, 2: 200}`                  |
| `(T,T)`  | Tuple                    | `(10, 20)`                          |
| `?T`     | Optional (some or none)  | `some(42)`, `none`                   |
| `!T`     | Result (ok or error)     | `ok(42)`, `error(0)`                |
| `channel<T>` | Channel for concurrency | `channel<i64>()`               |

### User-Defined Types

```forai
type Point { x: i64  y: i64 }

enum Color { Red Green Blue }
enum Shape { Circle(i64) Rect(i64, i64) }
```

## Functions

Every public function needs a `doc` block and a `test` block. The last expression in a function body is its return value. Explicit `return` is also supported.

```forai
doc add {
    summary: "Adds two integers together"
    params: { a: "first operand" b: "second operand" }
    returns: "the sum of a and b"
}
pub fn add(a: i64, b: i64) -> i64 {
    a + b
}
test add {
    assert add(2, 3) == 5
}
```

Private functions (without `pub`) do not require test blocks, but **still require doc blocks**:

```forai
doc helper {
    summary: "Doubles an integer"
    params: { x: "The value to double" }
    returns: "The doubled value"
}
fn helper(x: i64) -> i64 {
    x * 2
}
```

`main` is exempt from doc and test requirements. Functions without `-> Type` return void:

```forai
pub fn main() {
    print(add(10, 32))
}
```

## The Mandatory Contract

### Doc Blocks

Every function (both `pub fn` and `fn`), `type`, and `enum` needs a doc block. Sections:

| Section    | Required for                    | Value type             |
|------------|---------------------------------|------------------------|
| `summary`  | All items                       | `"text"`               |
| `params`   | Functions with parameters       | `{ name: "desc" }`    |
| `returns`  | Functions with non-unit return  | `"text"`               |
| `errors`   | Functions returning `!T`        | `{ name: "desc" }`    |
| `fields`   | Structs                         | `{ name: "desc" }`    |
| `variants` | Enums                           | `{ name: "desc" }`    |

```forai
doc safe_divide {
    summary: "Divides two integers safely"
    params: { a: "Dividend" b: "Divisor" }
    returns: "The quotient or an error"
    errors: { division_by_zero: "Returned when divisor is zero" }
}
pub fn safe_divide(a: i64, b: i64) -> !i64 {
    match b {
        0 -> error(0)
        _ -> ok(a / b)
    }
}
```

### Test Blocks

Every `pub fn` (except `main`) needs a corresponding `test` block:

```forai
test safe_divide {
    let ok_result = safe_divide(10, 2)
    assert ok_result? == 5
    let err_result = safe_divide(10, 0)
    let covered = "division_by_zero"
    assert 1 == 1
}
```

### Error Coverage

If a doc block has an `errors` section, each error key must appear as a string literal in the corresponding test block. In the example above, `"division_by_zero"` appears in the test to satisfy this.

### Comments

`//` comments are only allowed inside `test` and `doc` blocks. They are not allowed in function bodies or at the top level.

```forai
test add {
    // This comment is fine inside a test block
    assert add(2, 3) == 5
}
```

## Variables

```forai
let x = 10           // immutable binding
let mut counter = 0   // mutable binding
counter = counter + 1 // assignment (only for mut bindings)
```

## Control Flow

### Match (the only branching construct)

forai has no `if/else`. Use `match` for all branching:

```forai
// Integer matching
match value {
    0 -> handle_zero()
    1 -> handle_one()
    _ -> handle_other()
}

// Boolean matching (replaces if/else)
match x > 0 {
    true -> handle_positive()
    false -> handle_non_positive()
}

// String matching
match name {
    "alice" -> 1
    "bob" -> 2
    _ -> 0
}

// Result matching
match result {
    ok(v) -> v
    err(e) -> 0
}

// Optional matching
match opt {
    some(v) -> v
    none -> 0
}

// Enum matching
match color {
    Color::Red -> 1
    Color::Green -> 2
    Color::Blue -> 3
}

// Enum with payload
match shape {
    Shape::Circle(r) -> r
    Shape::Rect(w, h) -> w * h
}
```

Match arms use `->` (not `=>`). The wildcard `_` catches everything.

### Match: Important Rules

**Assignments and `break` must be wrapped in blocks.** A match arm's right-hand side must be an expression. Bare assignments and `break` are statements, so they need `{ }`:

```forai
// WRONG — will not parse:
match condition {
    true -> count = count + 1
    false -> break
}

// CORRECT — wrap in blocks:
match condition {
    true -> { count = count + 1 }
    false -> { break }
}
```

**All match arms must produce the same type.** When using match in a statement context where arms contain assignments (which produce unit), every arm must also produce unit. If some arms return a value and others do assignments, add a trailing `0` to the assignment arms to unify the types:

```forai
// WRONG — type mismatch (some arms produce str, others produce unit):
match kind {
    "header" -> { output = output + render_header(line) }
    "blank" -> 0
    _ -> { state = "para" }
}

// CORRECT — all arms produce i64:
match kind {
    "header" -> {
        output = output + render_header(line)
        0
    }
    "blank" -> 0
    _ -> {
        state = "para"
        0
    }
}
```

### Loops

```forai
// Infinite loop (use break to exit)
each {
    break
}

// Map over array (each-collect)
let arr = [1, 2, 3]
let doubled = each item in arr -> collect {
    item * 2
}
```

## Error Handling

### Result Type (`!T`)

```forai
pub fn divide(a: i64, b: i64) -> !i64 {
    match b {
        0 -> error(0)
        _ -> ok(a / b)
    }
}

// Match to extract
let result = divide(10, 2)
match result {
    ok(v) -> v
    err(e) -> 0
}

// ? operator to unwrap (returns error if err)
let value = divide(10, 2)?
```

### Optional Type (`?T`)

```forai
let x = some(42)     // create optional with value
let y = none          // create empty optional

// Match to extract
match x {
    some(v) -> v
    none -> 0
}

// ? operator to unwrap
let value = x?
```

## Structs

```forai
type Point { x: i64  y: i64 }

doc Point {
    summary: "A 2D point"
    fields: { x: "X coordinate" y: "Y coordinate" }
}

// Construction
let p = Point { x: 10, y: 20 }

// Field access
let x_val = p.x
```

## Enums

```forai
enum Color { Red Green Blue }

doc Color {
    summary: "A color"
    variants: { Red: "Red" Green: "Green" Blue: "Blue" }
}

// Unit variant
let c = Color::Red

// Payload variant
enum MyResult { Ok(i64) Err(i64) }
let r = MyResult::Ok(42)

// Matching
match r {
    MyResult::Ok(v) -> v
    MyResult::Err(e) -> e
}
```

## Arrays

```forai
let arr = [10, 20, 30]
let first = arr[0]        // index access
let length = arr.len()    // built-in method
```

## Tuples

```forai
let t = (10, 20, 30)
let first = t.0     // positional access
let second = t.1
```

## Maps

```forai
let m = {1: 100, 2: 200, 3: 300}
let val = m.get(2)        // returns 200
let length = m.len()      // returns 3
let exists = m.has(1)     // returns 1 (true) or 0 (false)

// String keys
let sm = {"name": "forai", "version": "0.1"}
```

## Generics

Function-level only. Type must be specified at the call site:

```forai
doc identity {
    summary: "Returns its argument unchanged"
    params: { x: "The value" }
    returns: "The same value"
}
pub fn identity<T>(x: T) -> T {
    return x
}

test identity {
    assert identity<i64>(42) == 42
    assert identity<bool>(true) == true
}
```

## UFCS (Uniform Function Call Syntax)

Any function can be called as a method. `x.f(y)` is equivalent to `f(x, y)`:

```forai
pub fn double(n: i64) -> i64 {
    n * 2
}

// Both are equivalent:
double(5)      // traditional call
5.double()     // UFCS call
```

## Strings

### Literals and Interpolation

```forai
let name = "world"
let greeting = "Hello, {name}!"   // string interpolation
let escaped = "Use \{ and \} for literal braces"
let combined = "Hello" + " " + "World"   // concatenation
```

### Built-in String Methods

These are available directly on any string expression:

```forai
let s = "hello world"

s.len()                    // 11 (length)
s.substring(0, 5)         // "hello" (aliases: .substr)
s.starts_with("hello")    // 1 (true) or 0 (false)
s.ends_with("world")      // 1 or 0
s.contains("lo")          // 1 or 0
s.trim()                  // removes leading/trailing whitespace
s.to_upper()              // "HELLO WORLD"
s.to_lower()              // "hello world"
s.split(" ")              // ["hello", "world"] (returns [str])
s.replace("world", "x")   // "hello x"
s.index_of("world")       // 6 (byte offset, or -1)
s.char_at(0)              // "h"
```

Array of strings:

```forai
let parts = ["a", "b", "c"]
parts.join("-")            // "a-b-c"
```

### String Equality

```forai
match "hello" == "hello" {
    true -> 1
    false -> 0
}
```

## Concurrency

### Channels

```forai
let ch = channel<i64>()

// Direct calls
send(ch, 42)
let val = receive(ch)

// UFCS style
ch.send(42)
let val = ch.receive()
```

### Go Blocks

Execute code concurrently (inline in WASM):

```forai
let ch = channel<i64>()
go {
    ch.send(99)
}
let val = ch.receive()
```

### Select

Wait on multiple channels:

```forai
select {
    msg from ch1 -> handle(msg)
    val from ch2 -> process(val)
    timeout(5000) -> default_value()
}
```

## Standard Library

Import with `use std.<module>`. Call functions as `module.function()`.

### std.str

String utilities (12 functions). All take a string as the first argument:

```forai
use std.str

str.len(s) -> i64                       // string length
str.contains(s, sub) -> i64             // 1 if contains, 0 if not
str.starts_with(s, prefix) -> i64       // 1 or 0
str.ends_with(s, suffix) -> i64         // 1 or 0
str.trim(s) -> str                      // strip whitespace
str.to_upper(s) -> str                  // uppercase
str.to_lower(s) -> str                  // lowercase
str.substr(s, start, end) -> str        // substring by index
str.replace(s, old, new) -> str         // replace occurrences
str.index_of(s, sub) -> i64            // find substring position
str.split(s, delim) -> [str]           // split into array
str.join(arr, sep) -> str              // join array with separator
```

### std.html

HTML generation (3 functions):

```forai
use std.html

html.escape(s) -> str                                   // escape HTML entities
html.element(tag, attrs, content) -> str                 // create HTML element
// attrs is {str:str} map, e.g. {"href": "https://example.com"}
html.template(tmpl, vars) -> str                         // template substitution
// vars is {str:str} map
```

### std.json

JSON encoding/decoding (4 functions):

```forai
use std.json

json.encode(s) -> str                   // JSON-encode a string
json.encode_map(m) -> str               // JSON-encode a {str:str} map
json.decode(s) -> !str                  // decode JSON string (returns Result)
json.decode_map(s) -> !{str:str}        // decode JSON to map (returns Result)
```

### std.http

HTTP server types and functions:

```forai
use std.http

// Types
type Request { method: str  path: str  body: str }
type Response { status: i64  body: str }

// Functions
http.listen(addr) -> i64                // start server, returns handle
http.accept(server) -> Request          // accept next request
http.respond(server, response)          // send response
http.ok(body) -> Response               // 200 response helper
http.not_found() -> Response            // 404 response helper
```

## Testing Features

### Approximate Equality

For floating-point comparisons, use `~=` instead of `==`:

```forai
test approx_add {
    assert approx_add(0.1, 0.2) ~= 0.3
}
```

### Property-Based Testing

Automatically run assertions with random inputs (100 iterations):

```forai
property add_commutative(a: i64, b: i64) {
    assert add(a, b) == add(b, a)
}

property add_identity(a: i64) {
    assert add(a, 0) == a
}
```

Property blocks count toward the test coverage requirement for their associated functions.

### Snapshot Testing

Capture an expression's result and compare against a saved snapshot:

```forai
snapshot compute_result {
    compute()
}
```

Run with `--update-snapshots` to save/update snapshot files in `__snapshots__/`.

## Module System

### Project Structure

```
my-project/
  src/
    main.fai     # entry point
    math.fai     # additional module
  dependencies.fai  # package dependencies (optional)
```

### Imports

```forai
// In src/main.fai
use math              // import sibling module

// Use qualified names
let result = math.add(1, 2)

// Standard library
use std.str
use std.html
use std.json
use std.http
```

Only `pub` items are accessible from other modules.

## Printing

```forai
print(42)              // print integer
print("hello")         // print string
```

## Operators

| Operator | Description          |
|----------|----------------------|
| `+`      | Add / string concat  |
| `-`      | Subtract             |
| `*`      | Multiply             |
| `/`      | Divide               |
| `%`      | Modulo               |
| `==`     | Equal                |
| `!=`     | Not equal            |
| `<`      | Less than            |
| `>`      | Greater than         |
| `<=`     | Less or equal        |
| `>=`     | Greater or equal     |
| `~=`     | Approximately equal  |
| `?`      | Unwrap result/optional |
| `!`      | (prefix) not         |

## Development Workflow: Test-Driven Development

forai enforces tests by design — the compiler rejects code without them. Lean into this. Write tests *as you write code*, not after. Run `forai build` after every meaningful change to catch errors immediately.

### The TDD cycle in forai

1. **Write the doc block and test block first.** Before implementing the function body, define what it should do and assert the expected behavior.
2. **Write the minimal function body** to make the test pass.
3. **Run `forai build`** (or `forai run`) to compile and execute tests. Fix any failures before moving on.
4. **Repeat** for each function. Never batch — build after every function.

### Example cycle

Start with the contract:

```forai
doc add {
    summary: "Adds two integers"
    params: { a: "First operand" b: "Second operand" }
    returns: "The sum"
}
pub fn add(a: i64, b: i64) -> i64 {
    0
}
test add {
    assert add(2, 3) == 5
}
```

Run `forai build src/main.fai` — the test fails. Now implement:

```forai
pub fn add(a: i64, b: i64) -> i64 {
    a + b
}
```

Run `forai build src/main.fai` again — the test passes. Move to the next function.

### Key habits

- **Build after every function.** Do not write three functions and then build. The compiler runs tests at compile time — use this.
- **Use `forai check`** for fast syntax/parse validation when you only want to check structure without running tests.
- **Use `forai dev`** during active development — it watches for file changes and rebuilds automatically.
- **Test error paths.** If a function returns `!T`, write test cases for both `ok` and `err` paths. The error coverage checker will enforce this if you have an `errors` doc section.
- **Keep tests meaningful.** `assert 1 == 1` satisfies the compiler but tests nothing. Assert actual behavior.

## CLI

```bash
forai build file.fai           # compile to WASM
forai run file.fai             # compile and execute
forai check file.fai           # parse/check only
forai new project-name         # scaffold new project
forai docs file.fai            # generate docs/api.md and docs/api.json
forai dev file.fai             # watch + rebuild + rerun
forai dev --web file.fai       # watch + rebuild + serve on :8080
forai fetch                    # fetch dependencies
forai clean                    # remove cached dependencies
```

## Keywords

```
fn  pub  let  mut  return  break  match  test  doc  type  enum  module  use  dep
go  each  in  select  timeout  channel  send  receive  assert  property  snapshot
true  false  ok  err  error  some  none
i32  i64  f32  f64  bool  str  byte  unit  never
```

## Complete Example

A full working program demonstrating the core pattern:

```forai
type Point { x: i64  y: i64 }
doc Point {
    summary: "A 2D point with integer coordinates"
    fields: { x: "The x coordinate" y: "The y coordinate" }
}

doc distance_squared {
    summary: "Computes the squared distance between two points"
    params: { a: "First point" b: "Second point" }
    returns: "The squared Euclidean distance"
}
pub fn distance_squared(a: Point, b: Point) -> i64 {
    let dx = a.x - b.x
    let dy = a.y - b.y
    dx * dx + dy * dy
}

test distance_squared {
    let p1 = Point { x: 0, y: 0 }
    let p2 = Point { x: 3, y: 4 }
    assert distance_squared(p1, p2) == 25
}

doc safe_divide {
    summary: "Divides two integers, returning an error for zero divisor"
    params: { a: "Dividend" b: "Divisor" }
    returns: "The quotient"
    errors: { division_by_zero: "When divisor is zero" }
}
pub fn safe_divide(a: i64, b: i64) -> !i64 {
    match b {
        0 -> error(0)
        _ -> ok(a / b)
    }
}

test safe_divide {
    assert safe_divide(10, 2)? == 5
    let err_result = safe_divide(10, 0)
    let covered = "division_by_zero"
    match err_result {
        ok(v) -> assert 1 == 0
        err(e) -> assert e == 0
    }
}

doc counter {
    summary: "Increments a mutable counter to a target"
    params: { target: "The target value" }
    returns: "The target value reached by incrementing"
}
pub fn counter(target: i64) -> i64 {
    let mut count = 0
    let mut i = 0
    each {
        match i == target {
            true -> break
            false -> {
                count = count + 1
                i = i + 1
            }
        }
    }
    count
}

test counter {
    assert counter(5) == 5
    assert counter(0) == 0
}

pub fn main() {
    let p1 = Point { x: 0, y: 0 }
    let p2 = Point { x: 3, y: 4 }
    print(distance_squared(p1, p2))
}
```
