# 8. 컬렉션과 반복자

프로그램은 대부분 여러 값을 모으고, 순회하고, 변환합니다. 이 장에서는 표준 컬렉션(`List`, `Dict`, `Set`과 그 친척), 슬라이싱, 그리고 이 모두를 하나로 묶는 `Iterator`를 다룹니다.

## List

`[1, 2, 3]`은 동적 배열 `List<T>`입니다. 인덱스는 0부터 시작하고, 음수 인덱스는 끝에서부터 셉니다. 범위를 벗어난 `xs[i]`는 panic을 일으킵니다. 그래서 없을 수도 있는 위치는 `T | none`을 반환하는 `get`으로 읽습니다.

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

`List`, `Dict`, `Set`은 **참조 핸들(reference handle)**입니다. `let ys = xs;`로 대입하거나 함수 인자로 넘기면 같은 컬렉션을 공유합니다. 독립된 복사본이 필요하면 `clone()`을 호출합니다.

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

## 슬라이싱

`xs[start:end:step]`은 Python과 같은 뜻입니다. 경계는 길이에 맞게 잘리므로(clamp) 슬라이싱은 범위 때문에 panic을 일으키지 않습니다. `List`를 슬라이싱하면 원래 버퍼를 가리키는 `Slice<T>` 뷰가 나옵니다. 독립된 리스트가 필요하면 `to_list()`로 복사합니다.

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

`Slice`는 읽기 전용입니다. 리스트의 일부를 채우거나 순서를 바꾸는 함수는 대신 `SliceMut<T>`를 받습니다. Rust의 `&mut [T]`와 같은, 쓸 수 있는 뷰입니다(프렐류드에 없으므로 `rok::core::collections`에서 가져옵니다). `SliceMut`를 기대하는 자리에서는 `xs`가 리스트 전체를, `xs[a:b]`가 그 범위를 넘깁니다. 아무것도 복사하지 않으며, 쓰기는 리스트에 반영됩니다.

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

리스트의 크기가 바뀌면(`push`, `insert`, `clear` 등) 그 뷰는 더 쓸 수 없습니다. 크기가 바뀐 뒤에 뷰를 쓰면, 컴파일러가 볼 수 있는 곳에서는 에러이고 볼 수 없는 곳에서는 panic입니다. 크기를 바꾼 뒤에는 뷰를 다시 만듭니다.

## Dict와 Set

`{"key": value}`는 해시 맵 `Dict<K, V>`이고, `{1, 2}`는 해시 집합 `Set<T>`입니다. 빈 `{}`는 블록이므로 빈 컬렉션은 `Dict::new()` / `Set::new()`로 만듭니다. `d[k]`는 키가 없으면 panic을 일으키고, `d.get(k)`는 `V | none`을 반환합니다. `k in d`는 키가 있는지, `x in s`는 집합에 드는지 검사합니다(반대는 `!in`).

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
println(f"{s & t:?} {s | t:?} {s - t:?} {s ^ t:?}");
println(f"{counts["z", default: 0]} {[1, 2] + [3]:?}");
```

```output
30 -1 true 2
a=3 b=1
true 2 4
{2, 3} {1, 2, 3, 4} {1} {1, 4}
0 [1, 2, 3]
```

`d[k, default: v]`는 키가 없으면 `v`를 읽습니다. `=`나 `+=`의 대상, 또는 메서드의 수신자로 쓰면 먼저 `v`를 넣습니다. 집합의 `&`, `|`, `-`, `^`는 교집합, 합집합, 차집합, 대칭차이고, `+`는 두 리스트를 이어 새 리스트를 만듭니다.

`ages.remove("lee")` 다음 넷째 줄에 `println(f"{ages["lee"]}");`를 쓰면 panic이 일어나고, 메시지가 대신 쓸 코드를 알려 줍니다.

```text
thread 'main' panicked at main.rok:4:12:
key "lee" not found; use `d.get(k)` to get `i32 | none`
```

> [!NOTE]
> `Dict`와 `Set`은 삽입 순서대로 순회합니다. 이미 있는 키를 다시 넣으면 자리가 그대로이고, 지웠다가 다시 넣은 키는 맨 뒤로 갑니다. 해시 값 자체는 실행할 때마다 달라지므로 거기에 기대면 안 됩니다.

## 삽입 순서와 Deque

`Dict`와 `Set`이 삽입 순서를 지키므로 순서를 지키는 타입은 따로 없습니다. `iter()`는 `(키, 값)` 쌍을 넣은 순서대로 줍니다. `Deque`는 양쪽 끝에서 넣고 뺄 수 있는 큐입니다.

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

## 반복자와 어댑터

`for x in collection`은 `Iterator`로 동작합니다. 컬렉션의 `iter()`가 반환한 반복자 위에 `map`, `filter`, `enumerate`, `zip`, `take`, `skip`, `rev`, `step_by`, `chain` 같은 **어댑터(adapter)**를 이어 붙일 수 있습니다. 어댑터는 미리 계산하지 않습니다. `collect`, `sum`, `fold`, `count`처럼 값을 실제로 소비하는 메서드를 호출해야 실행됩니다.

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
let uniq: Set<i32> = [1, 1, 2].collect();
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
println(words.enumerate(start: 1).map(|(i, w)| f"{i}.{w}").to_list().join(" "));
println([1, 2, 3].combinations(2).to_list(), [1, 2, 3].pairwise().to_list());
```

```output
["apple", "fig", "fig", "kiwi"] ["apple", "kiwi", "fig", "fig"]
[("fig", 2)] {4: ["kiwi"], 3: ["fig", "fig"], 5: ["apple"]}
1.kiwi 2.fig 3.apple 4.fig
[[1, 2], [1, 3], [2, 3]] [(1, 2), (2, 3)]
```

컬렉션은 반복자 메서드에 직접 답합니다. `xs.sum()`은 `xs.iter().sum()`이고 전과 같이 미리 계산하지 않습니다(`contains`, `windows` 같은 `List` 자신의 메서드가 먼저입니다). `to_list()`는 항목을 새 `List`에 모읍니다. `collect`는 받는 쪽의 타입(위의 `Set<i32>`)으로 만들고, 정하는 것이 없으면 `List`로 만듭니다. `sorted`, `sorted_by_key`, `counts`(Python의 `Counter`, 순위는 `most_common`), `group_by`, `enumerate(start:)`는 새 컬렉션을 돌려주거나 원하는 수부터 셉니다. `combinations(k)`, `permutations(k)`, `pairwise()`는 Python의 `itertools`처럼 미리 계산하지 않습니다. `min`과 `max`는 `PartialOrd`만 있으면 되므로 실수에도 쓸 수 있고, 실수의 `sum`은 정해진 순서로 둘씩 짝지어 더합니다.

## 반복자 직접 만들기: `Iterator`와 `yield`

`Iterator` 트레잇에 필요한 것은 연관 타입 `Item`과 메서드 `next(ref self) -> Item | none` 하나뿐입니다. 어댑터는 모두 기본 메서드로 딸려 옵니다.

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

상태를 직접 관리하기 번거로우면 **제너레이터(generator)**를 씁니다. `yield fn`은 본체가 `yield value;`로 값을 하나씩 내보내는 함수입니다. 호출하면 반복자(`GenIter<T>`)를 반환합니다. 끝없이 값을 내는 제너레이터여도 됩니다. 필요한 만큼만 `take`로 가져가면 됩니다. 표현식 자리에서는 `yield { ... }` 블록으로 바로 만들 수도 있습니다.

```rok
yield fn fib() -> i64 {
    let a = 0;
    let b = 1;
    loop {
        yield a;
        (a, b) = (b, a + b);
    }
}

let first = fib().take(10).to_list();
println(f"{first:?}");

let squares = yield { for i in 1..5 { yield i * i; } };
for v in squares { print(f"{v} "); }
println("");
```

```output
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
1 4 9 16
```

생성기는 값을 내놓는 동안 자원을 쥐고 있을 수 있습니다. 루프가 직접 쥔 생성기(`for r in rows()`처럼 호출로 받은 것)는 `for` 루프가 `break`, `return`, 에러로 일찍 끝나면 그 자리에서 마무리됩니다. 생성기의 `using` 블록([10장](10-errors.html#using-자원-정리))과 `finally` 블록이 루프 다음 문장보다 먼저 실행됩니다. 변수에 담아 둔 생성기는 열린 채로 남으므로 다음 루프에서 이어서 쓸 수 있습니다.

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

## 심화: 브로드캐스트 연산자

처음 읽을 때는 이 절을 건너뛰어도 됩니다. `.+`, `.*`, `.>`처럼 점이 붙은 연산자는 원소별 연산입니다. 이 연산자는 `DotAdd`, `DotMul`, `DotGt` 같은 트레잇 호출로 풀립니다. 그래서 내 타입에 이 트레잇을 구현하면 바로 연산자를 쓸 수 있습니다.

```rok
struct Vector { xs: List<f64>; }

impl ops::DotAdd for Vector {
    type Output = Vector;
    fn dot_add(self, o: Vector) -> Vector {
        Vector { xs: self.xs.zip(o.xs.iter()).map(|(a, b)| a + b).to_list() }
    }
}

impl ops::DotMul<f64> for Vector {
    type Output = Vector;
    fn dot_mul(self, k: f64) -> Vector {
        Vector { xs: self.xs.map(|x| x * k).to_list() }
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

사양은 `List`, 고정 크기 배열, `Array`에 이 연산자를 표준으로 정의합니다. 함수 뒤에 점을 붙여 모든 원소에 적용하는 `f.(xs)`도 함께 정의합니다. 이 연산자를 이어 쓴 식은 루프 하나로 돌고, `ys .= expr`, `ys .+= xs` 같은 `.op=` 형태는 새 리스트 없이 기존 리스트에 씁니다([21장](21-arrays-and-numerics.html)).

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

## 정리

- `List`에는 `push` / `pop` / `insert` / `remove` / `sort` / `get`이, `Dict`에는 `d[k]` / `get` / `d[k, default: v]`가, `Set`에는 집합 연산(`&`, `|`, `-`, `^`)이 있습니다.
- 컬렉션은 참조 핸들입니다. 복사본이 필요하면 `clone()`을 호출합니다.
- `xs[a:b:s]` 슬라이싱은 Python처럼 동작하고 범위를 벗어나도 panic을 일으키지 않습니다(간격이 0이면 panic입니다). 결과는 `Slice<T>` 뷰이고, 리스트에 쓰려면 `SliceMut<T>`를 씁니다.
- `Dict`와 `Set`은 삽입 순서를 지킵니다. 양방향 큐가 필요하면 `Deque`를 씁니다.
- 컬렉션이나 `iter()` 뒤에 어댑터를 이어 붙이고 `to_list` / `collect` / `sum` / `fold`로 소비합니다. `next` 하나만 구현하면 내 타입이 반복자가 됩니다.
- `yield fn`과 `yield { }`를 쓰면 반복자를 짧게 작성할 수 있습니다.

## 더 읽을거리

- [stdlib 2.9. 컬렉션](../spec/refs-standard-libraries/200-core/209-collections.html), [참조 핸들](../spec/refs-standard-libraries/200-core/209-collections.html#참조-핸들)
- [3.6. 인덱싱과 슬라이싱](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)
- [stdlib 2.7. iter](../spec/refs-standard-libraries/200-core/207-iter.html), [4.11. 제너레이터](../spec/refs-lang/004-functions/411-generators.html)
- [3.3. 브로드캐스트 연산자](../spec/refs-lang/003-expressions/303-broadcast-operators.html), [stdlib 2.6. broadcast](../spec/refs-standard-libraries/200-core/206-broadcast.html)
- API: [`List`](../../std/core/collections/list.html#struct.List), [`Dict`](../../std/core/collections/dict.html#struct.Dict), [`Set`](../../std/core/collections/set.html#struct.Set), [`Deque`](../../std/core/collections/deque.html#struct.Deque), [`Slice`](../../std/core/collections/slice.html#struct.Slice), [`Iterator`](../../std/core/iter.html#trait.Iterator), [`GenIter`](../../std/core/iter.html#struct.GenIter)
- 다음 장: [9. 문자열과 f-string](09-strings.html)
