# 10. Error Handling

Things that can fail, like a missing file or bad input, are everywhere. rok has a single error model. A function that can throw writes `throws` in its signature, and the caller marks that call with `try`. As a result, every point where an error can flow is visible in the code.

## `throws`, `throw`, and `try`

Write `throws` after the parameter list, before `->`. In the body, `throw` throws an error value. For most code the standard library's `ValueError { msg }` is all you need; you don't have to define an error type.

You must put `try` on every call to a function that throws. If the call fails, the error goes to the nearest `catch`; if there is none, it propagates out of the enclosing function, which must then be `throws` too.

```rok
fn parse_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw ValueError { msg: f"age {n} is negative" }; }
    n
}

fn total_age(xs: List<str>) throws -> i32 {
    let sum = 0;
    for x in xs {
        sum += try parse_age(x);     // an error leaves total_age here
    }
    sum
}

println(f"{try total_age(["30", "12"])}");
let t = try total_age(["30", "-1"]) catch e { println(f"error: {e}"); 0 };
println(f"{t}");
```

```output
42
error: age -1 is negative
0
```

`catch e { .. }` handles the error on the spot, and its block's value is used instead. `try` covers the entire expression to its right: `try a(b())` marks both calls together, and `try { .. }` covers every call inside the block.

A plain `throws` means the function can throw any error type (short for `throws (any Error)`); `s.parse()` throws a `ParseError`, and `parse_age` passes it on unchanged. Listing the exact error types is covered [below](#your-own-error-types).

### Errors at the top of a script

A program's top-level statements form an implicit `main` that `throws`. So in a script, `try` alone is enough: an error nobody catches ends the program with a message on stderr and exit code 1.

```rok build
let n: i32 = try "abc".parse();
println(f"{n}");
```

```text
$ rok run
error: `main` returned an uncaught error: cannot parse "abc": invalid digit
  --> src/main.rok:1:18
$ echo $?
1
```

An explicit `fn main()` that is not `throws` must handle every error itself; write `fn main() throws { .. }` to let them out.

## `try?` and `??`: a default instead of an error

Often you just want a fallback value. `try? e` discards the error and yields `none`, and `??` replaces `none` with a default. Close the `try?` with parentheses before `??`.

```rok
fn parse_port(s: str) -> i32 {
    (try? s.parse::<i32>()) ?? 8080
}

println(f"{parse_port("3000")} {parse_port("abc")}");
```

```output
3000 8080
```

To ignore an error on purpose, write `let _ = try? f();`. `try! e` turns an error into a panic; use it only when failing there would be a bug.

| Form     | On error                              |
|----------|---------------------------------------|
| `try e`  | Nearest `catch`, or propagate         |
| `try? e` | Discard the error and yield `none`    |
| `try! e` | Panic (treated as a program bug)      |

```rok
fn save(n: i32) throws {
    if n < 0 { throw ValueError { msg: "negative" }; }
    println(f"saved {n}");
}

let _ = try? save(1);
let _ = try? save(-1);         // the error is dropped
let n: i32 = try! "42".parse();
println(f"{n}");
```

```output
saved 1
42
```

## `catch`: selecting by type

`catch` patterns are tested from top to bottom, and the first matching block runs. `e: T` selects by type, and inside that branch `e` has that type, so you can read its fields. A final `catch e` takes everything else.

```rok
fn read_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw ValueError { msg: f"age {n} is negative" }; }
    n
}

for s in ["30", "x", "-1"] {
    let age = try read_age(s)
    catch e: ValueError { println(f"  {e.msg}"); -1 }
    catch e { println(f"  {e}"); -1 };
    println(f"{s} -> {age}");
}
```

```output
30 -> 30
  cannot parse "x": invalid digit
x -> -1
  age -1 is negative
-1 -> -1
```

## Block form and `finally`

To wrap several steps at once, use the `try { .. } catch .. { .. }` form. A `finally` block runs last no matter how control leaves: normal completion, an error, `return`, and so on.

```rok
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

for input in [8, 6] {
    let r = try {
        let a = try half(input);
        let b = try half(a);
        a + b
    } catch e {
        println(f"failed: {e}");
        0
    } finally {
        println(f"done with {input}");
    };
    println(f"result {r}");
}
```

```output
done with 8
result 6
failed: 3 is odd
done with 6
result 0
```

## Your own error types

When callers need to tell failures apart, define an error type: a struct or enum with `Debug` and `Display` that implements the `Error` trait (the `Display` shape is the one from [chapter 6](06-structs-enums-traits.html#operator-traits)). A signature can then list exactly what it throws, `throws (E)`, or several kinds as a sum, `throws (A | B)`. The compiler checks that every listed type is handled or passed on.

```rok
#[derive(Debug)]
struct NotFound { key: str; }
impl Display for NotFound {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(f"no key {self.key}"); }
}
impl Error for NotFound {}

#[derive(Debug)]
struct Invalid { reason: str; }
impl Display for Invalid {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.reason); }
}
impl Error for Invalid {}

fn lookup(key: str) throws (NotFound | Invalid) -> i32 {
    if key == "" { throw Invalid { reason: "empty key" }; }
    if key != "a" { throw NotFound { key: key }; }
    1
}

fn describe(key: str) -> str {
    try f"found {lookup(key)}"
    catch e: NotFound { f"missing: {e.key}" }
    catch e: Invalid { f"invalid: {e}" }
}

fn main() {
    for k in ["a", "b", ""] {
        println(describe(k));
    }
}
```

```output
found 1
missing: b
invalid: empty key
```

The error kinds of `lookup` are closed as `NotFound | Invalid`, so `describe`, which covers both types, becomes a function that no longer throws. The compiler checks this. Use a plain `throws` when the set of error kinds is hard to close, as in a public API; the receiver catches everything with `catch e` or narrows first, as in `catch e: ValueError` ([4.14.3](../spec/refs-lang/004-functions/414-throws.html#4143-untyped-throws)).

Any pattern can follow `catch`, so a struct pattern can pull out fields directly.

```rok
#[derive(Debug)]
struct NotFound { key: str; }
impl Display for NotFound {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(f"no key {self.key}"); }
}
impl Error for NotFound {}

fn lookup(key: str) throws (NotFound) -> i32 {
    if key != "a" { throw NotFound { key: key }; }
    1
}

for key in ["a", "b"] {
    let n = try lookup(key)
    catch NotFound { key } { println(f"missing: {key}"); 0 };
    println(f"{n}");
}
```

```output
1
missing: b
0
```

## `rethrows`: throwing only as much as the argument does

A higher-order function that takes a function as an argument can be declared `rethrows`. If the function passed in does not throw, the call needs no `try`; you add `try` only when passing a throwing function.

```rok
fn apply_all(xs: List<i32>, f: fn(i32) throws -> i32) rethrows -> List<i32> {
    let out: List<i32> = [];
    for x in xs { out.push(try f(x)); }
    out
}

fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

let doubled = apply_all([1, 2, 3], |x| x * 2);
println(f"{doubled:?}");
let halves = try apply_all([2, 4], half) catch _ { [] };
println(f"{halves:?}");
let bad = try apply_all([2, 3], half) catch e { println(f"{e}"); [] };
println(f"{bad:?}");
```

```output
[2, 4, 6]
[1, 2]
3 is odd
[]
```

## `using`: cleaning up resources

The GC reclaims memory, but resources such as files, sockets, and locks must be closed at a definite point. `using x = acquire_expr { .. }` calls `x`'s `Drop::drop` however the block is exited (normally, via `return`, or by error propagation). Multiple resources are separated by commas and cleaned up in reverse order.

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn open(name: str) -> Conn {
    println(f"open {name}");
    Conn { name: name }
}

fn work() throws -> i32 {
    using a = open("db"), b = open("cache") {
        println("working");
        throw ValueError { msg: "query failed" };
    }
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
open db
open cache
working
close cache
close db
work failed: query failed
```

`using` is an expression, so it can also produce a value, as in `let text = using f = try fs::open(path) { try f.read_to_end() };`. Remember that ordinary `let` bindings are not `drop`ped when they go out of scope. When you need deterministic cleanup, use `using`.

## Errors and panics

An error from `throws` is "a failure that can happen". By contrast, an out-of-range index, division by zero, a failed `try!`, or a call to `panic(..)` is a **panic**, which is treated as a program bug. A panic cannot be caught with `catch`; it writes a message to stderr and exits with code 101. Cleanup in `finally` and `using` does not run either.

```rok build
let xs = [1, 2, 3];
let i = 7;
println(f"{xs[i]}");
```

```text
thread 'main' panicked at main.rok:3:12:
List index out of bounds
```

Use `Result<T, E>` only when you need to hold an error as a value (to collect several results or handle them later). The rule is: "if you throw it, `throws`; if you keep it, `Result`".

```rok
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

for n in [4, 5] {
    let r = Result::catching(|| try half(n));
    println(f"{n}: ok={r.is_ok()} value={r.unwrap_or(-1)}");
}
```

```output
4: ok=true value=2
5: ok=false value=-1
```

## Summary

- A function that can throw declares `throws` (before `->`) and throws with `throw`; `ValueError { msg }` covers most needs.
- Calls are marked with `try`. An uncaught error propagates to the enclosing function; at the top of a script it ends the program with exit code 1.
- `(try? e) ?? default` gives a fallback; `let _ = try? f();` ignores an error; `try!` turns it into a panic.
- `catch` is an expression that selects by pattern, and `finally` always runs.
- Define your own error types (`Display` + `Error`) when callers must tell failures apart, and list them with `throws (A | B)`.
- `rethrows` throws only when the function it received throws.
- Resources bound with `using` are cleaned up however the block is exited.
- A panic is an uncatchable bug signal. Use `Result` only when you need to hold an error as a value.

## Further reading

- Next chapter: [11. Modules and Packages](11-modules-packages.html)
- Spec: [Error control flow](../spec/refs-lang/003-expressions/314-error-control-flow.html), [`throws` signatures](../spec/refs-lang/004-functions/414-throws.html), [`using` expressions](../spec/refs-lang/003-expressions/317-using-expression.html), [Panics and fatal errors](../spec/refs-lang/003-expressions/314-error-control-flow.html#3148-panics-and-fatal-errors)
- Standard library spec: [error](../spec/refs-standard-libraries/200-core/214-error.html)
- API: [`Error`](../std/core/error.html#trait.Error), [`ValueError`](../std/core/error.html#struct.ValueError), [`Result`](../std/core/error.html#enum.Result), [`Drop`](../std/core/marker.html#trait.Drop)
