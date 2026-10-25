# 4. Control Flow

This chapter covers branching (`if`, `match`), looping (`while`, `for`, `loop`), and destructuring. The key point is that `if`, `match`, and `loop` in rok are all expressions that produce values.

## `if` is an expression

The condition of an `if` needs no parentheses, and the body is always written in braces. The last expression of each branch becomes the value of the whole `if`, so there is no separate ternary operator.

```rok
let temp = 23;
let feel = if temp >= 28 {
    "hot"
} else if temp >= 15 {
    "mild"
} else {
    "cold"
};
println(f"{temp} degrees: {feel}");
```

```output
23 degrees: mild
```

When used as a value, all branches must have the same type. To mix different types, annotate the receiving variable with a sum type such as `i32 | str`.

## `match` and patterns

`match` compares a value against patterns from top to bottom and runs the first arm that matches. You can use literals, several patterns joined with `|`, ranges (`4..=9`), bindings (`k @ pattern`), guards (`if condition`), and the wildcard `_`.

```rok
fn describe(n: i32) -> str {
    match n {
        0 => "zero",
        1 | 2 | 3 => "small",
        k @ 4..=9 => f"digit {k}",
        k if k < 0 => "negative",
        _ => "large",
    }
}

for n in [0, 2, 7, -5, 42] {
    println(f"{n}: {describe(n)}");
}
```

```output
0: zero
2: small
7: digit 7
-5: negative
42: large
```

The compiler checks that every case is handled. `i32` has too many values to list, so a final `_` is required. Tuples can also be destructured and matched with patterns.

```rok
fn locate(p: (i32, i32)) -> str {
    match p {
        (0, 0) => "origin",
        (x, 0) => f"on the x-axis at {x}",
        (0, y) => f"on the y-axis at {y}",
        (a, b) if a == b => "on the diagonal",
        _ => "somewhere else",
    }
}

println(locate((0, 0)));
println(locate((5, 0)));
println(locate((2, 2)));
println(locate((1, 9)));
```

```output
origin
on the x-axis at 5
on the diagonal
somewhere else
```

When the value can be one of several types (`i32 | str | none`), `match` splits on its type with patterns such as `n: i32`. Those patterns, along with `is`, `if let`, and `let else`, come with sum types in the [next chapter](05-sum-types.html).

## `while` and `for`

`while` repeats as long as the condition is true. `for` takes elements one at a time from a range, list, or iterator. `a..b` excludes `b`; `a..=b` includes it. `continue` moves to the next iteration, and `break` leaves the loop.

```rok
let i = 0;
while i < 3 {
    print(f"{i} ");
    i += 1;
}
println("");

for k in 0..10 {
    if k % 2 == 1 { continue; }
    if k > 6 { break; }
    print(f"{k} ");
}
println("");

for (idx, name) in ["ada", "grace"].iter().enumerate() {
    println(f"{idx}: {name}");
}
```

```output
0 1 2 
0 2 4 6 
0: ada
1: grace
```

`while let` repeats as long as the pattern matches. `pop()` returns `T | none`, which makes draining a list until it is empty simple.

```rok
let stack = [1, 2, 3];
while let top: i32 = stack.pop() {
    print(f"{top} ");
}
println("");
```

```output
3 2 1 
```

## `loop` and `break` with a value

`loop` repeats forever until a `break`. If you exit with `break value`, that value is the result of the `loop` expression. This suits computations whose result is determined at the moment they finish.

```rok
let n = 27;
let steps = 0;
let total = loop {
    if n == 1 { break steps; }
    n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
    steps += 1;
};
println(f"Collatz(27) reaches 1 in {total} steps");
```

```output
Collatz(27) reaches 1 in 111 steps
```

## Labels

To end an outer loop directly from a nested one, attach a `'name:` label and use `break 'name` or `continue 'name`. Labels on `loop`s and blocks can also carry a value out.

```rok
let pair = 'search: loop {
    for a in 1..10 {
        for b in a..10 {
            if a * b == 42 { break 'search (a, b); }
        }
    }
    break (0, 0);
};
println(f"{pair:?}");

let limit = 50;
let checked = 'check: {
    if limit > 100 { break 'check -1; }
    limit * 2
};
println(f"{checked}");
```

```output
(6, 7)
100
```

## Destructuring

The left side of `let` and `for` can be any pattern that always matches (an irrefutable pattern). Unpack nested tuples, collect the rest with `..`, or discard with `_`.

```rok
let ((a, b), c) = ((1, 2), 3);
let (first, ..rest) = (10, 20, 30, 40);
let (_, second) = ("skip", "keep");
println(f"{a} {b} {c} {first} {rest:?} {second}");

let points = [(1, 2), (3, 4)];
for (x, y) in points {
    println(f"({x}, {y}) -> {x + y}");
}
```

```output
1 2 3 10 (20, 30, 40) keep
(1, 2) -> 3
(3, 4) -> 7
```

## Summary

- `if`, `match`, `loop`, and blocks are expressions that produce values. Branches must have the same type.
- `match` supports literals, `|`, ranges, `@` bindings, and guards, and checks exhaustiveness.
- `while let` repeats while the pattern matches.
- `for x in a..b` / `a..=b` iterates over ranges; `.iter().enumerate()` iterates with indices.
- `break value` and `'label:` let you exit nested loops and return a result.

## Further reading

- [`if` expressions](../spec/refs-lang/003-expressions/310-if-expression.html), [`match` expressions](../spec/refs-lang/003-expressions/312-match-expression.html), [patterns](../spec/refs-lang/003-expressions/311-patterns.html)
- [Loop expressions](../spec/refs-lang/003-expressions/313-loop-expressions.html), [range expressions](../spec/refs-lang/003-expressions/307-range-expressions.html), [labeled blocks](../spec/refs-lang/003-expressions/309-block-expressions.html)
- [Destructuring assignment](../spec/refs-lang/003-expressions/315-destructuring.html), [`let-else`](../spec/refs-lang/003-expressions/316-let-else.html)
- [`is` and type tests on slots](../spec/refs-lang/002-types/202-any-slot.html), [sum types](../spec/refs-lang/002-types/206-sum-types.html)
- Next chapter: [5. Sum Types and none](05-sum-types.html)
