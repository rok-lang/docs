# 8. Collections and Iterators

Most programs collect, traverse, and transform many values. This chapter covers the standard collections (`List`, `Dict`, `Set`, and their relatives), slicing, and the `Iterator` that ties them together.

## List

`[1, 2, 3]` is a dynamic array, `List<T>`. Indices start at 0, and negative indices count from the end. An out-of-range `xs[i]` panics, so read positions that may not exist with `get`, which returns `T | none`.

```rok
let xs = [5, 3, 8, 1];
xs.push(7);
println(f"{xs:?} len={xs.len()}");
println(f"{xs[0]} {xs[-1]} {xs.get(10):?}");

xs.sort();
let top = xs.pop();
println(f"{xs:?} popped={top:?}");

xs.insert(0, 0);
xs.remove(1);
println(f"{xs:?} first={xs.first():?}");
```

```output
[5, 3, 8, 1, 7] len=5
5 7 none
[1, 3, 5, 7] popped=8
[0, 3, 5, 7] first=0
```

`List`, `Dict`, and `Set` are **reference handles**. `let ys = xs;` and passing as a function argument share the same collection; call `clone()` when you need an independent copy.

```rok
let xs = [1, 2];
let shared = xs;
let copy = xs.clone();
shared.push(3);
copy.push(99);
println(f"{xs:?} {copy:?}");
```

```output
[1, 2, 3] [1, 2, 99]
```

## Slicing

`xs[start:end:step]` means the same as in Python. Bounds are clamped to the length, so slicing never panics on range. Slicing a `List` yields a `Slice<T>` view into the original buffer; copy it with `to_list()` when you need an independent list.

```rok
let xs = [10, 20, 30, 40, 50];
println(f"{xs[1:3]:?} {xs[:2]:?} {xs[-2:]:?}");
println(f"{xs[::2]:?} {xs[::-1]:?} {xs[3:100]:?}");

let part = xs[1:4];
println(f"{part.len()} {part[0]} {part.to_list():?}");
```

```output
[20, 30] [10, 20] [40, 50]
[10, 30, 50] [50, 40, 30, 20, 10] [40, 50]
3 20 [20, 30, 40]
```

## Dict and Set

`{"key": value}` is a hash map, `Dict<K, V>`, and `{1, 2}` is a hash set, `Set<T>`. An empty `{}` is a block, so create empty collections with `Dict::new()` / `Set::new()`. `d[k]` panics if the key is missing; `d.get(k)` returns `V | none`.

```rok
let ages: Dict<str, i32> = {"kim": 30, "lee": 25};
ages["park"] = 41;
ages.remove("lee");
println(f"{ages["kim"]} {ages.get("choi") ?? -1} {ages.contains_key("park")} {ages.len()}");

let counts: Dict<str, i32> = Dict::new();
for w in ["a", "b", "a", "a"] {
    counts.entry(w).and_modify(|c| c + 1).or_insert(1);
}
println(f"a={counts["a"]} b={counts["b"]}");

let s: Set<i32> = {1, 2, 3};
let t: Set<i32> = {2, 3, 4};
println(f"{s.contains(2)} {s.intersection(t).len()} {s.union(t).len()}");
```

```output
30 -1 true 2
a=3 b=1
true 2 4
```

> [!WARNING]
> The iteration order of `Dict` and `Set` is unspecified. When order matters, use `OrderedDict` / `OrderedSet`, which preserve insertion order.

## OrderedDict, OrderedSet, Deque

`OrderedDict` and `OrderedSet` iterate in insertion order. `Deque` is a queue you can push to and pop from at both ends.

```rok
let od: OrderedDict<str, i32> = OrderedDict::new();
od.insert("zebra", 1);
od.insert("apple", 2);
for (k, v) in od.entries() { println(f"{k}={v}"); }

let tags = OrderedSet::from(["b", "a", "b", "c"]);
println(f"{tags:?}");

let q: Deque<i32> = Deque::new();
q.push_back(1);
q.push_back(2);
q.push_front(0);
println(f"{q:?}");
let front = q.pop_front();
let back = q.pop_back();
println(f"{front:?} {back:?} {q:?}");
```

```output
zebra=1
apple=2
{"b", "a", "c"}
[0, 1, 2]
0 2 [1]
```

## Iterators and adapters

`for x in collection` drives an `Iterator`. On the iterator returned by a collection's `iter()`, you can chain **adapters** such as `map`, `filter`, `enumerate`, `zip`, `take`, `skip`, `rev`, `step_by`, and `chain`. Adapters are lazy: they run only when a method that actually consumes the values, such as `collect`, `sum`, `fold`, or `count`, is called.

```rok
let xs = [1, 2, 3, 4, 5, 6];

let evens: List<i32> = xs.iter().filter(|x| x % 2 == 0).map(|x| x * 10).collect();
println(f"{evens:?}");
println(f"{xs.iter().sum()} {xs.iter().count()} {xs.iter().any(|x| x > 5)}");
println(f"{xs.iter().fold(0, |acc, x| acc + x * x)}");
println(f"{xs.iter().find(|x| x > 3):?} {xs.iter().position(|x| x == 3):?} {xs.iter().max():?}");

for (i, x) in xs.iter().enumerate().skip(4) { println(f"{i}: {x}"); }

let names = ["a", "b", "c"];
let pairs: List<str> = names.iter().zip(xs.iter()).map(|p| f"{p.0}{p.1}").collect();
println(f"{pairs:?}");

let back: List<i32> = xs.iter().rev().take(2).collect();
let uniq: Set<i32> = [1, 1, 2].iter().collect();
println(f"{back:?} {uniq.len()}");
```

```output
[20, 40, 60]
21 6 true
91
4 2 6
4: 5
5: 6
["a1", "b2", "c3"]
[6, 5] 2
```

`collect` builds whatever the receiving type is (`List<i32>`, `Set<i32>`, ...), which is why the `let` carries a type annotation.

## Writing your own iterators: `Iterator` and `yield`

The `Iterator` trait requires only the associated type `Item` and one method, `next(ref self) -> Item | none`. All the adapters come along as default methods.

```rok
struct Countdown { n: i32; }

impl Iterator for Countdown {
    type Item = i32;
    fn next(ref self) -> i32 | none {
        if self.n == 0 { return none; }
        self.n -= 1;
        self.n + 1
    }
}

let c: List<i32> = Countdown { n: 3 }.collect();
println(f"{c:?}");
for v in (Countdown { n: 5 }).filter(|v| v % 2 == 1) { println(f"{v}"); }
```

```output
[3, 2, 1]
5
3
1
```

When managing state by hand gets tedious, use a **generator**. A `yield fn` is a function whose body emits values one at a time with `yield value;`, and calling it returns an iterator (`GenIter<T>`). Even an infinite generator is fine: take only as many values as you need with `take`. In expression position, you can also create one directly with a `yield { ... }` block.

```rok
yield fn fib() -> i64 {
    let a = 0i64;
    let b = 1i64;
    loop {
        yield a;
        let t = a + b;
        a = b;
        b = t;
    }
}

let first: List<i64> = fib().take(10).collect();
println(f"{first:?}");

let squares = yield { for i in 1..5 { yield i * i; } };
for v in squares { print(f"{v} "); }
println("");
```

```output
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
1 4 9 16
```

## Advanced: broadcast operators

You can skip this section on a first read. Dotted operators such as `.+`, `.*`, and `.>` are element-wise operations. They desugar to traits such as `DotAdd`, `DotMul`, and `DotGt`, so implementing those for your type makes the operators available right away.

```rok
struct Vec { xs: List<f64>; }

impl DotAdd for Vec {
    type Output = Vec;
    fn dot_add(self, o: Vec) -> Vec {
        Vec { xs: self.xs.iter().zip(o.xs.iter()).map(|p| p.0 + p.1).collect() }
    }
}

impl DotMul<f64> for Vec {
    type Output = Vec;
    fn dot_mul(self, k: f64) -> Vec {
        Vec { xs: self.xs.iter().map(|x| x * k).collect() }
    }
}

let a = Vec { xs: [1.0, 2.0, 3.0] };
let b = Vec { xs: [10.0, 20.0, 30.0] };
let c = a .+ b .* 2.0;
println(f"{c.xs:?}");
```

```output
[21, 42, 63]
```

The spec defines these operators for `List` and fixed-size arrays as standard, along with `f.(xs)`, which applies a function to every element by adding a dot after it.

```rok
let a = [1.0, 2.0, 3.0];
let b = [10.0, 20.0, 30.0];
let c = a .+ b;              // [11.0, 22.0, 33.0]
let d = a .* 2.0;            // [2.0, 4.0, 6.0]
let r = f64::sqrt.(a);       // [1.0, 1.414..., 1.732...]
println(f"{c:?} {d:?} {r:?}");
```

```output
[11, 22, 33] [2, 4, 6] [1, 1.4142135623730951, 1.7320508075688772]
```

## Summary

- `List` provides `push` / `pop` / `insert` / `remove` / `sort` / `get`, `Dict` provides `d[k]` / `get` / `entry`, and `Set` provides set operations.
- Collections are reference handles. Call `clone()` when you need a copy.
- `xs[a:b:s]` slicing works like Python and never panics. The result is a `Slice<T>` view.
- For ordering, use `OrderedDict` / `OrderedSet`; for a double-ended queue, `Deque`.
- Chain adapters after `iter()` and consume with `collect` / `sum` / `fold`. Implement just `next` and your own type becomes an iterator.
- `yield fn` and `yield { }` make iterators easy to write.

## Learn more

- [stdlib 2.9. Collections](../spec/refs-standard-libraries/200-core/209-collections.html), [Reference handles](../spec/refs-standard-libraries/200-core/209-collections.html#reference-handles)
- [3.6. Indexing and slicing](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)
- [stdlib 2.7. iter](../spec/refs-standard-libraries/200-core/207-iter.html), [4.11. Generators](../spec/refs-lang/004-functions/411-generators.html)
- [3.3. Broadcast operators](../spec/refs-lang/003-expressions/303-broadcast-operators.html), [stdlib 2.6. broadcast](../spec/refs-standard-libraries/200-core/206-broadcast.html)
- API: [`List`](../std/core/collections/list.html#struct.List), [`Dict`](../std/core/collections/dict.html#struct.Dict), [`Set`](../std/core/collections/set.html#struct.Set), [`OrderedDict`](../std/core/collections/ordered_dict.html#struct.OrderedDict), [`OrderedSet`](../std/core/collections/ordered_set.html#struct.OrderedSet), [`Deque`](../std/core/collections/deque.html#struct.Deque), [`Slice`](../std/core/collections/slice.html#struct.Slice), [`Iterator`](../std/core/iter.html#trait.Iterator), [`GenIter`](../std/core/iter.html#struct.GenIter)
