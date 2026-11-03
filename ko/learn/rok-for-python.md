# Python 사용자를 위한 Rok

Python 프로그래머를 위한 안내입니다. 그대로 통하는 것, Rok이 다르게 쓰는 것, Python 표준 라이브러리 모듈이 Rok의 어디에 있는지를 정리했습니다. 아래 블록은 모두 완결된 프로그램이니 `rok run file.rok`으로 실행해 보세요. 각 부분은 이 가이드의 해당 장에서 자세히 다룹니다.

## 그대로 통하는 것

- 파일은 `main` 없이 위에서 아래로 실행됩니다. 스크립트처럼 `rok file.rok`으로 실행하고, `#!/usr/bin/env rok` 줄을 쓸 수 있으며, 의존성도 파일 안에 적습니다(`rok add json file.rok`).
- f-string과 같은 서식 지정: `f"{x:.2f}"`, `f"{name:>10}"`, `f"{x=}"`.
- 음수 인덱스와 step이 있는 슬라이스: `xs[-1]`, `xs[1:3]`, `xs[::-1]`.
- 기본 인자와 이름 붙은 인자: `fn f(a: i32, sep: str = " ")`를 `f(1, sep: ", ")`로 호출합니다.
- `in`, 연쇄 비교(`0 <= i < n`), 튜플 분해와 교환(`(a, b) = (b, a)`).
- `try` / `catch` / `finally`, 제너레이터(`yield fn`), `async` / `await`, guard가 있는 `match`.
- 큰 정수: `int`에는 크기 제한이 없습니다.
- REPL(`rok repl`)과 Jupyter 커널(`rok jupyter`).

## 처음 놀라는 점

- `print`는 줄을 바꾸지 않고, `println`이 줄을 바꿉니다. 둘 다 값을 여러 개 받고 `sep:`도 받습니다.
- 모든 값의 타입을 컴파일러가 압니다. 타입을 적지 않은 매개변수는 제네릭이라, 본문이 부르는 메서드가 있는 타입이면 무엇이든 받습니다. 컴파일 시점에 검사하는 덕 타이핑입니다.
- 블록은 들여쓰기가 아니라 `{ }`로 묶고, 새 이름은 `let`으로 선언합니다.
- truthiness가 없습니다. 조건은 `bool`이어야 하니 `!xs.is_empty()`, `n != 0`, `x != none`처럼 씁니다.
- 값이 없음은 `none`이고, 그것을 담을 수 있는 타입은 `T | none`으로 씁니다.
- 정수의 `/`는 0 쪽으로 버립니다(`7 / 2`는 `3`). 내림 나눗셈은 `n.div_floor(2)`이고, `%%`가 내림 나머지입니다. `//`는 주석의 시작입니다.
- `i32`가 넘치면 panic이 납니다. `i64`를 쓰거나, 크기 제한이 없는 `int`를 쓰세요.
- 구조체는 값입니다. `let q = p;`는 복사합니다. `List`, `Dict`, `Set`은 Python처럼 공유되는 핸들입니다.
- `str` 슬라이스는 문자가 아니라 바이트를 셉니다. 문자 단위로 다니려면 `s.chars()`를 씁니다.

```rok
let name = "Ada";
let xs = [1, 2, 3];
print("no newline, ");
println("then one");
println("total:", xs.len(), xs);
println(1, 2, 3, sep: ", ");
println(f"{name} has {xs.len()} items: {xs:?}");
let x = 5;
println(f"{x=}");
```

```output
no newline, then one
total: 3 [1, 2, 3]
1, 2, 3
Ada has 3 items: [1, 2, 3]
x=5
```

## 제어 흐름

`if`는 식이라 `a if c else b` 자리도 대신합니다. `elif`는 `else if`이고, `for i in range(n)`은 `for i in 0..n`입니다.

```rok
fn grade(score: i32) -> str {
    if score >= 90 { "A" } else if score >= 80 { "B" } else { "C" }
}

let people = [("ada", 92), ("bob", 78)];
for (name, score) in people {
    println(f"{name}: {grade(score)}");
}
for (i, (name, _)) in people.enumerate() {
    println(f"{i} {name}");
}
for i in 0..3 {
    if i == 1 { continue; }
    println(i);
}
let label = if people.len() > 1 { "many" } else { "one" };
println(label);
let c = 5;
if 0 < c < 10 && c != 7 { println("in range"); }
if 3 in [1, 2, 3] { println("member"); }
```

```output
ada: A
bob: C
0 ada
1 bob
0
2
many
in range
member
```

`and`, `or`, `not`은 `&&`, `||`, `!`이고, `not in`은 `!in`입니다.

## 컴프리헨션은 반복자 체인으로

Rok에는 리스트, 딕셔너리, 집합 컴프리헨션이 없습니다. 컬렉션에 반복자 메서드가 있고, `to_list()`, `to_dict()`, `to_set()`으로 결과를 모읍니다. `sum`, `min`, `max`, `sorted`, `any`, `all`도 메서드입니다.

```rok
let xs = [1, 2, 3, 4, 5];
// [x * x for x in xs if x % 2 == 1]
let odd_squares = xs.filter(|x| x % 2 == 1).map(|x| x * x).to_list();
println(f"{odd_squares:?}");
println(f"{xs.sum()} {xs.min() ?? 0} {xs.max() ?? 0}");

let words = ["pear", "fig", "apple"];
println(f"{words.sorted():?}");
println(f"{words.sorted_by_key(|w| w.len()):?}");
println(words.join(", "));
println(f"{xs.any(|x| x > 4)} {xs.all(|x| x > 0)}");

// {w: len(w) for w in words}
let lengths = words.map(|w| (w, w.len())).to_dict();
println(lengths["fig"]);
println([1, 1, 2].to_set().len());

let (a, b) = (1, 2);
(a, b) = (b, a);
println(f"{a} {b}");
```

```output
[1, 9, 25]
15 1 5
["apple", "fig", "pear"]
["fig", "pear", "apple"]
pear, fig, apple
true true
3
2
2 1
```

Python의 `lambda x: x + 1`은 클로저 `|x| x + 1`입니다.

## `None` 대신 `none`

아무것도 못 찾을 수 있는 함수는 `T | none`을 반환합니다. `??`는 기본값을 주고, `x == none`은 검사하고, `if let`은 값을 꺼냅니다.

```rok
let xs = ["a", "b"];
let i = xs.position(|x| x == "b");
if let k: usize = i { println(f"found at {k}"); }
println(xs.position(|x| x == "z") ?? 99);
if !xs.is_empty() { println(f"{xs.len()} items"); }
```

```output
found at 1
99
2 items
```

## 에러

실패할 수 있는 함수는 `throws`라고 밝히고, 그 함수를 부를 때마다 `try`를 붙입니다. `throw "message"`는 간단한 에러를 던지고, `try .. catch`가 에러를 처리하며, `try?`는 실패를 `none`으로 바꿉니다.

```rok
fn parse_port(s: str) throws -> i64 {
    let n = try s.parse::<i64>();
    if n > 65535 { throw "port out of range"; }
    n
}

println(try parse_port("8080") catch _ { 0 });
let p = try parse_port("99999") catch e {
    println(f"error: {e}");
    0
};
println(p);
println(try? parse_port("x") ?? -1);
```

```output
8080
error: port out of range
0
-1
```

`int("42")`는 `try "42".parse::<int>()`이고, `float(s)`는 `try s.parse::<f64>()`입니다.

## 클래스는 구조체, 트레이트, `impl`로

구조체가 필드를 담고, `impl` 블록이 메서드를 담으며, `#[derive(..)]`가 `__eq__`, `__repr__`, `__hash__`를 대신 씁니다. 값을 바꾸는 메서드는 `ref self`를 받습니다.

```rok
#[derive(Debug, PartialEq)]
struct Point { x: f64; y: f64; }

impl Point {
    fn new(x: f64, y: f64) -> Point { Point { x: x, y: y } }
    fn norm(self) -> f64 { (self.x * self.x + self.y * self.y).sqrt() }
    fn scale(ref self, k: f64) { self.x *= k; self.y *= k; }
}

let p = Point::new(3.0, 4.0);
println(p.norm());
p.scale(2.0);
println(f"{p:?}");
println(p == Point { x: 6.0, y: 8.0 });
```

```output
5
Point { x: 6.0, y: 8.0 }
true
```

상속은 없습니다. 트레이트가 타입이 제공할 메서드를 나열하고(원하면 기본 본문도 줍니다), `any Shape`는 그것을 구현한 어떤 타입이든 담습니다.

```rok
trait Shape {
    fn area(self) -> f64;
    fn describe(self) -> str { f"area {self.area():.1}" }
}
struct Square { side: f64; }
struct Circle { r: f64; }
impl Shape for Square { fn area(self) -> f64 { self.side * self.side } }
impl Shape for Circle { fn area(self) -> f64 { f64::PI * self.r * self.r } }

let shapes: List<any Shape> = [Square { side: 2.0 }, Circle { r: 1.0 }];
for s in shapes { println(s.describe()); }

fn loud(x) -> str { x.to_uppercase() }
println(loud("hi"));
```

```output
area 4.0
area 3.1
HI
```

## 값, 핸들, 숫자

```rok
struct P { x: i32; }
let a = [1];
let b = a;
b.push(2);
println(f"{a:?}");
let p = P { x: 1 };
let q = p;
q.x = 9;
println(f"{p.x} {q.x}");

println(7 / 2);
println((-7).div_floor(2));
println(-7 %% 2);
println(7.0 / 2.0);
println((7 as f64) / 2.0);
println(2 ** 10);
```

```output
[1, 2]
1 9
3
-4
1
3.5
3.5
1024
```

`**`는 Python처럼 거듭제곱 연산자입니다. `2 ** 10`이고, `-x ** 2`는 `-(x ** 2)`입니다([21장](21-arrays-and-numerics.html)). 정수 밑은 `u32` 지수를 받으므로 `2 ** -1`은 에러이며, `2.0 ** -1.0`으로 쓰라고 알려 줍니다.

## `with`는 `using`으로

`using`으로 묶은 값은 스코프가 끝날 때 닫힙니다. `with` 블록이 파일을 닫는 것과 같습니다.

```rok
struct Conn { name: str; }
impl Drop for Conn {
    fn drop(ref self) { println(f"closed {self.name}"); }
}
fn work() {
    using c = Conn { name: "db" };
    println(f"using {c.name}");
}
work();
println("done");
```

```output
using db
closed db
done
```

## 파일과 텍스트

```rok
use rok::std::{env, fs, process};

let path = env::temp_dir() / f"pyrok_{process::id()}.txt";
try fs::write(path, "one\ntwo\n");
let text = try fs::read_to_str(path);
for line in text.lines() { println(line.to_uppercase()); }
try fs::remove_file(path);

println(42.to_str() + "px");
println(f"{3.14159:.2}");
println("ab".repeat(3));
println("a,b".split(",").to_list().len());
```

```output
ONE
TWO
42px
3.14
ababab
2
```

## Python 관용구를 Rok으로

| Python | Rok |
|---|---|
| `print(x)` / `print(x, end="")` | `println(x)` / `print(x)` |
| `print(a, b, sep=", ")` | `println(a, b, sep: ", ")` |
| `f"{x!r}"` | `f"{x:?}"` |
| `a and b`, `a or b`, `not a` | `a && b`, `a \|\| b`, `!a` |
| `x not in xs` | `x !in xs` |
| `x is None`, `x is not None` | `x == none`, `x != none` |
| `None`, `True`, `False` | `none`, `true`, `false` |
| `a if c else b` | `if c { a } else { b }` |
| `elif` | `else if` |
| `def f(x):` / `pass` | `fn f(x) { }` (빈 블록에는 `pass`가 필요 없습니다) |
| `lambda x: x + 1` | `\|x\| x + 1` |
| `if xs:` | `if !xs.is_empty()` |
| `len(xs)`, `sorted(xs)`, `sum(xs)` | `xs.len()`, `xs.sorted()`, `xs.sum()` |
| `min(a, b)`, `abs(x)`, `round(x)` | `a.min(b)`, `x.abs()`, `x.round()` |
| `str(x)` | `x.to_str()` 또는 `f"{x}"` |
| `int(s)`, `float(s)` | `try s.parse::<int>()`, `try s.parse::<f64>()` |
| `", ".join(xs)` | `xs.join(", ")` |
| `s * 3` | `s.repeat(3)` |
| `range(n)`, `range(a, b)` | `0..n`, `a..b` |
| `for i, x in enumerate(xs)` | `for (i, x) in xs.enumerate()` |
| `for k, v in d.items()` | `for (k, v) in d` |
| `[f(x) for x in xs if c]` | `xs.filter(\|x\| c).map(\|x\| f(x)).to_list()` |
| `x ** 2`, `n // 2` | `x ** 2`, `n.div_floor(2)` |
| `raise ValueError("m")` / `except E as e` | `throw "m"` / `catch e: E` |
| `with open(p) as f:` | `using f = try File::open(p);` |
| `assert x == 2, "m"` | `assert(x == 2, "m");` |
| `self`를 쓰는 `class C:` | `struct C { .. }`와 `impl C { fn m(self) { .. } }` |
| `C("h", 1)` | `C { host: "h", port: 1 }` |
| `import m` / `from m import x` | `use m;` / `use m::x;` |

이 Python 형태를 Rok에 쓰면 빌드가 에러로 멈추고, 에러의 help가 대신 쓸 Rok 형태를 보여 줍니다.

## Python 모듈을 Rok에서

| Python | Rok |
|---|---|
| `math` | 숫자의 메서드(`x.sqrt()`, `x.sin()`, `x.floor()`)와 `f64::PI`. import할 것이 없습니다 |
| `os`, `sys` | `rok::std::env`(`env::args()`, `env::var(..)`), `rok::std::process`(`process::exit(..)`, `process::id()`) |
| `os.path`, `pathlib` | `rok::std::path::Path`(`Path::new("a") / "b"`), `rok::std::fs` |
| `open`, `shutil` | `rok::std::fs`(`fs::read_to_str`, `fs::write`, `File::open`) |
| `subprocess` | `rok::std::process::Command` |
| `random` | `rok::std::random`(`random::range(1..7)`, `random::choice(xs)`, `random::shuffle(xs)`) |
| `time` | `rok::core::time`(`Instant::now()`, `Duration`) |
| `threading` | `rok::std::thread`(`thread::spawn(..)`, `join`) |
| `asyncio` | 언어에 들어 있는 `async fn`과 `await` |
| `collections` | 기본으로 있는 `Dict`, `Set`, `Deque`, `BinaryHeap` |
| `itertools`, `functools` | 반복자 메서드: `zip`, `chain`, `windows`, `chunks`, `fold`, `reduce`, `scan` |
| `json` | `json` stone: `rok add json` |
| `input()` | `try io::input("prompt")` |
| `unittest`, `pytest` | `#[test]` 함수와 `rok test` |
| `black`, `ruff` | `rok fmt`, `rok lint` |
| `pip`, `venv` | `rok add`, `rok env` |

`re`, `datetime`, `logging`, `argparse`에 해당하는 것은 아직 표준 라이브러리에 없습니다.
