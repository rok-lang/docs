# Rust 사용자를 위한 Rok

Rust 프로그래머를 위한 안내입니다. 그대로 통하는 것, 버려야 할 습관, Cargo의 도구가 Rok의 어디에 있는지를 정리했습니다. Rok의 겉모습은 Rust와 많이 닮았지만 메모리 모델은 다릅니다. 메모리는 GC가 관리하므로 라이프타임도 borrow checker도 없습니다. 아래 블록은 모두 완결된 프로그램이니 `rok run file.rok`으로 실행해 보세요. 각 부분은 이 가이드의 해당 장에서 자세히 다룹니다.

## 그대로 통하는 것

- `fn`, `struct`, `enum`, `trait`, `impl Trait for Type`, 바운드와 `where` 절이 있는 제네릭, 연관 타입, const 제네릭.
- guard가 있는 `match`, `if let`, `while let`, `let else`, 그리고 튜플, 구조체, enum variant를 분해하는 패턴.
- `#[derive(Debug, Clone, PartialEq, Eq, Hash, Default)]`, `#[repr(C)]`, `#[cfg(..)]`, `#[test]`, `#[inline]`.
- `mod`, `use a::b::{c, d}`, `pub`, `pub use` 재내보내기, 그리고 `crate::`에 해당하는 패키지 기준 경로(`pkg::`).
- 정수 타입 `i8` .. `i64`, `u8` .. `u64`, `isize`, `usize`, `f32`, `f64`. 리터럴 접미사(`7u8`, `2.5f32`), `_` 구분자, `as` 변환, `checked_add`, `wrapping_add`, `saturating_add`.
- 연산자 우선순위, 범위 `a..b`와 `a..=b`, 0 쪽으로 버리는 `/`와 `%`, 그리고 panic이 나는 오버플로.
- 반복자 체인(`map`, `filter`, `fold`, `zip`, `enumerate`, `windows`), 클로저 `|x| x + 1`, `async fn`과 `await`, `unsafe` 블록과 `extern "C"`.
- 빌드, 실행, 테스트, 포맷, 린트, 문서까지 맡는 패키지 관리자. Cargo의 자리는 `rok`이 맡습니다.

## 먼저 버려야 할 습관

- 라이프타임도, `&`나 `&mut`도, borrow checker도 없습니다. 메모리는 GC가 해제하고, 값은 무언가가 가리키는 동안 살아 있습니다.
- `mut`이 없습니다. 모든 변수는 다시 대입할 수 있고, 이미 있는 이름을 `let`으로 선언하면 Rust의 shadowing처럼 새 변수가 생깁니다.
- 구조체, enum, 튜플은 대입할 때 복사되는 값이고, 따로 선택할 것이 없습니다. `List`, `Dict`, `Set`은 공유되는 객체 하나를 가리키는 핸들입니다. 함수가 호출한 쪽의 구조체를 바꾸려면 `ref`를 씁니다.
- 에러는 반환하는 값이 아닙니다. 실패할 수 있는 함수는 `throws`라고 밝히고, 그 함수를 부를 때마다 `try`를 붙입니다. 에러에 쓰는 `?`는 없습니다.
- `Option`이 없습니다. 없을 수도 있는 값의 타입은 `T | none`이고, 줄여서 `T?`로 씁니다.
- 접미사 없는 정수 리터럴은 뒤에 쓰인 곳에서 타입을 추론하지 않고 `i32`가 됩니다. 숫자는 암묵적으로 변환되지 않습니다. `i32`에서 `i64`로도 마찬가지입니다.
- `Drop`이 있는 타입(파일, 소켓, 채널 sender)만 소유자가 하나이고 move됩니다. 닫기가 실패할 수 있는 파일은 `using`이나 `try f.close()`로 명시적으로 닫습니다.
- 문장은 Rust처럼 `;`로 끝나지만, 구조체 필드도 `;`로 끝납니다: `struct P { x: i32; y: i32; }`.

```rok
let x = 5;
x = x + 1;                 // no `mut` needed
let x = x * 2;             // a new `x`, as shadowing in Rust
println(x);

let words = ["a", "bb"];
let w = words;             // the same list, not a move
w.push("ccc");
println(f"{words:?} {words.len()}");
```

```output
12
["a", "bb", "ccc"] 3
```

그래도 `mut`을 쓰면 빌드가 멈추고, help가 `mut`을 빼라고 알려 줍니다.

```text check=checks/rust-let-mut.rok
error[E0012]: rok has no `let mut`: variables are mutable by default
  --> main.rok:1:5
   |
 1 | let mut total = 0;
   |     ^^^
   |
   = help: drop `mut`: `let total = ..;`
```

## 메모리: GC, 값과 핸들

구조체는 값입니다. 대입하거나, 인자로 넘기거나, 리스트에서 꺼내면 복사되고 원래 값은 그대로 쓸 수 있습니다. move되는 것은 `Drop`이 있는 타입뿐이니(다음 절), 다시 쓰려고 clone할 일이 없습니다. 호출한 쪽의 구조체를 바꿔야 하는 함수는 `ref`를 받고, 호출하는 쪽에서도 `ref`를 써서 무엇이 바뀔 수 있는지 읽는 사람이 알 수 있게 합니다. 받는 값을 바꾸는 메서드는 `&mut self`에 해당하는 `ref self`를 받고, 호출하는 쪽에는 표시가 필요 없습니다. 그냥 `self`는 Rust에서 `Copy` 타입의 `self`처럼 복사본입니다.

```rok
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Point {
    fn moved(self, dx: i32) -> Point { Point { x: self.x + dx, y: self.y } }
    fn shift(ref self, dx: i32) { self.x += dx; }
}

fn reset(ref p: Point) { p.x = 0; p.y = 0; }

let a = Point { x: 1, y: 2 };
let b = a;                 // a copy; `a` stays usable
b.shift(10);
println(f"{a:?} {b:?} {a.moved(5):?}");
reset(ref b);
println(f"{b:?}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for ref p in ps { p.x *= 10; }      // `for p in ps.iter_mut()`
println(f"{ps:?}");
```

```output
Point { x: 1, y: 2 } Point { x: 11, y: 2 } Point { x: 6, y: 2 }
Point { x: 0, y: 0 }
[Point { x: 10, y: 0 }, Point { x: 20, y: 0 }]
```

Rust라면 `&`와 `&mut` 가운데 하나를 고르게 했을 자리에서, Rok은 실수를 에러로 만듭니다. 복사본의 필드를 바꾸고 그 뒤로 아무도 읽지 않으면(`fn inc(self) { self.n += 1; }`, `for p in ps { p.x = 0; }`) 컴파일되지 않고, help가 `ref self`나 `for ref`를 알려 줍니다([6장](06-structs-enums-traits.html#changing-the-callers-value-ref)).

락이나 다른 스레드와 공유한 핸들도 검사합니다. `Mutex` guard의 `g.get()`이 돌려주는 것은 보호받는 객체의 view라서, 락을 쥔 `using` 블록 밖으로 나갈 수 없습니다([E0106](../errors/E0106.html)). 대신 `g.get_clone()`을 씁니다. `r.lines()` 같은 스트림 view는 `r`을 통해 읽으므로 `r`의 스코프 안에 머뭅니다([E0140](../errors/E0140.html)). 반환하려면 소유하는 형태인 `r.into_lines()`를 씁니다. Rok이 borrow 비슷한 것을 추적하는 곳은 이 몇 군데뿐이고, 에러마다 무엇을 쓰면 되는지 알려 줍니다.

## 리소스: 스코프 끝의 `Drop` 대신 `using`

GC는 메모리를 해제하지만 파일을 닫지는 않습니다. `Drop`을 구현한 타입은 Rust의 모든 값처럼 소유자가 하나입니다. `let b = a`는 값을 move하고, 그 뒤로 `a`는 쓸 수 없습니다. `using x = e { .. }`는 블록을 어떻게 빠져나가든 `x`를 닫고, 문장 형태 `using x = e;`는 둘러싼 블록의 나머지를 덮습니다. Rust가 지역 변수를 drop하듯 역순으로 닫습니다. `using` 없이 쓴 `Drop` 값은 소유자가 스코프를 벗어나는 곳에서 해제됩니다.

```rok
struct Conn { name: str; }

impl Drop for Conn {
    fn drop(ref self) { println(f"close {self.name}"); }
}

fn work() throws -> i32 {
    using a = Conn { name: "db" };
    using b = Conn { name: "cache" };
    println(f"using {a.name} and {b.name}");
    throw "query failed";
}

try work() catch e { println(f"work failed: {e}"); };
```

```output
using db and cache
close cache
close db
work failed: query failed
```

파일 닫기는 실패할 수 있고(닫을 때까지 버퍼에 있던 쓰기가 디스크에 닿지 못할 수 있습니다), Rust의 `Drop`은 그 에러를 삼킵니다. Rok은 이런 타입을 대신 닫아 주지 않습니다. `try f.close()`를 쓰거나, 닫기 에러를 던지는 `using`으로 묶으세요. `File`을 연 채로 스코프를 벗어나면 컴파일 에러입니다([E0135](../errors/E0135.html)).

```rok
use rok::std::{env, fs, process};

let path = env::temp_dir() / f"rustrok_{process::id()}.txt";
using f = try fs::File::create(path) {
    try f.write_all("hello\n".as_bytes());
}
println(try fs::read_to_str(path));
try fs::remove_file(path);
```

```output
hello

```

## 에러: `Result`와 `?` 대신 `throws`와 `try`

`fn f() -> Result<T, E>`는 `fn f() throws -> T`이고, `f()?`는 `try f()`입니다. `try`는 실패할 수 있는 호출마다 붙으므로, 함수가 일찍 빠져나갈 수 있는 지점이 읽는 사람 눈에 모두 보입니다. `throw "message"`는 `ValueError`를 던집니다. 직접 만든 에러 타입은 `Display`와 `Error`를 구현하고 `throws (A | B)`로 나열합니다.

```rok
fn parse_port(s: str) throws -> i32 {
    let n = try s.parse::<i32>();
    if n < 1 || n > 65535 { throw f"port {n} out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });             // match on Err(_)
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);                       // .ok().unwrap_or(-1)
println(try! parse_port("443"));                           // .unwrap()
```

```output
8080
error: port 99999 out of range
0
-1
443
```

`catch e: IoError { .. }`는 타입으로 골라 잡고, `finally`는 항상 실행됩니다. 에러를 값으로 들고 있어야 할 때, 예를 들어 여러 결과를 모을 때를 위해 `Result<T, E>`도 있습니다: `Result::catching(|| try f())`. `panic(..)`, 범위를 벗어난 인덱스, 실패한 `try!`는 panic이고, Rust에서처럼 `catch`로 잡히지 않습니다.

## `Option` 대신 `T | none`과 합 타입

`Option<T>`는 `T | none`이고, 줄여서 `T?`로 씁니다. `Some(x)`는 그냥 `x`, `None`은 `none`입니다. `??`는 `unwrap_or`, `?.`는 필드나 메서드에 대한 `map`이고, `T?`를 반환하는 함수에서 후위 `?`는 Rust의 `Option`에 쓰는 `?`처럼 `none`을 반환합니다.

`A | B`는 아무 타입이나 합친 합 타입이고, enum을 선언할 필요가 없습니다. `match` arm이나 `is`가 타입을 고릅니다. `enum`은 경우마다 이름을 붙이고, Rust의 enum처럼 데이터를 담습니다.

```rok
struct User { name: str; nick: str?; }

fn nick_len(u: User?) -> usize? {
    let user = u?;
    user.nick?.len()
}

fn show(v: i32 | str | none) -> str {
    match v {
        n: i32 if n < 0 => "negative",
        n: i32 => f"int {n}",
        s: str => f"str {s}",
        none => "nothing",
    }
}

let u = User { name: "ada", nick: "bob" };
println(f"{nick_len(u):?} {nick_len(none):?}");
println(u.nick ?? "-");
if let n: str = u.nick { println(n.to_uppercase()); }
println(f"{show(-1)}, {show(7)}, {show("x")}, {show(none)}");

enum Shape { Circle(f64), Rect { w: f64; h: f64; }, Empty }
let s = Shape::Rect { w: 2.0, h: 3.0 };
let area = match s {
    Circle(r) => 3.0 * r * r,
    Rect { w, h } => w * h,
    Empty => 0.0,
};
println(area);
```

```output
3 none
bob
BOB
negative, int 7, str x, nothing
6
```

`match`하는 enum의 variant는 그냥 이름만 씁니다(`Shape::Circle(r)`이 아니라 `Circle(r)`). `if let x: T = e`는 합 타입에서 `T`를 꺼냅니다. Rust의 `if let Some(x) = e`에 해당합니다.

## 트레이트, 제네릭, `impl`

트레이트, `impl Trait for Type`, 제네릭 바운드, `where` 절, 기본 메서드, 연관 타입은 Rust와 같고, orphan rule도 같습니다. `impl`에는 트레이트나 타입 가운데 하나가 내 패키지에 있어야 합니다([E0076](../errors/E0076.html)). 다른 점은 이렇습니다.

- 트레이트는 타입이 아닙니다. `dyn Trait`은 `any Trait`이고 `Box`가 필요 없습니다. `List<any Shape>`는 `Shape`를 구현한 어떤 타입의 값이든 담습니다. 인자나 반환 자리의 `impl Trait`은 없으니 제네릭 `<T: Shape>`를 받습니다.
- 타입을 적지 않은 매개변수는 제네릭입니다. `fn loud(x) -> str { x.to_uppercase() }`는 본문이 부르는 메서드가 있는 타입이면 무엇이든 받습니다.
- `Fn`, `FnMut`, `FnOnce`는 함수나 클로저를 담는 한 타입 `fn(i32) -> i32`입니다. 다른 스레드로 옮겨도 되는 것은 `fn() + Send`입니다.
- 클로저는 값을 복사해서 캡처합니다. `[ref x]`는 변수 자체를 캡처해서 클로저가 바꿀 수 있게 하고, `[move x]`는 값을 넘겨줍니다. 클로저를 만든 쪽은 그 뒤로 `x`를 쓸 수 없습니다.
- `#[derive(..)]`는 Rust처럼 `impl`을 대신 씁니다. 모든 구조체가 이미 `let y = x;`로 복사되므로 `Copy`는 표시일 뿐입니다.

```rok
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

fn largest<T: Shape>(xs: List<T>) -> f64 {
    xs.map(|s| s.area()).fold(0.0, |a, b| a.max(b))
}

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }
println(largest([Square { side: 1.0 }, Square { side: 3.0 }]));

fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
println(apply(adder(5), 10));

let count = 0;
let snapshot = || count + 1;
let inc = [ref count] || { count += 1; };
inc();
inc();
println(f"{snapshot()} {count}");
```

```output
area 4.0
area 3.1
9
15
1 2
```

## 숫자

접미사 없는 정수 리터럴은 들어가는 자리(`let b: u8 = 200;`, 인자, 다른 타입의 피연산자)에서 타입을 얻고, 아무것도 정해 주지 않으면 `i32`입니다. 실수 리터럴은 같은 방식으로 `f64`입니다. 길이와 인덱스는 `usize`이므로, `i32`인 `total`로 `total / xs.len()`을 하려면 `as`가 필요합니다. `int`는 크기 제한이 없는 정수입니다.

암묵적 변환은 없습니다. `i32 + i64`는 에러이고, help가 `as`를 써 줍니다. `as`는 잘라 내고, `.into()`는 잃는 것이 없을 때만 허용되며, `.try_into::<T>()`는 값이 맞지 않으면 에러를 던지므로 `try_from` 자리에는 `try`가 붙습니다.

```rok
let a = 7;                    // i32
let b: u8 = 200;
let n = 3i64;
println(f"{(a as i64) + n} {b as i32 + 100} {300 as u8}");
let w: i64 = a.into();
let small: u8 = try 255i32.try_into::<u8>();
println(f"{w} {small}");
println(try? 256i32.try_into::<u8>() ?? 0);
println(f"{-7 / 2} {-7 % 2} {-7 %% 2} {7.pow(2)}");
let big: int = 2;
println(big.pow(100));
```

```output
10 300 44
7 255
0
-3 -1 1 49
1267650600228229401496703205376
```

`%%`는 내림 나머지로, 나누는 수가 양수일 때 Rust의 `rem_euclid`와 같습니다. `x.pow(n)`은 Rust처럼 메서드입니다.

## 연산자

산술, 비트, 비교 연산자와 그 우선순위는 Rust와 같고, 연산자 트레이트도 `Add`, `Mul`, `Neg`, `Index` 등입니다. Rok은 몇 가지를 더합니다.

- 연쇄 비교: `0 <= i < n`.
- 포함 여부를 보는 `in`과 `!in`: `x in xs`, `k !in dict`.
- 리스트의 각 원소에 연산자를 적용하는 원소별 "점" 연산자 `.+`, `.*`, `.>` 등([8장](08-collections-iterators.html#advanced-broadcast-operators)).
- 기본값을 주는 `??`와 `none`을 건너 체이닝하는 `?.`.
- `Path`에 쓰는 `/`는 경로를 잇습니다: `dir / "file.txt"`.

```rok
let a = [1.0, 2.0, 3.0];
let b = [10.0, 20.0, 30.0];
println(f"{a .+ b:?} {a .* 2.0:?}");
let i = 3;
println(0 <= i < 5 && i !in [1, 2]);
```

```output
[11.0, 22.0, 33.0] [2.0, 4.0, 6.0]
true
```

## 모듈과 가시성

`mod name;`은 파일 옆의 `name.rok`를 불러오고, `name/`에 그 하위 모듈이 들어갑니다. `mod.rs`가 없는 Rust 2018 방식과 같습니다. 패키지 루트에서 시작하는 경로는 Rust의 `crate::` 대신 `pkg::`로 시작하고, `sup::`는 `super::`입니다. 항목은 자기 모듈 안에서만 보입니다. `pub(sub)`은 하위 모듈까지, `pub(pkg)`는 패키지 전체(`pub(crate)`)에, `pub`은 모두에게 엽니다. 표준 라이브러리의 몇몇 `pub` 항목은 `#[internal]`이라 toolchain 자체 패키지만 쓸 수 있습니다.

```rok
mod geo {
    pub struct Point { pub x: f64; pub y: f64; }
    pub fn dist_sq(a: Point, b: Point) -> f64 { sq(a.x - b.x) + sq(a.y - b.y) }
    fn sq(v: f64) -> f64 { v * v }
    pub mod unit {
        pub fn origin() -> sup::Point { sup::Point { x: 0.0, y: 0.0 } }
    }
}

use geo::{Point, unit};
println(geo::dist_sq(unit::origin(), Point { x: 3.0, y: 4.0 }));
```

```output
25
```

`println`, `List`, `Dict`, `Mutex`, `channel`, `task`, `par`는 prelude에 있습니다. 운영체제와 이야기하는 `rok::std`의 모듈(`fs`, `env`, `process`, `thread`, `net`)은 `use rok::std::fs;`처럼 가져옵니다.

## 속성과 매크로

속성은 Rust처럼 씁니다. `macro_rules!`나 procedural macro를 찾을 자리에는 대신 메타 정의라는 한 가지 방식이 있습니다. `@` 토큰이 들어간 Rok 코드 템플릿입니다. `#[impl_derive(T)]`는 derive를, `#[impl_attribute(name)]`는 속성을, `#[impl_macro(name)]`는 `name!(..)` 호출을 정의합니다([6.8](../spec/refs-lang/006-meta-system/608-meta-blocks.html)). 컴파일 시점에 토큰 스트림을 다루며 실행되는 코드는 없습니다. `println`, `assert`, 그리고 `format`식 문자열은 보통 함수와 f-string이라 `!`를 붙이지 않습니다.

```rok
#[impl_macro(twice)] @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let (a, b) = twice!(21 * 2);
println(f"{a} {b}");
assert(a == b, "same value");
```

```output
42 42
```

## 동시성

스레드는 Rust와 같습니다. `thread::spawn`은 클로저를 받고, `join`은 그 반환값을 돌려줍니다(스레드가 panic으로 끝났으면 에러를 던집니다). `Send`와 `Sync`가 있고 컬렉션은 `Sync`가 아니므로, 다른 스레드에 줄 리스트를 캡처하는 클로저는 `[move rows]`로 가져갑니다. `Mutex`, `RwLock`, `Atomic`, `channel`은 prelude에 있고, 락이나 채널에 넘긴 값은 그 뒤로 쓸 수 없습니다. Rust에서 `Mutex`로 move하는 것을 대신하는 넘겨주기 검사입니다.

`async fn`과 `await`는 런타임이 내장되어 있어 Tokio가 필요 없습니다. `task::spawn`은 태스크를 시작하고, `task::join`은 여러 태스크를 기다리며, 최상위에 `await`가 있는 스크립트는 `async fn main`으로 실행됩니다. `await`는 전위 연산자라 `f().await`가 아니라 `await f()`로 씁니다. `par::map`, `par::filter`, `par::for_each`는 Rayon처럼 작업을 풀에 나눠 줍니다.

```rok
use rok::std::thread;

let rows = [3, 1, 2];
let h = thread::spawn([move rows] || {
    rows.sort();
    rows[0]
});
println(f"smallest: {try h.join()}");

let m = Mutex::new(0);
par::for_each(1..=100, |n| { using g = m.lock() { g.set(g.get() + n); } });
println(using g = m.lock() { g.get() });

async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms(5));
    f"item-{id}"
}
let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2))];
println(f"{await task::join(tasks):?}");
println(f"{par::map(1..=4, |x| x * x):?}");
```

```output
smallest: 1
5050
["item-1", "item-2"]
[1, 4, 9, 16]
```

## `unsafe`와 FFI

`extern "C" { .. }`는 C 함수를 선언하고, 그 함수는 언제나 `unsafe { .. }` 안에서 부릅니다. `#[repr(C)]`, 원시 포인터(`rok::bare::ptr`의 `Ptr<T>`, `ConstPtr<T>`), `#[link("m")]`, `extern "C" fn` 콜백은 Rust와 같습니다. `Pod`(`rok::bare::mem::Pod`, plain old data)는 `bytemuck::Pod` 같은 `unsafe trait`입니다. `#[derive(Pod)]`나 `unsafe impl Pod for T {}`로 구현하고, 컴파일러는 그래도 레이아웃을 검사합니다. `List`, `Dict`, 클로저는 GC가 옮길 수 있으므로 C로 넘길 수 없습니다. 포인터와 길이를 넘기세요.

```rok
extern "C" {
    fn abs(x: i32) -> i32;
    fn strlen(s: str) -> usize;
}

println(unsafe { abs(-42) });
println(unsafe { strlen("hello") });
```

```output
42
5
```

## Cargo, crate, `rok.rokon`

패키지 관리자는 `rok`이고, 매니페스트는 TOML이 아니라 Rok의 데이터 형식인 ROKON으로 쓴 `rok.rokon`입니다. crate가 crates.io에 올라가듯 [rokstones.io](https://rokstones.io)에 올린 패키지를 stone이라고 부릅니다. `rok.lock`은 `Cargo.lock`처럼 버전을 고정합니다. `rok publish`는 소스 아카이브를 올리지 않고, git 저장소의 커밋을 가리키는 줄 하나를 인덱스에 더합니다. 파일 하나도 프로그램이 될 수 있습니다. `rok file.rok`은 패키지도 `main`도 없이 실행하고, `rok add json file.rok`은 의존성을 파일 안에 적습니다.

```rokon
Package { name: "app", version: "0.1.0", toolchain: "0.2", deps: {
    "json": "^1.0",
    "geom": Dep::path("../geom"),
} }
```

| Cargo | Rok |
|---|---|
| `cargo new app`, `cargo new --lib geom` | `rok new app`, `rok new --lib geom` |
| `cargo build`, `cargo run`, `cargo check` | `rok build`, `rok run`, `rok check` |
| `cargo test`, `cargo bench` | `rok test`, `rok bench` |
| `cargo fmt`, `cargo clippy` | `rok fmt`, `rok lint` |
| `cargo doc` | `rok doc` |
| `cargo add serde`, `cargo update` | `rok add json`, `rok update` |
| `cargo install`, `cargo publish` | `rok install`, `rok publish` |
| `rustup toolchain install` | `rok toolchain install` |
| `[features]`, `--features` | `rok.rokon`의 `features:`, `--features`([18장](18-features-and-stones.html)) |

`rok fmt`는 들여쓰기, 줄 끝 공백, 빈 줄만 고칩니다. 토큰과 한 줄 안의 간격은 쓴 그대로 둡니다. 테스트는 패키지 어디에 있든 `#[test]` 함수이고, `#[cfg(test)] mod tests`로 감쌀 필요가 없습니다. 보통 빌드에서는 빠지고, `rok test`는 테스트마다 따로 프로세스를 띄우므로 하나가 panic이 나도 나머지는 계속 돕니다.

```rok
fn gcd(a: i32, b: i32) -> i32 { if b == 0 { a } else { gcd(b, a % b) } }

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
}

println(gcd(12, 18));
```

```output
6
```

## Rust 관용구를 Rok으로

| Rust | Rok |
|---|---|
| `let mut x = 0;` | `let x = 0;` |
| `let x = x.trim();` (shadowing) | `let x = x.trim();` |
| `&T`, `&mut T` 매개변수 | `T`(구조체는 복사본, `List`는 같은 객체), `ref p: T` |
| `&mut self` | `ref self` |
| `for x in xs.iter_mut()` | `for ref x in xs` |
| 다시 쓰려고 `x.clone()` | 필요 없음: 구조체는 복사됨. 따로 둘 `List`에는 `.clone()` |
| `Box<dyn Trait>`, `Vec<Box<dyn Trait>>` | `any Trait`, `List<any Trait>` |
| `fn f(x: impl Trait)` | `fn f<T: Trait>(x: T)` |
| `impl Fn(i32) -> i32`, `Box<dyn FnMut()>` | `fn(i32) -> i32`, `fn()` |
| `move \|\| ..` | `[move x] \|\| ..` |
| 캡처한 지역 변수를 바꾸는 클로저 | `[ref x] \|\| { x += 1; }` |
| `Option<T>`, `Some(x)`, `None` | `T?`, `x`, `none` |
| `opt.unwrap_or(d)`, `opt.map(\|v\| v.f())` | `opt ?? d`, `opt?.f()` |
| `if let Some(x) = opt` | `if let x: T = opt` |
| `fn f() -> Result<T, E>` | `fn f() throws -> T` |
| `f()?`, `f().unwrap()`, `f().ok()` | `try f()`, `try! f()`, `try? f()` |
| `Err("msg".into())`, `anyhow!("msg")` | `throw "msg"` |
| `match r { Ok(v) => v, Err(e) => .. }` | `try f() catch e { .. }` |
| `impl Drop` + 스코프 끝 | `using x = e;`, 파일은 `try f.close()` |
| `Vec<T>`, `HashMap<K, V>`, `HashSet<T>` | `List<T>`, `Dict<K, V>`, `Set<T>` |
| `vec![1, 2]`, `String`, `&str` | `[1, 2]`, `str`(문자열 타입은 하나) |
| `println!("{x:?}")`, `format!(..)` | `println(f"{x:?}")`, `f".."` |
| `assert_eq!(a, b)` | `assert_eq(a, b)` |
| `x as i64`, `i64::from(x)` | `x as i64`, `x.into()` |
| `u8::try_from(x)?` | `try x.try_into::<u8>()` |
| `x.rem_euclid(n)` | `x %% n` |
| `crate::a::b`, `super::c`, `pub(crate)` | `pkg::a::b`, `sup::c`, `pub(pkg)` |
| `macro_rules! m` | `#[impl_macro(m)] @macro { .. }` |
| `fut.await`, `tokio::spawn` | `await fut`, `task::spawn` |
| `rayon` `par_iter().map(..)` | `par::map(xs, ..)` |
| `struct P { x: i32, y: i32 }` | `struct P { x: i32; y: i32; }` |
