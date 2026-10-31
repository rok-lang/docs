# 16. 도구

이 장에서는 코드를 쓰면서 늘 곁에 두는 도구를 둘러봅니다. 포매터, 린터, 테스트와 벤치마크, 문서 생성, 에디터 지원(LSP), REPL과 Jupyter, 그리고 컴파일러 안을 들여다보는 디버그 옵션입니다. 모두 `rok` 명령 하나에서 시작합니다.

이 장의 출력은 아래 패키지를 실제로 실행해 얻은 것입니다. `rok new mathx`로 만들고 `src/main.rok`을 다음과 같이 바꿨습니다.

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
fn fib_20(b: Bench) { b.iter(|| fib(20)); }
```

## `rok test`: 테스트

`#[test]`를 붙인 함수가 테스트입니다. 테스트 함수는 보통 빌드에서 빠지고, `rok test`가 테스트 함수를 모아 따로 빌드합니다. 검사는 `assert`, `assert_eq`, `assert_ne`로 합니다. 검사가 실패하면 panic이 일어납니다. 마지막 인자로 메시지를 줄 수 있습니다.

각 테스트는 자기 프로세스에서 실행되므로 하나가 panic으로 끝나도 나머지는 계속 실행됩니다. 위 패키지의 `fib_wrong`은 일부러 틀리게 만든 것입니다.

```text
$ rok test
   Compiling mathx (.)
    Finished in 0.69s

running 3 tests
test fib_works ... ok
test fib_wrong ... FAILED
test gcd_works ... ok

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

하나라도 실패하면 종료 코드가 1이므로 CI에서 그대로 쓸 수 있습니다. 이름의 일부(또는 `^`, `$`, `.*`를 쓰는 작은 정규식)를 인자로 주면 맞는 테스트만 실행합니다.

```text
$ rok test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.22s
```

통과한 테스트의 출력도 보려면 `--show-output`을, 동시에 실행할 테스트 수를 정하려면 `--threads=N`을 씁니다. CI 대시보드 같은 도구에는 `--format json`을 씁니다. 표준 출력에 테스트마다 하나, 마지막에 요약 하나씩 JSON 객체를 한 줄에 하나씩 쓰고, 사람이 읽는 글은 표준 에러로 보냅니다.

```text
$ rok test --format json 2>/dev/null
{"schema":1,"type":"case","kind":"test","name":"fib_works","result":"ok","ms":87.057}
{"schema":1,"type":"case","kind":"test","name":"fib_wrong","result":"failed","ms":87.144}
{"schema":1,"type":"case","kind":"test","name":"gcd_works","result":"ok","ms":87.05}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"filtered_out":0,"ms":87.276}
```

## `rok bench`: 벤치마크

`#[bench]` 함수는 `Bench`를 받아, 시간을 잴 코드를 `b.iter`에 넘깁니다. `rok bench`는 이 함수들을 최적화 빌드(`-O2`)로 만들고, 각 클로저를 여러 번 호출해 호출당 중앙값 시간과 그 편차를 출력합니다. `b.iter`를 한 번도 호출하지 않는 벤치마크에는 경고가 나옵니다. 수치는 기계와 실행마다 다릅니다.

```text
$ rok bench
   Compiling mathx (.)
    Finished in 0.42s

benchmarking 1 bench
  fib_20      12.99 us +/- 120.52 ns

bench result: ok. 1 run; 0 failed; 0 filtered out; finished in 2.63s
```

`--iters=N`은 표본당 호출 횟수(기본값은 100ms쯤 걸리는 횟수), `--samples=N`은 표본 수(기본값 10)입니다. 필터는 `rok test`와 같이 동작합니다. 출력은 터미널에서나 파이프에서나 같습니다. 결과를 읽는 도구는 `--format=rokbench`를 주면 벤치마크마다 `#rokbench` 줄 하나를 받고, `--format json`을 주면 `rok test`와 같은 JSON 줄을 받습니다. 벤치마크마다 `median_ns`, `mad_ns`, `samples`가 들어 있습니다.

## `rok fmt`: 포매터

`rok fmt`는 패키지의 `.rok`과 `.rokon` 파일을 표준 형식으로 다시 씁니다. Rok의 포매터는 일부러 보수적입니다. **토큰, 줄 안의 공백, 주석은 그대로 두고** 들여쓰기(괄호 깊이마다 4칸), 줄 끝 공백, 연속된 빈 줄만 고칩니다. 그래서 포매터가 코드의 의미나 손으로 맞춘 정렬을 바꾸는 일은 없습니다.

들여쓰기가 엉망이고 탭이 섞이고 줄 끝 공백이 있는 `fib`를 `--check`로 검사해 봅니다. 포매터는 파일을 건드리지 않고 바뀔 줄을 보여 준 뒤 종료 코드 1로 끝납니다. 이 모드는 CI에 잘 맞습니다.

```text
$ rok fmt --check
Diff in ./src/main.rok:
-   10   if n < 2 { return n; }
-   11   let unused = 1;
-   12 	fib(n - 1) + fib(n - 2)
+   10     if n < 2 { return n; }
+   11     let unused = 1;
+   12     fib(n - 1) + fib(n - 2)
        note: 1 file needs formatting; `rok fmt` without `--check` rewrites it
$ rok fmt
   Formatted 1 file, 1 already formatted
$ rok fmt --check
        note: 2 files already formatted
```

`--check` 없이 실행하면 파일을 고치고 몇 개를 고쳤는지 출력합니다. `-q`를 주면 그 줄을 생략합니다. 에디터에서 저장할 때마다 실행하면 편리합니다.

패키지 안에서는 어느 하위 디렉터리에서 실행해도 `rok fmt`가 패키지 전체를 포맷합니다. `target/`, 숨김 디렉터리, `node_modules/`, `.gitignore`에 있는 경로는 건너뜁니다. 패키지 밖에서는 포맷할 대상을 지정합니다. 파일(`rok fmt hi.rok`)이나 디렉터리(`rok fmt scripts/`)를 줍니다. 경로가 없으면 멈추고 그렇게 알려 줍니다.

```text
$ rok fmt
rok: error: not inside a package; pass a file or directory: `rok fmt <path>`
```

## `rok lint`와 `rok check`

`rok lint`는 컴파일하지 않고 타입 검사와 린트만 실행합니다. 위에 남겨 둔 `let unused = 1;`을 잡아냅니다.

```text
$ rok lint
    Checking mathx (.)
warning[E503]: unused variable `unused`
   --> src/main.rok:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `#[warn(unused_variables)]` on by default
     Checked mathx: 0 errors, 1 warning
    Finished in 0.05s
```

모든 린트의 기본 수준은 `warn`입니다. 수준은 세 곳에서 바꿀 수 있고, 뒤의 것이 우선합니다.

1. `rok.rokon`의 `lints` 필드: `lints: { "unused": "deny", "dead_code": "allow" }`
2. 명령줄의 `-A` / `-W` / `-D` (allow / warn / deny)
3. 소스의 속성: 항목에 `#[allow(dead_code)]`, 파일 맨 위에 `#![deny(unused)]`

린트 하나(`unused_variables`) 대신 그룹(`unused`, `nonstandard_style`, `warnings`, `all`)을 지정할 수도 있습니다. `deny` 린트가 하나라도 걸리면 종료 코드는 1입니다.

```text
$ rok lint -D unused
    Checking mathx (.)
error[E503]: unused variable `unused`
   --> src/main.rok:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `-D unused` on the command line
     Checked mathx: 1 error, 0 warnings
    Finished in 0.04s
```

경고가 있으면 CI를 실패시키려면 `warnings`를 deny하세요(`lints: { "warnings": "deny" }` 또는 `-D warnings`). rustc와 같이 이 그룹은 이후 릴리스가 추가하는 린트까지 포함해 경고 수준의 린트를 모두 담습니다. 새 릴리스 때문에 CI 결과가 바뀌면 안 된다면 `rok.rokon`에 toolchain을 정확한 버전으로 고정하세요(`toolchain: "=0.1.6"`). `all`은 모든 린트입니다. `rok.rokon`에 `"warnings": "deny"`를 적으면 마지막 note가 그 수준이 어디서 왔는지 알려 줍니다.

```text
$ rok lint
    Checking mathx (.)
error[E503]: unused variable `unused`
   --> src/main.rok:11:9
    |
 11 |     let unused = 1;
    |         ^^^^^^
    |
    = note: if this is intentional, prefix it with an underscore: `_unused`
    = note: `"warnings": "deny"` in `lints` of rok.rokon
     Checked mathx: 1 error, 0 warnings
    Finished in 0.11s
```

`deprecated`는 `#[deprecated]`가 붙은 항목을 쓰면 경고합니다. 함수, 메서드, 필드, 타입, 상수, static 모두 해당하고, 내 코드든 의존성이든 표준 라이브러리든 상관없습니다. 에디터는 그런 사용처에 취소선을 긋습니다. 전체 목록은 `rokc lint --list`로 봅니다. 린트마다 추가된 릴리스도 함께 나옵니다. 출력을 `head`로 파이프해도 됩니다. 읽는 쪽이 먼저 끝나면 `rok`과 `rokc`는 에러 없이 조용히 멈춥니다. `rok check`는 한 걸음 더 나아가 코드 생성까지 실행하지만, 링크하거나 실행 파일을 만들지는 않습니다. 빌드보다 빠르므로 컴파일되는지만 확인할 때 씁니다. 여기서부터의 예제는 에러와 저수준 출력이 `mathx`를 어지럽히지 않도록 버려도 되는 `hello` 패키지를 씁니다.

```text
$ rok check
    Checking hello (.)
error[E200]: mismatched types: `let s` expects `str`, found `i32`
  --> src/main.rok:2:14
   |
 2 | let s: str = square(7);
   |              ^^^^^^^^^
   |
   = help: to make text from it, write `f"{square(7)}"`
     Checked hello: 1 error, 0 warnings
    Finished in 0.04s
```

`rok build`와 `rok run`도 같은 진단을 출력하고, 빌드에 실패하면 `Finished` 대신 개수를 세어 끝냅니다.

```text
$ rok build
   Compiling hello (.)
...
error: could not build hello (1 error, 0 warnings)
```

기계가 읽을 출력이 필요하면 `--message-format=json`을 붙입니다. 진단 하나가 JSON 한 줄(`"schema": 1`)로 출력됩니다. 코드와 메시지, 시작과 끝의 줄·열이 담긴 위치, `help` 줄, 진단이 제안하는 `fixes`, 린트 경고라면 그 `lint` 이름, 규칙의 사양 절이 들어 있습니다.

대괄호 안의 코드는 에러의 종류를 가리킵니다. [에러 코드 목록](../errors/index.html)에 코드마다 예제와 고치는 법을 담은 페이지가 있습니다.

## `rok doc`: 문서 주석과 API 문서

`///`는 바로 아래 항목을 문서화하고, `//!`는 주석이 들어 있는 모듈(파일)을 문서화합니다. 위 패키지에서는 첫 줄 `//! Small math helpers.`가 모듈 설명이 되고, `gcd`와 `fib` 위의 `///`가 함수 설명이 됩니다. 문서 텍스트에는 제목, 목록, 인라인 코드, 코드 블록, 링크 같은 Markdown 문법을 쓸 수 있습니다.

```text
$ rok doc
 Documenting mathx (.)
    Finished in 0.10s
   Generated ./target/doc/index.html
```

`rok doc --open`은 만든 문서를 브라우저에서 엽니다.

`target/doc/`에 패키지와 그 의존성의 공개(`pub`) 항목의 HTML과 검색 색인이 생깁니다. 이 사이트의 [표준 라이브러리 API](../../std/core/index.html)도 같은 방식으로 라이브러리 소스의 문서 주석에서 만들었습니다.

## 에디터: `rokls`

`rokls`는 LSP(Language Server Protocol) 서버입니다. 진단, 호버, 정의로 이동, 참조 찾기, 이름 바꾸기, 문서 심볼, 자동 완성, 시그니처 도움말, 빠른 수정, 포맷 기능이 있습니다. 자동 완성은 범위 안의 지역 변수, 값의 멤버(`xs.`는 `List`의 공개 메서드를 보여 줍니다), `use` 뒤의 경로를 제안합니다. 호버와 정의로 이동은 표준 라이브러리 안까지 따라갑니다. 빠른 수정은 진단이 제안한 수정을 적용하고, 빠진 `;`나 `use`를 넣고, 없는 stone에는 `rok add`를 실행합니다. 이름 바꾸기는 라이브러리 항목, 키워드, 다른 파일의 이름이면 거절하고 이유를 알려 줍니다. `rok lsp`와 `rokc lsp`도 같은 서버입니다. 표준 입출력으로 통신하므로 LSP를 지원하는 에디터라면 실행할 명령만 알려 주면 됩니다. 터미널에서 직접 실행하면 `rokls`는 에디터가 실행하는 서버라고만 알려 줍니다.

Neovim(0.11 이상)에서는 `init.lua`에 다음을 넣습니다.

```text
vim.filetype.add({ extension = { rok = "rok", rokon = "rokon" } })
vim.lsp.config("rokls", {
  cmd = { "rokls" },
  filetypes = { "rok", "rokon" },
  root_markers = { "rok.rokon" },
})
vim.lsp.enable("rokls")
```

Helix에서는 `languages.toml`에 다음을 넣습니다.

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

VS Code 같은 다른 에디터에서는 범용 LSP 클라이언트 확장에 `rokls`를 등록합니다. `rokls`는 `PATH`에 있어야 합니다.

## REPL과 Jupyter

1장의 REPL(`rok` 또는 `rok repl`)을 좀 더 자세히 봅니다. 터미널에서는 두 줄로 시작합니다. toolchain 버전과 호스트, 그리고 도움말 안내입니다. `rok repl -q`는 이 두 줄을 생략합니다. 입력이 파이프로 들어오면 처음부터 나오지 않습니다.

```text
$ rok repl
Rok 0.1.6 (9103a4909 2026-10-31) on aarch64-apple-darwin
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

프롬프트는 `>>>`이고, 셀이 이어지는 동안에는 `...`입니다. 입력이 분명히 끝나지 않은 경우가 아니면 셀은 줄 끝에서 끝납니다. 끝나지 않은 경우는 세 가지입니다. 괄호, 문자열, 블록 주석이 아직 열려 있는 경우, 줄이 연산자, `.`, `,`, `=`로 끝나는 경우, `fn f(x: i32)`처럼 본문을 기다리는 항목 머리인 경우입니다. 셀의 마지막 문장이 `let`이나 대입이면 `;`을 생략할 수 있습니다.

각 셀은 한 번 실행됩니다. 최상위 `let` 바인딩과 `fn` / `struct` / `impl` 정의는 이후 셀로 이어집니다. 셀의 최상위에서 `await`를 쓸 수 있습니다. 식 셀은 값을 `Display`로, 없으면 `Debug`로, 그것도 없으면 타입만 출력합니다. 끝에 `?`를 붙이면 대신 `Debug` 형태를 출력합니다. `let` 셀은 아무것도 보여 주지 않습니다. 문장이 여럿인 셀은 마지막이 식으로 끝나면 그 식의 값을 보여 줍니다. 셀이 보여 준 값은 `it`으로 남으므로 다음 셀에서 쓸 수 있습니다. 직접 정의한 `it`이 우선합니다.

일반 소스와 달리 같은 이름을 `let`으로 다시 선언할 수 있고, 타입이 달라도 됩니다. `fn`이나 `struct`를 다시 정의하면 새 정의로 바뀌고, 이전 셀의 코드도 새 버전을 호출합니다. 컴파일에 실패하거나 panic이 일어나거나 에러를 던진 셀은 세션을 바꾸지 않습니다. 이미 쓴 출력은 남습니다. `:reset`은 빈 세션에서 다시 시작합니다.

에러는 입력한 셀 안을 `<repl>:line:col`로 가리킵니다. panic과 이전 셀의 `fn` 안에서 난 에러는 그 셀을 `<repl cell N>:line:col`로 가리킵니다. N은 지금까지 입력한 셀의 번호입니다. Jupyter에서 N은 노트북 셀의 실행 번호이고, REPL 셀 여러 개로 나뉘는 노트북 셀은 `N.1`, `N.2`처럼 번호를 붙입니다.

```text
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

| 명령 | 하는 일 |
|------|---------|
| `expr?` | `Display` 대신 `Debug` 형태를 출력 |
| `:type expr` (`:t`) | 실행하지 않고 타입을 출력 |
| `:doc name` | 이름의 시그니처와 문서 주석을 출력. 메서드는 `:doc xs.len` |
| `:time expr` | 식을 실행하고 단계마다 걸린 시간을 출력 |
| `:vars` | 바인딩을 `name: type`으로 나열한 뒤 항목을 나열 |
| `:load file`, `:reload` | 프로그램을 세션에 로드. 마지막 것을 다시 로드. `:save`로 만든 스크립트는 셀 단위로 다시 실행 |
| `:edit` | `$VISUAL` / `$EDITOR`에서 셀을 쓴 뒤 실행 |
| `:save file` | 세션을 셀마다 표시한 스크립트로 저장 |
| `:opt [level]` | 이후 셀의 최적화 수준을 보거나 설정(`0` ~ `3`, `s`, `z`) |
| `:reset` | 모든 셀을 잊고 다시 시작 |
| `:add name ..` | 의존성 추가. 환경에는 `rok install`, 패키지에서는 `rok add` |
| `:help` (`:h`) | 모든 명령을 나열 |
| `:exit`, `:quit` (`:q`) | 종료. `exit`나 `quit`만 쓰거나 입력이 끝나도 종료 |

변수를 캡처하는 클로저는 셀의 최상위 `let`에 남겨 둘 수 없습니다. 클로저마다 이름 없는 자기 타입이 있기 때문입니다. REPL은 그런 셀을 거부합니다.

```text
>>> let x = 3
>>> let f = |n: i32| n * x
error: `f: closure(i32) -> i32` cannot be kept for later cells (its type has no name)
```

바인딩에 함수 타입을 주면(`let f: fn(i32) -> i32 = |n: i32| n * x`) 이후 셀로 이어집니다. `fn`도 이어지지만 `x` 같은 셀 변수는 보지 못하므로 인자로 넘기세요. `xs.iter().map(|n| n * x)`처럼 셀 안에서 쓰는 클로저는 평소대로 동작합니다.

터미널에서 REPL은 줄 편집을 직접 합니다.

- Tab은 이름을 완성합니다. 대상은 범위 안의 지역 변수, 세션의 바인딩과 항목, core 라이브러리, 키워드이고, `x.` 뒤에서는 `x`의 공개 필드와 메서드, `a::` 뒤에서는 모듈 `a`의 항목입니다. 줄 맨 앞에서 Tab은 들여씁니다.
- 기록은 셀 단위로 남고, 세션이 끝나도 유지됩니다(`$ROK_HOME/repl_history`). 위와 아래 화살표로 오갑니다. 텍스트를 조금 입력해 두면 그 텍스트로 시작하는 셀만 거칩니다. Ctrl-R은 거꾸로 검색합니다.
- 붙여 넣은 블록은 줄마다 실행되지 않고, Enter를 누르면 셀 하나가 됩니다.
- 입력은 이 가이드처럼 색칠됩니다. 색을 끄려면 `NO_COLOR`를 설정합니다.

표준 입력을 읽는 셀은 터미널을 읽습니다. REPL 자신의 입력이 파이프로 들어오면 셀은 빈 입력을 봅니다.

`rok repl path`는 패키지나 파일을 세션에 로드합니다. 라이브러리라면 패키지를 의존성으로 추가하고 `use name::*;`를 먼저 실행합니다. `:reset` 뒤에도 다시 실행합니다. 라이브러리를 만들면서 함수를 시험해 보기에 편리합니다. 프로그램이라면 프로그램을 실행한 뒤 그 바인딩과 항목을 세션에 남깁니다. 패키지 안에서 `rok repl .`은 현재 패키지를 로드하고, 그냥 `rok repl`은 빈 상태로 시작합니다.

같은 셀 엔진을 Jupyter 노트북에서도 쓸 수 있습니다. 커널을 한 번 등록하면 Jupyter, VS Code 노트북 등에서 `rok` 커널을 고를 수 있습니다.

```text
$ rok jupyter install
   Installed kernel `rok` (/Users/me/Library/Jupyter/kernels/rok/kernel.json)
```

macOS에서는 `~/Library/Jupyter` 아래에, Windows에서는 `%APPDATA%\jupyter`에, 그 밖에서는 `~/.local/share/jupyter`에 설치됩니다. `JUPYTER_DATA_DIR`로 위치를 바꿀 수 있습니다. `rok jupyter install --env name`은 이름 있는 환경의 커널을 설치합니다.

노트북 셀은 REPL의 규칙을 따르고 마지막 식의 값을 보여 줍니다. 출력은 끝에서만이 아니라 셀이 실행되는 동안 나타납니다. 표준 입력을 읽는 셀은 노트북에 입력 상자를 띄웁니다. Tab 완성과 Shift-Tab 도움말은 REPL과 같은 이름을 씁니다. 노트북 프런트엔드가 Rok을 알게 될 때까지 코드는 Rust 규칙으로 강조됩니다.

## 컴파일러 안 들여다보기

빌드 단계마다 걸리는 시간을 보려면 `-v`를 붙입니다.

```text
$ rok build -v
         Std prebuilt (/Users/me/.rok/toolchains/0.1.6-aarch64-apple-darwin/lib/rok-cache/5cbf37af5441372c-fe4fdd5b06e0f175-3074122f20802b47/rok.rokhir)
   Compiling hello (.)
   Stage: front end (34ms)
      Reused: 93 of 94 files parsed, 0 of 2 bodies lowered, the library's items collected
    Building ./target/debug/hello
       Units 0 reused, 3 compiled
   Stage: codegen + link (24ms)
    Finished in 0.07s
```

`--emit`은 중간 산출물을 `target/debug/build/`에 씁니다. 종류는 `source`(메타 전개와 디슈거링을 마친 패키지 전체를 Rok 소스로 쓴 것), `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expansions`(메타 전개 하나하나의 출력)입니다. 예전 이름인 `ast`와 `expanded`도 아직 동작하지만 경고가 나오고, 0.2.0에서 없어집니다.

```text
$ rok emit --emit=llvmir
   Compiling hello (.)
       Wrote ./target/debug/build/hello.ll
    Finished in 0.04s
$ grep define target/debug/build/hello.ll | head -2
define internal i32 @_Y6cfff936__square_(i32 %0, ptr %1) #0 {
define internal void @_Y6cfff936__main_(ptr %0) #0 {
```

`_Y`로 시작하는 이름은 Rok의 맹글링된 심볼입니다. 디버거나 프로파일러에서 만나면 `rokc demangle`로 푸세요.

```text
$ rokc demangle _Y6cfff936__square_
package:  6cfff936
path:     (root)
name:     square
generics: (none)
```

`-g`로 빌드하면 DWARF 디버그 정보가 들어가므로 `lldb` / `gdb`에서 소스 줄 단위로 따라갈 수 있습니다. gdb는 합 타입이나 enum의 활성 변형만 스스로 보여 줍니다. lldb용으로는 toolchain에 같은 일을 하는 포매터가 들어 있습니다. `rokc --print lldbinit`이 그 포매터를 로드하는 명령을 출력합니다.

```text
$ lldb -O "$(rokc --print lldbinit)" ./target/debug/hello
$ rokc --print lldbinit >> ~/.lldbinit      # or load them every time
```

## 도움말 보기

`rok help`는 명령을 묶어서 보여 주고, `rok help <command>`(또는 `rok <command> --help`)는 명령 하나의 사용법, 예, 옵션을 보여 줍니다.

```text
$ rok help fmt
usage: rok fmt [path] [options]

Format .rok and .rokon files in place: <path> is one file, or a directory
whose files are all formatted (default: the current one).

examples:
  rok fmt                      format everything here
  rok fmt --check              exit 1 if a file would change (CI)
  rok fmt --diff src/main.rok  show the change, write nothing

fmt options:    --check (exit 1 if a file would change), --diff

global options (-q, -v, --color, ...): `rok help`
```

명령이나 옵션의 철자를 틀리면 가장 가까운 이름을 알려 줍니다.

```text
$ rok tset
rok: unknown command `tset`; did you mean `test`?
run `rok help` for usage
```

`-q`, `-v`, `--color` 같은 전역 옵션은 명령 앞에도 뒤에도 쓸 수 있습니다.

## 요약

- `rok test` / `rok bench`는 `#[test]` / `#[bench]` 함수를 따로 빌드해 실행합니다. 테스트는 각자 자기 프로세스에서 돕니다.
- `rok fmt`는 들여쓰기와 빈 줄만 고칩니다. CI에서는 `rok fmt --check`를 씁니다. 패키지 밖에서는 경로를 줍니다.
- `rok lint`는 린트를 실행하고, `rok check`는 코드 생성까지 갑니다. 린트 수준은 `rok.rokon` < `-A/-W/-D` < 소스 속성 순으로 정해집니다.
- `rok doc`은 `///`와 `//!` 문서 주석으로 `target/doc/`에 HTML을 생성합니다.
- 에디터에 `rokls`를 LSP 서버로 등록합니다.
- REPL의 프롬프트는 `>>>`이고, 마지막 값을 `it`에 두며, `:type`, `:doc`, `:vars`, `:reset` 같은 명령이 있습니다. `rok repl .`은 현재 패키지를 로드하고, 노트북에는 `rok jupyter install`을 씁니다.
- 문제를 파고들 때는 `-v`, `--emit`, `rokc demangle`, `-g`를 씁니다. lldb에서는 `rokc --print lldbinit`과 함께 씁니다.
- `rok help <command>`는 명령 하나의 사용법과 옵션을 보여 줍니다.

## 더 읽을거리

- 사양: [rokc CLI](../spec/refs-lang/009-tools/901-rokc-cli.html), [표준 포맷](../spec/refs-lang/009-tools/901-rokc-cli.html#9111-표준-형식rokc-fmt), [린트 제어](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-린트-제어), [Rok 프런트엔드](../spec/refs-lang/009-tools/910-rok-frontend.html), [JIT 셀과 REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [REPL 줄 편집](../spec/refs-lang/009-tools/907-jit-cells.html#줄-편집), [REPL 명령](../spec/refs-lang/009-tools/909-debug-tools.html#999-repl--jupyter-디버그), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [디버그 도구](../spec/refs-lang/009-tools/909-debug-tools.html), [진단](../spec/refs-lang/009-tools/909-debug-tools.html#994-진단--에러-메시지), [심볼 맹글링](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- [에러 코드](../errors/index.html)
- API: [rok::core의 assert 함수](../../std/core/panic.html)
- 다음 장: [17. 조건부 컴파일](17-conditional-compilation.html)
