# Rok for Rustaceans

A map for Rust programmers: what carries over, what to unlearn, and where Cargo's tools live. Much of Rok's surface looks like Rust, but its memory model does not: memory is garbage collected, so there are no lifetimes and no borrow checker. Every block below is a complete program; run it with `rok run file.rok`. The chapters of this guide explain each part in full.

## What carries over

- `fn`, `struct`, `enum`, `trait`, `impl Trait for Type`, generics with bounds and `where` clauses, associated types and const generics.
- `match` with guards, `if let`, `while let`, `let else`, and patterns that destructure tuples, structs and enum variants.
- `#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]`, `#[repr(C)]`, `#[cfg(..)]`, `#[test]` and `#[inline]`.
- `mod`, `use a::b::{c, d}`, `pub`, `pub use` re-exports, and a package-relative path for `crate::` (`pkg::`).
- Integer types `i8` .. `i64`, `u8` .. `u64`, `isize`, `usize`, `f32`, `f64`; literal suffixes (`7u8`, `2.5f32`), `_` separators, `as` casts, `checked_add`, `wrapping_add`, `saturating_add`.
- Operator precedence, ranges `a..b` and `a..=b`, `/` and `%` truncating toward zero, and overflow that panics.
- Iterator chains (`map`, `filter`, `fold`, `zip`, `enumerate`, `windows`), closures `|x| x + 1`, `async fn` and `await`, `unsafe` blocks and `extern "C"`.
- A package manager that builds, runs, tests, formats, lints and documents: `rok` plays Cargo's part.

## The first things to unlearn

- There are no lifetimes, no `&` or `&mut`, and no borrow checker. The GC frees memory; a value lives as long as something refers to it.
- There is no `mut`. Every variable can be reassigned, and a `let` of a name you already have declares a new one, as shadowing does in Rust.
- Structs, enums and tuples are values that copy on assignment, with nothing to opt into. `List`, `Dict` and `Set` are handles to one shared object. `ref` lets a function change its caller's struct.
- Errors are not values you return. A function that can fail says `throws`, and each call to it says `try`. There is no `?` for errors.
- There is no `Option`. A value that may be missing has the type `T | none`, and `T?` is short for it.
- An unsuffixed integer literal takes its type from its uses in the function, as in Rust, and is `i64` when nothing fixes it. Numbers never convert implicitly, not even `i32` to `i64`.
- Only a type with `Drop` (a file, a socket, a channel sender) has a single owner and moves. A file, whose close can fail, is closed explicitly, with `using` or `try f.close()`.
- A statement ends with `;`, as in Rust, but struct fields do too: `struct P { x: i32; y: i32; }`.

```rok
let x = 5;
x = x + 1;                 // no `mut` needed
let x = x * 2;             // a new `x`, as shadowing in Rust
println(x);

let words = ["a", "bb"];
let w = words;             // the same list, not a move
w.push("ccc");
println(f"{words:?} {words.len()}");
```

```output
12
["a", "bb", "ccc"] 3
```

Writing `mut` anyway stops the build with a help that drops it:

```text check=checks/rust-let-mut.rok
error[E0012]: rok has no `let mut`: variables are mutable by default
  --> main.rok:1:5
   |
 1 | let mut total = 0;
   |     ^^^
   |
   = help: drop `mut`: `let total = ..;`
```

The other Rust spellings stop the build the same way, and each help writes the Rok form: `&x` and `&mut x`, `&T` and `&mut self`, lifetimes, `impl Trait`, `Box<dyn Trait>`, `Fn(i32) -> i32`, `move ||`, `println!`, `vec![..]`, `format!`, `macro_rules!`, `pub(crate)`, `crate::` and `super::`, `Vec`, `String` and `HashMap`, `Ok(x)` and `Err(e)` in a `throws` function, and `.unwrap()` on a call that throws.

## Memory: a GC, values and handles

A struct is a value: assigning it, passing it or reading it out of a list copies it, and the original stays usable. Only a type with `Drop` moves (next section), so there is nothing to clone before reuse. A function that should change its caller's struct takes `ref`, and the call site writes `ref` too, so the reader sees what may change. A method that changes its receiver takes `ref self`, the counterpart of `&mut self`; the call site needs no marker. `self` alone is a copy, like Rust's `self` on a `Copy` type.

```rok
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Point {
    fn moved(self, dx: i32) -> Point { Point { x: self.x + dx, y: self.y } }
    fn shift(ref self, dx: i32) { self.x += dx; }
}

fn reset(ref p: Point) { p.x = 0; p.y = 0; }

let a = Point { x: 1, y: 2 };
let b = a;                 // a copy; `a` stays usable
b.shift(10);
println(f"{a:?} {b:?} {a.moved(5):?}");
reset(ref b);
println(f"{b:?}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for ref p in ps { p.x *= 10; }      // `for p in ps.iter_mut()`
println(f"{ps:?}");
```

```output
Point { x: 1, y: 2 } Point { x: 11, y: 2 } Point { x: 6, y: 2 }
Point { x: 0, y: 0 }
[Point { x: 10, y: 0 }, Point { x: 20, y: 0 }]
```

Where Rust would make you choose between `&` and `&mut`, Rok makes the mistake an error instead: setting a field on a copy that nothing reads afterwards (`fn inc(self) { self.n += 1; }`, or `for p in ps { p.x = 0; }`) does not compile, and the help names `ref self` or `for ref` ([chapter 6](06-structs-enums-traits.html#changing-the-callers-value-ref)).

A handle shared with a lock or another thread is checked too. What a `Mutex` guard's `g.get()` returns is a view of the protected object, so it cannot leave the `using` block that holds the lock ([E0106](../errors/E0106.html)); take `g.get_clone()` instead. A stream view such as `r.lines()` reads through `r` and stays in `r`'s scope ([E0140](../errors/E0140.html)); return the owning form `r.into_lines()`. These are the few places where Rok tracks something like a borrow, and each error says what to write.

## Resources: `using` instead of `Drop` at scope end

The GC frees memory but never closes a file. A type that implements `Drop` has one owner, as every Rust value does: `let b = a` moves it, and `a` is gone. `using x = e { .. }` closes `x` however the block is left, and the statement form `using x = e;` covers the rest of the enclosing block, in reverse order, as Rust drops locals. A `Drop` value without `using` is released where its owner leaves scope.

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn work() throws -> i32 {
    using a = Conn { name: "db" };
    using b = Conn { name: "cache" };
    println(f"using {a.name} and {b.name}");
    throw "query failed";
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
using db and cache
close cache
close db
work failed: query failed
```

Closing a file can fail (a write buffered until close may not reach the disk), and Rust's `Drop` swallows that error. Rok does not close such a type for you: write `try f.close()`, or bind it with `using`, which throws the close error. Leaving the scope with a `File` still open is a compile error ([E0135](../errors/E0135.html)).

```rok
use rok::std::{env, fs, process};

let path = env::temp_dir() / f"rustrok_{process::id()}.txt";
using f = try fs::File::create(path) {
    try f.write_all("hello\n".as_bytes());
}
println(try fs::read_to_str(path));
try fs::remove_file(path);
```

```output
hello

```

## Errors: `throws` and `try` instead of `Result` and `?`

`fn f() -> Result<T, E>` is `fn f() throws -> T`, and `f()?` is `try f()`. `try` marks every call that can fail, so a reader sees each point where the function may leave early. `throw "message"` throws a `ValueError`; your own error types implement `Display` and `Error` and are listed as `throws (A | B)`.

```rok
fn parse_port(s: str) throws -> i32 {
    let n = try s.parse::<i32>();
    if n < 1 || n > 65535 { throw f"port {n} out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });             // match on Err(_)
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);                       // .ok().unwrap_or(-1)
println(try! parse_port("443"));                           // .unwrap()
```

```output
8080
error: port 99999 out of range
0
-1
443
```

`catch e: IoError { .. }` selects by type, and `finally` always runs. `Result<T, E>` still exists for keeping an error as a value, to collect several outcomes: `Result::catching(|| try f())`. `panic(..)`, an out-of-range index and a failed `try!` are panics, which `catch` does not catch, as in Rust.

## `T | none` and sum types instead of `Option`

`Option<T>` is `T | none`, written `T?` for short. `Some(x)` is just `x`, and `None` is `none`. `??` is `unwrap_or`, `?.` is `map` over a field or a method, and postfix `?` returns `none` from a function that returns `T?`, as Rust's `?` does on an `Option`.

`A | B` is a sum of any types, with no enum to declare: a `match` arm or `is` picks the type. An `enum` gives each case a name and carries data as Rust's does.

```rok
struct User { name: str; nick: str?; }

fn nick_len(u: User?) -> usize? {
    let user = u?;
    user.nick?.len()
}

fn show(v: i32 | str | none) -> str {
    match v {
        n: i32 if n < 0 => "negative",
        n: i32 => f"int {n}",
        s: str => f"str {s}",
        none => "nothing",
    }
}

let u = User { name: "ada", nick: "bob" };
println(f"{nick_len(u):?} {nick_len(none):?}");
println(u.nick ?? "-");
if let n: str = u.nick { println(n.to_uppercase()); }
println(f"{show(-1)}, {show(7)}, {show("x")}, {show(none)}");

enum Shape { Circle(f64), Rect { w: f64; h: f64; }, Empty }
let s = Shape::Rect { w: 2.0, h: 3.0 };
let area = match s {
    Circle(r) => 3.0 * r * r,
    Rect { w, h } => w * h,
    Empty => 0.0,
};
println(area);
```

```output
3 none
bob
BOB
negative, int 7, str x, nothing
6
```

The variants of the enum a `match` is on are written bare (`Circle(r)`, not `Shape::Circle(r)`). `if let x: T = e` takes the `T` out of a sum, where Rust writes `if let Some(x) = e`.

## Traits, generics and `impl`

Traits, `impl Trait for Type`, generic bounds, `where` clauses, default methods and associated types work as in Rust, and so does the orphan rule: an `impl` needs the trait or the type to come from your package ([E0076](../errors/E0076.html)). The differences:

- A trait is not a type. `dyn Trait` is `any Trait`, with no `Box`: `List<any Shape>` holds values of any type that implements `Shape`. `impl Trait` is `some Trait`, in argument and return position: `fn area(s: some Shape)` is a generic parameter with no name, `fn evens() -> some Iterator<Item = i32>` hides the type it returns.
- A parameter with no type is generic: `fn loud(x) -> str { x.to_uppercase() }` accepts any type with the methods the body calls.
- `Fn`, `FnMut` and `FnOnce` are one type, `fn(i32) -> i32`, which holds a function or a closure. `fn() + Send` is one that may move to another thread.
- A closure captures by copying values. `[ref x]` captures the variable itself, so the closure can change it, and `[move x]` hands the value over: the code that made the closure cannot use `x` afterwards.
- `#[derive(..)]` writes the `impl`s as Rust's does. `Copy` is only a marker, since every struct already copies on `let y = x;`.

```rok
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

fn largest<T: Shape>(xs: List<T>) -> f64 {
    xs.map(|s| s.area()).fold(0.0, |a, b| a.max(b))
}

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }
println(largest([Square { side: 1.0 }, Square { side: 3.0 }]));

fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
println(apply(adder(5), 10));

let count = 0;
let snapshot = || count + 1;
let inc = [ref count] || { count += 1; };
inc();
inc();
println(f"{snapshot()} {count}");
```

```output
area 4.0
area 3.1
9
15
1 2
```

## Numbers

An unsuffixed integer literal takes its type from where it goes (`let b: u8 = 200;`, an argument, an operand of another type) or from its later uses, and is `i64` when nothing gives it one. A float literal is `f64` the same way. Lengths and indexes are `usize`, so `total / xs.len()` with an `i32` `total` needs `as`. `int` is an integer of any size.

Nothing converts implicitly: `i32 + i64` is an error whose help writes the `as`. `as` truncates, `.into()` is allowed only where nothing is lost, and `.try_into::<T>()` throws when the value does not fit, so `try_from` needs a `try`.

```rok
let a = 7;                    // i64: nothing else gives it a type
let b: u8 = 200;
let n = 3i32;
println(f"{a + 1} {(n as i64) + a} {b as i32 + 100} {300 as u8}");
let w: i64 = n.into();
let small: u8 = try 255i32.try_into::<u8>();
println(f"{w} {small}");
println(try? 256i32.try_into::<u8>() ?? 0);
println(f"{-7 / 2} {-7 % 2} {-7 %% 2} {7.pow(2)}");
let big: int = 2;
println(big.pow(100));
```

```output
8 10 300 44
3 255
0
-3 -1 1 49
1267650600228229401496703205376
```

`%%` is the floored remainder, Rust's `rem_euclid` for a positive divisor. `x.pow(n)` is a method, as in Rust.

## Operators

The arithmetic, bitwise and comparison operators and their precedence are Rust's, and the operator traits are `Add`, `Mul`, `Neg`, `Index` and so on. Rok adds a few:

- Chained comparisons: `0 <= i < n`.
- `in` and `!in` for membership: `x in xs`, `k !in dict`.
- Element-wise "dot" operators, `.+`, `.*`, `.>` and others, which apply an operator to each element of a list ([chapter 8](08-collections-iterators.html#advanced-broadcast-operators)).
- `??` for a default and `?.` for chaining through `none`.
- `/` on a `Path` joins paths: `dir / "file.txt"`.

```rok
let a = [1.0, 2.0, 3.0];
let b = [10.0, 20.0, 30.0];
println(f"{a .+ b:?} {a .* 2.0:?}");
let i = 3;
println(0 <= i < 5 && i !in [1, 2]);
```

```output
[11.0, 22.0, 33.0] [2.0, 4.0, 6.0]
true
```

## Modules and visibility

`mod name;` loads `name.rok` beside the file, and `name/` holds its submodules, as Rust 2018 does with no `mod.rs`. A path from the package root starts with `pkg::` where Rust writes `crate::`, and `sup::` is `super::`. An item is private to its module; `pub(sub)` opens it to submodules, `pub(pkg)` to the package (`pub(crate)`), and `pub` to everyone. A few `pub` items of the standard library are `#[internal]`: only the toolchain's own packages may use them.

```rok
mod geo {
    pub struct Point { pub x: f64; pub y: f64; }
    pub fn dist_sq(a: Point, b: Point) -> f64 { sq(a.x - b.x) + sq(a.y - b.y) }
    fn sq(v: f64) -> f64 { v * v }
    pub mod unit {
        pub fn origin() -> sup::Point { sup::Point { x: 0.0, y: 0.0 } }
    }
}

use geo::{Point, unit};
println(geo::dist_sq(unit::origin(), Point { x: 3.0, y: 4.0 }));
```

```output
25
```

`println`, `List`, `Dict`, `Mutex`, `channel`, `task` and `par` are in the prelude; the modules of `rok::std` that talk to the operating system (`fs`, `env`, `process`, `thread`, `net`) are imported with `use rok::std::fs;`.

## Attributes and macros

Attributes are written as in Rust. A program that would reach for `macro_rules!` or a procedural macro has one mechanism instead: a meta definition, a template of Rok code with `@` tokens. `#[impl_derive(T)]` defines a derive, `#[impl_attribute(name)]` an attribute, and `#[impl_macro(name)]` a call `name!(..)` ([6.8](../spec/refs-lang/006-meta-system/608-meta-blocks.html)). There is no code that runs at compile time on a token stream. `println`, `assert` and `format`-style strings are ordinary functions and f-strings, so they take no `!`.

```rok
#[impl_macro(twice)] @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let (a, b) = twice!(21 * 2);
println(f"{a} {b}");
assert(a == b, "same value");
```

```output
42 42
```

## Concurrency

Threads are Rust's: `thread::spawn` takes a closure and `join` returns its value (and throws if the thread panicked). `Send` and `Sync` exist, and the collections are not `Sync`: a closure that captures a list for another thread takes it with `[move rows]`. `Mutex`, `RwLock`, `Atomic` and `channel` are in the prelude, and a value handed to a lock or a channel cannot be used afterwards, the hand-over check that replaces Rust's move into a `Mutex`.

`async fn` and `await` come with the runtime built in, so no Tokio: `task::spawn` starts a task, `task::join` waits for several, and a script with a top-level `await` runs as `async fn main`. `await` is a prefix operator: `await f()`, not `f().await`. `par::map`, `par::filter` and `par::for_each` spread work over a pool, as Rayon does.

```rok
use rok::std::thread;

let rows = [3, 1, 2];
let h = thread::spawn([move rows] || {
    rows.sort();
    rows[0]
});
println(f"smallest: {try h.join()}");

let m = Mutex::new(0);
par::for_each(1..=100, |n| { using g = m.lock() { g.set(g.get() + n); } });
println(using g = m.lock() { g.get() });

async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms(5));
    f"item-{id}"
}
let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2))];
println(f"{await task::join(tasks):?}");
println(f"{par::map(1..=4, |x| x * x):?}");
```

```output
smallest: 1
5050
["item-1", "item-2"]
[1, 4, 9, 16]
```

## `unsafe` and FFI

`extern "C" { .. }` declares C functions, and each call to one is inside `unsafe { .. }`. `#[repr(C)]`, raw pointers (`Ptr<T>`, `ConstPtr<T>` in `rok::bare::ptr`), `#[link("m")]` and `extern "C" fn` callbacks work as in Rust. `Pod` (`rok::bare::mem::Pod`, plain old data) is an `unsafe trait` like `bytemuck::Pod`: `#[derive(Pod)]` or `unsafe impl Pod for T {}`, and the compiler still checks the layout. A `List`, `Dict` or closure cannot cross into C, since the collector may move it; pass a pointer and a length.

```rok
extern "C" {
    fn abs(x: i32) -> i32;
    fn strlen(s: str) -> usize;
}

println(unsafe { abs(-42) });
println(unsafe { strlen("hello") });
```

```output
42
5
```

## Cargo, crates and `rok.rokon`

`rok` is the package manager, and the manifest is `rok.rokon`, written in ROKON, Rok's data format, instead of TOML. Packages published to [rokstones.io](https://rokstones.io) are called stones, as crates are on crates.io. `rok.lock` pins versions as `Cargo.lock` does. `rok publish` uploads no source archive: it adds a line to the index that points at a commit of your git repository. A single file can be a program too: `rok file.rok` runs it with no package and no `main`, and `rok add json file.rok` records the dependency in the file itself.

```rokon
Package { name: "app", version: "0.1.0", toolchain: "0.2", deps: {
    "json": "^1.0",
    "geom": Dep::path("../geom"),
} }
```

| Cargo | Rok |
|---|---|
| `cargo new app`, `cargo new --lib geom` | `rok new app`, `rok new --lib geom` |
| `cargo build`, `cargo run`, `cargo check` | `rok build`, `rok run`, `rok check` |
| `cargo test`, `cargo bench` | `rok test`, `rok bench` |
| `cargo fmt`, `cargo clippy` | `rok fmt`, `rok lint` |
| `cargo doc` | `rok doc` |
| `cargo add serde`, `cargo update` | `rok add json`, `rok update` |
| `cargo install`, `cargo publish` | `rok install`, `rok publish` |
| `rustup toolchain install` | `rok toolchain install` |
| `[features]`, `--features` | `features:` in `rok.rokon`, `--features` ([chapter 18](18-features-and-stones.html)) |

`rok fmt` only fixes indentation, trailing whitespace and blank lines; it leaves the tokens and spacing within a line as you wrote them. A test is a `#[test]` function anywhere in the package, with no `#[cfg(test)] mod tests` around it: normal builds leave it out, and `rok test` runs each test in its own process, so one panic does not stop the others.

```rok
fn gcd(a: i32, b: i32) -> i32 { if b == 0 { a } else { gcd(b, a % b) } }

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
}

println(gcd(12, 18));
```

```output
6
```

## Rust idioms in Rok

| Rust | Rok |
|---|---|
| `let mut x = 0;` | `let x = 0;` |
| `let x = x.trim();` (shadowing) | `let x = x.trim();` |
| `&T`, `&mut T` parameter | `T` (a copy of a struct, the same object for a `List`), `ref p: T` |
| `&mut self` | `ref self` |
| `for x in xs.iter_mut()` | `for ref x in xs` |
| `x.clone()` before reuse | nothing: structs copy; `.clone()` for a separate `List` |
| `Box<dyn Trait>`, `Vec<Box<dyn Trait>>` | `any Trait`, `List<any Trait>` |
| `fn f(x: impl Trait)` | `fn f(x: some Trait)` |
| `fn f() -> impl Trait` | `fn f() -> some Trait` |
| `impl Fn(i32) -> i32`, `Box<dyn FnMut()>` | `some fn(i32) -> i32`, `fn(i32) -> i32`, `fn()` |
| `move \|\| ..` | `[move x] \|\| ..` |
| closure mutating a captured local | `[ref x] \|\| { x += 1; }` |
| `Option<T>`, `Some(x)`, `None` | `T?`, `x`, `none` |
| `opt.unwrap_or(d)`, `opt.map(\|v\| v.f())` | `opt ?? d`, `opt?.f()` |
| `if let Some(x) = opt` | `if let x: T = opt` |
| `fn f() -> Result<T, E>` | `fn f() throws -> T` |
| `f()?`, `f().unwrap()`, `f().ok()` | `try f()`, `try! f()`, `try? f()` |
| `Err("msg".into())`, `anyhow!("msg")` | `throw "msg"` |
| `match r { Ok(v) => v, Err(e) => .. }` | `try f() catch e { .. }` |
| `impl Drop` + scope end | `using x = e;`, or `try f.close()` for a file |
| `Vec<T>`, `HashMap<K, V>`, `HashSet<T>` | `List<T>`, `Dict<K, V>`, `Set<T>` |
| `vec![1, 2]`, `String`, `&str` | `[1, 2]`, `str` (one string type) |
| `println!("{x:?}")`, `format!(..)` | `println(f"{x:?}")`, `f".."` |
| `assert_eq!(a, b)` | `assert_eq(a, b)` |
| `x as i64`, `i64::from(x)` | `x as i64`, `x.into()` |
| `u8::try_from(x)?` | `try x.try_into::<u8>()` |
| `x.rem_euclid(n)` | `x %% n` |
| `crate::a::b`, `super::c`, `pub(crate)` | `pkg::a::b`, `sup::c`, `pub(pkg)` |
| `macro_rules! m` | `#[impl_macro(m)] @macro { .. }` |
| `fut.await`, `tokio::spawn` | `await fut`, `task::spawn` |
| `rayon` `par_iter().map(..)` | `par::map(xs, ..)` |
| `struct P { x: i32, y: i32 }` | `struct P { x: i32; y: i32; }` |
