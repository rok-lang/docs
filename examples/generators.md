# Generators

Write the Fibonacci sequence, a prime stream, a hand-made `take_while` adapter and a recursive tree walk as lazy iterators with `yield fn`.

## Code

```rok
// An endless Fibonacci stream.
yield fn fibonacci() -> i64 {
    let a = 0i64;
    let b = 1i64;
    loop {
        yield a;
        let next = a + b;
        a = b;
        b = next;
    }
}

// Primes by trial division against the primes found so far.
yield fn primes() -> i64 {
    let found: List<i64> = [];
    let n = 2i64;
    loop {
        let is_prime = true;
        for p in found {
            if p * p > n { break; }
            if n % p == 0 { is_prime = false; break; }
        }
        if is_prime {
            found.push(n);
            yield n;
        }
        n += 1;
    }
}

// A generic adapter: pass items through until `pred` fails.
yield fn take_while<I: Iterator>(it: I, pred: fn(I::Item) -> bool) -> I::Item {
    for x in it {
        if !pred(x) { return; }
        yield x;
    }
}

// A nested list, walked depth-first by a recursive generator.
enum Tree {
    Leaf(str),
    Branch(List<Tree>),
}

yield fn leaves(t: Tree) -> str {
    match t {
        Tree::Leaf(name) => yield name,
        Tree::Branch(kids) => for k in kids {
            for name in leaves(k) { yield name; }
        },
    }
}

let first: List<i64> = fibonacci().take(12).collect();
println(f"fib: {first:?}");

let small: List<i64> = take_while(primes(), |p| p < 50).collect();
println(f"primes < 50: {small:?}");

let even_fib_sum = take_while(fibonacci(), |x| x < 4_000_000)
    .filter(|x| x % 2 == 0)
    .sum();
println(f"sum of even fib < 4M: {even_fib_sum}");

// The 1000th prime: nothing past it is ever computed.
println(f"1000th prime: {primes().nth(999) ?? -1}");

// An inline generator block.
let squares = yield {
    for i in 1i64..=5 { yield i * i; }
};
println(f"squares: {squares.collect::<List<i64>>():?}");

let tree = Tree::Branch([
    Tree::Leaf("a"),
    Tree::Branch([Tree::Leaf("b"), Tree::Branch([Tree::Leaf("c")]), Tree::Leaf("d")]),
    Tree::Leaf("e"),
]);
let names: List<str> = leaves(tree).collect();
println(f"leaves: {names:?}");
```

```output
fib: [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
primes < 50: [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
sum of even fib < 4M: 4613732
1000th prime: 7919
squares: [1, 4, 9, 16, 25]
leaves: ["a", "b", "c", "d", "e"]
```

## Walkthrough

- Calling a `yield fn` doesn't run its body right away; it returns an iterator, `GenIter<T>`. The `-> i64` is not a return value but the type of the yielded items ([4.11.3 named generator functions](../spec/refs-lang/004-functions/411-generators.html#4113-named-generator-functions)).
- `fibonacci()` and `primes()` are infinite generators. Their locals (`a`, `b`, `found`) are preserved between `yield`s, and the consumer pulls only as many items as it needs with `take`, `nth` and the like ([4.11.9 infinite generators](../spec/refs-lang/004-functions/411-generators.html#4119-infinite-generators)).
- Generators implement [`Iterator`](../std/core/iter.html#trait.Iterator), so they combine with adapters such as `filter`, `sum` and `collect`. A `take_while`, which the standard iterator lacks, is a single generic `yield fn` that ends iteration with `return;` ([Collections and iterators](../learn/08-collections-iterators.html), [Generics](../learn/07-generics.html)).
- `yield { ... }` is an inline generator block that builds an iterator right where an expression is expected ([4.11.4](../spec/refs-lang/004-functions/411-generators.html#4114-inline-yield-blocks-values)).
- `leaves` calls itself and re-yields the values of each subtree. Recursive generators are a common way to flatten nested structures ([Sum types](../learn/05-sum-types.html)).

> [!NOTE]
> Defining the tree as an enum that contains itself directly, like `Node(Tree, i64, Tree)`, currently makes the compiler stop with a stack overflow. This example keeps the children in a `List<Tree>` to express the same structure.
