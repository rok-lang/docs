# 15. ROKON

이 장에서는 Rok의 데이터 형식인 ROKON(Rok Object Notation)으로 설정 파일을 읽고 쓰는 법을 다룹니다. 새로 배울 문법은 없습니다. ROKON 파일은 Rok 소스에서 값을 적을 때와 같은 리터럴 문법을 씁니다. 패키지 매니페스트 `rok.rokon`도 이 형식입니다.

## 형식 한눈에 보기

파일 하나에 값 하나가 들어갑니다. 보통은 struct 하나입니다.

```rokon
// app.rokon
Config {
    name: "demo",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

| 형태 | 예 |
|------|----|
| 기본 값 | `none`, `true`, `42`, `0xff`, `3.14`, `'a'`, `"text"` |
| 리스트 / 튜플 | `[1, 2, 3]`, `("x", 1)` |
| dict | `{ "k": 1, "j": 2 }` (키는 값) |
| struct | `Point { x: 1, y: 2 }`, `Pair(1, 2)` |
| enum 변형 | `Mode::Fast`, `Shape::Circle(1.0)`, `Shape::Rect { w: 2, h: 3 }` |

`//` 주석과 끝에 붙는 쉼표는 쓸 수 있습니다. 변수, 연산자, 함수 호출, f-string 같은 "코드"는 쓸 수 없습니다. ROKON은 실행되는 일이 없는 순수한 데이터입니다.

## `#[derive(Rokon)]`과 `from_str` / `to_string`

struct나 enum에 `#[derive(Rokon)]`을 붙이면 `rok::std::rokon`의 `from_str`로 읽고 `to_string`으로 쓸 수 있습니다. 어떤 타입으로 읽을지는 변수의 타입 표기가 정합니다.

```rok
use rok::std::rokon;
use rok::std::rokon::RokonError;

#[derive(Rokon)]
enum Mode { Fast, Slow }

#[derive(Rokon)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Rokon)]
struct Config {
    name: str;
    mode: Mode;
    servers: List<Server>;
    limits: OrderedDict<str, i32>;
}

fn main() throws (RokonError) {
    let text = r#"
Config {
    name: "demo",
    mode: Mode::Fast,
    servers: [
        Server { host: "a.example", port: 8080, tls: true },
        Server { host: "b.example" },
    ],
    limits: { "cpu": 2, "mem": 512 },
}
"#;
    let cfg: Config = try rokon::from_str(text);
    println(f"{cfg.name}: {cfg.servers.len()} servers");
    for s in cfg.servers {
        println(f"  {s.host}:{s.port} tls={s.tls ?? false}");
    }
    println(f"cpu limit: {cfg.limits["cpu"]}");
    println(rokon::to_string(cfg));
}
```

```output
demo: 2 servers
  a.example:8080 tls=true
  b.example:80 tls=false
cpu limit: 2
Config {
    name: "demo",
    mode: Mode::Fast,
    servers: [
        Server { host: "a.example", port: 8080, tls: true },
        Server { host: "b.example", port: 80, tls: none },
    ],
    limits: { "cpu": 2, "mem": 512 },
}
```

중첩된 struct, 리스트, enum도 모두 같은 방식으로 읽습니다. 두 번째 서버처럼 필드가 빠져 있으면 다음 규칙을 따릅니다.

- 필드에 `= default`가 있으면 그 값(`port: u16 = 80`).
- 타입이 `T | none`이면 `none`(`tls`).
- 그 밖에는 에러.

`to_string`은 **정규 형식**(canonical form)으로 씁니다. 들여쓰기는 4칸입니다. 값이 100열 안에 들어가면 한 줄로 쓰고, 아니면 원소마다 한 줄씩 끝에 쉼표를 붙여 씁니다. 정규 형식을 다시 읽으면 같은 값이 됩니다.

## enum, 튜플 struct, Dict

enum 변형은 언제나 `Enum::Variant`처럼 전체 경로로 적습니다. 튜플 struct는 `Name(a, b)`로 적습니다. 표준 컬렉션과 튜플은 derive 없이 바로 읽을 수 있습니다.

```rok
use rok::std::rokon;

#[derive(Rokon)]
enum Shape {
    Dot,
    Circle(f64),
    Rect { w: i32; h: i32; },
}

#[derive(Rokon)]
struct Point(i32, i32);

let shapes: List<Shape> = try! rokon::from_str("[Shape::Dot, Shape::Circle(1.5), Shape::Rect { w: 3, h: 2 }]");
for s in shapes {
    let d = match s {
        Shape::Dot => "dot",
        Shape::Circle(r) => f"circle r={r}",
        Shape::Rect { w, h } => f"rect {w}x{h}",
    };
    println(d);
}

println(rokon::to_string(Point(3, -4)));
let p: Point = try! rokon::from_str("Point(1, 2)");
println(f"{p.0} {p.1}");

let ages: Dict<str, i32> = try! rokon::from_str(r#"{ "kim": 31, "lee": 27 }"#);
println(f"lee is {ages["lee"]}");
let pair: (str, List<i32>) = try! rokon::from_str(r#"("primes", [2, 3, 5])"#);
println(f"{pair.0}: {pair.1:?}");
```

```output
dot
circle r=1.5
rect 3x2
Point(3, -4)
1 2
lee is 27
primes: [2, 3, 5]
```

> [!TIP]
> `Dict`와 `Set`은 해시 순서로 쓰이므로 `to_string` 출력의 순서가 실행마다 같다는 보장이 없습니다. 파일에 쓸 때 순서를 지키려면 `OrderedDict` / `OrderedSet`을 쓰세요.

## 잘못된 입력과 에러

`from_str`은 무언가 잘못되면 `RokonError`를 던집니다. 메시지는 `line:column: description` 형태이므로 파일에서 그 자리를 바로 찾을 수 있습니다.

```rok
use rok::std::rokon;

#[derive(Rokon)]
struct Server { host: str; port: u16; }

fn check(text: str) {
    let s: Server = try rokon::from_str(text) catch e {
        println(f"error: {e}");
        return;
    };
    println(f"ok: {s.host}:{s.port}");
}

check(r#"Server { host: "h", port: 22 }"#);
check(r#"Server { host: "h" }"#);
check(r#"Server { host: "h", port: 70000 }"#);
check(r#"Server { host: h, port: 22 }"#);
check(r#"Server { host: "h", port: 22, user: "me" }"#);
check(r#"Client { host: "h", port: 22 }"#);
```

```output
ok: h:22
error: 1:1: missing field `port` in Server
error: 1:27: integer 70000 is out of range for u16
error: 1:16: bare identifier `h` is not allowed in ROKON (strings are quoted, variants are written `Enum::h`)
error: 1:31: unknown field `user` in Server
error: 1:1: expected Server, found `Client { .. }`
```

모르는 필드도 에러입니다. 그래서 설정 키를 잘못 적어도 조용히 무시되지 않습니다.

## 타입 없이 읽기: `Value`

구조를 미리 알 수 없는 데이터는 `rokon::parse`로 `Value` enum에 읽어 들일 수 있습니다. `Value`에는 `None`, `Int`, `Str`, `List`, `Dict`, `Struct`, `Variant` 같은 변형이 있습니다. `kind()`는 그 종류의 이름을 돌려줍니다.

```rok
use rok::std::rokon;
use rok::std::rokon::Value;

let v = try! rokon::parse(r#"Site { title: "rok", pages: 12, draft: false }"#);
match v {
    Value::Struct { name, fields } => {
        println(f"struct {name ?? "(anonymous)"}");
        for (k, x) in fields {
            println(f"  {k}: {x.kind()} = {x}");
        }
    },
    _ => println(f"not a struct: {v.kind()}"),
}
```

```output
struct Site
  title: string = "rok"
  pages: integer = 12
  draft: bool = false
```

## 설정 파일 읽고 쓰기

13장의 `fs`와 함께 쓰면 설정 파일을 다룰 수 있습니다. 이 사이트를 만드는 프로그램이 바로 그렇게 합니다. 사이트 저장소의 `site.rokon`을 `#[derive(Rokon)] struct Site`로 읽습니다. 기본값이 있는 필드는 파일에서 생략할 수 있습니다.

아래 예제는 설정을 임시 파일에 쓰고 읽어서 고친 뒤 다시 씁니다. `load`는 `IoError`와 `RokonError`를 모두 던질 수 있고, 호출하는 쪽은 각 종류를 따로 잡습니다.

```rok
use rok::std::fs;
use rok::std::env;
use rok::std::rokon;
use rok::std::rokon::RokonError;
use rok::std::io::IoError;

#[derive(Rokon)]
struct Site {
    title: str;
    tagline: str;
    spec: str = "../spec/ko";
    out: str = "public";
}

fn load(path: str) throws (IoError | RokonError) -> Site {
    try rokon::from_str(try fs::read_to_str(path))
}

let path = (env::temp_dir() / f"rok_learn15_{env::pid()}.rokon").as_str();
try! fs::write(path, r#"// Site configuration
Site {
    title: "rok",
    tagline: "statically typed, rich library",
}
"#);
let site = try! load(path);
println(f"{site.title} -> {site.out}");

site.out = "dist";
try! fs::write(path, rokon::to_string(site) + "\n");
print(try! fs::read_to_str(path));

try! fs::write(path, "Site { title: 1 }");
try load(path) catch e: RokonError { println(f"bad config: {e}"); };
try! fs::remove_file(path);
try load(path) catch e: IoError { println(f"missing: {e.kind():?}"); };
```

```output
rok -> public
Site { title: "rok", tagline: "statically typed, rich library", spec: "../spec/ko", out: "dist" }
bad config: 1:15: expected str, found integer
missing: NotFound
```

`to_string`으로 다시 쓰면 주석이 사라집니다. 사람이 편집하는 파일은 프로그램이 덮어쓰지 않는 편이 좋습니다. `rok fmt`는 `.rokon` 파일의 들여쓰기만 고치고, 주석과 배치는 그대로 둡니다.

## 요약

- ROKON은 Rok 리터럴 문법을 그대로 쓰는 데이터 형식이고, 확장자는 `.rokon`입니다.
- `#[derive(Rokon)]`을 붙이고, `rokon::from_str`로 읽고, `rokon::to_string`으로 씁니다.
- 빠진 필드는 `= default` 값을, 타입이 `T | none`이면 `none`을 받고, 그 밖에는 에러입니다. 모르는 필드도 에러입니다.
- 에러 메시지는 `line:column: description` 형태입니다.
- 구조를 모르면 `rokon::parse`로 `Value`를 얻습니다.
- 순서가 중요한 맵에는 `OrderedDict`를 씁니다.

## 더 읽을거리

- 사양: [3.9 ROKON](../spec/refs-standard-libraries/300-std/309-rokon.html), [필드 기본값](../spec/refs-lang/002-types/203-struct.html#233-필드-기본값), [rok.rokon 매니페스트](../spec/refs-lang/009-tools/910-rok-frontend.html)
- API: [rok::std::rokon](../../std/std/rokon.html)
- 다음 장: [16. 도구](16-tools.html)
