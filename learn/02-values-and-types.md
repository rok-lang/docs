# 2. Values and Types

This chapter covers `let` for declaring variables, basic types such as numbers and strings, and tuples and lists for grouping values. Rok is statically typed, but most types are inferred, so you rarely write them out.

## `let` and reassignment

`let` creates a new variable; to write to an existing variable, use `=` without `let`. Variables are mutable by default (there is no `mut` marker).

```rok
let name = "rok";
let count = 0;
count = count + 1;
count += 1;
println(f"{name}: {count}");
```

```output
rok: 2
```

Because declaration and reassignment are syntactically distinct, a typo never silently creates a new variable. Assigning to a name that was never declared is a compile error.

```text check=checks/02-undeclared.rok
error[E0057]: `coutn` is not declared; a variable is created only by `let`
  --> main.rok:2:1
   |
 2 | coutn = count + 1;
   | ^^^^^
   |
   = help: did you mean `count`? To declare a new variable, write `let coutn = count + 1;`
```

A few more rules:

- An initial value is required. `let x;` is an error. If there is no value yet, state the absence explicitly: `let x: i32 | none = none;`.
- A variable's type is fixed at declaration and never changes. Assigning a string to an `i32` variable is an error.
- A `let` of a name you already have makes a new variable that hides the old one, in the same scope or an inner block. Its initial value can read the old one, and its type may differ, so a value can keep its name through each step.
- Keywords cannot be names. Six of them are reserved for later and have no syntax yet: `macro`, `defer`, `dyn`, `move`, `gen` and `do`. To use a keyword as a name anyway, put `r#` in front (`r#move`, `r#type`); the error suggests it.

```rok
let a = 1;
{
    let a = 42;
    a = 10;
    println(f"inner a = {a}");
}
println(f"outer a = {a}");
```

```output
inner a = 10
outer a = 1
```

```rok
let line = "  42  ";
let line = line.trim();          // reads the old `line`
let line: i32 = try! line.parse();
println(f"{line + 1}");
```

```output
43
```

To change a variable's value, assign without `let`. A redeclaration leaves the old variable as it was ([5.1.4](../spec/refs-lang/005-modules/501-variable-declaration.html#514-redeclaration-in-the-same-scope)): a file you opened is not closed by it, so close it first or move it into the new value, as in `let f = BufReader::new(f);` (chapter 13).

## Numeric types and literals

Integers are `i8` `i16` `i32` `i64` `isize` and the unsigned `u8` `u16` `u32` `u64` `usize`; floating-point numbers are `f32` `f64`. A literal with a type-name suffix has that type; without a suffix it follows the **expected type**. A literal you bind with `let` takes its type from how the variable is used later in the function; only when nothing says, integers default to `i64` and floats to `f64`.

```rok
let a = 42;            // i64
let b: u8 = 200;       // u8, from the annotation
let c = 7i64;          // suffix
let d = 2.5f32;
let e = 3.14;          // f64
let big = 1_000_000u64;
println(f"{a} {b} {c} {d} {e} {big}");
println(f"{0xFF} {0b1010} {0o17} {1e3}");
println(f"{7 / 2} {-7 / 2} {7.0 / 2.0}");
```

```output
42 200 7 2.5 3.14 1000000
255 10 15 1000
3 -3 3.5
```

Because the type comes from the uses, you rarely write it:

```rok
fn count_big(xs: List<i32>) -> i32 {
    let c = 0;                 // i32: the function returns it
    for x in xs { if x > 100 { c += 1; } }
    c
}
let hits = Atomic::new(0);     // the counter type comes from fetch_add below
let step: u8 = 2;
hits.fetch_add(step);
println(f"{count_big([5, 500, 900])} {hits.load()}");
```

```output
2 2
```

Rok still never converts: a variable holds one type. Using `let t = 0;` as an `f64` is an error that tells you to write `let t = 0.0;` at the declaration.

`_` is a digit separator, and `0x` / `0b` / `0o` mean hexadecimal / binary / octal. A leading zero does not make a number octal, and `0755` is a compile error: write `0o755` for octal or `755` for decimal. Integer division truncates toward zero. A literal outside its type's range (`let x: u8 = 256;`) is a compile error.

## No implicit conversions: `as`

Rok never mixes different numeric types automatically. Adding an `i32` and an `i64` is an error.

```text check=checks/02-mixed-int.rok
error[E0054]: operator `+` cannot be applied to `i32` and `i64`
  --> main.rok:3:12
   |
 3 | println(f"{a + b}");
   |            ^^^^^
   |
   = help: rok never converts numbers implicitly; convert one side: `(a as i64) + b`
```

This is by design, to prevent silent loss of precision. You write conversions explicitly. `as` always succeeds but may truncate or change the value; `.into()` is only allowed for lossless conversions.

```rok
let x: i32 = 10;
let y: i64 = 3;
println(f"{(x as i64) + y}");
let w: i64 = x.into();
println(f"{w}");
println(f"{2.9 as i32} {300 as u8} {x as f64 / 4.0}");
```

```output
13
10
2 44 2.5
```

`2.9 as i32` drops the fractional part, and `300 as u8` keeps only the low 8 bits. For a conversion that must not lose anything, `.try_into::<T>()` checks the range and throws an error instead; [chapter 10](10-errors.html) shows how to handle errors.

A number never casts to `bool`: `n as bool` is an error. Write the comparison you mean, `n != 0` (or `x != 0.0` for a float).

## Integer gotchas

A few integer rules surprise people coming from Python, Go, or JavaScript.

**`len()` is a `usize`.** Lengths and indexes are `usize`. A literal takes its type from its uses, so `let total = 6;` divided by `xs.len()` is a `usize` too, but a value whose type is already fixed (an `i64` field, parameter or annotation) is not converted: `total / xs.len()` with an `i64` `total` is an error. Convert one side with `as`.

```text check=checks/02-len-usize.rok
error[E0054]: operator `/` cannot be applied to `i64` and `usize`
```

```rok
let xs: List<i64> = [10, 20, 30];
let total = 0;
for x in xs { total += x; }
let avg = total / (xs.len() as i64);
println(f"average {avg}");

for i in 0..xs.len() {       // i is a usize, so xs[i] just works
    print(f"{i}:{xs[i]} ");
}
println("");
```

```output
average 20
0:10 1:20 2:30 
```

**`/` and `%` truncate toward zero.** `-7 / 2` is `-3` and `-7 % 2` is `-1`: the remainder takes the sign of the left side, as in C, Go, and Rust. For Python's `%`, whose result takes the sign of the divisor, use `%%` (floored modulo). It is handy for wrapping an index around.

```rok
println(f"{-7 / 2} {-7 % 2} {-7 %% 2}");
let len = 5;
let i = -1;
println(f"{i % len} {i %% len}");
```

```output
-3 -1 1
-1 4
```

**Operators bind as in Rust.** `*` `/` `%` bind tighter than `+` `-`, those tighter than the shifts `<<` `>>`, then `&`, `^`, `|`, and all of them tighter than comparisons. So `1 + 2 << 3` is `(1 + 2) << 3`, and `x & mask == 0` is `(x & mask) == 0`, not C's `x & (mask == 0)`.

```rok
println(f"{1 + 2 << 3} {6 & 3 == 2}");
```

```output
24 true
```

**Overflow panics.** Going past the range of a fixed-width integer panics in every build, debug or `--release`, so a wrong value never slips through silently.

```rok build
let x: i32 = 2147483647;
println(f"{x + 1}");
```

```text
$ rok run --release
thread 'main' panicked at src/main.rok:2:12:
attempt to add with overflow
```

The checks cost a few percent in tight integer loops. A program that needs that back turns them off with `-C overflow-checks=no`, or for one profile in `rok.rokon` with `profile: { "release": { overflow_checks: false } }`; then overflow wraps around. When overflow is expected, say what you want with `checked_add` (gives `none`), `wrapping_add`, or `saturating_add`, or use `int` (below), which never overflows.

```rok
let x: i32 = 2147483647;
println(f"{x.checked_add(1):?} {x.wrapping_add(1)} {x.saturating_add(1)}");
```

```output
none -2147483648 2147483647
```

## bool, char, str

`bool` is `true` / `false`, combined with `&&`, `||`, and `!`. Comparisons in one direction chain: `0 <= x < 10` is `0 <= x && x < 10`, with `x` evaluated once (`a < b > c` is an error). `x in xs` and `x !in xs` test membership in a list, set, dictionary (its keys), string or range. `char` is a single Unicode character, written in single quotes; `b'A'` is that ASCII character's byte, a `u8`. `str` is a UTF-8 string, written in double quotes.

```rok
let ok = 3 < 5 && !false;
let x = 7;
println(f"{ok} {0 <= x < 10} {x in [1, 7]} {x !in 0..5}");
let ch = '가';
println(f"{ch} {ch as u32} {65u8 as char} {b'A'}");

let s = "héllo";
println(f"{s.len()} bytes, {s.char_count()} chars");
let t = s + ", rok";
println(f"{t} {t.to_uppercase()} {t.contains("rok")}");
println("tab:\tend");
println(r"raw: C:\path\n");
```

```output
true true true true
가 44032 A 65
6 bytes, 5 chars
héllo, rok HÉLLO, ROK true
tab:	end
raw: C:\path\n
```

The Hangul syllable `'가'` (U+AC00, code point 44032) shows that a `char` holds any Unicode character, not just ASCII. `len()` counts bytes; use `char_count()` when you need the number of characters. `r"..."` is a raw string that does not process escapes. Other `str` methods (`trim`, `split`, `replace`, `parse`, and so on) are in the [standard library docs](../std/core/str.html).

## Tuples, arrays, and lists

A **tuple** groups a few values of different types. Access them with `.0`, `.1`, or destructure with `let`.

```rok
let p = (1, "one", true);
println(f"{p.0} {p.1} {p:?}");
let (n, word, _) = p;
println(f"{word} = {n}");
```

```output
1 one (1, "one", true)
one = 1
```

A `[...]` literal is a **`List`** by default. It is a growable list, and every variable that refers to the same list sees changes to it. A `List` cannot be printed with `{}`, so use the Debug format `{:?}`.

```rok
let names = ["ada", "grace"];
names.push("linus");
println(f"{names:?} {names.len()} {names[0]}");

let empty: List<i32> = [];
println(f"{empty.is_empty()}");
```

```output
["ada", "grace", "linus"] 3 ada
true
```

If the variable's type is `[T; N]`, the same literal becomes a fixed-length **array**. Its size is known at compile time, so use it for fixed-shape data such as matrices and buffers.

```rok
let grid: [[i32; 2]; 2] = [[1, 2], [3, 4]];
println(f"{grid:?} {grid[1][0]} {grid.len()}");
```

```output
[[1, 2], [3, 4]] 3 2
```

When the element type can't be inferred from the elements, as with an empty list, write the type: `let xs: List<i32> = [];`.

## Big integers: `int`

Fixed-width types like `i32` are fast but have a range. When you need exact integers without worrying about range, use `int`. Small values are stored like fixed-width integers and automatically switch to big integers as they grow. An `int` only arises where the type is written out, and mixing it with fixed-width types also requires explicit conversion.

```rok
let big: int = 1;
for _ in 0..100 {
    big = big * 2;
}
println(f"2^100 = {big}");
```

```output
2^100 = 1267650600228229401496703205376
```

An `int` becomes an `f64` with `big.to_f64()`, which rounds to the nearest value. Rounding is not lossless, so `into()` and `as f64` do not compile for it; `try f64::try_from(big)` throws instead of giving `inf` when the value is past `f64::MAX`.

> [!NOTE]
> Advanced: `int` also has number-theory helpers such as `isqrt`, `modpow`, `gcd`, and `lcm`. They are listed in the [`int` API](../std/core/num/bigint.html).

## Constants

`const` is a value fixed at compile time. Unlike `let`, the type must be written, and it can appear at the top level of a module. The value is inlined at every use, so each use is a new value: changing one (`XS.push(4)` on a `const` list) is an error; copy it into a variable first (`let xs = XS;`).

```rok
const MAX_USERS: i32 = 100;
const GREETING: str = "welcome";

fn capacity_left(used: i32) -> i32 {
    MAX_USERS - used
}

println(f"{GREETING}: {capacity_left(42)} seats left");
```

```output
welcome: 58 seats left
```

## Summary

- `let` declares, `=` reassigns. A new `let` of a name makes a new variable, and an initial value is required.
- Unsuffixed literals follow the expected type, defaulting to `i64` / `f64`.
- There is no automatic conversion between numeric types. Use `as`, `.into()`, or `.try_into::<T>()`.
- `len()` and indexes are `usize`; convert with `as` to mix them with `i64`.
- `/` and `%` truncate toward zero; `%%` is floored modulo. Overflow panics in every build unless overflow checks are turned off.
- `[...]` is a `List`; with a `[T; N]` type it is a fixed array. Print a `List` with `{:?}`.
- `int` is an unbounded integer; `const` is a compile-time constant.

## Further reading

- [Variable declarations](../spec/refs-lang/005-modules/501-variable-declaration.html), [`const`](../spec/refs-lang/005-modules/502-const.html)
- [Numeric literals](../spec/refs-lang/001-lexical/106-numeric-literals.html), [string literals](../spec/refs-lang/001-lexical/107-string-literals.html), [character literals](../spec/refs-lang/001-lexical/108-character-literals.html)
- [Casts and conversions](../spec/refs-lang/002-types/214-cast-and-conversion.html), [type expressions](../spec/refs-lang/002-types/201-type-expressions.html)
- [Numeric types and `int`](../spec/refs-standard-libraries/200-core/201-numbers.html), [formatted output](../spec/refs-standard-libraries/200-core/210-fmt.html)
- Standard library API: [`str`](../std/core/str.html), [`List`](../std/core/collections/list.html)
- Next chapter: [3. Functions](03-functions.html)
