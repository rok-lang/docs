# Rok for Pythonistas

A map for Python programmers: what carries over, what Rok writes differently, and where the Python standard library's modules live. Every block below is a complete program; run it with `rok run file.rok`. The chapters of this guide explain each part in full.

## What carries over

- A file runs top to bottom with no `main`, as a script does: `rok file.rok`, a `#!/usr/bin/env rok` line, and dependencies declared in the file itself (`rok add json file.rok`).
- f-strings, with the same format specs: `f"{x:.2f}"`, `f"{name:>10}"`, `f"{x=}"`.
- Slices with negative indices and steps: `xs[-1]`, `xs[1:3]`, `xs[::-1]`.
- Default and named arguments: `fn f(a: i32, sep: str = " ")`, called `f(1, sep: ", ")`.
- `in`, chained comparisons (`0 <= i < n`), tuple unpacking and swapping (`(a, b) = (b, a)`).
- `try` / `catch` / `finally`, generators (`yield fn`), `async` / `await`, `match` with guards.
- Big integers: `int` has no size limit.
- A REPL (`rok repl`) and a Jupyter kernel (`rok jupyter`).

## The first surprises

- `print` adds no newline; `println` does. Both take several values and `sep:`.
- Every value has a type the compiler knows. Parameters with no type are generic: they accept any type with the methods the body calls, which is duck typing checked at compile time.
- Blocks use `{ }`, not indentation, and a new name is declared with `let`.
- There is no truthiness: a condition is a `bool`. Write `!xs.is_empty()`, `n != 0`, or `x != none`.
- A missing value is `none`, and a type that may hold it is written `T | none`.
- `/` on integers rounds toward zero (`7 / 2` is `3`); floor division is `n.div_floor(2)` and `%%` is the floored remainder. `//` starts a comment.
- An `i32` that overflows panics; use `i64`, or `int` for any size.
- Structs are values: `let q = p;` copies. `List`, `Dict` and `Set` are shared handles, as in Python.
- `str` slices count bytes, not characters; walk characters with `s.chars()`.

```rok
let name = "Ada";
let xs = [1, 2, 3];
print("no newline, ");
println("then one");
println("total:", xs.len(), xs);
println(1, 2, 3, sep: ", ");
println(f"{name} has {xs.len()} items: {xs:?}");
let x = 5;
println(f"{x=}");
```

```output
no newline, then one
total: 3 [1, 2, 3]
1, 2, 3
Ada has 3 items: [1, 2, 3]
x=5
```

## Control flow

`if` is an expression, so it also takes the place of `a if c else b`. `elif` is `else if`, and `for i in range(n)` is `for i in 0..n`.

```rok
fn grade(score: i32) -> str {
    if score >= 90 { "A" } else if score >= 80 { "B" } else { "C" }
}

let people = [("ada", 92), ("bob", 78)];
for (name, score) in people {
    println(f"{name}: {grade(score)}");
}
for (i, (name, _)) in people.enumerate() {
    println(f"{i} {name}");
}
for i in 0..3 {
    if i == 1 { continue; }
    println(i);
}
let label = if people.len() > 1 { "many" } else { "one" };
println(label);
let c = 5;
if 0 < c < 10 && c != 7 { println("in range"); }
if 3 in [1, 2, 3] { println("member"); }
```

```output
ada: A
bob: C
0 ada
1 bob
0
2
many
in range
member
```

`and`, `or` and `not` are `&&`, `||` and `!`; `not in` is `!in`.

## Comprehensions are iterator chains

Rok has no list, dict or set comprehensions. A collection has the iterator methods, and `to_list()`, `to_dict()` and `to_set()` collect the result. `sum`, `min`, `max`, `sorted`, `any` and `all` are methods too.

```rok
let xs = [1, 2, 3, 4, 5];
// [x * x for x in xs if x % 2 == 1]
let odd_squares = xs.filter(|x| x % 2 == 1).map(|x| x * x).to_list();
println(f"{odd_squares:?}");
println(f"{xs.sum()} {xs.min() ?? 0} {xs.max() ?? 0}");

let words = ["pear", "fig", "apple"];
println(f"{words.sorted():?}");
println(f"{words.sorted_by_key(|w| w.len()):?}");
println(words.join(", "));
println(f"{xs.any(|x| x > 4)} {xs.all(|x| x > 0)}");

// {w: len(w) for w in words}
let lengths = words.map(|w| (w, w.len())).to_dict();
println(lengths["fig"]);
println([1, 1, 2].to_set().len());

let (a, b) = (1, 2);
(a, b) = (b, a);
println(f"{a} {b}");
```

```output
[1, 9, 25]
15 1 5
["apple", "fig", "pear"]
["fig", "pear", "apple"]
pear, fig, apple
true true
3
2
2 1
```

Closures are written `|x| x + 1` where Python writes `lambda x: x + 1`.

## `none` instead of `None`

A function that may find nothing returns `T | none`. `??` gives a default, `x == none` tests, and `if let` takes the value out.

```rok
let xs = ["a", "b"];
let i = xs.position(|x| x == "b");
if let k: usize = i { println(f"found at {k}"); }
println(xs.position(|x| x == "z") ?? 99);
if !xs.is_empty() { println(f"{xs.len()} items"); }
```

```output
found at 1
99
2 items
```

## Errors

A function that can fail says `throws`, and each call to it is marked with `try`. `throw "message"` raises a plain error; `try .. catch` handles one, and `try?` turns a failure into `none`.

```rok
fn parse_port(s: str) throws -> i64 {
    let n = try s.parse::<i64>();
    if n > 65535 { throw "port out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);
```

```output
8080
error: port out of range
0
-1
```

`int("42")` is `try "42".parse::<int>()`, and `float(s)` is `try s.parse::<f64>()`.

## Classes are structs, traits and `impl`

A struct holds the fields, an `impl` block holds the methods, and `#[derive(..)]` writes `__eq__`, `__repr__` and `__hash__`. A method that changes its value takes `ref self`.

```rok
#[derive(Debug, PartialEq)]
struct Point { x: f64; y: f64; }

impl Point {
    fn new(x: f64, y: f64) -> Point { Point { x: x, y: y } }
    fn norm(self) -> f64 { (self.x * self.x + self.y * self.y).sqrt() }
    fn scale(ref self, k: f64) { self.x *= k; self.y *= k; }
}

let p = Point::new(3.0, 4.0);
println(p.norm());
p.scale(2.0);
println(f"{p:?}");
println(p == Point { x: 6.0, y: 8.0 });
```

```output
5
Point { x: 6.0, y: 8.0 }
true
```

There is no inheritance. A trait lists the methods a type provides, with default bodies where it wants them, and `any Shape` holds any type that implements it.

```rok
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }

fn loud(x) -> str { x.to_uppercase() }
println(loud("hi"));
```

```output
area 4.0
area 3.1
HI
```

## Values, handles and numbers

```rok
struct P { x: i32; }
let a = [1];
let b = a;
b.push(2);
println(f"{a:?}");
let p = P { x: 1 };
let q = p;
q.x = 9;
println(f"{p.x} {q.x}");

println(7 / 2);
println((-7).div_floor(2));
println(-7 %% 2);
println(7.0 / 2.0);
println((7 as f64) / 2.0);
println(2 ** 10);
```

```output
[1, 2]
1 9
3
-4
1
3.5
3.5
1024
```

`**` is the power operator, as in Python: `2 ** 10`, and `-x ** 2` is `-(x ** 2)` ([chapter 21](21-arrays-and-numerics.html)). An integer base takes a `u32` exponent, so `2 ** -1` is an error that says to write `2.0 ** -1.0`.

## `with` is `using`

`using` binds a value that is closed when its scope ends, as a `with` block closes its file.

```rok
struct Conn { name: str; }
impl Drop for Conn {
    fn drop(ref self) { println(f"closed {self.name}"); }
}
fn work() {
    using c = Conn { name: "db" };
    println(f"using {c.name}");
}
work();
println("done");
```

```output
using db
closed db
done
```

## Files and text

```rok
use rok::std::{env, fs, process};

let path = env::temp_dir() / f"pyrok_{process::id()}.txt";
try fs::write(path, "one\ntwo\n");
let text = try fs::read_to_str(path);
for line in text.lines() { println(line.to_uppercase()); }
try fs::remove_file(path);

println(42.to_str() + "px");
println(f"{3.14159:.2}");
println("ab".repeat(3));
println("a,b".split(",").to_list().len());
```

```output
ONE
TWO
42px
3.14
ababab
2
```

## Python idioms in Rok

| Python | Rok |
|---|---|
| `print(x)` / `print(x, end="")` | `println(x)` / `print(x)` |
| `print(a, b, sep=", ")` | `println(a, b, sep: ", ")` |
| `f"{x!r}"` | `f"{x:?}"` |
| `a and b`, `a or b`, `not a` | `a && b`, `a \|\| b`, `!a` |
| `x not in xs` | `x !in xs` |
| `x is None`, `x is not None` | `x == none`, `x != none` |
| `None`, `True`, `False` | `none`, `true`, `false` |
| `a if c else b` | `if c { a } else { b }` |
| `elif` | `else if` |
| `def f(x):` / `pass` | `fn f(x) { }` (an empty block needs no `pass`) |
| `lambda x: x + 1` | `\|x\| x + 1` |
| `if xs:` | `if !xs.is_empty()` |
| `len(xs)`, `sorted(xs)`, `sum(xs)` | `xs.len()`, `xs.sorted()`, `xs.sum()` |
| `min(a, b)`, `abs(x)`, `round(x)` | `a.min(b)`, `x.abs()`, `x.round()` |
| `str(x)` | `x.to_str()` or `f"{x}"` |
| `int(s)`, `float(s)` | `try s.parse::<int>()`, `try s.parse::<f64>()` |
| `", ".join(xs)` | `xs.join(", ")` |
| `s * 3` | `s.repeat(3)` |
| `range(n)`, `range(a, b)` | `0..n`, `a..b` |
| `for i, x in enumerate(xs)` | `for (i, x) in xs.enumerate()` |
| `for k, v in d.items()` | `for (k, v) in d` |
| `[f(x) for x in xs if c]` | `xs.filter(\|x\| c).map(\|x\| f(x)).to_list()` |
| `x ** 2`, `n // 2` | `x ** 2`, `n.div_floor(2)` |
| `raise ValueError("m")` / `except E as e` | `throw "m"` / `catch e: E` |
| `with open(p) as f:` | `using f = try File::open(p);` |
| `assert x == 2, "m"` | `assert(x == 2, "m");` |
| `class C:` with `self` | `struct C { .. }` and `impl C { fn m(self) { .. } }` |
| `C("h", 1)` | `C { host: "h", port: 1 }` |
| `import m` / `from m import x` | `use m;` / `use m::x;` |

Each of these Python forms, typed in Rok, stops the build with an error whose help shows the Rok form to write.

## Python modules in Rok

| Python | Rok |
|---|---|
| `math` | methods on numbers (`x.sqrt()`, `x.sin()`, `x.floor()`) and `f64::PI`; no import |
| `os`, `sys` | `rok::std::env` (`env::args()`, `env::var(..)`), `rok::std::process` (`process::exit(..)`, `process::id()`) |
| `os.path`, `pathlib` | `rok::std::path::Path` (`Path::new("a") / "b"`), `rok::std::fs` |
| `open`, `shutil` | `rok::std::fs` (`fs::read_to_str`, `fs::write`, `File::open`) |
| `subprocess` | `rok::std::process::Command` |
| `random` | `rok::std::random` (`random::range(1..7)`, `random::choice(xs)`, `random::shuffle(xs)`) |
| `time` | `rok::core::time` (`Instant::now()`, `Duration`) |
| `threading` | `rok::std::thread` (`thread::spawn(..)`, `join`) |
| `asyncio` | `async fn` and `await`, built in |
| `collections` | `Dict`, `Set`, `Deque`, `BinaryHeap`, built in |
| `itertools`, `functools` | iterator methods: `zip`, `chain`, `windows`, `chunks`, `fold`, `reduce`, `scan` |
| `json` | the `json` stone: `rok add json` |
| `input()` | `try io::input("prompt")` |
| `unittest`, `pytest` | `#[test]` functions and `rok test` |
| `black`, `ruff` | `rok fmt`, `rok lint` |
| `pip`, `venv` | `rok add`, `rok env` |

`re`, `datetime`, `logging` and `argparse` have no counterpart in the standard library yet.
