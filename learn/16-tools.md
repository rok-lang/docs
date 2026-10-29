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

`rok fmt` rewrites the package's `.rok` and `.rokon` files into the standard format. Rok's formatter is deliberately conservative: it **leaves tokens, spacing within a line, and comments alone**, and only fixes indentation (4 spaces per bracket depth), trailing whitespace, and runs of blank lines. So the formatter never changes the meaning of code or alignment a person set up by hand.

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

A closer look at the REPL (`rok` or `rok repl`) from chapter 1. On a terminal it starts with two lines: the toolchain version and host, and a hint. `rok repl -q` leaves them out, and piped input never gets them.

```text
$ rok repl
Rok 0.1.4 (3f2a1c9 2026-10-28) on aarch64-apple-darwin
Type :help for help, :exit to quit.
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> it * 10
30
>>> xs.sort(); xs?
[1, 2, 3]
>>> :type xs
List<i32>
>>> fn twice(x: i32) -> i32 {
...     x * 2
... }
>>> twice(21)
42
>>> exit
```

The prompt is `>>>`, and `...` while a cell goes on. A cell ends at the end of a line unless the input is clearly unfinished: a bracket, string or block comment is still open, the line ends in an operator, `.`, `,` or `=`, or it is an item header such as `fn f(x: i32)` still waiting for its body. When the cell's last statement is a `let` or an assignment, you can leave out its `;`.

Each cell runs once. Top-level `let` bindings and `fn` / `struct` / `impl` definitions carry over to later cells, and a cell may `await` at the top level. An expression cell prints its value with `Display`, else with `Debug`, else just its type. End it with `?` to print the `Debug` form instead. A `let` cell shows nothing. A cell of several statements shows the value of its last expression, if it ends in one. The value a cell shows is kept as `it`, so the next cell can use it; once you define your own `it`, yours wins.

Unlike regular source, you can `let` the same name again, even with another type. Defining an `fn` or a `struct` again replaces it, and code from earlier cells calls the new version. A cell that fails to compile, panics or throws leaves the session as it was. Output it already wrote stays written. `:reset` starts over from an empty session.

Errors point into the cell you typed as `<repl>:line:col`. Panics, and errors in an `fn` from an earlier cell, name that cell as `<repl cell N>:line:col`, where N counts the cells entered so far.

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| Command | What it does |
|------|---------|
| `expr?` | Print the `Debug` form instead of `Display` |
| `:type expr` (`:t`) | Print the type without running it |
| `:doc name` | Print the signature and doc comment of a name; `:doc xs.len` for a method |
| `:time expr` | Run the expression and print how long each stage took |
| `:vars` | List the bindings as `name: type`, then the items |
| `:load file`, `:reload` | Load a program into the session; load the last one again |
| `:edit` | Write cells in `$VISUAL` / `$EDITOR`, then run them |
| `:save file` | Write the session out as a script |
| `:opt [level]` | Show or set the optimization level for later cells (`0` to `3`, `s`, `z`) |
| `:reset` | Forget all cells and start over |
| `:add name ..` | Add dependencies: `rok install` into the environment, or `rok add` in a package |
| `:help` (`:h`) | List every command |
| `:exit`, `:quit` (`:q`) | Quit; so does `exit` or `quit` alone, or end of input |

A closure that captures variables can't be kept in a cell's top-level `let`, because each closure has its own unnamed type. The cell is rejected:

```text
>>> let x = 3
>>> let f = |n: i32| n * x
error: `f: closure(i32) -> i32` cannot be kept for later cells (its type has no name)
```

Give the binding a function type, `let f: fn(i32) -> i32 = |n: i32| n * x`, which carries over. (An `fn` carries over too, but it cannot see cell variables such as `x`; pass them as arguments.) Closures used inside a cell, as in `xs.iter().map(|n| n * x)`, work as usual.

On a terminal the REPL edits the line itself:

- Tab completes names: the session's bindings and items, the core library, keywords, and after `x.` the fields and methods of `x`. At the start of a line, Tab indents.
- History keeps whole cells, also across sessions (`$ROK_HOME/repl_history`). Up and Down walk it; with some text typed, they step only through cells that start with it. Ctrl-R searches backwards.
- A pasted block becomes one cell when you press Enter, instead of running line by line.
- Input is coloured as in this guide. Set `NO_COLOR` to turn colours off.

A cell that reads standard input reads the terminal. When the REPL's own input is piped, a cell sees an empty input.

`rok repl path` loads a package or a file into the session. For a library, the package is added as a dependency and `use name::*;` runs first, again after `:reset`. Handy for trying out functions while you build a library. For a program, it runs the program, then keeps its bindings and items in the session. Inside a package, `rok repl .` loads the current one; a plain `rok repl` starts empty.

The same cell engine is available in Jupyter notebooks. Register the kernel once, and you can pick the `rok` kernel in Jupyter, VS Code notebooks, and so on.

```text
$ rok jupyter install
   Installed kernel `rok` (/Users/me/Library/Jupyter/kernels/rok/kernel.json)
```

It is installed under `~/Library/Jupyter` on macOS, `%APPDATA%\jupyter` on Windows, and `~/.local/share/jupyter` elsewhere; `JUPYTER_DATA_DIR` changes the location. `rok jupyter install --env name` installs a kernel for a named environment.

A notebook cell follows the REPL's rules and shows the value of its last expression. Output appears while the cell runs, not only at the end. A cell that reads standard input gets an input box in the notebook. Tab completion and Shift-Tab help use the same names as the REPL. Code is highlighted with Rust rules until notebook front ends know Rok.

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

Names starting with `_Y` are Rok's mangled symbols. When you meet one in a debugger or profiler, decode it with `rokc demangle`.

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
- The REPL prompts with `>>>`, keeps the last value in `it`, and has commands such as `:type`, `:doc`, `:vars` and `:reset`. `rok repl .` loads the current package; for notebooks, `rok jupyter install`.
- To dig into problems: `-v`, `--emit`, `rokc demangle`, `-g` (with `rokc --print lldbinit` for lldb).

## Further reading

- Spec: [rokc CLI](../spec/refs-lang/009-tools/901-rokc-cli.html), [Standard format](../spec/refs-lang/009-tools/901-rokc-cli.html#9111-standard-format-rokc-fmt), [Lint control](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-lint-control), [Rok front-end](../spec/refs-lang/009-tools/910-rok-frontend.html), [JIT cells and the REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [REPL line editing](../spec/refs-lang/009-tools/907-jit-cells.html#line-editing), [REPL commands](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-debugging), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [Debug tools](../spec/refs-lang/009-tools/909-debug-tools.html), [Symbol mangling](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- API: [assert functions in rok::core](../std/core/panic.html)
- Next chapter: [17. Conditional Compilation](17-conditional-compilation.html)
