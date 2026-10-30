# 명령줄 도구

`grep`을 닮은 작은 도구 `mgrep`은 명령줄 인자를 직접 파싱해서 구조체에 담고, `--help`와 잘못된 인자를 `throws`로 처리합니다.

## 코드

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

인자 없이 실행하면 내장된 예시 호출 몇 가지를 차례로 시도합니다.

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

## 설명

- `env::args()`는 `[0]`에 프로그램 이름이 들어 있는 `List<str>`이므로, `1..argv.len()`만 복사합니다([`env::args`](../../std/std/env.html#fn.args)).
- 옵션은 필드 기본값을 가진 `Options` 구조체에 담깁니다. 기본값이 있는 필드는 `Options { pattern: "" }`처럼 생략할 수 있습니다([구조체, enum, 트레잇](../learn/06-structs-enums-traits.html), [2.3.3 필드 기본값](../spec/refs-lang/002-types/203-struct.html#233-필드-기본값)).
- `match a { "-i" | "--ignore-case" => ... }`처럼 문자열 리터럴 패턴을 `|`로 이으면 짧은 형태와 긴 형태를 한 번에 받습니다. 대입은 표현식이 아니라 문장이므로, 그 가지의 본체는 `{ ...; }` 블록으로 감쌉니다([제어 흐름](../learn/04-control-flow.html)).
- 잘못된 인자는 `ArgError` enum으로 던집니다. `run`은 `throws (ArgError | IoError)`를 선언하고, `main`은 타입마다 `catch`를 하나씩 두어 종료 코드(2 또는 1)를 고릅니다([에러 처리](../learn/10-errors.html), [3.14.4 `catch`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3144-catch-후위-처리-식)).
- `-m` 뒤의 숫자는 `parse::<usize>()`로 읽습니다. 실패하면 `catch _ { throw ...; }`가 더 알기 쉬운 에러로 바꿉니다. `opts.max`는 `usize | none`이므로, 비교할 때 `?? usize::MAX`로 기본값을 줍니다.

## 실행하기

실제 파일을 검색하려면 실행 파일을 빌드하고 인자를 넘깁니다.

```bash
rokc build mgrep.rok -o mgrep
./mgrep -n -i rok notes.txt todo.txt
./mgrep --count error app.log
```
