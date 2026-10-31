# 제너레이터

피보나치 수열, 소수 스트림, 직접 만든 `take_while` 어댑터, 재귀 트리 순회를 `yield fn`을 써서 지연 반복자로 만듭니다.

## 코드

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

## 설명

- `yield fn`을 호출해도 본체가 바로 실행되지는 않습니다. 호출은 반복자 `GenIter<T>`를 돌려줍니다. `-> i64`는 반환값이 아니라 `yield`로 내보내는 항목의 타입입니다([4.11.3 명명된 제너레이터 함수](../spec/refs-lang/004-functions/411-generators.html#4113-명명된-제너레이터-함수)).
- `fibonacci()`와 `primes()`는 무한 제너레이터입니다. 지역 변수(`a`, `b`, `found`)는 `yield` 사이에 보존됩니다. 소비하는 쪽은 `take`, `nth` 등으로 필요한 만큼만 항목을 꺼냅니다([4.11.9 무한 제너레이터](../spec/refs-lang/004-functions/411-generators.html#4119-무한-제너레이터)).
- 제너레이터는 [`Iterator`](../../std/core/iter.html#trait.Iterator)를 구현하므로 `filter`, `sum`, `collect` 같은 어댑터와 조합할 수 있습니다. 표준 반복자에 없는 `take_while`은 `return;`으로 반복을 끝내는 제네릭 `yield fn` 하나로 만듭니다([컬렉션과 반복자](../learn/08-collections-iterators.html), [제네릭](../learn/07-generics.html)).
- `yield { ... }`는 표현식 자리에서 바로 반복자를 만드는 인라인 제너레이터 블록입니다([4.11.4](../spec/refs-lang/004-functions/411-generators.html#4114-인라인-yield-블록-값)).
- `leaves`는 자기 자신을 호출해서 각 하위 트리의 값을 다시 `yield`로 내보냅니다. 재귀 제너레이터는 중첩된 구조를 평평하게 펼칠 때 흔히 쓰는 방법입니다([합 타입](../learn/05-sum-types.html)).

> [!NOTE]
> `Node(Tree, i64, Tree)`처럼 자기 자신을 직접 담는 enum은 크기가 무한하므로 컴파일러가 거부합니다(``recursive type `Tree` has infinite size``). 그래서 이 예제는 자식을 참조 타입인 `List<Tree>`에 담아 같은 구조를 나타냅니다.
