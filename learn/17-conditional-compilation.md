# 17. Conditional Compilation

One source tree often has to build in several shapes: per operating system, debug against release, with or without an optional feature. Rok decides this at compile time with `#[cfg(..)]`. Code whose condition is false is removed right after parsing, as if it had never been written, so it costs nothing at run time and may even refer to names that do not exist in that build.

## Conditions

A condition is a flag, a `key = "value"` pair, or a combination of them with `all(..)`, `any(..)` and `not(..)`.

| Condition                       | True when                                              |
|---------------------------------|--------------------------------------------------------|
| `target_os = "linux"`           | Building for Linux (`"macos"`, `"windows"`, `"wasi"`, ..) |
| `target_arch = "aarch64"`       | Building for that CPU (`"x86_64"`, ..)                 |
| `target_family = "unix"`        | Unix-like targets (`"windows"`, `"wasm"`)              |
| `target_pointer_width = "64"`   | 64-bit targets                                         |
| `feature = "tls"`               | The package's feature `tls` is enabled ([chapter 18](18-features-and-stones.html)) |
| `debug_assertions`              | Building at `-O0` (the default for `rok run`)          |
| `test`                          | Building for `rok test`                                |

`target_*` follow `--target`, so a cross build sees the target's values, not the host's.

## Where `#[cfg]` goes

`#[cfg]` attaches to items (functions, types, `mod`, `use`, `const`, `static`, `impl` blocks and their members) and also to the smaller pieces inside code: statements, `match` arms, struct fields, fields of a struct literal, and function parameters.

```rok
enum Op { Add, Trace, Quit }

struct Build {
    name: str;
    #[cfg(debug_assertions)]
    checks: bool;                         // a field only debug builds have
}

fn describe(op: Op) -> str {
    match op {
        Op::Add => "add",
        #[cfg(debug_assertions)]
        Op::Trace => "trace (debug build)", // an arm
        _ => "unsupported",
    }
}

let b = Build {
    name: "demo",
    #[cfg(debug_assertions)]
    checks: true,                         // a struct literal field
};

#[cfg(debug_assertions)]
println(f"debug build of {b.name}");      // a statement
#[cfg(not(debug_assertions))]
println(f"release build of {b.name}");

println(describe(Op::Trace));
```

```output
debug build of demo
trace (debug build)
```

Built with `-O2`, the same file prints `release build of demo` and `unsupported`: the `checks` field, its initializer and the `Trace` arm are all gone.

A parameter with a false condition is removed from the signature, and calls pass only the remaining arguments. This lets an optional feature add a parameter without a second copy of the function.

```rok
fn connect(host: str, #[cfg(feature = "tls")] verify: bool) -> str {
    #[cfg(feature = "tls")]
    if verify { return f"{host} (verified)"; }
    host
}

// Without the `tls` feature, `connect` takes one argument.
println(connect("example.com"));
```

```output
example.com
```

Closure parameters take no attributes.

## Two definitions, one build

To give a function a different body per platform, write it twice with conditions that exclude each other. Only one survives in any build; if both survive, it is the ordinary duplicate-definition error.

```rok
#[cfg(target_family = "windows")]
fn path_sep() -> str { "\\" }

#[cfg(not(target_family = "windows"))]
fn path_sep() -> str { "/" }

println(["usr", "local", "bin"].join(path_sep()));
```

On Linux and macOS this prints `usr/local/bin`.

A whole module can be conditional too. `#[cfg(target_os = "linux")] mod linux;` does not even read `linux.rok` in other builds.

## `cfg!`: a compile-time `bool`

When both branches compile everywhere and only the choice differs, `cfg!(..)` is shorter. It is a constant `bool`, so the compiler drops the dead branch; unlike `#[cfg]`, both sides must still type-check.

```rok
const DEBUG: bool = cfg!(debug_assertions);

let level = if cfg!(feature = "verbose") { 2 } else { 1 };
println(f"debug={DEBUG} level={level}");
```

```output
debug=true level=1
```

`#[cfg_attr(cond, attr)]` attaches another attribute only when the condition holds, for example `#[cfg_attr(test, derive(Debug))]`.

## Summary

- `#[cfg(..)]` keeps or removes code at compile time. Conditions are `key = "value"`, flags (`test`, `debug_assertions`) and `all` / `any` / `not`.
- It attaches to items, fields, variants, statements, match arms, struct literal fields and parameters. A removed parameter disappears from the signature.
- Write mutually exclusive definitions for per-platform bodies.
- `cfg!(..)` is a constant `bool` for an ordinary `if`.
- `feature = ".."` conditions are set by the package's features, the subject of the next chapter.

## Further reading

- Spec: [6.3 `#[cfg]`](../spec/refs-lang/006-meta-system/603-cfg.html), [6.3.3 Attachment positions](../spec/refs-lang/006-meta-system/603-cfg.html#633-attachment-positions), [6.3.5 `cfg!`](../spec/refs-lang/006-meta-system/603-cfg.html#635-cfg-compile-time-bool-value), [Platform targets](../spec/refs-lang/000-meta/006-platform-targets.html)
- Next chapter: [18. Features and Stones](18-features-and-stones.html)
