# 11. Modules and Packages

As a program grows, you need to split the code up and use libraries written by other people. In rok, the unit for dividing code is the **module**, and the unit for building and distributing is the **package**. A package published to the registry is called a **stone**.

## `mod` and paths

`mod name { .. }` defines a module in place. Items inside a module are referred to with a `module::item` path, and modules can contain further modules.

```rok
mod geom {
    pub struct Point { pub x: f64; pub y: f64; }

    pub fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    pub mod dist {
        use sup::Point;

        pub fn manhattan(a: Point, b: Point) -> f64 {
            abs(a.x - b.x) + abs(a.y - b.y)
        }

        fn abs(v: f64) -> f64 { if v < 0.0 { -v } else { v } }
    }
}

let p = geom::Point { x: 3.0, y: -4.0 };
println(f"{geom::dist::manhattan(geom::origin(), p)}");
```

```output
7
```

Only certain words can start a path.

| Start    | Meaning                                                      |
|----------|--------------------------------------------------------------|
| `pkg::`  | From the root of the current package                         |
| `sup::`  | From the parent module (`sup::sup::` to go further up)       |
| `rok::`  | The standard library (`rok::core`, `rok::std`)               |
| a name   | An item or child module of the current module, or a name brought in with `use` |

Name lookup does not search parent modules automatically. That is why using `Point` inside `dist` requires bringing it in explicitly with `use sup::Point;`. Thanks to this, you can always tell from within a file where a name comes from.

## Importing with `use`

Instead of writing long paths every time, bring names in with `use`. You can import several with `{..}`, rename with `as`, and import all public items of a module with `*`.

```rok
mod shapes {
    pub fn square(x: i32) -> i32 { x * x }
    pub fn cube(x: i32) -> i32 { x * x * x }
    pub const UNIT: i32 = 1;
}

mod report {
    use pkg::shapes::{square, cube as pow3};

    pub fn show(x: i32) {
        println(f"{x}: {square(x)} {pow3(x)}");
    }
}

use report::show;
use shapes::*;

show(3);
println(f"{cube(UNIT + 1)}");
```

```output
3: 9 27
8
```

A name you import by name must not also be defined in the same module; the definition would hide the import without a word. A glob (`use shapes::*;`) simply gives way to the module's own items.

```rok norun
use shapes::square;
fn square(x: f64) -> f64 { x * x }
```

```text
error[E102]: `square` is both imported from `shapes::square` and defined in this module
  --> main.rok:1:13
   |
 1 | use shapes::square;
   |             ^^^^^^ imported here
 2 | fn square(x: f64) -> f64 { x * x }
   | -- defined here, which would hide the import
   |
   = help: import it under another name, `use shapes::square as shapes_square;`, or remove this `use` if the module does not need it
```

Names like `List`, `Dict`, and `println`, which we have used so far without `use`, work thanks to the `use rok::core::**;` implicitly present in every file. Modules in `rok::std` such as files, networking, and threads are imported explicitly, as in `use rok::std::fs;`.

## Visibility

By default, an item is visible only within its own module. To make it visible outside, add a visibility keyword.

| Keyword    | Visible from                            |
|------------|-----------------------------------------|
| (none)     | The same module                         |
| `pub(sub)` | The same module and its submodules      |
| `pub(pkg)` | Anywhere in the same package            |
| `pub`      | Other packages too                      |

Struct fields must also be made public individually. That is why `Point` above marks each field with `pub`, as in `pub x: f64`. If you use a private item from outside, the compiler tells you where to add `pub`.

```rok
mod bank {
    pub struct Account { pub owner: str; balance: i64; }

    pub fn open(owner: str) -> Account { Account { owner: owner, balance: 0 } }

    pub fn deposit(a: Account, amount: i64) -> Account {
        Account { owner: a.owner, balance: a.balance + check(amount) }
    }

    pub fn balance(a: Account) -> i64 { a.balance }

    fn check(amount: i64) -> i64 { if amount < 0 { 0 } else { amount } }
}

let a = bank::deposit(bank::open("kim"), 500);
println(f"{a.owner}: {bank::balance(a)}");
```

```output
kim: 500
```

The `balance` field and the `check` function are not visible outside `bank`, so the balance changes only through `bank`'s functions.

## Splitting into files

In a real package, modules are split into files. Declaring a module without a body, as in `mod shapes;`, makes `shapes.rok` in the same directory the body of that module. If it has submodules, put a `shapes/` directory with the same name next to `shapes.rok` (there is no special file name like `mod.rok`).

```text
geom/
├── rok.rokon
└── src/
    ├── lib.rok          // mod shapes;
    ├── shapes.rok       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.rok   // shapes::circle
```

A package's sources live in `src/`. The package root module is `src/main.rok` for an executable and `src/lib.rok` for a library. Executable top-level statements may appear only in `src/main.rok`; other modules contain only items.

## Packages and `rok.rokon`

`rok new` creates a package: `src/main.rok` for an executable, or with `--lib`, `src/lib.rok` holding an example function and one `#[test]`. The manifest `rok.rokon` is in [ROKON](../spec/refs-standard-libraries/300-std/309-rokon.html) format.

```bash
rok new --lib geom
rok new app
```

```text
     Created library package (geom)
     Created executable package (app)
```

Let's create two modules in `geom`. If you re-export a submodule's items from the package root with `pub use`, users do not need to know the internal structure.

```rok norun
// geom/src/lib.rok
mod shapes;

pub use shapes::Circle;

pub(pkg) const PI: f64 = 3.14159;

pub fn area_sum(cs: List<Circle>) -> f64 {
    cs.iter().map(|c| c.area()).sum()
}
```

```rok norun
// geom/src/shapes.rok
pub struct Circle { pub r: f64; }

impl Circle {
    pub fn area(self) -> f64 { sup::PI * self.r * self.r }
}
```

When you add a dependency in `app` with `rok add`, it goes into `deps` in `rok.rokon`. The name of the dependency package becomes its top-level path (`-` turns into `_`).

```bash
cd app
rok add geom --path ../geom
```

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "geom": Dep::path("../geom"),
} }
```

```rok norun
// app/src/main.rok
use geom::{Circle, area_sum};

println(f"{area_sum([Circle { r: 1.0 }, Circle { r: 2.0 }])}");
```

```text
$ rok run
   Compiling geom (../geom)
   Compiling app (.)
    Finished in 1.97s
     Running `./target/debug/app`
15.70795
```

`PI` is `pub(pkg)`, so it can be used anywhere inside `geom` but is not visible from `app`. A git repository is added with something like `rok add parser --git <url> --tag v1.2`. For git and stone dependencies, the exact commit is pinned in `rok.lock`, so commit `rok.lock` to version control as well. Only `rok update` moves pinned versions forward.

## Stones and rokstones.io

A package on the registry is called a stone, and the default registry is [rokstones.io](https://rokstones.io). You search by name and add it together with a version requirement. Version requirements work as in Cargo: `^1.2` means `>=1.2.0, <2.0.0`. If you omit the requirement, the newest version is recorded with a `^`.

```text
$ rok search json
    Updating index https://github.com/rok-lang/rokstones.io-index
json 0.1.0 - JSON (RFC 8259): Value tree, strict parser, streaming tokenizer, typed encode/decode, JSON Pointer
$ rok add json@^0.1
       Added json to deps of rok.rokon
```

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1"),
} }
```

A plain string is shorthand for `Dep::version`, and a requirement with no operator means `^`, so these three lines ask for the same thing. Stone READMEs often use the short form (`"json": "0.1"`, which means `>=0.1.0, <0.2.0`).

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

A path is not a version: write `Dep::path("../geom")`, not `"../geom"`.

A library can offer **features** that its users turn on and off, and you can publish your own library as a stone with `rok login` and `rok publish`. Both are covered in [chapter 18](18-features-and-stones.html), together with the stones the rok project maintains (`json`, `toml`, `http`, `markdown`).

## Single-file scripts and environments

For a small script that does not warrant a package, its dependencies live in a ```` ```rokon ```` block inside the `//!` comment at the very top of the file. `rok add` with the script as the last argument writes the block for you; a path is written relative to the script, and the lockfile is kept in the cache.

```bash
rok add geom --path ../geom tool.rok
```

```text
       Added geom to tool.rok
```

````rok norun
//! ```rokon
//! Package {
//!     deps: {
//!         "geom": Dep::path("../geom"),
//!     },
//! }
//! ```
mod helper;
use geom::{Circle, area_sum};
println(f"{area_sum([Circle { r: helper::radius() }])}");
````

```bash
rok run tool.rok
```

A script can have file modules too: `mod helper;` loads `helper.rok` beside it, just as in a package's `src/`. A module file may carry its own `//!` block (`rok add <name> helper.rok`), and the blocks of all the files are merged. Scripts never see libraries installed with `rok install`; only their blocks count. When a script outgrows one file, `rok init --script tool.rok` turns it into a package: the blocks become `rok.rokon`, and `tool.rok` and its module files move into `src/`.

Named environments are for the REPL and Jupyter. Install libraries into one with `rok install`, and pick it with `--env` or change the default with `rok env use`; there is no activation step.

```text
$ rok env create sci
     Created environment `sci` (~/.rok/envs/sci)
$ rok install --env sci geom --path ~/work/geom
$ rok env use sci
$ rok env list
* sci
```

A package never uses an environment: inside a package, its own `rok.rokon` is all that counts.

## Installing programs

`rok install` also installs programs, like `cargo install`: a package with `src/main.rok` (from a stone, git or a path) or a script file is built with `--release` into `~/.rok/bin`. Installing the same stone version or git commit again does nothing, and says how to rebuild; a new version replaces the old one. A path or a script file is always rebuilt.

```text
$ rok install tool
   Installed tool 0.1.0 (~/.rok/bin/tool)
$ rok install tool
`tool` 0.1.0 is installed already (~/.rok/bin/tool); to rebuild it, run `rok install tool --force`
$ rok install tool --force
 Reinstalled tool 0.1.0 (~/.rok/bin/tool)
$ rok install tool@0.2
    Upgraded tool from 0.1.0 to 0.2.0 (~/.rok/bin/tool)
```

`rok install --list` shows what is installed, and `rok uninstall tool` removes it.

## Summary

- Create modules with `mod`, and refer to names with `pkg::` / `sup::` / `rok::` paths and `use`.
- Items are private by default. Widen their scope with `pub(sub)`, `pub(pkg)`, and `pub`.
- `mod x;` maps to `x.rok` (and `x/`). The root is `main.rok` or `lib.rok`.
- Create packages with `rok new [--lib]` and add dependencies (path, git, stone) with `rok add`. Commit `rok.lock`.
- Work with stones on rokstones.io using `rok search` / `rok add name@^1.2`; features and publishing are in [chapter 18](18-features-and-stones.html).
- Scripts use their `//! ```rokon` blocks (`rok add <name> <file.rok>`), and `rok init --script` turns one into a package; `rok env` is for the REPL / Jupyter.

## Further reading

- Spec: [Modules](../spec/refs-lang/005-modules/504-modules.html), [`use`](../spec/refs-lang/005-modules/505-use.html), [Visibility](../spec/refs-lang/005-modules/506-visibility.html), [Paths](../spec/refs-lang/005-modules/507-paths.html), [Re-exports](../spec/refs-lang/005-modules/508-re-export.html)
- Tools: [`rok` front-end](../spec/refs-lang/009-tools/910-rok-frontend.html), [`rok.rokon` / `rok.lock`](../spec/refs-lang/009-tools/910-rok-frontend.html#9103-rokrokon--roklock), [Stones and rokstones.io](../spec/refs-lang/009-tools/910-rok-frontend.html#9104a-stones-and-rokstonesio), [Environments](../spec/refs-lang/009-tools/910-rok-frontend.html#9105-environments), [Inline manifests](../spec/refs-lang/009-tools/910-rok-frontend.html#9107-inline-manifest-for-single-file-scripts)
- [1. Getting Started](01-getting-started.html)
