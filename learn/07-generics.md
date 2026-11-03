# 7. Generics

Use generics when you want the same logic to work for many types. Rok generics are compiled separately for each concrete type they're called with (monomorphization), so they run as fast as hand-written per-type functions.

## Generic functions

Declare type parameters in `<T>` after the function name and use them in the parameter and return types. At the call site you usually don't write the type; the compiler infers it from the arguments.

```rok
fn first_or<T>(xs: List<T>, fallback: T) -> T {
    xs.first() ?? fallback
}

println(f"{first_or([10, 20], 0)}");
println(first_or(["a", "b"], "-"));
let empty: List<f64> = [];
println(f"{first_or(empty, 1.5)}");
```

```output
10
a
1.5
```

Within one call, `T` is a single type. Passing different types, as in `first_or([1, 2], "x")`, is a compile error.

## Trait bounds

With an unbounded `T`, all you can do is move values around. To compare or add them, require the needed traits with a **bound** such as `T: Ord`. Join multiple bounds with `+`. The compiler checks a generic body against its bounds alone, so the body cannot break no matter which type it's called with.

```rok
fn largest<T: Ord>(xs: List<T>) -> T? {
    let best = xs.first()?;
    for x in xs {
        if x > best { best = x; }
    }
    best
}

println(f"{largest([3, 9, 2]):?}");
println(f"{largest(["pear", "apple"]):?}");
let none_here: List<i32> = [];
println(f"{largest(none_here):?}");
```

```output
9
"pear"
none
```

With `T: num::Zero`, you can also call static methods on the type parameter, such as `T::zero()`.

```rok
fn sum<T: num::Zero + ops::Add<Output = T>>(xs: List<T>) -> T {
    let acc = T::zero();
    for x in xs { acc = acc + x; }
    acc
}

println(f"{sum([1, 2, 3])} {sum([0.5, 0.25])}");
```

```output
6 0.75
```

### The body is checked against its bounds

A generic body is type-checked once, on its own, before any call is looked at. Inside it, `T` supports only what its bounds promise, plus what every type supports: moving, storing into `any`, forming `T | none`, and `is`. Forgetting a bound is an error in the body, even if every call passes a type that would work.

```rok norun
fn double<T>(x: T) -> T { x + x }
```

```text check=checks/07-missing-bound.rok
error[E0054]: operator `+` on `T` needs the bound `T: Add`
  --> main.rok:1:27
   |
 1 | fn double<T>(x: T) -> T { x + x }
   |                           ^^^^^
```

With the bound added, the same body compiles for every type that has `Add`. The operations that need no bound work without one.

```rok
fn double<T: ops::Add<T, Output = T>>(x: T) -> T { x + x }
fn keep<T>(x: T, yes: bool) -> T | none { if yes { x } else { none } }
fn boxed<T>(x: T) -> any { x }

println(f"{double(21)} {double(1.5)}");
println(f"{keep("a", true):?} {keep(1, false):?}");
println(f"{boxed(3) is i64}");
```

```output
42 3
"a" none
true
```

Because of this rule, a library's generic function that compiles for you compiles for everyone who calls it; an error never surfaces at a distant call site. Trait default methods are checked the same way, with `Self` known only through the trait. A function with untyped parameters (an implicit generic, below) is the exception.

A bound can name the same trait twice with different parameters. Each use in the body picks the bound whose parameter matches the argument's type, just as an operator picks among a type's impls.

```rok
struct Meters { v: f64; }
impl ops::Add<i32> for Meters { type Output = Meters; fn add(self, rhs: i32) -> Meters { Meters { v: self.v + rhs as f64 } } }
impl ops::Add<f64> for Meters { type Output = Meters; fn add(self, rhs: f64) -> Meters { Meters { v: self.v + rhs } } }

fn nudge<T: ops::Add<i32, Output = T> + ops::Add<f64, Output = T>>(x: T) -> T {
    (x + 1) + 0.5          // Add<i32>, then Add<f64>
}

println(f"{nudge(Meters { v: 2.0 }).v}");
```

```output
3.5
```

## `where` clauses

When bounds get long, moving them to a `where` clause at the end of the signature makes it easier to read. The meaning is the same.

```rok
fn join_all<T>(xs: List<T>, sep: str) -> str
where
    T: Display,
{
    let out = StrBuf::new();
    for i in 0..xs.len() {
        if i > 0 { out.push_str(sep); }
        out.push_str(f"{xs[i]}");
    }
    out.to_str()
}

println(join_all([1, 2, 3], ", "));
println(join_all([true, false], " / "));
```

```output
1, 2, 3
true / false
```

## Generic structs and impl

Structs and enums can take type parameters too. When adding methods, declare the parameters on the `impl` as well, as in `impl<A, B> Pair<A, B>`.

```rok
struct Pair<A, B> { first: A; second: B; }

impl<A, B> Pair<A, B> {
    fn swap(self) -> Pair<B, A> {
        Pair { first: self.second, second: self.first }
    }
}

let p = Pair { first: 1, second: "one" };
let q = p.swap();
println(f"{q.first} {q.second}");
```

```output
one 1
```

## Associated types

To let each implementation of a trait pick one type, declare an **associated type** `type Item;`. Inside the trait, refer to it as plain `Item`; in generic code outside, as `C::Item`. The standard `Iterator` works this way (next chapter).

```rok
trait Container {
    type Item;
    fn get(self, i: usize) -> Item | none;
    fn first(self) -> Item | none { self.get(0) }
}

struct Shelf { books: List<str>; }

impl Container for Shelf {
    type Item = str;
    fn get(self, i: usize) -> str? { self.books.get(i) }
}

fn first_of<C: Container>(c: C) -> C::Item | none { c.first() }

let s = Shelf { books: ["Dune", "Emma"] };
println(f"{first_of(s):?} {s.get(5):?}");
```

```output
"Dune" none
```

It is an associated type rather than a type parameter because `Item` is fixed to exactly one type for a given implementor. Users don't need to spell out `Shelf`'s element type.

## Advanced: const generics

You can skip this section on a first read. A parameter can also be a compile-time **value** instead of a type. Use this when a value is part of a type, like the length of a fixed-size array `[T; N]`.

```rok
fn total<const N: usize>(a: [i32; N]) -> i32 {
    let s = 0;
    for i in 0..N { s += a[i]; }
    s
}

let a: [i32; 4] = [1, 2, 3, 4];
let z: [f64; 3] = [0.5; 3];
println(f"{total(a)} {a.len()} {z:?}");
```

```output
10 4 [0.5, 0.5, 0.5]
```

`const` parameters can only be integers, `bool`, or `char`.

## Hiding a return type: `some`

Iterator adapters and closures have long types you don't want in a signature. `-> some Trait` returns one concrete type that callers see only as `Trait`: the body picks it, calls stay static, and you can change the body without changing the signature. It works on `pub` functions too.

```rok
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k
}

println(evens(10).map(|x| x * x).sum());
println(adder(3)(4));
```

```output
120
7
```

A caller may use only what the bound gives: `evens(10).contains(4)` is an error, since `Iterator` has no `contains`. The body returns one type; to return different types from different paths, write `-> any Trait`.

## Taking any type with a trait: `some` parameters

A parameter written `x: some Trait` takes any type with `Trait`, without naming a type parameter: `fn show(x: some Display)` is `fn show<T: Display>(x: T)`. Each call has one static type, the argument's, and each `some` parameter is its own type.

```rok
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a} {b}" }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

println(show(5));
println(both("hi", 2.5));
println(twice(|x| x * 3, 2));
```

```output
<5>
hi 2.5
18
```

The type has no name, so a call can't choose it with `::<..>`; declare `<T: Trait>` when it should. `some` is the static counterpart of `any Trait`: `some` is one type, fixed at each call or by the body, while `any` holds a value of any such type and dispatches at run time.

## Naming a hidden type: `type X = some Trait`

A `-> some Trait` type has no name, so you can't keep it in a struct field or write it in another signature. `type X = some Trait;` gives it one. The functions of X's module that return `X` define it: their bodies pick the one concrete type, as with `-> some`. Every other module can store `X` in fields, `let x: X` and `List<X>`, pass it and return it, and sees only the bound.

```rok
mod nums {
    pub type Evens = some Iterator<Item = i64>;

    pub fn evens(n: i64) -> Evens {
        (0..n).filter(|x| x % 2 == 0)
    }
}

struct Batch {
    name: str;
    items: nums::Evens;
}

fn total(b: Batch) -> i64 { b.items.sum() }

let batches = [
    Batch { name: "small", items: nums::evens(5) },
    Batch { name: "large", items: nums::evens(10) },
];
for b in batches {
    println(f"{b.name}: {total(b)}");
}
```

```output
small: 6
large: 20
```

Inside `nums`, `Evens` is the concrete filter type, so its functions can use all of it. Outside, only `Iterator` is there: `b.items.len()` is an error, and so is `let n: i64 = ..` for a `some Display` type ([E0147](../errors/E0147.html)). The module needs at least one function that returns `Evens`, and all of them must give the same type ([E0146](../errors/E0146.html)). Calls and fields stay static: a `Batch` holds the filter itself, with no box. `type X = any Trait` is something else, a plain alias of a dynamic slot whose values may each have a different type.

`some` goes in those three places only. A field or a type argument written `some Trait` is an error; name the type with `type`, as `Evens` does, or declare `<T: Trait>`:

```rok norun
struct Label {
    text: some Display;
}
```

```text check=checks/07-some-field.rok
error[E0010]: `some Trait` is only a function's parameter or return type, or the whole right side of a `type` alias
  --> main.rok:2:11
   |
 2 |     text: some Display;
   |           ^^^^
   |
   = help: for a variable, leave the type out (`let x = ..`); for a field or a type argument, declare `<T: Trait>` and write `T`, or name the type with `type Name = some Trait;`; for a value of several types, write `any Trait`
```

## Generics and `any`

Generics are static. When you instead need a slot whose value's type may change at runtime, write `any` **explicitly**. Narrow `any` values with `is` before use. Dynamic dispatch costs appear only where you wrote `any`.

```rok
let v: any = 1;
println(f"{v is i64}");
v = "text";
if v is str {
    println(f"now a string, length {v.len()}");
}
```

```output
true
now a string, length 4
```

The spec defines a function with omitted parameter types, `fn show_sum(x, y)`, as an implicit generic, not as `any`. Only a function that is not `pub` or `pub(pkg)` may omit them; a `pub` one writes every parameter type ([E0139](../errors/E0139.html)). No bounds are inferred: the function is monomorphized at each call site, and each copy of the body is checked with that call's argument types, so an error points into the body and at the call that caused it.

```rok
fn show_sum(x, y) { println(f"{x + y}"); }   // implicit generic: fn show_sum<T, U>(x: T, y: U)

show_sum(1, 2);       // 3
show_sum(1.5, 2.0);   // 3.5
```

```output
3
3.5
```

## Summary

- Declare generics with `fn f<T>(..)`, `struct S<A, B>`, and `impl<A, B> S<A, B>`. Type arguments are usually inferred.
- Bounds like `T: Ord + Clone` determine which operations the body may use. Move long ones to a `where` clause.
- An associated type `type Item;` is fixed to one type per implementation and is written `C::Item` from outside.
- `const N: usize` takes a value as a type parameter.
- `-> some Trait` returns one hidden concrete type that callers use only through `Trait`; `x: some Trait` takes any type with `Trait`, one per call; `type X = some Trait;` names a hidden type, fixed by its module's functions, for fields and signatures.
- Generics are static (monomorphized); `any` is a dynamic slot that exists only where you write it.

## Further reading

- [2.10. Generics](../spec/refs-lang/002-types/210-generics.html): [2.10.3. Trait bounds](../spec/refs-lang/002-types/210-generics.html#2103-trait-bounds), [2.10.4. where clauses](../spec/refs-lang/002-types/210-generics.html#2104-where-clauses), [2.10.9. Body checking](../spec/refs-lang/002-types/210-generics.html#2109-body-checking), [2.10.7. Interaction with gradual typing](../spec/refs-lang/002-types/210-generics.html#2107-combination-with-gradual-typing)
- [2.8.3. Associated types](../spec/refs-lang/002-types/208-trait.html#283-associated-types)
- [2.11. `const` generics](../spec/refs-lang/002-types/211-const-generics.html)
- [2.2. The `any` slot](../spec/refs-lang/002-types/202-any-slot.html)
- Next chapter: [8. Collections and Iterators](08-collections-iterators.html)
