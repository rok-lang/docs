# ROKON 설정

ROKON 설정을 기본값이 채워진 `#[derive(Rokon)]` struct로 읽어 검증하고 고친 다음 다시 ROKON으로 씁니다.

## 코드

```rok
// Load a server configuration from ROKON, validate it, change it and write it back.
use rok::std::rokon;
use rok::std::rokon::RokonError;

#[derive(Rokon, Debug, PartialEq)]
enum Level { Debug, Info, Warn, Error }

#[derive(Rokon, Debug)]
struct Listen {
    host: str = "127.0.0.1";
    port: u16 = 8080;
}

#[derive(Rokon, Debug)]
struct Route {
    path: str;
    target: str;
    timeout_ms: u32 = 5000;
}

#[derive(Rokon, Debug)]
struct Tls {
    cert: str;
    key: str;
}

#[derive(Rokon, Debug)]
struct Config {
    name: str;
    listen: Listen;
    workers: i32 = 4;
    log: Level = Level::Info;
    routes: List<Route> = [];
    limits: Dict<str, i64> = Dict::new();
    tls: Tls | none;
}

const SOURCE: str = "// gateway.rokon
Config {
    name: \"gateway\",
    listen: { port: 9000 },
    log: Level::Debug,
    routes: [
        Route { path: \"/api\", target: \"http://10.0.0.5:8000\" },
        Route { path: \"/static\", target: \"file:///srv/www\", timeout_ms: 500 },
    ],
    limits: { \"body_kb\": 512, \"conns\": 1000 },
}";

fn validate(c: Config) -> List<str> {
    let problems: List<str> = [];
    if c.listen.port == 0u16 { problems.push("listen.port must not be 0"); }
    if c.workers < 1 || c.workers > 64 { problems.push(f"workers must be 1..=64, got {c.workers}"); }
    for r in c.routes {
        if !r.path.starts_with("/") { problems.push(f"route '{r.path}' must start with '/'"); }
    }
    if c.log == Level::Debug && !(c.tls is none) { problems.push("debug logging with TLS leaks secrets"); }
    problems
}

fn load(text: str) throws (RokonError) -> Config {
    let c: Config = try rokon::from_str(text);
    c
}

fn report(label: str, text: str) {
    let c = try load(text) catch e {
        println(f"{label}: parse error {e}");
        return;
    };
    let problems = validate(c);
    if problems.is_empty() { println(f"{label}: ok"); }
    for p in problems { println(f"{label}: {p}"); }
}

fn main() throws (RokonError) {
    let c = try load(SOURCE);
    println(f"{c.name} on {c.listen.host}:{c.listen.port}, {c.workers} workers, log {c.log:?}");
    for r in c.routes {
        println(f"  {r.path:<8} -> {r.target} ({r.timeout_ms} ms)");
    }
    println(f"  tls off: {c.tls is none}  body limit: {c.limits["body_kb"]} KB");

    report("good", SOURCE);
    report("bad", "Config { name: \"x\", listen: { port: 0 }, workers: 100, routes: [Route { path: \"api\", target: \"t\" }] }");
    report("typo", "Config { name: \"x\", listen: {}, wokers: 2 }");
    report("type", "Config {\n    name: \"x\",\n    listen: { port: \"80\" },\n}");

    // Modify and serialize back; defaults are written out explicitly.
    c.log = Level::Warn;
    c.workers = 8;
    c.tls = Tls { cert: "/etc/gw/cert.pem", key: "/etc/gw/key.pem" };
    c.routes.push(Route { path: "/health", target: "internal:health", timeout_ms: 100u32 });
    let out = rokon::to_string(c);
    println(out);

    let back = try load(out);
    println(f"round trip: {rokon::to_string(back) == out}");
}
```

```output
gateway on 127.0.0.1:9000, 4 workers, log Debug
  /api     -> http://10.0.0.5:8000 (5000 ms)
  /static  -> file:///srv/www (500 ms)
  tls off: true  body limit: 512 KB
good: ok
bad: listen.port must not be 0
bad: workers must be 1..=64, got 100
bad: route 'api' must start with '/'
typo: parse error 1:33: unknown field `wokers` in Config
type: parse error 3:21: expected u16, found string
Config {
    name: "gateway",
    listen: Listen { host: "127.0.0.1", port: 9000 },
    workers: 8,
    log: Level::Warn,
    routes: [
        Route { path: "/api", target: "http://10.0.0.5:8000", timeout_ms: 5000 },
        Route { path: "/static", target: "file:///srv/www", timeout_ms: 500 },
        Route { path: "/health", target: "internal:health", timeout_ms: 100 },
    ],
    limits: { "body_kb": 512, "conns": 1000 },
    tls: Tls { cert: "/etc/gw/cert.pem", key: "/etc/gw/key.pem" },
}
round trip: true
```

## 설명

- `#[derive(Rokon)]` 하나로 struct와 enum을 ROKON과 서로 변환할 수 있습니다. 중첩 struct(`Listen`), enum(`Level`), `List`, `Dict`, `Tls | none`이 모두 그대로 대응합니다([ROKON](../learn/15-rokon.html), [3.9.2 Rok 타입과 대응](../spec/refs-standard-libraries/300-std/309-rokon.html#392-rok-타입과-대응)).
- 문서에 없는 필드는 struct의 기본값(`workers: i32 = 4`)을 받습니다. 기본값이 없는 `T | none` 필드는 `none`이 됩니다. `listen: { port: 9000 }`처럼 타입 이름 없이 적은 중첩 값도 받아들입니다.
- `rokon::from_str`은 알 수 없는 필드나 타입이 틀린 값을 만나면 `RokonError`를 던집니다. 메시지에는 `line:col` 위치를 담습니다. `wokers` 같은 오타를 조용히 무시하지 않습니다([`rokon::from_str`](../../std/std/rokon.html#fn.from_str)).
- 형식을 넘어선 규칙은 `validate`가 모아서 한꺼번에 알립니다. 포트는 0이 아니어야 하고 워커 수는 범위 안에 있어야 한다는 규칙입니다. `T | none` 값이 비어 있는지는 `c.tls is none`으로 확인합니다.
- `rokon::to_string`은 정규 형태로 씁니다. 기본값으로 채운 필드도 함께 쓰므로 출력을 다시 읽으면 같은 값이 나옵니다. `Dict`는 키 순서를 지키므로 limits는 읽은 순서 그대로 나옵니다([`rokon::to_string`](../../std/std/rokon.html#fn.to_string)).
