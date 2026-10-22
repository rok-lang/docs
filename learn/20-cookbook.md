# 20. How Do I...

Short answers to everyday tasks, each a complete program you can paste into a file and run with `rok run file.rok`. Each recipe links to the chapter that explains it.

## Read a file

`fs::read_to_str` reads a whole file as text; `fs::write` writes one. Both throw an `IoError`, so the calls are marked with `try` ([chapter 13](13-io.html#reading-and-writing-files)).

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

## Read stdin line by line

Each item of `io::stdin().lines()` is a line without its newline, or the error that ended the input; `try item.get()` takes the line out ([chapter 13](13-io.html#standard-io-and-the-environment)). The output below is `0 lines` because this page builds with no input; run it locally and pipe a file in (`rok run count.rok < file`).

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

Run it as `printf 'a\nb\n' | rok run lines.rok` to see it number the lines.

## Get command-line arguments

`env::args()` is a `List<str>`; element 0 is the program itself. Pass arguments after `--`: `rok run greet.rok -- ada grace` ([chapter 13](13-io.html#standard-io-and-the-environment), [example](../examples/cli-args.html)). The output below is `0 arguments` because this page builds with none; run it locally with some.

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

## Exit with a code

`env::exit(code)` ends the program at once. `finally` blocks and `Drop` do not run. An uncaught error at the top of a script exits with code 1 by itself ([chapter 10](10-errors.html#errors-at-the-top-of-a-script)).

```rok build
use rok::std::env;

let args = env::args();
if args.len() < 2 {
    eprintln("usage: greet NAME");
    env::exit(2);
}
println(f"hello, {args[1]}");
```

## Sleep, and measure how long something takes

`Instant::now()` reads a monotonic clock, and `elapsed()` gives a `Duration`. `thread::sleep` blocks the current thread; in async code use `await task::sleep(..)` instead ([chapter 12](12-concurrency.html#async-and-tasks), [`Duration` API](../std/core/task/time.html#struct.Duration)).

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

## Run tasks concurrently and wait for them

`task::spawn` starts a future as a task, and `task::join` waits for all of them and returns their results in spawn order. Any top-level `await` makes the script's `main` async, which `task::spawn` needs ([chapter 12](12-concurrency.html#async-and-tasks)).

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

## Share a counter or a list across threads

Use `Atomic` for a number and `Mutex` for anything else. A lock guard is bound with `using`, which releases it at the end of the block. A `[ref x]` capture won't compile here, because it isn't safe across threads ([chapter 12](12-concurrency.html#mutex-and-atomic)).

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

## Handle errors

Write `throws` on a function that can fail and `try` on each call. `catch` handles an error where it happens; `(try? ..) ?? default` swaps it for a default ([chapter 10](10-errors.html)).

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

## Parse JSON

JSON lives in the `json` stone on [rokstones.io](https://rokstones.io), not in the standard library. Add it with `rok add json` in a package, or `rok add json app.rok` for a script ([chapter 18](18-features-and-stones.html#the-rok-stones)).

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

## Make an HTTP request, or serve HTTP

HTTP is the `http` stone, added the same way as `json` above (`rok add http`, or `rok add http app.rok` for a script). Its client and server are async, so the script uses `await` ([chapter 12](12-concurrency.html#async-and-tasks), [chapter 18](18-features-and-stones.html#the-rok-stones)).

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "http": Dep::version("^0.1.0"),
} }
```

A request:

```rok norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

A server: one `async fn` handles every request, each in its own task.

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

## Further reading

- [Examples](../examples/index.html): longer complete programs, among them a [command-line tool](../examples/cli-args.html), a [TCP echo server](../examples/tcp-echo.html), and a [JSON parser](../examples/json-parser.html) written from scratch.
- Standard library API: [`fs`](../std/std/fs.html), [`io`](../std/std/io.html), [`env`](../std/std/env.html), [`thread`](../std/std/thread.html), [`task`](../std/core/task.html), [`sync`](../std/core/sync.html)
