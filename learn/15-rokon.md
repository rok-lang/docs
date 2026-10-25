# 15. ROKON

This chapter shows how to read and write configuration files in ROKON (Rok Object Notation), rok's data format. There is no new syntax to learn: a ROKON file uses the same literal syntax you write values with in rok source. The package manifest `rok.rokon` uses this format too.

## The format at a glance

A file holds one value, usually a single struct.

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

| Form | Example |
|------|----|
| Basic values | `none`, `true`, `42`, `0xff`, `3.14`, `'a'`, `"text"` |
| List / tuple | `[1, 2, 3]`, `("x", 1)` |
| dict | `{ "k": 1, "j": 2 }` (keys are values) |
| struct | `Point { x: 1, y: 2 }`, `Pair(1, 2)` |
| enum variant | `Mode::Fast`, `Shape::Circle(1.0)`, `Shape::Rect { w: 2, h: 3 }` |

`//` comments and trailing commas are allowed. "Code" such as variables, operators, function calls, and f-strings is not. ROKON is pure data that never executes.

## `#[derive(Rokon)]` with `from_str` / `to_string`

Put `#[derive(Rokon)]` on a struct or enum and you can read it with `from_str` and write it with `to_string` from `rok::std::rokon`. The variable's type annotation decides which type is read.

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

Nested structs, lists, and enums are all read the same way. When a field is missing, as in the second server, these rules apply:

- If the field has a `= default`, that value (`port: u16 = 80`).
- If the type is `T | none`, `none` (`tls`).
- Otherwise, an error.

`to_string` writes the **canonical form**: 4-space indentation, one line if it fits in 100 columns, otherwise one element per line with a trailing comma. Reading the canonical form back yields the same value.

## Enums, tuple structs, Dict

Enum variants are always written with their full path, like `Enum::Variant`. Tuple structs are written `Name(a, b)`. Standard collections and tuples are read directly, without a derive.

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
> `Dict` and `Set` are written in hash order, so the order of `to_string` output is not guaranteed to be the same across runs. To keep the order when writing to a file, use `OrderedDict` / `OrderedSet`.

## Bad input and errors

`from_str` throws `RokonError` when something is wrong. Messages have the form `line:column: description`, so a person can find the spot in the file right away.

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

Note that an unknown field is also an error, so a misspelled configuration key is never silently ignored.

## Reading without a type: `Value`

Data whose structure you don't know in advance can be read into the `Value` enum with `rokon::parse`. `Value` has variants such as `None`, `Int`, `Str`, `List`, `Dict`, `Struct`, and `Variant`, and `kind()` returns the name of the kind.

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

## Reading and writing configuration files

Combined with `fs` from chapter 13, this handles configuration files. The program that builds this site does exactly that: it reads `site.rokon` from the site repository into a `#[derive(Rokon)] struct Site`, and fields with defaults can be omitted from the file.

The example below writes a configuration to a temporary file, reads it, modifies it, and writes it back. `load` can throw both `IoError` and `RokonError`, and the caller catches each kind separately.

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
    tagline: "small language, rich library",
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
Site { title: "rok", tagline: "small language, rich library", spec: "../spec/ko", out: "dist" }
bad config: 1:15: expected str, found integer
missing: NotFound
```

Notice also that rewriting with `to_string` drops comments. Files that people edit are better left for programs not to overwrite. `rok fmt` only fixes indentation in `.rokon` files and leaves comments and layout alone.

## Summary

- ROKON is a data format that uses rok literal syntax as is; the file extension is `.rokon`.
- Add `#[derive(Rokon)]`, read with `rokon::from_str`, write with `rokon::to_string`.
- A missing field takes its `= default`, or `none` if its type is `T | none`; otherwise it is an error. Unknown fields are errors too.
- Error messages have the form `line:column: description`.
- If you don't know the structure, use `rokon::parse` to get a `Value`.
- Use `OrderedDict` for maps whose order matters.

## Further reading

- Spec: [3.9 ROKON](../spec/refs-standard-libraries/300-std/309-rokon.html), [Field defaults](../spec/refs-lang/002-types/203-struct.html#233-field-default-values), [The rok.rokon manifest](../spec/refs-lang/009-tools/910-rok-frontend.html)
- API: [rok::std::rokon](../std/std/rokon.html)
- Next chapter: [16. Tools](16-tools.html)
