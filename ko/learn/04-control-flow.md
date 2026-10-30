# 4. 제어 흐름

이 장에서는 분기(`if`, `match`), 반복(`while`, `for`, `loop`), 분해를 다룹니다. 가장 중요한 점은 Rok의 `if`, `match`, `loop`가 모두 값을 내는 표현식이라는 것입니다.

## `if`는 표현식입니다

`if`의 조건에는 괄호가 필요 없고, 본체는 항상 중괄호로 씁니다. 각 분기의 마지막 식이 `if` 전체의 값이 됩니다. 그래서 삼항 연산자가 따로 없습니다.

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

값으로 쓸 때는 모든 분기의 타입이 같아야 합니다. 서로 다른 타입을 섞으려면 받는 변수에 `i32 | str` 같은 합 타입을 적습니다.

## `match`와 패턴

`match`는 값을 위에서 아래로 패턴과 비교해 처음 맞는 가지를 실행합니다. 리터럴, `|`로 이은 여러 패턴, 범위(`4..=9`), 바인딩(`k @ pattern`), 가드(`if condition`), 와일드카드 `_`를 쓸 수 있습니다.

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

컴파일러는 모든 경우를 처리했는지 검사합니다. `i32`는 값이 너무 많아 다 나열할 수 없으므로 마지막 `_`가 필요합니다. 튜플도 패턴으로 분해하고 매칭할 수 있습니다.

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

값이 여러 타입 중 하나일 수 있으면(`i32 | str | none`) `match`에서 `n: i32` 같은 패턴으로 타입에 따라 나눕니다. 이런 패턴과 `is`, `if let`, `let else`는 [다음 장](05-sum-types.html)에서 합 타입과 함께 다룹니다.

## `while`과 `for`

`while`은 조건이 참인 동안 반복합니다. `for`는 범위, 리스트, 반복자에서 원소를 하나씩 꺼냅니다. `a..b`는 `b`를 빼고, `a..=b`는 `b`를 포함합니다. `continue`는 다음 반복으로 넘어가고, `break`는 루프를 빠져나갑니다.

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

`while let`은 패턴이 맞는 동안 반복합니다. `pop()`이 `T | none`을 반환하므로 리스트가 빌 때까지 원소를 꺼내는 코드가 짧아집니다.

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

## `loop`와 값을 가진 `break`

`loop`는 `break`를 만날 때까지 끝없이 반복합니다. `break value`로 빠져나가면 그 값이 `loop` 표현식의 결과가 됩니다. 반복이 끝나는 순간에 결과가 정해지는 계산에 알맞습니다.

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

## 라벨

중첩 루프 안에서 바깥 루프를 바로 끝내려면 `'name:` 라벨을 붙이고 `break 'name`이나 `continue 'name`을 씁니다. `loop`와 블록에 라벨을 붙이면 값을 가지고 나올 수도 있습니다.

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

## 분해

`let`과 `for`의 왼쪽에는 항상 맞는 패턴(반증 불가능 패턴)이면 무엇이든 올 수 있습니다. 중첩 튜플을 풀거나, `..`로 나머지를 모으거나, `_`로 버릴 수 있습니다.

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

## 정리

- `if`, `match`, `loop`, 블록은 값을 내는 표현식입니다. 분기의 타입은 같아야 합니다.
- `match`에는 리터럴, `|`, 범위, `@` 바인딩, 가드를 쓸 수 있고, 컴파일러는 모든 경우를 처리했는지 검사합니다.
- `while let`은 패턴이 맞는 동안 반복합니다.
- `for x in a..b` / `a..=b`는 범위를 돌고, `.iter().enumerate()`는 인덱스와 함께 돕니다.
- `break value`와 `'label:`로 중첩 루프를 빠져나오면서 결과를 돌려줄 수 있습니다.

## 더 읽을거리

- [`if` 표현식](../spec/refs-lang/003-expressions/310-if-expression.html), [`match` 표현식](../spec/refs-lang/003-expressions/312-match-expression.html), [패턴](../spec/refs-lang/003-expressions/311-patterns.html)
- [루프 표현식](../spec/refs-lang/003-expressions/313-loop-expressions.html), [범위 표현식](../spec/refs-lang/003-expressions/307-range-expressions.html), [라벨 블록](../spec/refs-lang/003-expressions/309-block-expressions.html)
- [분해 대입](../spec/refs-lang/003-expressions/315-destructuring.html), [`let-else`](../spec/refs-lang/003-expressions/316-let-else.html)
- [`is`와 슬롯의 타입 검사](../spec/refs-lang/002-types/202-any-slot.html), [합 타입](../spec/refs-lang/002-types/206-sum-types.html)
- 다음 장: [5. 합 타입과 none](05-sum-types.html)
