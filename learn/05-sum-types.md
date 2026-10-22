# 5. Sum Types and none

When a value is "either this or that", rok joins the types with `|`, as in `A | B`. This chapter covers sum types and the tools for working with the most common one, `T | none`, which means "there may be no value".

## Sum types `A | B`

A value of a sum type belongs to exactly one of the listed variants. The value carries a tag recording which one, so `match` can extract it with a type-test pattern `name: Type`.

```rok
fn describe(v: i32 | str | bool) -> str {
    match v {
        n: i32 => f"integer {n}",
        s: str => f"string {s}",
        b: bool => f"boolean {b}",
    }
}

println(describe(42));
println(describe("rok"));
println(describe(true));
```

```output
integer 42
string rok
boolean true
```

Sum types are **closed**. The compiler knows every possible variant, so removing the `bool` arm from the `match` above is a compile error saying not all cases are handled. When you add a new variant, the compiler points you to every place that needs updating.

Sum types behave like sets. `i32 | str` and `str | i32` are the same type, and `i32 | i32` is just `i32`. Long sums can be named with `type`.

```rok
type Number = i32 | f64;

fn half(n: Number) -> f64 {
    match n {
        i: i32 => i as f64 / 2.0,
        f: f64 => f / 2.0,
    }
}

println(f"{half(7)} {half(1.5)}");
```

```output
3.5 0.75
```

## `none` and `T?`

`none` is the literal for "no value". Where other languages use `Option<T>` or `null`, rok uses the sum type `T | none`, written `T?` for short. There is no `Some(..)` wrapper, so you return the value as is.

```rok
fn find(xs: List<i32>, target: i32) -> usize? {
    for i in 0..xs.len() {
        if xs[i] == target { return i; }
    }
    none
}

let xs = [3, 1, 4];
println(f"{find(xs, 4):?}");
println(f"{find(xs, 7):?}");
```

```output
2
none
```

The standard library follows the same convention. Lookups that can fail, such as `List::get`, `Dict::get`, and `List::pop`, all return `T | none`.

> [!NOTE]
> Advanced: `T | none` is nearly free. For types with unused bit patterns, such as `bool`, pointers, and field-less `enum`s, `none` is stored in one of those patterns with no separate tag ([niche optimization](../spec/refs-lang/002-types/207-niche-optimization.html)). That is why `bool?` is 1 byte.

## Narrowing with `is`

`x is T` is a `bool` expression that tests whether the value is currently the `T` variant. When an `if` condition tests a local variable, the variable is **narrowed** to `T` inside that branch, so you can use `T`'s operations directly. The same holds for later conditions joined with `&&`.

```rok
let v: i32 | str = 7;
if v is i32 && v > 5 {
    println(f"big number: {v + 1}");
}

let w: i32 | str = "hello";
if w is str {
    println(f"length: {w.len()}");
}
println(f"{w is i32}");
```

```output
big number: 8
length: 5
false
```

Inside the narrowed branch you can still assign to the variable. A value of the narrowed type keeps the narrowing; a value of another member of the declared type ends it, and from there to the end of the branch the variable has its declared type again.

```rok
let v: i32 | str = 5;
if v is i32 {
    v = v + 1;              // still i32
    println(f"{v * 2}");
    v = "six";              // ends the narrowing: v is i32 | str again
    println(f"{v is str} {v:?}");
}
```

```output
12
true "six"
```

After `v = "six"`, writing `v + 1` would be a type error, because `+` is not defined on `i32 | str`.

## `if let` and `match`

To test and bind the value to a new name at the same time, use `if let name: T = expr`. In `match`, write `none` as a literal pattern.

```rok
fn lookup(id: i32) -> str? {
    if id == 1 { "kim" } else { none }
}

if let name: str = lookup(1) {
    println(f"found: {name}");
} else {
    println("not found");
}

for id in [1, 2] {
    match lookup(id) {
        name: str => println(f"{id} → {name}"),
        none => println(f"{id} → (none)"),
    }
}
```

```output
found: kim
1 → kim
2 → (none)
```

## `let else`: bail out early

`let pattern = expr else { ... };` binds the name if the pattern matches and otherwise runs the `else` block, which must leave the scope with `return`, `break`, `continue`, or `throw`. This lets you write "bail out early if missing" without extra indentation, and after it the name has the narrowed type.

```rok
fn first_even(xs: List<i32>) -> i32 | none {
    for x in xs {
        if x % 2 == 0 { return x; }
    }
    none
}

fn report(xs: List<i32>) -> str {
    let n: i32 = first_even(xs) else {
        return "no even number";
    };
    f"first even is {n}, half is {n / 2}"
}

println(report([1, 3, 8, 10]));
println(report([1, 3]));
```

```output
first even is 8, half is 4
no even number
```

## Defaults with `??` and chaining with `?.`

Writing `match` every time gets tedious. `a ?? b` returns `b` if `a` is `none`, and `a`'s value otherwise. `x?.field` and `x?.method()` evaluate to `none` when `x` is `none`, and only perform the access when there is a value. Combined, "take it if present, otherwise use a default" fits on one line.

```rok
struct User { name: str; nick: str?; }

let u: User? = User { name: "kim", nick: none };
let nobody: User? = none;

println(u?.name ?? "anonymous");
println(nobody?.name ?? "anonymous");
println(f"{u?.name?.len():?}");

let scores: Dict<str, i32> = {"kim": 90};
println(f"{scores.get("kim") ?? 0} {scores.get("lee") ?? 0}");
```

```output
kim
anonymous
3
90 0
```

`?.` short-circuits. If the left side is `none`, the method after it is never called.

## Postfix `?`: return early on none

When a function has to unwrap several `T | none` values in sequence, use postfix `?`. `e?` extracts the value of `e`, and if it is `none`, immediately returns `none` from the enclosing function. The function's return type must therefore include `none`.

```rok
struct User { name: str; nick: str?; }

fn nick_len(u: User?) -> usize? {
    let user = u?;
    let nick = user.nick?;
    nick.len()
}

println(f"{nick_len(User { name: "a", nick: "bob" }):?}");
println(f"{nick_len(User { name: "a", nick: none }):?}");
println(f"{nick_len(none):?}");
```

```output
3
none
none
```

`?` works only on `T | none`. It is not a way to propagate errors; errors are handled with `throws` / `try` ([error control flow](../spec/refs-lang/003-expressions/314-error-control-flow.html)).

> [!NOTE]
> Advanced: if a collection stores `none` itself (`List<i32?>`), the result of `get` is a two-level optional `(i32 | none)?` that distinguishes "no element" from "a stored none". `??`, `?.`, and `?` act only on the outer level. See [nested optionals](../spec/refs-lang/002-types/206-sum-types.html#nested-optional) for the full rules.

## Summary

- `A | B` is a closed sum type. Variant order and duplicates don't matter, and `match` must handle every variant.
- When a value may be absent, use `T | none`, or `T?` for short. Return the value as is, without wrapping.
- `x is T` both tests and narrows. Extract values with `if let x: T = ...`, `let x: T = ... else { return .. };`, and the `n: T` / `none` patterns in `match`.
- `??` supplies a default, `?.` skips the access on none, and postfix `?` returns from the function on none.

## Learn more

- [2.6. Sum types](../spec/refs-lang/002-types/206-sum-types.html), [2.1.5a. Optional type `T?`](../spec/refs-lang/002-types/201-type-expressions.html#215a-optional-types-t)
- [2.2.6. Narrowing](../spec/refs-lang/002-types/202-any-slot.html#226-narrowing), [3.10.5. `if let`](../spec/refs-lang/003-expressions/310-if-expression.html#3105-if-let-combining-with-pattern-matching), [3.16. `let-else`](../spec/refs-lang/003-expressions/316-let-else.html), [3.11. Patterns](../spec/refs-lang/003-expressions/311-patterns.html)
- [3.8. Optional chaining and null coalescing](../spec/refs-lang/003-expressions/308-optional-chaining-and-coalescing.html), [3.14.10. The `?` operator](../spec/refs-lang/003-expressions/314-error-control-flow.html#31410-the--operator---early-return-of-none)
- [2.7. Niche optimization](../spec/refs-lang/002-types/207-niche-optimization.html)
- Next chapter: [6. Structs, Enums, and Traits](06-structs-enums-traits.html)
