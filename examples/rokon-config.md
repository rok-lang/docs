# ROKON configuration

Read a ROKON configuration into `#[derive(Rokon)]` structs with defaults filled in, validate it, change it and write it back as ROKON.

## Code

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
    limits: OrderedDict<str, i64> = OrderedDict::new();
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

## Walkthrough

- A single `#[derive(Rokon)]` makes structs and enums convert to and from ROKON. A nested struct (`Listen`), an enum (`Level`), `List`, `OrderedDict` and `Tls | none` all map directly ([ROKON](../learn/15-rokon.html), [3.9.2 mapping to rok types](../spec/refs-standard-libraries/300-std/309-rokon.html#392-correspondence-with-rok-types)).
- Fields missing from the document take the struct's default (`workers: i32 = 4`); a `T | none` field without a default becomes `none`. Nested values written without a type name, like `listen: { port: 9000 }`, are accepted too.
- `rokon::from_str` throws a `RokonError` for an unknown field or a value of the wrong type, with a `line:col` position in the message. A typo such as `wokers` is not silently ignored ([`rokon::from_str`](../std/std/rokon.html#fn.from_str)).
- Rules beyond the format (port must not be 0, worker count in range) are collected by `validate` and reported together. Whether a `T | none` value is empty is checked with `c.tls is none`.
- `rokon::to_string` writes the canonical form. Fields filled from defaults are written out too, so reading the output gives the same value back. `OrderedDict` is used instead of `Dict` to keep the key order ([`rokon::to_string`](../std/std/rokon.html#fn.to_string)).
