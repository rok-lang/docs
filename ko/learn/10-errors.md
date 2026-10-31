# 10. 에러 처리

파일이 없거나 입력이 잘못되는 것처럼, 실패할 수 있는 일은 어디에나 있습니다. Rok의 에러 모델은 하나뿐입니다. 에러를 던질 수 있는 함수는 시그니처에 `throws`를 적고, 호출하는 쪽은 그 호출에 `try`를 붙입니다. 그래서 에러가 흐를 수 있는 지점이 모두 코드에 드러납니다.

## `throws`, `throw`, `try`

`throws`는 매개변수 목록 뒤, `->` 앞에 씁니다. 본체에서는 `throw`로 에러 값을 던집니다. 대부분의 코드는 표준 라이브러리의 `ValueError { msg }`로 충분하므로 에러 타입을 따로 정의할 필요가 없습니다.

에러를 던지는 함수를 호출할 때는 언제나 `try`를 붙여야 합니다. 호출이 실패하면 에러는 가장 가까운 `catch`로 갑니다. `catch`가 없으면 에러는 감싸는 함수 밖으로 전파됩니다. 이때 그 함수도 `throws`여야 합니다.

```rok
fn parse_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw ValueError { msg: f"age {n} is negative" }; }
    n
}

fn total_age(xs: List<str>) throws -> i32 {
    let sum = 0;
    for x in xs {
        sum += try parse_age(x);     // an error leaves total_age here
    }
    sum
}

println(f"{try total_age(["30", "12"])}");
let t = try total_age(["30", "-1"]) catch e { println(f"error: {e}"); 0 };
println(f"{t}");
```

```output
42
error: age -1 is negative
0
```

`catch e { .. }`는 그 자리에서 에러를 처리하고, 호출 결과 대신 블록의 값을 씁니다. `try`는 오른쪽의 식 전체를 덮습니다. `try a(b())`는 두 호출을 함께 표시하고, `try { .. }`는 블록 안의 모든 호출을 덮습니다.

`throws`라고만 쓰면 그 함수는 어떤 에러 타입이든 던질 수 있습니다. `throws (any Error)`의 축약입니다. `s.parse()`는 `ParseError`를 던지고, `parse_age`는 그 에러를 그대로 넘깁니다. 던지는 에러 타입을 정확히 나열하는 방법은 [아래](#직접-만드는-에러-타입)에서 다룹니다.

### 스크립트 최상위의 에러

프로그램의 최상위 문장은 `throws`인 암묵적 `main`의 본체가 됩니다. 그래서 스크립트에서는 `try`만 붙이면 됩니다. 아무도 잡지 않은 에러는 stderr에 메시지를 남기고 종료 코드 1로 프로그램을 끝냅니다.

```rok build
let n: i32 = try "abc".parse();
println(f"{n}");
```

```text
$ rok run
error: `main` returned an uncaught error: cannot parse "abc": invalid digit
  --> src/main.rok:1:18
$ echo $?
1
```

직접 쓴 `fn main()`이 `throws`가 아니면 모든 에러를 그 안에서 처리해야 합니다. 에러를 밖으로 내보내려면 `fn main() throws { .. }`로 씁니다.

## `try?`와 `??`: 에러 대신 기본값

대체 값만 있으면 되는 경우가 많습니다. `try? e`는 에러를 버리고 `none`을 내놓습니다. `??`는 `none`을 기본값으로 바꿉니다. `??` 앞에서는 `try?` 식을 괄호로 감쌉니다. `try? f() ?? d`는 `try? (f() ?? d)`로 읽혀 에러가 나면 `none`이 됩니다. rokc가 경고하고, `f()`가 옵셔널이 아니면 컴파일 에러입니다.

```rok
fn parse_port(s: str) -> i32 {
    (try? s.parse::<i32>()) ?? 8080
}

println(f"{parse_port("3000")} {parse_port("abc")}");
```

```output
3000 8080
```

에러를 일부러 무시하려면 `let _ = try? f();`로 씁니다. `try! e`는 에러를 panic으로 바꿉니다. 그 자리에서 실패하는 것이 곧 버그일 때만 쓰세요.

| 형태     | 에러가 나면                              |
|----------|------------------------------------------|
| `try e`  | 가장 가까운 `catch`로, 없으면 전파      |
| `try? e` | 에러를 버리고 `none`을 내놓음           |
| `try! e` | panic (프로그램 버그로 취급)             |

```rok
fn save(n: i32) throws {
    if n < 0 { throw ValueError { msg: "negative" }; }
    println(f"saved {n}");
}

let _ = try? save(1);
let _ = try? save(-1);         // the error is dropped
let n: i32 = try! "42".parse();
println(f"{n}");
```

```output
saved 1
42
```

## `catch`: 타입으로 고르기

`catch` 패턴은 위에서 아래로 검사되고, 처음 맞는 블록이 실행됩니다. `e: T`는 타입으로 고릅니다. 그 가지 안에서 `e`는 그 타입이므로 필드를 읽을 수 있습니다. 마지막의 `catch e`는 나머지를 모두 받습니다.

```rok
fn read_age(s: str) throws -> i32 {
    let n: i32 = try s.parse();
    if n < 0 { throw ValueError { msg: f"age {n} is negative" }; }
    n
}

for s in ["30", "x", "-1"] {
    let age = try read_age(s)
    catch e: ValueError { println(f"  {e.msg}"); -1 }
    catch e { println(f"  {e}"); -1 };
    println(f"{s} -> {age}");
}
```

```output
30 -> 30
  cannot parse "x": invalid digit
x -> -1
  age -1 is negative
-1 -> -1
```

## 블록 형태와 `finally`

여러 단계를 한꺼번에 감싸려면 `try { .. } catch .. { .. }` 형태를 씁니다. `finally` 블록은 정상 완료, 에러, `return` 등 어떻게 빠져나가든 마지막에 실행됩니다.

```rok
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

for input in [8, 6] {
    let r = try {
        let a = try half(input);
        let b = try half(a);
        a + b
    } catch e {
        println(f"failed: {e}");
        0
    } finally {
        println(f"done with {input}");
    };
    println(f"result {r}");
}
```

```output
done with 8
result 6
failed: 3 is odd
done with 6
result 0
```

## 직접 만드는 에러 타입

호출자가 실패를 구별해야 한다면 에러 타입을 정의합니다. 에러 타입은 `Debug`와 `Display`를 갖추고 `Error` 트레잇을 구현한 struct나 enum입니다. `Display`의 모양은 [6장](06-structs-enums-traits.html#연산자-트레잇)에서 본 것과 같습니다. 에러 타입을 정의하면 던지는 에러를 시그니처에 정확히 적을 수 있습니다. 한 종류라면 `throws (E)`, 여러 종류라면 합 타입으로 `throws (A | B)`라고 씁니다. 컴파일러는 나열한 타입을 모두 처리하거나 넘기는지 검사합니다.

```rok
#[derive(Debug)]
struct NotFound { key: str; }
impl Display for NotFound {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(f"no key {self.key}"); }
}
impl Error for NotFound {}

#[derive(Debug)]
struct Invalid { reason: str; }
impl Display for Invalid {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.reason); }
}
impl Error for Invalid {}

fn lookup(key: str) throws (NotFound | Invalid) -> i32 {
    if key == "" { throw Invalid { reason: "empty key" }; }
    if key != "a" { throw NotFound { key: key }; }
    1
}

fn describe(key: str) -> str {
    try f"found {lookup(key)}"
    catch e: NotFound { f"missing: {e.key}" }
    catch e: Invalid { f"invalid: {e}" }
}

fn main() {
    for k in ["a", "b", ""] {
        println(describe(k));
    }
}
```

```output
found 1
missing: b
invalid: empty key
```

`lookup`의 에러 종류는 `NotFound | Invalid`로 닫혀 있습니다. 그래서 두 타입을 모두 처리하는 `describe`는 에러를 던지지 않는 함수가 되고, 컴파일러가 이 점을 검사합니다. 공개 API처럼 에러 종류를 닫기 어려울 때는 그냥 `throws`를 씁니다. 받는 쪽은 `catch e`로 모두 받거나, `catch e: ValueError`처럼 먼저 좁힙니다([4.14.3](../spec/refs-lang/004-functions/414-throws.html#4143-타입-없는-throws)).

`catch` 뒤에는 어떤 패턴이든 올 수 있으므로 struct 패턴으로 필드를 바로 꺼낼 수 있습니다.

```rok
#[derive(Debug)]
struct NotFound { key: str; }
impl Display for NotFound {
    fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(f"no key {self.key}"); }
}
impl Error for NotFound {}

fn lookup(key: str) throws (NotFound) -> i32 {
    if key != "a" { throw NotFound { key: key }; }
    1
}

for key in ["a", "b"] {
    let n = try lookup(key)
    catch NotFound { key } { println(f"missing: {key}"); 0 };
    println(f"{n}");
}
```

```output
1
missing: b
0
```

## `rethrows`: 인자가 던지는 만큼만 던지기

함수를 인자로 받는 고차 함수는 `rethrows`로 선언할 수 있습니다. 넘긴 함수가 에러를 던지지 않으면 호출에 `try`가 필요 없습니다. 에러를 던지는 함수를 넘길 때만 `try`를 붙입니다.

```rok
fn apply_all(xs: List<i32>, f: fn(i32) throws -> i32) rethrows -> List<i32> {
    let out: List<i32> = [];
    for x in xs { out.push(try f(x)); }
    out
}

fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

let doubled = apply_all([1, 2, 3], |x| x * 2);
println(f"{doubled:?}");
let halves = try apply_all([2, 4], half) catch _ { [] };
println(f"{halves:?}");
let bad = try apply_all([2, 3], half) catch e { println(f"{e}"); [] };
println(f"{bad:?}");
```

```output
[2, 4, 6]
[1, 2]
3 is odd
[]
```

## `using`: 자원 정리

메모리는 GC가 회수하지만 파일, 소켓, 잠금 같은 자원은 정해진 시점에 닫아야 합니다. `using x = acquire_expr { .. }`는 블록을 어떻게 빠져나가든 `x`의 `Drop::drop`을 호출합니다. 정상 종료, `return`, 에러 전파가 모두 해당합니다. 여러 자원은 쉼표로 구분하고, 획득한 역순으로 정리됩니다.

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn open(name: str) -> Conn {
    println(f"open {name}");
    Conn { name: name }
}

fn work() throws -> i32 {
    using a = open("db"), b = open("cache") {
        println("working");
        throw ValueError { msg: "query failed" };
    }
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
open db
open cache
working
close cache
close db
work failed: query failed
```

`using`은 식이므로 `let text = using f = try fs::open(path) { try f.read_to_end() };`처럼 값을 내놓을 수도 있습니다. 일반 `let` 바인딩은 스코프를 벗어나도 `drop`을 호출하지 않습니다. 정해진 시점에 정리해야 하면 `using`을 씁니다.

## 에러와 panic

`throws`의 에러는 "일어날 수 있는 실패"입니다. 반면 범위를 벗어난 인덱스, 0으로 나누기, 실패한 `try!`, `panic(..)` 호출은 **panic**이고 프로그램 버그로 취급합니다. panic은 `catch`로 잡을 수 없습니다. 프로그램은 stderr에 메시지를 쓰고 종료 코드 101로 끝납니다. `finally`와 `using`의 정리도 실행되지 않습니다.

```rok build
let xs = [1, 2, 3];
let i = 7;
println(f"{xs[i]}");
```

```text
thread 'main' panicked at main.rok:3:12:
index 7 out of bounds for length 3
```

`Result<T, E>`는 에러를 값으로 담아 두어야 할 때에만 씁니다. 여러 결과를 모으거나 나중에 처리할 때입니다. 규칙은 "던지면 `throws`, 담아 두면 `Result`"입니다.

```rok
fn half(n: i32) throws -> i32 {
    if n % 2 != 0 { throw ValueError { msg: f"{n} is odd" }; }
    n / 2
}

for n in [4, 5] {
    let r = Result::catching(|| try half(n));
    println(f"{n}: ok={r.is_ok()} value={r.unwrap_or(-1)}");
}
```

```output
4: ok=true value=2
5: ok=false value=-1
```

## 정리

- 에러를 던질 수 있는 함수는 `->` 앞에 `throws`를 선언하고 `throw`로 던집니다. 대부분은 `ValueError { msg }`로 충분합니다.
- 호출은 `try`로 표시합니다. 잡지 않은 에러는 감싸는 함수로 전파되고, 스크립트 최상위에서는 종료 코드 1로 프로그램을 끝냅니다.
- `(try? e) ?? default`는 대체 값을 주고(괄호가 없으면 rokc가 경고하거나 거부합니다), `let _ = try? f();`는 에러를 무시하며, `try!`는 에러를 panic으로 바꿉니다.
- `catch`는 패턴으로 고르는 식이고, `finally`는 항상 실행됩니다.
- 호출자가 실패를 구별해야 하면 에러 타입(`Display` + `Error`)을 직접 정의하고 `throws (A | B)`로 나열합니다.
- `rethrows`는 받은 함수가 던질 때에만 던집니다.
- `using`으로 바인딩한 자원은 블록을 어떻게 빠져나가든 정리됩니다.
- panic은 잡을 수 없는 버그 신호입니다. 에러를 값으로 담아 두어야 할 때에만 `Result`를 씁니다.

## 더 읽을거리

- 사양: [에러 제어 흐름](../spec/refs-lang/003-expressions/314-error-control-flow.html), [`throws` 시그니처](../spec/refs-lang/004-functions/414-throws.html), [`using` 식](../spec/refs-lang/003-expressions/317-using-expression.html), [panic과 fatal 에러](../spec/refs-lang/003-expressions/314-error-control-flow.html#3148-panic과-fatal-에러)
- 표준 라이브러리 사양: [error](../spec/refs-standard-libraries/200-core/214-error.html)
- API: [`Error`](../../std/core/error.html#trait.Error), [`ValueError`](../../std/core/error.html#struct.ValueError), [`Result`](../../std/core/error.html#enum.Result), [`Drop`](../../std/core/marker.html#trait.Drop)
- 다음 장: [11. 모듈과 패키지](11-modules-packages.html)
