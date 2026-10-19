# 3. Functions

This chapter covers declaring and calling functions, default values and named arguments, and closures you can pass around as values. In rok, a function body is also a block expression, so its last expression is the return value.

## Declarations and return values

Declare a function with `fn name(param: Type, ...) -> ReturnType { body }`. The last expression of the body (one without a semicolon) is the return value; use `return` only to exit early.

```rok
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn sign(n: i32) -> str {
    if n < 0 {
        return "negative";
    }
    if n == 0 { "zero" } else { "positive" }
}

fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

println(f"{add(2, 3)} {sign(-4)} {sign(0)}");
let (q, r) = divmod(17, 5);
println(f"17 = 5 * {q} + {r}");
```

```output
5 negative zero
17 = 5 * 3 + 2
```

To return several values, group them in a tuple and destructure it at the call site.

Return and parameter types can be omitted. An omitted return type is inferred from the last expression (`()` if it ends with `;`). An untyped parameter makes the function generic, compiled separately for each argument type it is called with.

```rok
fn double(x: i32) { x * 2 }   // return type inferred: i32
fn id(x) { x }                // generic: fn id<T>(x: T) -> T
println(f"{double(4)} {id("hi")}");
```

```output
8 hi
```

A block `{ ... }` is an expression outside function bodies too. It is handy for keeping temporaries inside the block and taking out only the result.

```rok
let area = {
    let w = 3;
    let h = 4;
    w * h
};
println(f"{area}");
```

```output
12
```

## Default values and named arguments

A parameter can have a default value with `= value`, and a call can pass arguments by name with `name: value`. Positional arguments come first, named arguments after.

```rok
fn greet(name: str, greeting: str = "Hello", punct: str = "!") -> str {
    f"{greeting}, {name}{punct}"
}

println(greet("rok"));
println(greet("rok", punct: "?"));
println(greet(name: "Ada", greeting: "Hi"));
```

```output
Hello, rok!
Hello, rok?
Hi, Ada!
```

Default values must be known at compile time, such as literals or `const`s. A variable number of arguments is taken with `...T`; inside the body it is a read-only slice.

```rok
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {
        total += x;
    }
    total
}

println(f"{sum()} {sum(1, 2, 3)}");
```

```output
0 6
```

## Closures

A closure is an anonymous function value, written `|params| expr` or `|params| { block }`. Parameter types are inferred from context; without context, write them out.

```rok
let double = |x: i32| x * 2;
let describe = |x: i32| -> str {
    let half = x / 2;
    f"{x} is {half} * 2"
};
println(f"{double(21)}");
println(describe(10));
```

```output
42
10 is 5 * 2
```

Closures capture surrounding variables **by copying their values**. If an outer variable changes after the closure is created, the closure still remembers the value from creation time. To modify the outer variable itself, request a by-reference capture with a `[ref name]` capture list.

```rok
let count = 0;
let snapshot = || count + 1;
count = 10;
println(f"snapshot: {snapshot()}");

let inc = [ref count] || { count += 1; };
inc();
inc();
println(f"count: {count}");
```

```output
snapshot: 1
count: 12
```

If you try to assign to a variable captured by value, the compiler tells you to add `[ref count]`. Heap objects such as `List` are different: the variable is a reference to the object, so even a by-value capture can mutate the object itself, for example with `push`.

## Function types

Functions and closures are values of type `fn(ParamTypes) -> ReturnType`. You can store them in variables, pass them as arguments, return them, and put them in lists.

```rok
fn apply(f: fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x| x + n
}

let add5 = make_adder(5);
println(f"{apply(add5, 10)}");

let ops: List<fn(i32, i32) -> i32> = [|a, b| a + b, |a, b| a * b];
for op in ops {
    println(f"{op(6, 7)}");
}
```

```output
15
13
42
```

The closure returned by `make_adder` stays alive with `n` captured.

## Trailing closures

If the last argument is a closure, it can be moved outside the parentheses: `f(args) |x| { ... }` with parameters, `f(args) { ... }` without. Patterns such as iteration and resource management then read like built-in syntax.

```rok
fn repeat(times: i32, body: fn(i32)) {
    for i in 0..times {
        body(i);
    }
}

fn measure(label: str, body: fn() -> i32) -> i32 {
    let r = body();
    println(f"{label}: {r}");
    r
}

repeat(3) |i| {
    println(f"tick {i}");
};
let total = measure("sum") { 1 + 2 + 3 };
let evens: List<i32> = [1, 2, 3, 4].iter().filter |x| { x % 2 == 0 }.collect();
println(f"{total} {evens:?}");
```

```output
tick 0
tick 1
tick 2
sum: 6
6 [2, 4]
```

## A preview of methods

Functions inside an `impl` block are methods. If the first parameter is `self`, call it as `value.method()`; without `self`, it is a static method called as `Type::function()`. To modify fields inside a method, take `ref self`. Structs and methods are covered in detail in a later chapter.

```rok
struct Counter { n: i32 }

impl Counter {
    fn new() -> Counter { Counter { n: 0 } }
    fn bump(ref self) { self.n += 1; }
    fn get(self) -> i32 { self.n }
}

let c = Counter::new();
c.bump();
c.bump();
println(f"{c.get()}");
```

```output
2
```

## Nested functions and the entry point

You can declare a function inside a function. A nested function is visible only within its block and, unlike a closure, does **not** capture outer variables. Use it to hide helpers that need no outer state.

```rok
fn hypot_sq(a: i32, b: i32) -> i32 {
    fn sq(x: i32) -> i32 { x * x }
    sq(a) + sq(b)
}
println(f"{hypot_sq(3, 4)}");
```

```output
25
```

A program starts at the top-level statements of `src/main.rok` (or a standalone file) or at `fn main()`. Top-level statements become the body of an implicit `main` generated by the compiler, so top-level `let` variables are **local variables** and are not visible inside other function bodies. If a function needs such a value, pass it as an argument or capture it in a closure.

```rok
let base = 100;
let shifted = |x: i32| x + base;

fn scale(x: i32, factor: i32) -> i32 {
    x * factor
}

println(f"{shifted(1)} {scale(base, 3)}");
```

```output
101 300
```

## Summary

- The last expression of a function body is its return value. Use `return` to exit early.
- Parameters can have default values, and calls can name arguments with `name: value`. `...T` takes variadic arguments.
- A closure `|x| expr` captures by copying values by default; `[ref x]` requests a by-reference capture.
- A function value has type `fn(A) -> R`. A closure in the last argument position can be written in trailing form.
- A top-level `let` is a local of the implicit `main`, so other functions cannot see it.

The next chapter, [4. Control Flow](04-control-flow.html), covers `if`, `match`, and loops.

## Further reading

- [Function declarations](../spec/refs-lang/004-functions/401-function-declaration.html), [parameters](../spec/refs-lang/004-functions/402-parameters.html), [return](../spec/refs-lang/004-functions/403-return.html)
- [Function types](../spec/refs-lang/004-functions/404-function-type.html), [closures](../spec/refs-lang/004-functions/405-closure.html), [capture semantics](../spec/refs-lang/004-functions/406-capture-semantics.html)
- [Trailing closures](../spec/refs-lang/004-functions/408-trailing-closure.html), [methods](../spec/refs-lang/004-functions/409-method.html), [nested functions](../spec/refs-lang/004-functions/410-nested-functions.html)
- [Entry point](../spec/refs-lang/004-functions/416-entry-point.html), [block expressions](../spec/refs-lang/003-expressions/309-block-expressions.html)
