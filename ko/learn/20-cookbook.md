# 20. 어떻게 하나요

일상적인 작업에 대한 짧은 답입니다. 각각 파일에 붙여 넣고 `rok run file.rok` 으로 실행할 수 있는 완전한 프로그램이며, 설명하는 장으로 가는 링크가 붙어 있습니다.

## 파일 읽기

`fs::read_to_str` 은 파일 전체를 텍스트로 읽고, `fs::write` 는 파일을 씁니다. 둘 다 `IoError` 를 던지므로 호출에 `try` 를 붙입니다 ([13장](13-io.html#파일-읽고-쓰기)).

```rok
use rok::std::{env, fs};

let path = env::temp_dir() / f"rok_cookbook_{env::pid()}.txt";
try fs::write(path, "one\ntwo\nthree\n");

let text = try fs::read_to_str(path);
for line in text.lines() {
    println(line);
}
try fs::remove_file(path);
```

```output
one
two
three
```

## 표준 입력을 한 줄씩 읽기

`io::stdin().lines()` 의 각 원소는 줄바꿈이 빠진 한 줄이거나, 입력을 끝낸 에러입니다. `try item.get()` 이 줄을 꺼냅니다 ([13장](13-io.html#표준-io-와-환경)). 이 페이지는 입력 없이 빌드되므로 아래 출력은 `0 lines` 입니다. 직접 실행하면서 파일을 파이프로 넣어 보십시오 (`rok run count.rok < file`).

```rok
use rok::std::io;

let count = 0;
for item in io::stdin().lines() {
    let line = try item.get();
    count += 1;
    println(f"{count}: {line}");
}
println(f"{count} lines");
```

```output
0 lines
```

`printf 'a\nb\n' | rok run lines.rok` 로 실행하면 줄에 번호를 붙이는 것을 볼 수 있습니다.

## 명령줄 인자 얻기

`env::args()` 는 `List<str>` 이며, 원소 0 은 프로그램 자신입니다. 인자는 `--` 뒤에 넘깁니다: `rok run greet.rok -- ada grace` ([13장](13-io.html#표준-io-와-환경), [예제](../examples/cli-args.html)). 이 페이지는 인자 없이 빌드되므로 아래 출력은 `0 arguments` 입니다. 직접 인자를 주어 실행해 보십시오.

```rok
use rok::std::env;

let args = env::args();
for name in args.iter().skip(1) {
    println(f"hello, {name}");
}
println(f"{args.len() - 1} arguments");
```

```output
0 arguments
```

## 종료 코드와 함께 끝내기

`env::exit(code)` 는 프로그램을 즉시 끝냅니다. `finally` 블록과 `Drop` 은 실행되지 않습니다. 스크립트의 최상위에서 잡히지 않은 에러는 저절로 코드 1 로 종료합니다 ([10장](10-errors.html#스크립트-최상위의-에러)).

```rok build
use rok::std::env;

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    env::exit(2);
}
println(f"hello, {args[1]}");
```

## 잠들기, 그리고 걸린 시간 재기

`Instant::now()` 는 단조 시계 (monotonic clock) 를 읽고, `elapsed()` 는 `Duration` 을 줍니다. `thread::sleep` 은 현재 스레드를 막습니다. async 코드에서는 대신 `await task::sleep(..)` 을 씁니다 ([12장](12-concurrency.html#async-와-task), [`Duration` API](../../std/core/task/time.html#struct.Duration)).

```rok
use rok::std::thread;

let start = Instant::now();
thread::sleep(Duration::ms(20));
let took = start.elapsed();
println(f"slept at least 20 ms: {took.as_ms() >= 20}");
println(f"{took.as_secs_f64() < 5.0}");
```

```output
slept at least 20 ms: true
true
```

## 태스크를 동시에 실행하고 기다리기

`task::spawn` 은 future 를 태스크로 시작하고, `task::join` 은 그것들을 모두 기다려 결과를 spawn 한 순서대로 돌려줍니다. 최상위에 `await` 가 하나라도 있으면 스크립트의 `main` 이 async 가 되며, `task::spawn` 에는 그것이 필요합니다 ([12장](12-concurrency.html#async-와-task)).

```rok
async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms(10));
    f"item-{id}"
}

let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2)), task::spawn(fetch(3))];
let items = await task::join(tasks);
println(f"{items:?}");
```

```output
["item-1", "item-2", "item-3"]
```

## 스레드 사이에서 카운터나 리스트 공유하기

수에는 `Atomic` 을, 그 밖의 것에는 `Mutex` 를 씁니다. 잠금 가드는 `using` 으로 바인딩하며, 블록 끝에서 해제됩니다. 여기서 `[ref x]` 캡처는 스레드 사이에서 안전하지 않으므로 컴파일되지 않습니다 ([12장](12-concurrency.html#mutex-와-atomic)).

```rok
use rok::std::thread;

let hits = Atomic::new(0);
let seen = Mutex::new([]);

let hs: List<thread::JoinHandle<()>> = [];
for i in 1..=4 {
    hs.push(thread::spawn(|| {
        hits.fetch_add(1);
        using g = seen.lock() { g.get().push(i); }
    }));
}
for h in hs { try h.join(); }

let all = using g = seen.lock() { g.get().clone() };
all.sort();
println(f"{hits.load()} {all:?}");
```

```output
4 [1, 2, 3, 4]
```

## 에러 처리하기

실패할 수 있는 함수에는 `throws` 를, 각 호출에는 `try` 를 씁니다. `catch` 는 에러가 난 자리에서 처리하고, `(try? ..) ?? default` 는 에러를 기본값으로 바꿉니다 ([10장](10-errors.html)).

```rok
fn parse_port(s: str) throws -> u16 {
    let n: u16 = try s.parse();
    if n < 1024 { throw ValueError { msg: f"port {n} is reserved" }; }
    n
}

let a = try parse_port("8080");
let b = try parse_port("80") catch e { println(f"error: {e}"); 8000 };
let c = (try? parse_port("abc")) ?? 3000;
println(f"{a} {b} {c}");
```

```output
error: port 80 is reserved
8080 8000 3000
```

## JSON 파싱하기

JSON 은 표준 라이브러리가 아니라 [rokstones.io](https://rokstones.io) 의 `json` stone 에 있습니다. 패키지에서는 `rok add json` 으로, 스크립트에서는 `rok add json app.rok` 으로 추가합니다 ([18장](18-features-and-stones.html#rok-프로젝트의-stone)).

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
} }
```

```rok norun
#[derive(Json)]
struct User { name: str; age: i32; tags: List<str> = []; }

// Untyped: walk a Value tree.
let v = try json::parse("{\"name\": \"ada\", \"langs\": [\"rok\", \"c\"]}");
println(v.get("name")?.as_str() ?? "?");          // ada
println(v.pointer("/langs/0")?.as_str() ?? "?");  // rok

// Typed: decode into your own struct, and back.
let u: User = try json::from_str("{\"name\": \"grace\", \"age\": 85}");
println(f"{u.name} {u.age}");                      // grace 85
println(json::to_string(u));                       // {"name":"grace","age":85,"tags":[]}
```

## HTTP 요청 보내기, HTTP 서비스하기

HTTP 는 `http` stone 이며, 위의 `json` 과 같은 방법으로 추가합니다 (`rok add http`, 스크립트는 `rok add http app.rok`). 클라이언트와 서버가 async 이므로 스크립트는 `await` 를 씁니다 ([12장](12-concurrency.html#async-와-task), [18장](18-features-and-stones.html#rok-프로젝트의-stone)).

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

요청:

```rok norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

서버: `async fn` 하나가 모든 요청을 처리하며, 요청마다 자기 태스크에서 실행됩니다.

```rok norun
use http::{Request, Response};

async fn handle(req: Request) -> Response {
    Response::text(f"hello from {req.uri.path}\n")
}

println("listening on http://127.0.0.1:8080");
try await http::serve("127.0.0.1:8080", handle);
```

```text
$ curl http://127.0.0.1:8080/hi
hello from /hi
```

## 더 읽을거리

- [예제](../examples/index.html): 더 긴 완전한 프로그램들입니다. [명령줄 도구](../examples/cli-args.html), [TCP echo 서버](../examples/tcp-echo.html), 처음부터 작성한 [JSON parser](../examples/json-parser.html) 등이 있습니다.
- 표준 라이브러리 API: [`fs`](../../std/std/fs.html), [`io`](../../std/std/io.html), [`env`](../../std/std/env.html), [`thread`](../../std/std/thread.html), [`task`](../../std/core/task.html), [`sync`](../../std/core/sync.html)
