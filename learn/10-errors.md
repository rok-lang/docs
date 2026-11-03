# 10. Error Handling

Things that can fail, like a missing file or bad input, are everywhere. Rok has a single error model. A function that can throw writes `throws` in its signature, and the caller marks that call with `try`. As a result, every point where an error can flow is visible in the code.

## `throws`, `throw`, and `try`

Write `throws` after the parameter list, before `->`. In the body, `throw` throws an error value. For most code a message is all you need: `throw "msg"` (any `str`, f-strings too) throws the standard library's `ValueError::new("msg")` (unless the `throws` list or a `catch e: str` takes a `str` as it is), so you don't have to define an error type.

You must put `try` on every call to a function that throws. If the call fails, the error goes to the nearest `catch`; if there is none, it propagates out of the enclosing function, which must then be `throws` too.

```rok
fn parse_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw f"age {n} is negative"; }
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

An explicit `fn main()` with no return type throws the same way, without writing `throws`:

```rok build
fn main() {
    let n: i32 = try "42".parse();
    if n > 40 { throw f"{n} is too big"; }
    println("unreached");
}
```

```text
error: `main` returned an uncaught error: 42 is too big
```

A bare `return;` in the top-level statements ends the script with exit code 0. With `let .. else` it stops a script whose input is missing:

```rok
let first = ["a", "b"].first() else {
    return;
};
println(first);
```

```output
a
```

`return v;` there is an error (E0018): use `process::exit(v)` for another exit code.

## `try?` and `??`: a default instead of an error

Often you just want a fallback value. `try? e` discards the error and yields `none`, and `??` replaces `none` with a default. `??` ends the operand of `try?` (and of `try!`), so `try? f() ?? d` is `(try? f()) ?? d`: `d` on an error. `try` still covers everything to its right, `??` included.

```rok
fn parse_port(s: str) -> i32 {
    try? s.parse::<i32>() ?? 8080
}

println(f"{parse_port("3000")} {parse_port("abc")}");
```

```output
3000 8080
```

If `f()` itself returns `T | none`, `try? f()` is the two-level `(T | none)?`: the outer `none` means an error was thrown, the inner one is `f`'s own `none`, and `??` strips only the outer level. Write `try? f() ?? none` when both should mean the same thing.

To ignore an error on purpose, write `let _ = try? f();`. `try! e` turns an error into a panic; use it only when failing there would be a bug.

| Form     | On error                              |
|----------|---------------------------------------|
| `try e`  | Nearest `catch`, or propagate         |
| `try? e` | Discard the error and yield `none`    |
| `try! e` | Panic (treated as a program bug)      |

```rok
fn save(n: i32) throws {
    if n < 0 { throw "negative"; }
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
    if n < 0 { throw f"age {n} is negative"; }
    n
}

for s in ["30", "x", "-1"] {
    let age = try read_age(s)
    catch e: ValueError { println(f"  {e.message()}"); -1 }
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
    if n % 2 != 0 { throw f"{n} is odd"; }
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

A `return`, `break` or `continue` that leaves the `finally` block itself is an error: it would drop an error that is on its way out. Put the jump after the `try` expression.

## `try for`: loops whose steps can fail

Some iterators can fail at each step: a generator that throws, or bytes read from a stream. Such an iterator is a `TryIterator`, and a `for` over it is written `try for`. The `try` covers the iterable and each step, not the body: a call in the body that can throw needs its own `try`.

```rok
yield fn numbers(words: List<str>) throws (ParseError) -> i32 {
    for w in words { yield try w.parse::<i32>(); }
}

fn total(words: List<str>) throws -> i32 {
    let sum = 0;
    try for n in numbers(words) {   // a failed step propagates
        sum += n;
    }
    sum
}

println(f"{try! total(["1", "2", "3"])} {try? total(["1", "x"]) ?? -1}");
```

```output
6 -1
```

A `TryIterator` held in a variable is stepped in place: a loop that leaves early with `break` leaves the variable where the loop stopped, and a second loop over it goes on from there. `try! for` panics on a failed step, and `try? for` ends the loop there. To go on past errors, loop over `it.results()`, which gives each step as a `Result`.

## Your own error types

When callers need to tell failures apart, define an error type: a struct or enum with `Debug` and `Display` that implements the `Error` trait. `#[derive(Debug, Error)]` gives two of them, and `#[display("..")]` writes `Display` from an f-string over the fields; a hand-written `impl Display` has the shape from [chapter 6](06-structs-enums-traits.html#operator-traits). A signature can then list exactly what it throws, `throws (E)`, or several kinds as a sum, `throws (A | B)`. The compiler checks that every listed type is handled or passed on.

```rok
#[derive(Debug, Error)]
#[display("no key {self.key}")]
struct NotFound { key: str; }

#[derive(Debug, Error)]
#[display("{self.reason}")]
struct Invalid { reason: str; }

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
#[derive(Debug, Error)]
#[display("no key {self.key}")]
struct NotFound { key: str; }

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
    let out = [];
    for x in xs { out.push(try f(x)); }
    out
}

fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw f"{n} is odd"; }
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
        throw "query failed";
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

`using` is an expression, so it can also produce a value, as in `let text = using f = try fs::open(path) { try f.read_to_end() };`.

When the resource should live to the end of the enclosing block, the statement form `using x = e;` saves the extra block. It is the block form over the statements after it, so release happens at the block's end, in reverse order, on every way out:

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn report() {
    using a = Conn { name: "db" };
    using b = Conn { name: "cache" };
    println(f"using {a.name} and {b.name}");
}

report();
```

```output
using db and cache
close cache
close db
```

A value whose type implements `Drop` has one owner. `let b = a` moves it, and using `a` afterwards is an error; `a.clone()` makes a second value with its own owner. Without `using`, the compiler releases the value where its owner leaves scope, on every way out:

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn short_job() {
    let c = Conn { name: "tmp" };
    let d = c;               // moves: `c` is gone
    println("working");
}                            // the compiler releases `d` here

short_job();
```

```output
working
close tmp
```

A type whose close can fail, such as a `File`, is not closed for you, since its error would be lost: close it with `try f.close()`, or bind it with `using`. Leaving its scope with it still open is a compile error. For the same reason `try?` cannot drop an error that may hold such a value, like the `SendError<TcpStream>` of a failed `tx.send(conn)`, which gives the stream back: catch it with `try .. catch e { .. }`, take the value out and close it.

## Errors and panics

An error from `throws` is "a failure that can happen". By contrast, an out-of-range index, division by zero, a failed `try!`, or a call to `panic(..)` is a **panic**, which is treated as a program bug. A panic cannot be caught with `catch`; it writes a message to stderr and exits with code 101. Cleanup in `finally` and `using` does not run either.

```rok build
let xs = [1, 2, 3];
let i = 7;
println(f"{xs[i]}");
```

```text
thread 'main' panicked at main.rok:3:12:
index 7 out of bounds for length 3
```

Use `Result<T, E>` only when you need to hold an error as a value (to collect several results or handle them later). The rule is: "if you throw it, `throws`; if you keep it, `Result`".

```rok
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw f"{n} is odd"; }
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

- A function that can throw declares `throws` (before `->`) and throws with `throw`; `throw "msg"` throws a `ValueError`, which covers most needs. `fn main()` throws without declaring it.
- Calls are marked with `try`. An uncaught error propagates to the enclosing function; at the top of a script it ends the program with exit code 1.
- `try? e ?? default` gives a fallback; `let _ = try? f();` ignores an error; `try!` turns it into a panic. A bare `return;` ends a script.
- `catch` is an expression that selects by pattern, and `finally` always runs.
- Define your own error types (`Display` + `Error`) when callers must tell failures apart, and list them with `throws (A | B)`.
- `rethrows` throws only when the function it received throws.
- Resources bound with `using` are cleaned up however the block is exited; `using x = e;` covers the rest of its block. Other `Drop` values are released where their owner leaves scope; a `File` must be closed or bound with `using`.
- `try for` loops over an iterator whose steps can fail; the body marks its own `try`.
- A panic is an uncatchable bug signal. Use `Result` only when you need to hold an error as a value.

## Further reading

- Spec: [Error control flow](../spec/refs-lang/003-expressions/314-error-control-flow.html), [`throws` signatures](../spec/refs-lang/004-functions/414-throws.html), [`using` expressions](../spec/refs-lang/003-expressions/317-using-expression.html), [Panics and fatal errors](../spec/refs-lang/003-expressions/314-error-control-flow.html#3148-panics-and-fatal-errors)
- Standard library spec: [error](../spec/refs-standard-libraries/200-core/214-error.html)
- API: [`Error`](../std/core/error.html#trait.Error), [`ValueError`](../std/core/error.html#struct.ValueError), [`Result`](../std/core/error.html#enum.Result), [`Drop`](../std/core/marker.html#trait.Drop)
- Next chapter: [11. Modules and Packages](11-modules-packages.html)
