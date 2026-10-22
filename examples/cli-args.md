# Command-line tool

`mgrep`, a small `grep`-like tool, parses its command-line arguments by hand into a struct and handles `--help` and bad arguments with `throws`.

## Code

```rok
// mgrep: print the lines of FILEs that contain PATTERN.
use rok::std::env;
use rok::std::fs;
use rok::std::io::IoError;

const USAGE: str = "usage: mgrep [-i] [-n] [-c] [-m NUM] PATTERN [FILE]...
  -i, --ignore-case   match case-insensitively
  -n, --line-number   prefix each line with its number
  -c, --count         print only the number of matching lines
  -m, --max NUM       stop after NUM matches
  -h, --help          show this help";

// Used when no FILE is given (the demo never reads stdin).
const SAMPLE: str = "Roses are red,
violets are blue.
rok is fast,
and so are you.";

enum ArgError {
    Unknown(str),
    MissingValue(str),
    BadNumber(str),
    NoPattern,
}

impl Display for ArgError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(match self {
            ArgError::Unknown(a) => f"unknown option '{a}'",
            ArgError::MissingValue(a) => f"option '{a}' needs a value",
            ArgError::BadNumber(v) => f"'{v}' is not a number",
            ArgError::NoPattern => "missing PATTERN",
        })
    }
}

struct Options {
    pattern: str;
    ignore_case: bool = false;
    line_numbers: bool = false;
    count_only: bool = false;
    max: usize | none = none;
    files: List<str> = [];
    help: bool = false;
}

fn parse_args(args: List<str>) throws (ArgError) -> Options {
    let opts = Options { pattern: "" };
    let have_pattern = false;
    let i: usize = 0;
    while i < args.len() {
        let a = args[i];
        match a {
            "-h" | "--help" => { opts.help = true; },
            "-i" | "--ignore-case" => { opts.ignore_case = true; },
            "-n" | "--line-number" => { opts.line_numbers = true; },
            "-c" | "--count" => { opts.count_only = true; },
            "-m" | "--max" => {
                i += 1;
                if i >= args.len() { throw ArgError::MissingValue(a); }
                opts.max = try args[i].parse::<usize>() catch _ { throw ArgError::BadNumber(args[i]); };
            },
            _ => {
                if a.starts_with("-") && a.len() > 1 { throw ArgError::Unknown(a); }
                if have_pattern { opts.files.push(a); } else { opts.pattern = a; have_pattern = true; }
            },
        }
        i += 1;
    }
    if !have_pattern && !opts.help { throw ArgError::NoPattern; }
    opts
}

fn search(opts: Options, name: str, text: str) {
    let needle = if opts.ignore_case { opts.pattern.to_lowercase() } else { opts.pattern };
    let hits: usize = 0;
    let n: usize = 0;
    for line in text.lines() {
        n += 1;
        let hay = if opts.ignore_case { line.to_lowercase() } else { line };
        if !hay.contains(needle) { continue; }
        hits += 1;
        if !opts.count_only {
            let prefix = if opts.files.len() > 1 { f"{name}:" } else { "" };
            if opts.line_numbers { println(f"{prefix}{n}:{line}"); } else { println(f"{prefix}{line}"); }
        }
        if hits == (opts.max ?? usize::MAX) { break; }
    }
    if opts.count_only { println(f"{hits}"); }
}

fn run(args: List<str>) throws (ArgError | IoError) {
    let opts = try parse_args(args);
    if opts.help { println(USAGE); return; }
    if opts.files.is_empty() { search(opts, "-", SAMPLE); return; }
    for path in opts.files {
        search(opts, path, try fs::read_to_str(path));
    }
}

fn main() -> i32 {
    let argv = env::args();
    let args: List<str> = [];
    for i in 1..argv.len() { args.push(argv[i]); }
    // No arguments: demonstrate a few invocations on the built-in sample.
    let runs: List<List<str>> = if args.is_empty() {
        [["--help"], ["-n", "are"], ["-i", "-c", "ROK"], ["-m", "1", "are"], ["-x", "are"], ["--max"]]
    } else {
        [args]
    };
    let status = 0;
    for r in runs {
        if args.is_empty() {
            let line = StrBuf::new();
            line.push_str("$ mgrep");
            for a in r { line.push_str(f" {a}"); }
            println(line.to_str());
        }
        try run(r)
        catch e: ArgError { println(f"mgrep: {e}\n(try --help)"); status = 2; }
        catch e: IoError { println(f"mgrep: {e}"); status = 1; };
    }
    if args.is_empty() { 0 } else { status }
}
```

Run without arguments, it tries a few built-in sample invocations in turn.

```output
$ mgrep --help
usage: mgrep [-i] [-n] [-c] [-m NUM] PATTERN [FILE]...
  -i, --ignore-case   match case-insensitively
  -n, --line-number   prefix each line with its number
  -c, --count         print only the number of matching lines
  -m, --max NUM       stop after NUM matches
  -h, --help          show this help
$ mgrep -n are
1:Roses are red,
2:violets are blue.
4:and so are you.
$ mgrep -i -c ROK
1
$ mgrep -m 1 are
Roses are red,
$ mgrep -x are
mgrep: unknown option '-x'
(try --help)
$ mgrep --max
mgrep: option '--max' needs a value
(try --help)
```

## Walkthrough

- `env::args()` is a `List<str>` with the program name at `[0]`, so only `1..argv.len()` is copied ([`env::args`](../std/std/env.html#fn.args)).
- The options live in an `Options` struct with field defaults. Fields that have a default can be left out, as in `Options { pattern: "" }` ([Structs, enums and traits](../learn/06-structs-enums-traits.html), [2.3.3 field defaults](../spec/refs-lang/002-types/203-struct.html#233-field-default-values)).
- String literal patterns joined with `|`, as in `match a { "-i" | "--ignore-case" => ... }`, accept the short and long form at once. Assignment is a statement, not an expression, so those arm bodies are wrapped in a `{ ...; }` block ([Control flow](../learn/04-control-flow.html)).
- Bad arguments are thrown as the `ArgError` enum. `run` declares `throws (ArgError | IoError)`, and `main` uses one `catch` per type to choose the exit code (2 or 1) ([Errors](../learn/10-errors.html), [3.14.4 `catch`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3144-catch-postfix-handling-expression)).
- The number after `-m` is read with `parse::<usize>()`; on failure, `catch _ { throw ...; }` turns it into a friendlier error. `opts.max` is `usize | none`, so the comparison supplies a default with `?? usize::MAX`.

## Running it

To search real files, build an executable and pass arguments.

```bash
rokc build mgrep.rok -o mgrep
./mgrep -n -i rok notes.txt todo.txt
./mgrep --count error app.log
```
