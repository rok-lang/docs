# 1. Getting Started

In this chapter you install the rok toolchain, create and run your first package, and try a single-file script and the REPL. By the end you can run the examples in the rest of the guide yourself.

## Installation

### Prerequisites

`rokc` is self-contained: LLVM, MLIR and the lld linker are built into it, so you do not install LLVM ([9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-embedded-toolchain)).

- macOS: the Xcode Command Line Tools (`xcode-select --install`), for the SDK that the final link needs.
- Linux: nothing. The toolchain and the programs it builds need glibc 2.34 or later (Ubuntu 22.04, Debian 12, RHEL 9 and newer).
- Windows: nothing.

Release archives are built for macOS arm64, Linux x86_64 and arm64 (both glibc and static musl), and Windows x86_64. Other hosts, such as macOS x86_64 and Windows arm64, have no release archive yet; build Rok from source.

### Install script

The install script downloads the toolchain for your platform from the [GitHub releases](https://github.com/rok-lang/rok/releases), checks it against its SHA-256, and unpacks it into `~/.rok`.

```bash
curl -fsSL https://rok-lang.org/install.sh | sh
```

On Windows, run this in PowerShell. It installs into `%USERPROFILE%\.rok` and adds its `bin` to your user `PATH`.

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://rok-lang.org/install.ps1 | iex"
```

`install.sh --version <v>` and `install.ps1 -Version <v>` pick a release.

You can change the install location with the `ROK_HOME` environment variable (default `~/.rok`). When the script finishes, you get the following layout.

```text
~/.rok/
├── bin/                        # rok, rokc, rokls proxy
└── toolchains/
    └── 0.1.0-aarch64-apple-darwin/
        ├── bin/{rok, rokc, rokls}
        └── lib/                # runtime and standard library sources
```

The toolchain you just installed becomes the default. The three executables in `~/.rok/bin` are **proxies**: the real work is done by the executable of the same name in the active toolchain. This lets you install several toolchains side by side and pick one (the same approach as rustup). Add this directory to `PATH` (on Windows the installer already did).

```bash
# bash / zsh
echo 'export PATH="$HOME/.rok/bin:$PATH"' >> ~/.zshrc
# fish
fish_add_path ~/.rok/bin
```

Check the version in a new shell.

```bash
rok --version
rok toolchain list
```

Three executables are installed.

| Executable | What it does |
|------------|--------------|
| `rokc`     | The compiler. Build, run, test, format ([9.1](../spec/refs-lang/009-tools/901-rokc-cli.html)) |
| `rok`      | Package / build front-end. `rok.rokon`, dependencies, environments, REPL, Jupyter ([9.10](../spec/refs-lang/009-tools/910-rok-frontend.html)) |
| `rokls`    | Language server (LSP) |

### Managing toolchains

`rok toolchain` manages installed toolchains. The active toolchain is chosen, in order, by `+version` on the command line (`rok +0.1 build`), the `ROK_TOOLCHAIN` environment variable, the `toolchain` field of the nearest `rok.rokon`, and finally the default.

```bash
rok toolchain list                     # installed toolchains
rok toolchain default 0.1.0            # change the default
rok toolchain install 0.2.0 --from rok-0.2.0-aarch64-apple-darwin.tar.gz
rok toolchain update                   # download the latest release and make it the default
```

`rok toolchain update 0.2.0` installs that version instead. The download is checked against its published SHA-256, and if anything fails the previous default stays in place. Older toolchains are kept. Do not confuse it with `rok update`, which moves the versions pinned in a package's `rok.lock` ([chapter 11](11-modules-packages.html)).

### Building from source (optional)

You can also build a toolchain archive yourself and install it with the same script. This needs a Rust toolchain (`rustup`).

```bash
git clone https://github.com/rok-lang/rok && cd rok
./dist.sh                                  # dist/rok-<version>-<triple>.tar.gz
sh install.sh --archive dist/rok-*.tar.gz  # into ~/.rok
```

The first run also builds the pinned LLVM into `~/.cache/rok` (cmake and ninja, about an hour). On Linux, `./dist-linux.sh` (podman or docker) builds the same archive in a container so that it runs on glibc 2.34 and later; the release archives are built this way. The repository's README has the details.

If you are working on the compiler, you can use the build output directly without installing. Without the pinned LLVM, this `rokc` compiles at `-O0` only.

```bash
cargo build --release
export PATH="$PWD/target/release:$PATH"
```

## Your first package

`rok new` creates a package directory and initializes a git repository with a `.gitignore` that excludes `target/`. Package names use lowercase letters, digits, `-`, and `_`, and start with a letter. The name `rok` is taken by the standard library, so a package can't be called `rok`. In code, `-` becomes `_` (`rok new my-lib` is imported as `use my_lib::..`).

```bash
rok new hello
cd hello
```

```text
     Created executable package (hello)
```

There are two files. The manifest `rok.rokon` holds the package name, version, and dependencies.

```rokon
Package { name: "hello", version: "0.1.0", deps: {} }
```

The source lives in `src/`; `src/main.rok` is a single line that prints a greeting.

```rok
println("Hello, world!");
```

```output
Hello, world!
```

`rok run` builds and runs it. The output goes to `target/debug/`.

```bash
rok run
```

```text
   Compiling hello (.)
    Finished in 1.55s
     Running `./target/debug/hello`
Hello, world!
```

The top-level statements in `src/main.rok` run from top to bottom and form the program. Function and type declarations can go anywhere between top-level statements. Other common commands are `rok build` (build only), `rok check` (type check only), `rok test`, and `rok fmt`.

## A slightly bigger program

Change `src/main.rok` to the following and run `rok run` again.

```rok
let names = ["ada", "grace", "linus"];
for name in names {
    greet(name);
}
println(f"{names.len()} people greeted");

fn greet(name: str) {
    println(f"Hello, {name}!");
}
```

```output
Hello, ada!
Hello, grace!
Hello, linus!
3 people greeted
```

`f"..."` is an f-string: each expression inside `{}` is evaluated and inserted in place. `greet` is declared below its use, yet the call works, because declarations such as functions are visible regardless of order.

Instead of top-level statements, you can use `fn main()` as the entry point. Use one or the other.

```rok
fn main() {
    let total = add(2, 3);
    println(f"2 + 3 = {total}");
}

fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

```output
2 + 3 = 5
```

## Single-file scripts

Quick experiments don't need a package. Give `rok run` a file path and it runs that file on its own.

```bash
echo 'println("hi");' > hi.rok
rok run hi.rok
```

```text
   Compiling hi (.)
    Finished in 1.52s
     Running `/tmp/rokc-.../hi`
hi
```

Scripts can use dependencies too: `rok add json hi.rok` writes them into a `//!` comment at the top of the file (see [inline manifests](../spec/refs-lang/009-tools/910-rok-frontend.html#9107-inline-manifest-for-single-file-scripts) and [chapter 11](11-modules-packages.html#single-file-scripts-and-environments)).

## REPL

Running `rok` with no arguments opens the REPL. Enter an expression and it shows the value and type; a `let` just defines the name.

```text
$ rok
> let xs = [1, 2, 3];
> xs.len() * 2
usize: 6
> :type xs
List<i32>
> :quit
```

`:type expr` reports the type of an expression without running it. Whenever you wonder about a type while reading this guide, check it in the REPL. Unlike regular source, the REPL lets you redeclare the same name with `let` ([JIT cells](../spec/refs-lang/009-tools/907-jit-cells.html)).

## Summary

- Install into `~/.rok` with `curl -fsSL https://rok-lang.org/install.sh | sh` (`install.ps1` on Windows) and add `~/.rok/bin` to `PATH`.
- `rok new name` creates `rok.rokon` and `src/main.rok`; `rok run` builds and runs.
- The top-level statements of `src/main.rok` are the program. You can also use `fn main()`.
- `rok run file.rok` runs a script; `rok` alone starts the REPL.

## Further reading

- [Install layout](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-installation-layout), [pinning the toolchain](../spec/refs-lang/009-tools/910-rok-frontend.html#9106-toolchain)
- [`rok` front-end](../spec/refs-lang/009-tools/910-rok-frontend.html), [`rokc` CLI](../spec/refs-lang/009-tools/901-rokc-cli.html)
- [Entry point](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT cells and the REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- Next chapter: [2. Values and Types](02-values-and-types.html)
