# 5. 합 타입과 none

값이 "이것 아니면 저것"일 때 Rok은 `A | B`처럼 타입을 `|`로 잇습니다. 이 장에서는 합 타입(sum type)과 함께, 가장 흔한 합 타입인 `T | none`("값이 없을 수 있음")을 다루는 도구를 설명합니다.

## 합 타입 `A | B`

합 타입의 값은 나열된 항 가운데 정확히 하나에 속합니다. 값에는 어느 항인지 기록한 태그가 붙어 있으므로 `match`에서 타입 검사 패턴 `name: Type`으로 꺼낼 수 있습니다.

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

합 타입은 **닫혀** 있습니다. 컴파일러는 가능한 항을 모두 압니다. 그래서 위 `match`에서 `bool` 가지를 지우면 모든 경우를 처리하지 않았다는 컴파일 에러가 납니다. 새 항을 추가하면 고쳐야 할 곳을 컴파일러가 모두 짚어 줍니다.

합 타입은 집합처럼 동작합니다. `i32 | str`과 `str | i32`는 같은 타입이고, `i32 | i32`는 그냥 `i32`입니다. 긴 합에는 `type`으로 이름을 붙일 수 있습니다.

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

## `none`과 `T?`

`none`은 "값 없음"을 나타내는 리터럴입니다. 다른 언어에서 `Option<T>`나 `null`을 쓰는 자리에 Rok은 합 타입 `T | none`을 쓰고, 줄여서 `T?`로 적습니다. `Some(..)` 같은 감싸개가 없으므로 값을 그대로 반환하면 됩니다.

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

표준 라이브러리도 같은 관례를 따릅니다. `List::get`, `Dict::get`, `List::pop`처럼 실패할 수 있는 조회는 모두 `T | none`을 반환합니다.

> [!NOTE]
> 심화: `T | none`은 비용이 거의 없습니다. `bool`, 포인터, 필드 없는 `enum`처럼 쓰지 않는 비트 패턴이 있는 타입은 별도 태그 없이 그 패턴 하나에 `none`을 저장합니다([niche 최적화](../spec/refs-lang/002-types/207-niche-optimization.html)). 그래서 `bool?` 타입은 1바이트입니다.

## `is`로 좁히기

`x is T`는 값이 지금 `T` 항인지 검사하는 `bool` 식입니다. `if` 조건에서 지역 변수를 검사하면 그 분기 안에서 변수가 `T`로 **좁혀집니다**. 그래서 `T`의 연산을 바로 쓸 수 있습니다. `&&`로 이은 뒤쪽 조건에서도 마찬가지입니다.

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

좁혀진 분기 안에서도 변수에 대입할 수 있습니다. 좁혀진 타입의 값을 대입하면 좁히기가 유지됩니다. 선언된 타입의 다른 항 값을 대입하면 좁히기가 끝나고, 그때부터 분기 끝까지 변수는 다시 선언된 타입이 됩니다.

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

`v = "six"` 다음에 `v + 1`을 쓰면 타입 에러입니다. `+`는 `i32 | str`에 정의되어 있지 않기 때문입니다.

## `if let`과 `match`

검사하면서 값을 새 이름에 바인딩하려면 `if let name: T = expr`을 씁니다. `match`에서는 `none`을 리터럴 패턴으로 씁니다.

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

## `let else`: 일찍 빠져나가기

`let pattern = expr else { ... };`는 패턴이 맞으면 이름을 바인딩하고, 맞지 않으면 `else` 블록을 실행합니다. `else` 블록은 `return`, `break`, `continue`, `throw`로 스코프를 떠나야 합니다. 그래서 "없으면 일찍 빠져나가기"를 들여쓰기 없이 쓸 수 있습니다. 그 뒤에서 이름은 좁혀진 타입이 됩니다.

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

## `??`로 기본값, `?.`로 체이닝

매번 `match`를 쓰면 번거롭습니다. `a ?? b`는 `a`가 `none`이면 `b`를, 아니면 `a`의 값을 돌려줍니다. `x?.field`와 `x?.method()`는 `x`가 `none`이면 `none`으로 평가됩니다. 값이 있을 때만 필드나 메서드에 접근합니다. 둘을 합치면 "있으면 쓰고, 없으면 기본값"을 한 줄로 쓸 수 있습니다.

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

`?.`는 단축 평가를 합니다. 왼쪽이 `none`이면 그 뒤의 메서드는 호출되지 않습니다.

## 후위 `?`: none이면 일찍 반환

함수가 `T | none` 값 여러 개를 차례로 풀어야 할 때는 후위 `?`를 씁니다. `e?`는 `e`의 값을 꺼내고, `none`이면 감싼 함수에서 즉시 `none`을 반환합니다. 그러므로 함수의 반환 타입에 `none`이 들어 있어야 합니다.

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

`?`는 `T | none`에만 쓸 수 있고, 에러를 전파하는 데는 쓰지 않습니다. 에러는 `throws` / `try`로 처리합니다([에러 제어 흐름](../spec/refs-lang/003-expressions/314-error-control-flow.html)).

> [!NOTE]
> 심화: 컬렉션에 `none` 자체를 저장하면(`List<i32?>`) `get`의 결과는 두 층짜리 optional `(i32 | none)?`입니다. 바깥 층은 "원소 없음"을, 안쪽 층은 "저장된 none"을 나타냅니다. `??`, `?.`, `?`는 바깥 층에만 작용합니다. 전체 규칙은 [중첩 optional](../spec/refs-lang/002-types/206-sum-types.html#중첩-optional)에 있습니다.

## 정리

- `A | B`는 닫힌 합 타입입니다. 항의 순서와 중복은 상관없고, `match`는 모든 항을 처리해야 합니다.
- 값이 없을 수 있으면 `T | none`, 줄여서 `T?`를 씁니다. 값은 감싸지 않고 그대로 반환합니다.
- `x is T`는 검사와 좁히기를 함께 합니다. 값은 `if let x: T = ...`, `let x: T = ... else { return .. };`, `match`의 `n: T` / `none` 패턴으로 꺼냅니다.
- `??`는 기본값을 주고, `?.`는 none이면 접근을 건너뛰고, 후위 `?`는 none이면 함수에서 반환합니다.

## 더 읽을거리

- [2.6. 합 타입](../spec/refs-lang/002-types/206-sum-types.html), [2.1.5a. optional 타입 `T?`](../spec/refs-lang/002-types/201-type-expressions.html#215a-optional-타입t)
- [2.2.6. 좁히기](../spec/refs-lang/002-types/202-any-slot.html#226-좁히기narrowing), [3.10.5. `if let`](../spec/refs-lang/003-expressions/310-if-expression.html#3105-if-let-패턴-매칭과-결합), [3.16. `let-else`](../spec/refs-lang/003-expressions/316-let-else.html), [3.11. 패턴](../spec/refs-lang/003-expressions/311-patterns.html)
- [3.8. 옵셔널 체이닝과 null 합체](../spec/refs-lang/003-expressions/308-optional-chaining-and-coalescing.html), [3.14.10. `?` 연산자](../spec/refs-lang/003-expressions/314-error-control-flow.html#31410--연산자-none-조기-반환)
- [2.7. niche 최적화](../spec/refs-lang/002-types/207-niche-optimization.html)
- 다음 장: [6. struct, enum, 트레잇](06-structs-enums-traits.html)
