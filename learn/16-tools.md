# 16. Tools

This chapter tours the tools you keep at hand while writing code: the formatter, the linter, tests and benchmarks, documentation generation, editor support (LSP), the REPL and Jupyter, and debug options for looking inside the compiler. All of them start from the single `rok` command.

The output in this chapter comes from actually running the package below. It was created with `rok new mathx`, and `src/main.rok` was changed to this:

```rok build
//! Small math helpers.

/// Greatest common divisor of `a` and `b`.
pub fn gcd(a: i64, b: i64) -> i64 {
    if b == 0 { a } else { gcd(b, a % b) }
}

/// The `n`-th Fibonacci number.
pub fn fib(n: i64) -> i64 {
    if n < 2 { return n; }
    fib(n - 1) + fib(n - 2)
}

println(f"gcd(12, 18) = {gcd(12, 18)}, fib(10) = {fib(10)}");

#[test]
fn gcd_works() {
    assert_eq(gcd(12, 18), 6);
    assert_eq(gcd(7, 0), 7);
}

#[test]
fn fib_works() {
    assert_eq(fib(10), 55);
}

#[test]
fn fib_wrong() {
    assert_eq(fib(5), 6, "deliberately wrong");
}

#[bench]
fn fib_20() {
    fib(20);
}
```

## `rok test`: tests

A function marked `#[test]` is a test. Test functions are left out of normal builds; `rok test` collects them and builds them separately. Checks use `assert`, `assert_eq`, and `assert_ne`, which panic on failure. You can pass a message as the last argument.

Each test runs in its own process, so if one panics the rest keep running. `fib_wrong` in the package above is deliberately wrong.

```text
$ rok test
   Compiling mathx (.)
    Finished in 0.69s

running 3 tests
test gcd_works ... ok
test fib_works ... ok
test fib_wrong ... FAILED

failures:

---- fib_wrong ----
thread 'main' panicked at src/main.rok:29:5:
assertion `left == right` failed: deliberately wrong
  left: 5
 right: 6
(panicked (exit 101))

failures:
    fib_wrong

test result: FAILED. 2 passed; 1 failed; 0 filtered out; finished in 0.22s
```

If anything fails, the exit code is 1, so it works in CI as is. Passing part of a name (or a small regex using `^`, `$`, `.*`) as an argument runs only the matching tests.

```text
$ rok test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.22s
```

Use `--show-output` to also see output from passing tests, and `--threads=N` to set how many tests run concurrently.

## `rok bench`: benchmarks

`rok bench` calls each `#[bench]` function many times in an optimized build (`-O2`) and measures the average time per call. The numbers vary by machine and by run.

```text
$ rok bench
   Compiling mathx (.)
    Finished in 0.42s

running 1 bench
bench fib_20 ... bench:       34,204 ns/iter (+/- 1,712)

bench result: ok. 1 measured; 0 failed; 0 filtered out; finished in 0.36s
```

`--iters=N` is the number of calls per sample, and `--samples=N` is the number of samples. Filters work as in `rok test`.

## `rok fmt`: the formatter

`rok fmt` rewrites the package's `.rok` and `.rokon` files into the standard format. rok's formatter is deliberately conservative: it **leaves tokens, spacing within a line, and comments alone**, and only fixes indentation (4 spaces per bracket depth), trailing whitespace, and runs of blank lines. So the formatter never changes the meaning of code or alignment a person set up by hand.

Checking a `fib` with messed-up indentation, mixed tabs, and trailing whitespace using `--check` leaves the file untouched, shows the lines that would change, and exits with code 1. This mode is a good fit for CI.

```text
$ rok fmt --check
Diff in ./src/main.rok:
+   10     if n < 2 { return n; }
+   11     let unused = 1;
+   12     fib(n - 1) + fib(n - 2)
-   10   if n < 2 { return n; }
-   11   let unused = 1;
-   12 	fib(n - 1) + fib(n - 2)
$ rok fmt
$ rok fmt --check
$
```

Without `--check`, it quietly fixes the files. Running it on every save in your editor is convenient.

## `rok lint` and `rok check`

`rok lint` does not compile; it runs only type checking and lints. It catches the `let unused = 1;` left in above.

```text
$ rok lint
    Checking mathx (.)
warning[E503]: unused variable `unused`
   --> src/main.rok:11:5
    |
 11 |     let unused = 1;
    |     ^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `#[warn(unused_variables)]` on by default
     Checked mathx: 0 errors, 1 warning
    Finished in 0.38s
```

Every lint defaults to `warn`. The level can be changed in three places, and later ones win:

1. The `lints` field in `rok.rokon`: `lints: { "unused": "deny", "dead_code": "allow" }`
2. `-A` / `-W` / `-D` (allow / warn / deny) on the command line
3. Attributes in source: `#[allow(dead_code)]` on an item, `#![deny(unused)]` at the top of a file

Instead of a single lint (`unused_variables`), you can name a group (`unused`, `nonstandard_style`, `all`). If any `deny` lint fires, the exit code is 1.

```text
$ rok lint -D unused
    Checking mathx (.)
error[E503]: unused variable `unused`
   --> src/main.rok:11:5
    |
 11 |     let unused = 1;
    |     ^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `-D unused` on the command line
     Checked mathx: 1 error, 0 warnings
    Finished in 0.31s
```

See the full list with `rokc lint --list` (piping it into `head` is fine: when the reader of its output goes away, `rok` and `rokc` stop quietly instead of reporting an error). `rok check` goes further and also runs code generation, but does not link or produce an executable. It is faster than a build, so use it to answer "does it compile?" The examples from here on use a throwaway `hello` package, so the errors and low-level output do not disturb `mathx`.

```text
$ rok check
    Checking hello (.)
error[E200]: mismatched types: `let s` expects `str`, found `i32`
  --> src/main.rok:2:1
   |
 2 | let s: str = square(7);
   | ^^^
     Checked hello: 1 error, 0 warnings
    Finished in 0.32s
```

For machine-readable output, add `--message-format=json`. Each diagnostic is printed as one line of JSON.

## `rok doc`: doc comments and API docs

`///` documents the item directly below it, and `//!` documents the module (file) containing the comment. In the package above, the first line `//! Small math helpers.` becomes the module description, and the `///` above `gcd` and `fib` become the function descriptions. Doc text can use markdown such as headings, lists, inline code, code blocks, and links.

```text
$ rok doc
 Documenting mathx (.)
    Finished in 0.22s
   Generated ./target/doc/index.html
```

`target/doc/` gets HTML for the public (`pub`) items of the package and its dependencies, plus a search index. This site's [standard library API](../std/core/index.html) is generated the same way, from doc comments in the library source.

## Editors: `rokls`

`rokls` is an LSP (Language Server Protocol) server. It provides diagnostics, hover, go to definition, document symbols, completion, and formatting. `rok lsp` and `rokc lsp` are the same server. It communicates over standard IO, so any editor with LSP support only needs to be told the command to run.

For Neovim (0.11 or later), add this to `init.lua`:

```text
vim.filetype.add({ extension = { rok = "rok", rokon = "rokon" } })
vim.lsp.config("rokls", {
  cmd = { "rokls" },
  filetypes = { "rok", "rokon" },
  root_markers = { "rok.rokon" },
})
vim.lsp.enable("rokls")
```

For Helix, add this to `languages.toml`:

```text
[language-server.rokls]
command = "rokls"

[[language]]
name = "rok"
scope = "source.rok"
file-types = ["rok"]
roots = ["rok.rokon"]
language-servers = ["rokls"]
```

In other editors such as VS Code, register `rokls` with a generic LSP client extension. `rokls` must be on your `PATH`.

## REPL and Jupyter

A closer look at the REPL (`rok` or `rok repl`) from Chapter 1. Each cell runs once, and top-level `let` bindings and `fn` / `struct` definitions carry over to later cells. An expression cell shows `type: value`; a `let` cell shows nothing. A cell of several statements shows the value of its last expression, if it ends in one. Unlike regular source, you can `let` the same name again, and redefining an `fn` replaces it. A cell that errors leaves no trace.

```text
$ rok repl
> let xs = [3, 1, 2];
> xs.len()
usize: 3
> xs.sort(); xs
List<i32>: [1, 2, 3]
> :type xs
List<i32>
> fn twice(x: i32) -> i32 { x * 2 }
> twice(21)
i32: 42
> :quit
```

| Command | What it does |
|------|---------|
| `:type expr` | Print the type without evaluating |
| `:reset` | Forget all cells and start over |
| `:add name`, `:add name --path path` | Add a dependency to the current session (installed into the environment, or added to the package; never into a script) |
| `:help`, `:quit` | Help, quit |

A closure that captures variables can't be kept in a cell's top-level `let`, because each closure has its own unnamed type. The cell is rejected:

```text
> let x = 3;
> let f = |n: i32| n * x;
error: `f: closure(i32) -> i32` cannot be kept for later cells (its type has no name)
```

Give the binding a function type, `let f: fn(i32) -> i32 = |n: i32| n * x;`, which carries over. (An `fn` carries over too, but it cannot see cell variables such as `x`; pass them as arguments.) Closures used inside a cell, as in `xs.iter().map(|n| n * x)`, work as usual.

`rok repl --package path` starts with a library package attached and `use name::*;` already in place. Handy for trying out functions while building a library.

The same cell engine is available in Jupyter notebooks. Register the kernel once, and you can pick the `rok` kernel in Jupyter, VS Code notebooks, and so on.

```text
$ rok jupyter install
   Installed kernel `rok` (/Users/me/Library/Jupyter/kernels/rok/kernel.json)
```

It is installed under `~/Library/Jupyter` on macOS and `~/.local/share/jupyter` elsewhere; `JUPYTER_DATA_DIR` changes the location.

## Looking inside the compiler

To see how long each build stage takes, add `-v`.

```text
$ rok build -v
   Compiling hello (.)
   Stage: front end (250ms)
    Building ./target/debug/hello
   Stage: codegen + link (160ms)
    Finished in 0.41s
```

`--emit` writes intermediate artifacts to `target/debug/build/`. The kinds are `ast`, `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, and `expanded` (the result of meta expansion).

```text
$ rok emit --emit=llvmir
   Compiling hello (.)
       Wrote ./target/debug/build/hello.ll
    Finished in 0.32s
$ grep define target/debug/build/hello.ll | head -2
define i32 @_Y2cfee941__square_(i32 %0) {
define void @_Y2cfee941__main_() {
```

Names starting with `_Y` are rok's mangled symbols. When you meet one in a debugger or profiler, decode it with `rokc demangle`.

```text
$ rokc demangle _Y2cfee941__square_
package:  2cfee941
path:     (root)
name:     square
generics: (none)
```

Building with `-g` includes DWARF debug info, so you can step through source lines in `lldb` / `gdb`. gdb shows only the active variant of a sum or an enum on its own; for lldb, the toolchain ships formatters that do the same, and `rokc --print lldbinit` prints the command that loads them.

```text
$ lldb -O "$(rokc --print lldbinit)" ./target/debug/hello
$ rokc --print lldbinit >> ~/.lldbinit      # or load them every time
```

## Summary

- `rok test` / `rok bench` build and run `#[test]` / `#[bench]` functions separately. Each test gets its own process.
- `rok fmt` only fixes indentation and blank lines. In CI, use `rok fmt --check`.
- `rok lint` runs lints; `rok check` goes as far as code generation. Lint levels resolve in the order `rok.rokon` < `-A/-W/-D` < source attributes.
- From `///` and `//!` doc comments, `rok doc` generates HTML in `target/doc/`.
- Register `rokls` as the LSP server in your editor.
- The REPL has `:type` / `:reset` / `:add`; for notebooks, `rok jupyter install`.
- To dig into problems: `-v`, `--emit`, `rokc demangle`, `-g` (with `rokc --print lldbinit` for lldb).

## Further reading

- Spec: [rokc CLI](../spec/refs-lang/009-tools/901-rokc-cli.html), [Standard format](../spec/refs-lang/009-tools/901-rokc-cli.html#9111-standard-format-rokc-fmt), [Lint control](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-lint-control), [rok front-end](../spec/refs-lang/009-tools/910-rok-frontend.html), [JIT cells and the REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [Debug tools](../spec/refs-lang/009-tools/909-debug-tools.html), [Symbol mangling](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- API: [assert functions in rok::core](../std/core/panic.html)
- Next chapter: [17. Conditional Compilation](17-conditional-compilation.html)
