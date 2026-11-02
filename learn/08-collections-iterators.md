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

A `Slice` is read-only. A function that fills or reorders part of a list takes a `SliceMut<T>` instead, a writable view like Rust's `&mut [T]` (import it from `rok::core::collections`; it is not in the prelude). Where a `SliceMut` is expected, `xs` passes the whole list and `xs[a:b]` that range; nothing is copied, and writes go to the list.

```rok
use rok::core::collections::SliceMut;

fn zero(buf: SliceMut<i32>) { buf.fill(0); }

let ys = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = ys[1:4];
mid.sort();
mid[0] = 20;
zero(ys[4:]);
println(f"{ys:?}");
```

```output
[5, 20, 3, 4, 0]
```

A view does not survive a resize of its list (`push`, `insert`, `clear`, ...): using it after one is an error where the compiler can see it, and a panic where it cannot. Make the view again after the resize.

## Dict and Set

`{"key": value}` is a hash map, `Dict<K, V>`, and `{1, 2}` is a hash set, `Set<T>`. An empty `{}` is a block, so create empty collections with `Dict::new()` / `Set::new()`. `d[k]` panics if the key is missing; `d.get(k)` returns `V | none`. `k in d` tests for a key, `x in s` for a set member (`!in` for the opposite).

```rok
let ages: Dict<str, i32> = {"kim": 30, "lee": 25};
ages["park"] = 41;
ages.remove("lee");
println(f"{ages["kim"]} {ages.get("choi") ?? -1} {"park" in ages} {ages.len()}");

let counts: Dict<str, i32> = Dict::new();
for w in ["a", "b", "a", "a"] {
    counts[w, default: 0] += 1;
}
println(f"a={counts["a"]} b={counts["b"]}");

let s: Set<i32> = {1, 2, 3};
let t: Set<i32> = {2, 3, 4};
println(f"{2 in s} {s.intersection(t).len()} {s.union(t).len()}");
```

```output
30 -1 true 2
a=3 b=1
true 2 4
```

After `ages.remove("lee")`, a fourth line `println(f"{ages["lee"]}");` would panic, and the message says what to write instead:

```text
thread 'main' panicked at main.rok:4:12:
key "lee" not found; use `d.get(k)` to get `i32 | none`
```

> [!NOTE]
> `Dict` and `Set` iterate in insertion order. A key inserted again keeps its place, and a key removed and inserted again goes last. Hash values themselves change from run to run, so never rely on them.

## Insertion order and Deque

Because `Dict` and `Set` keep insertion order, there is no separate ordered type: `iter()` gives the `(key, value)` pairs in the order they went in. `Deque` is a queue you can push to and pop from at both ends.

```rok
let d: Dict<str, i32> = Dict::new();
d.insert("zebra", 1);
d.insert("apple", 2);
for (k, v) in d.iter() { println(f"{k}={v}"); }

let tags: Set<str> = Set::from(["b", "a", "b", "c"]);
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

let evens = xs.filter(|x| x % 2 == 0).map(|x| x * 10).to_list();
println(evens);
println(xs.sum(), xs.count(), xs.any(|x| x > 5));
println(xs.fold(0, |acc, x| acc + x * x));
println(f"{xs.iter().find(|x| x > 3):?} {xs.position(|x| x == 3):?} {xs.max():?}");

for (i, x) in xs.enumerate().skip(4) { println(f"{i}: {x}"); }

let names = ["a", "b", "c"];
let pairs = names.zip(xs.iter()).map(|(n, x)| f"{n}{x}").to_list();
println(pairs);

let back = xs.rev().take(2).to_list();
let uniq: Set<i32> = [1, 1, 2].iter().collect();
println(back, uniq.len());
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

```rok
let words = ["kiwi", "fig", "apple", "fig"];
println(words.sorted(), words.sorted_by_key(|w| w.len(), reverse: true));
println(words.counts().most_common(1), words.group_by(|w| w.len()));
println(words.enumerate(start: 1).map(|p| f"{p.0}.{p.1}").to_list().join(" "));
```

```output
["apple", "fig", "fig", "kiwi"] ["apple", "kiwi", "fig", "fig"]
[("fig", 2)] {4: ["kiwi"], 3: ["fig", "fig"], 5: ["apple"]}
1.kiwi 2.fig 3.apple 4.fig
```

A collection answers the iterator methods itself: `xs.sum()` is `xs.iter().sum()`, lazy as before (a `List`'s own methods such as `contains` and `windows` come first). `to_list()` gathers the items into a new `List`; `collect` builds whatever the receiving type is (`Set<i32>` above) and a `List` when nothing says. `sorted`, `sorted_by_key`, `counts` (Python's `Counter`, ranked with `most_common`), `group_by` and `enumerate(start:)` return new collections or count from where you say. `d[k, default: 0] += 1` stores `0` first when `k` is missing.

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

A generator can hold a resource while it yields. When a `for` loop over a generator it holds itself (a call such as `for r in rows()`) leaves early, by `break`, `return` or an error, the generator is finished on the spot: its `using` blocks ([chapter 10](10-errors.html#using-cleaning-up-resources)) and `finally` blocks run before the loop's next statement. A generator kept in a variable stays open, so a later loop can go on with it.

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

yield fn rows() -> i32 {
    using c = Conn { name: "db" };
    for i in 1..=5 { yield i; }
}

for r in rows() {
    if r == 3 { break; }
    println(f"row {r}");
}
println("done");
```

```output
row 1
row 2
close db
done
```

## Advanced: broadcast operators

You can skip this section on a first read. Dotted operators such as `.+`, `.*`, and `.>` are element-wise operations. They desugar to traits such as `DotAdd`, `DotMul`, and `DotGt`, so implementing those for your type makes the operators available right away.

```rok
struct Vector { xs: List<f64>; }

impl ops::DotAdd for Vector {
    type Output = Vector;
    fn dot_add(self, o: Vector) -> Vector {
        Vector { xs: self.xs.iter().zip(o.xs.iter()).map(|(a, b)| a + b).collect() }
    }
}

impl ops::DotMul<f64> for Vector {
    type Output = Vector;
    fn dot_mul(self, k: f64) -> Vector {
        Vector { xs: self.xs.iter().map(|x| x * k).collect() }
    }
}

let a = Vector { xs: [1.0, 2.0, 3.0] };
let b = Vector { xs: [10.0, 20.0, 30.0] };
let c = a .+ b .* 2.0;
println(f"{c.xs:?}");
```

```output
[21.0, 42.0, 63.0]
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
[11.0, 22.0, 33.0] [2.0, 4.0, 6.0] [1.0, 1.4142135623730951, 1.7320508075688772]
```

## Summary

- `List` provides `push` / `pop` / `insert` / `remove` / `sort` / `get`, `Dict` provides `d[k]` / `get` / `entry`, and `Set` provides set operations.
- Collections are reference handles. Call `clone()` when you need a copy.
- `xs[a:b:s]` slicing works like Python and never panics on out-of-range bounds (a step of 0 does). The result is a `Slice<T>` view; a `SliceMut<T>` writes the list.
- `Dict` and `Set` keep insertion order; for a double-ended queue, use `Deque`.
- Chain adapters after `iter()` and consume with `collect` / `sum` / `fold`. Implement just `next` and your own type becomes an iterator.
- `yield fn` and `yield { }` make iterators easy to write.

## Further reading

- [stdlib 2.9. Collections](../spec/refs-standard-libraries/200-core/209-collections.html), [Reference handles](../spec/refs-standard-libraries/200-core/209-collections.html#reference-handles)
- [3.6. Indexing and slicing](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)
- [stdlib 2.7. iter](../spec/refs-standard-libraries/200-core/207-iter.html), [4.11. Generators](../spec/refs-lang/004-functions/411-generators.html)
- [3.3. Broadcast operators](../spec/refs-lang/003-expressions/303-broadcast-operators.html), [stdlib 2.6. broadcast](../spec/refs-standard-libraries/200-core/206-broadcast.html)
- API: [`List`](../std/core/collections/list.html#struct.List), [`Dict`](../std/core/collections/dict.html#struct.Dict), [`Set`](../std/core/collections/set.html#struct.Set), [`Deque`](../std/core/collections/deque.html#struct.Deque), [`Slice`](../std/core/collections/slice.html#struct.Slice), [`Iterator`](../std/core/iter.html#trait.Iterator), [`GenIter`](../std/core/iter.html#struct.GenIter)
- Next chapter: [9. Strings and f-strings](09-strings.html)
