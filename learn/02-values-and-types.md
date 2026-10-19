# 2. Values and Types

This chapter covers `let` for declaring variables, basic types such as numbers and strings, and tuples and lists for grouping values. rok is statically typed, but most types are inferred, so you rarely write them out.

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

```text
error[E200]: `coutn` is not declared; a variable is created only by `let`
  --> main.rok:2:1
   |
 2 | coutn = count + 1;
   | ^^^^^
   |
   = help: write `let coutn = count + 1;`
```

A few more rules:

- An initial value is required. `let x;` is an error. If there is no value yet, state the absence explicitly: `let x: i32 | none = none;`.
- You cannot redeclare the same name with `let` in the same scope (there is no Rust-style shadowing). A `let` in an inner block creates a separate variable from the outer one.
- A variable's type is fixed at declaration and never changes. Assigning a string to an `i32` variable is an error.

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

## Numeric types and literals

Integers are `i8` `i16` `i32` `i64` `isize` and the unsigned `u8` `u16` `u32` `u64` `usize`; floating-point numbers are `f32` `f64`. A literal with a type-name suffix has that type; without a suffix it follows the **expected type**. With no expected type, integers default to `i32` and floats to `f64`.

```rok
let a = 42;            // i32
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

`_` is a digit separator, and `0x` / `0b` / `0o` mean hexadecimal / binary / octal. Integer division truncates toward zero. A literal outside its type's range (`let x: u8 = 256;`) is a compile error.

## No implicit conversions: `as`

rok never mixes different numeric types automatically. Adding an `i32` and an `i64` is an error.

```text
error[E200]: operator `+` cannot be applied to `i32` and `i64`
  --> main.rok:3:12
   |
 3 | println(f"{a + b}");
   |            ^^^^^
   |
   = help: rok never converts numbers implicitly; convert one side: `a + (b as i32)`
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

## Integer gotchas

A few integer rules surprise people coming from Python, Go, or JavaScript.

**`len()` is a `usize`.** Lengths and indexes are `usize`, while an unsuffixed literal is an `i32`. Since nothing converts implicitly, `total / xs.len()` with an `i32` `total` is an error. Convert one side with `as`.

```text
error[E200]: operator `/` cannot be applied to `i32` and `usize`
```

```rok
let xs = [10, 20, 30];
let total = 0;
for x in xs { total += x; }
let avg = total / (xs.len() as i32);
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

**Overflow depends on the build.** `rok run` and `rok build` compile without optimization (`-O0`), with overflow checks on, so going past the range panics. An optimized build (`-O1` and up; `rok build --release` is `-O2`) turns the checks off, and the same addition silently wraps around.

```rok build
let x: i32 = 2147483647;
println(f"{x + 1}");
```

```text
$ rok run
thread 'main' panicked at src/main.rok:2:12:
attempt to add with overflow
$ rok run --release
-2147483648
```

To keep the checks in an optimized build, add `-C overflow-checks=yes`. When overflow is expected, say what you want with `checked_add` (gives `none`), `wrapping_add`, or `saturating_add`, or use `int` (below), which never overflows.

```rok
let x: i32 = 2147483647;
println(f"{x.checked_add(1):?} {x.wrapping_add(1)} {x.saturating_add(1)}");
```

```output
none -2147483648 2147483647
```

## bool, char, str

`bool` is `true` / `false`, combined with `&&`, `||`, and `!`. `char` is a single Unicode character, written in single quotes. `str` is a UTF-8 string, written in double quotes.

```rok
let ok = 3 < 5 && !false;
let ch = '가';
println(f"{ok} {ch} {ch as u32} {65u8 as char}");

let s = "héllo";
println(f"{s.len()} bytes, {s.char_count()} chars");
let t = s + ", rok";
println(f"{t} {t.to_uppercase()} {t.contains("rok")}");
println("tab:\tend");
println(r"raw: C:\path\n");
```

```output
true 가 44032 A
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

> [!NOTE]
> Advanced: `int` also has number-theory helpers such as `isqrt`, `modpow`, `gcd`, and `lcm`. They are listed in the [`int` API](../std/core/num/bigint.html).

## Constants

`const` is a value fixed at compile time. Unlike `let`, the type must be written, and it can appear at the top level of a module. The value is inlined at every use.

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

- `let` declares, `=` reassigns. Redeclaring in the same scope is forbidden, and an initial value is required.
- Unsuffixed literals follow the expected type, defaulting to `i32` / `f64`.
- There is no automatic conversion between numeric types. Use `as`, `.into()`, or `.try_into::<T>()`.
- `len()` and indexes are `usize`; convert with `as` to mix them with `i32`.
- `/` and `%` truncate toward zero; `%%` is floored modulo. Overflow panics in a debug build and wraps in an optimized one.
- `[...]` is a `List`; with a `[T; N]` type it is a fixed array. Print a `List` with `{:?}`.
- `int` is an unbounded integer; `const` is a compile-time constant.

The next chapter, [3. Functions](03-functions.html), covers functions and closures.

## Further reading

- [Variable declarations](../spec/refs-lang/005-modules/501-variable-declaration.html), [`const`](../spec/refs-lang/005-modules/502-const.html)
- [Numeric literals](../spec/refs-lang/001-lexical/106-numeric-literals.html), [string literals](../spec/refs-lang/001-lexical/107-string-literals.html), [character literals](../spec/refs-lang/001-lexical/108-character-literals.html)
- [Casts and conversions](../spec/refs-lang/002-types/214-cast-and-conversion.html), [type expressions](../spec/refs-lang/002-types/201-type-expressions.html)
- [Numeric types and `int`](../spec/refs-standard-libraries/200-core/201-numbers.html), [formatted output](../spec/refs-standard-libraries/200-core/210-fmt.html)
- Standard library API: [`str`](../std/core/str.html), [`List`](../std/core/collections/list.html)
