# 3. 함수

이 장에서는 함수 선언과 호출, 기본값과 이름 붙인 인자(named argument), 값처럼 주고받을 수 있는 클로저를 다룹니다. Rok에서 함수 본체는 블록 표현식이기도 하므로 마지막 식이 반환값입니다.

## 선언과 반환값

함수는 `fn name(param: Type, ...) -> ReturnType { body }`로 선언합니다. 본체의 마지막 식(세미콜론이 없는 식)이 반환값입니다. `return`은 일찍 빠져나갈 때만 씁니다.

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

값을 여러 개 반환하려면 튜플로 묶고 호출하는 쪽에서 분해합니다.

블록 안의 문은 `;`으로 끝나지만, `}` 바로 앞의 마지막 문은 `;`을 생략해도 의미가 같습니다. 따라서 `while i < n { i += 1 }`은 `{ i += 1; }`과 같습니다. 마지막 `let`, `return`, `break`, `throw`도 마찬가지입니다. 블록에 값을 주는 것은 마지막 식뿐입니다. 위의 `a + b`가 그렇습니다.

반환 타입과 매개변수 타입은 생략할 수 있습니다. 생략한 반환 타입은 마지막 식에서 추론합니다(`;`으로 끝나면 `()`). 타입 없는 매개변수가 있으면 함수는 제네릭이 됩니다. 컴파일러는 호출에 쓰인 인자 타입마다 함수를 따로 컴파일합니다.

```rok
fn double(x: i32) { x * 2 }   // return type inferred: i32
fn id(x) { x }                // generic: fn id<T>(x: T) -> T
println(f"{double(4)} {id("hi")}");
```

```output
8 hi
```

블록 `{ ... }`은 함수 본체 밖에서도 표현식입니다. 임시 값은 블록 안에 두고 결과만 꺼낼 때 편합니다.

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

## 기본값과 named 인자

매개변수에 `= value`로 기본값을 줄 수 있고, 호출할 때 `name: value`로 인자를 이름으로 넘길 수 있습니다. 위치 인자가 먼저, 이름 붙인 인자가 나중에 옵니다.

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

기본값은 리터럴이나 `const`처럼 컴파일 타임에 알 수 있는 값이어야 합니다. 개수가 정해지지 않은 인자는 `...T`로 받습니다. 본체 안에서 이 인자는 읽기 전용 슬라이스입니다.

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

## 클로저

클로저는 이름 없는 함수 값이고 `|params| expr` 또는 `|params| { block }`으로 씁니다. 매개변수 타입은 문맥에서 추론하고, 문맥이 없으면 직접 적습니다.

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

클로저는 주변 변수를 **값을 복사해서** 캡처합니다. 클로저를 만든 뒤 바깥 변수가 바뀌어도 클로저는 만들 때의 값을 기억합니다. 바깥 변수 자체를 바꾸려면 `[ref name]` 캡처 리스트로 참조 캡처를 지정합니다.

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

값으로 캡처한 변수에 대입하려 하면 컴파일러가 `[ref count]`를 추가하라고 알려 줍니다. `List` 같은 힙 객체는 다릅니다. 변수에 담긴 것이 객체를 가리키는 참조이므로 값으로 캡처해도 `push`처럼 객체 자체를 바꿀 수 있습니다.

## 함수 타입

함수와 클로저는 `fn(ParamTypes) -> ReturnType` 타입의 값입니다. 변수에 담고, 인자로 넘기고, 반환하고, 리스트에 넣을 수 있습니다.

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

`make_adder`가 반환한 클로저는 `n`을 캡처한 채 살아 있습니다.

## 트레일링 클로저

마지막 인자가 클로저이면 괄호 밖으로 뺄 수 있습니다. 매개변수가 있으면 `f(args) |x| { ... }`, 없으면 `f(args) { ... }`입니다. 이렇게 쓰면 반복이나 자원 관리 같은 패턴이 내장 문법처럼 읽힙니다.

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

## 메서드 미리 보기

`impl` 블록 안의 함수는 메서드입니다. 첫 매개변수가 `self`이면 `value.method()`로 호출합니다. `self`가 없으면 `Type::function()`으로 호출하는 정적 메서드입니다. 메서드 안에서 필드를 바꾸려면 `ref self`를 받습니다. struct와 메서드는 뒤의 장에서 자세히 다룹니다.

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

## 중첩 함수와 진입점

함수 안에 함수를 선언할 수 있습니다. 중첩 함수는 자기 블록 안에서만 보이고, 클로저와 달리 바깥 변수를 캡처하지 **않습니다**. 바깥 상태가 필요 없는 도우미를 숨길 때 씁니다.

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

프로그램은 `src/main.rok`(또는 단독 파일)의 최상위 문이나 `fn main()`에서 시작합니다. 최상위 문은 컴파일러가 만드는 암묵 `main`의 본체가 됩니다. 그래서 최상위 `let` 변수는 **지역 변수**이고, 다른 함수 본체 안에서는 보이지 않습니다. 함수에 그런 값이 필요하면 인자로 넘기거나 클로저로 캡처합니다.

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

## 정리

- 함수 본체의 마지막 식이 반환값입니다. 일찍 빠져나갈 때 `return`을 씁니다.
- 매개변수에 기본값을 줄 수 있고, 호출에서 `name: value`로 인자에 이름을 붙일 수 있습니다. `...T`는 가변 인자를 받습니다.
- 클로저 `|x| expr`는 기본적으로 값을 복사해 캡처하고, `[ref x]`는 참조로 캡처합니다.
- 함수 값의 타입은 `fn(A) -> R`입니다. 마지막 인자 자리의 클로저는 트레일링 형태로 쓸 수 있습니다.
- 최상위 `let`은 암묵 `main`의 지역 변수이므로 다른 함수에서 보이지 않습니다.

## 더 읽을거리

- [함수 선언](../spec/refs-lang/004-functions/401-function-declaration.html), [매개변수](../spec/refs-lang/004-functions/402-parameters.html), [반환](../spec/refs-lang/004-functions/403-return.html)
- [함수 타입](../spec/refs-lang/004-functions/404-function-type.html), [클로저](../spec/refs-lang/004-functions/405-closure.html), [캡처 의미론](../spec/refs-lang/004-functions/406-capture-semantics.html)
- [트레일링 클로저](../spec/refs-lang/004-functions/408-trailing-closure.html), [메서드](../spec/refs-lang/004-functions/409-method.html), [중첩 함수](../spec/refs-lang/004-functions/410-nested-functions.html)
- [진입점](../spec/refs-lang/004-functions/416-entry-point.html), [블록 표현식](../spec/refs-lang/003-expressions/309-block-expressions.html)
- 다음 장: [4. 제어 흐름](04-control-flow.html)
