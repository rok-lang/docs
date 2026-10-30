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
fn fib_20() {
    fib(20);
}
```

## `rok test`: 테스트

`#[test]`를 붙인 함수가 테스트입니다. 테스트 함수는 보통 빌드에서 빠지고, `rok test`가 테스트 함수를 모아 따로 빌드합니다. 검사는 `assert`, `assert_eq`, `assert_ne`로 합니다. 검사가 실패하면 panic이 일어납니다. 마지막 인자로 메시지를 줄 수 있습니다.

각 테스트는 자기 프로세스에서 실행되므로 하나가 panic으로 끝나도 나머지는 계속 실행됩니다. 위 패키지의 `fib_wrong`은 일부러 틀리게 만든 것입니다.

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

하나라도 실패하면 종료 코드가 1이므로 CI에서 그대로 쓸 수 있습니다. 이름의 일부(또는 `^`, `$`, `.*`를 쓰는 작은 정규식)를 인자로 주면 맞는 테스트만 실행합니다.

```text
$ rok test gcd
   Compiling mathx (.)
    Finished in 0.37s

running 1 test
test gcd_works ... ok

test result: ok. 1 passed; 0 failed; 2 filtered out; finished in 0.22s
```

통과한 테스트의 출력도 보려면 `--show-output`을, 동시에 실행할 테스트 수를 정하려면 `--threads=N`을 씁니다.

## `rok bench`: 벤치마크

`rok bench`는 최적화 빌드(`-O2`)에서 각 `#[bench]` 함수를 여러 번 호출해 호출당 평균 시간을 잽니다. 수치는 기계와 실행마다 다릅니다.

```text
$ rok bench
   Compiling mathx (.)
    Finished in 0.42s

running 1 bench
bench fib_20 ... bench:       34,204 ns/iter (+/- 1,712)

bench result: ok. 1 measured; 0 failed; 0 filtered out; finished in 0.36s
```

`--iters=N`은 표본당 호출 횟수, `--samples=N`은 표본 수입니다. 필터는 `rok test`와 같이 동작합니다.

## `rok fmt`: 포매터

`rok fmt`는 패키지의 `.rok`과 `.rokon` 파일을 표준 형식으로 다시 씁니다. Rok의 포매터는 일부러 보수적입니다. **토큰, 줄 안의 공백, 주석은 그대로 두고** 들여쓰기(괄호 깊이마다 4칸), 줄 끝 공백, 연속된 빈 줄만 고칩니다. 그래서 포매터가 코드의 의미나 손으로 맞춘 정렬을 바꾸는 일은 없습니다.

들여쓰기가 엉망이고 탭이 섞이고 줄 끝 공백이 있는 `fib`를 `--check`로 검사해 봅니다. 포매터는 파일을 건드리지 않고 바뀔 줄을 보여 준 뒤 종료 코드 1로 끝납니다. 이 모드는 CI에 잘 맞습니다.

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

`--check` 없이 실행하면 아무 말 없이 파일을 고칩니다. 에디터에서 저장할 때마다 실행하면 편리합니다.

## `rok lint`와 `rok check`

`rok lint`는 컴파일하지 않고 타입 검사와 린트만 실행합니다. 위에 남겨 둔 `let unused = 1;`을 잡아냅니다.

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

모든 린트의 기본 수준은 `warn`입니다. 수준은 세 곳에서 바꿀 수 있고, 뒤의 것이 우선합니다.

1. `rok.rokon`의 `lints` 필드: `lints: { "unused": "deny", "dead_code": "allow" }`
2. 명령줄의 `-A` / `-W` / `-D` (allow / warn / deny)
3. 소스의 속성: 항목에 `#[allow(dead_code)]`, 파일 맨 위에 `#![deny(unused)]`

린트 하나(`unused_variables`) 대신 그룹(`unused`, `nonstandard_style`, `all`)을 지정할 수도 있습니다. `deny` 린트가 하나라도 걸리면 종료 코드는 1입니다.

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

전체 목록은 `rokc lint --list`로 봅니다. 출력을 `head`로 파이프해도 됩니다. 읽는 쪽이 먼저 끝나면 `rok`과 `rokc`는 에러 없이 조용히 멈춥니다. `rok check`는 한 걸음 더 나아가 코드 생성까지 실행하지만, 링크하거나 실행 파일을 만들지는 않습니다. 빌드보다 빠르므로 컴파일되는지만 확인할 때 씁니다. 여기서부터의 예제는 에러와 저수준 출력이 `mathx`를 어지럽히지 않도록 버려도 되는 `hello` 패키지를 씁니다.

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

기계가 읽을 출력이 필요하면 `--message-format=json`을 붙입니다. 진단 하나가 JSON 한 줄로 출력됩니다.

## `rok doc`: 문서 주석과 API 문서

`///`는 바로 아래 항목을 문서화하고, `//!`는 주석이 들어 있는 모듈(파일)을 문서화합니다. 위 패키지에서는 첫 줄 `//! Small math helpers.`가 모듈 설명이 되고, `gcd`와 `fib` 위의 `///`가 함수 설명이 됩니다. 문서 텍스트에는 제목, 목록, 인라인 코드, 코드 블록, 링크 같은 Markdown 문법을 쓸 수 있습니다.

```text
$ rok doc
 Documenting mathx (.)
    Finished in 0.22s
   Generated ./target/doc/index.html
```

`target/doc/`에 패키지와 그 의존성의 공개(`pub`) 항목의 HTML과 검색 색인이 생깁니다. 이 사이트의 [표준 라이브러리 API](../../std/core/index.html)도 같은 방식으로 라이브러리 소스의 문서 주석에서 만들었습니다.

## 에디터: `rokls`

`rokls`는 LSP(Language Server Protocol) 서버입니다. 진단, 호버, 정의로 이동, 문서 심볼, 자동 완성, 포맷 기능이 있습니다. `rok lsp`와 `rokc lsp`도 같은 서버입니다. 표준 입출력으로 통신하므로 LSP를 지원하는 에디터라면 실행할 명령만 알려 주면 됩니다.

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

1장의 REPL(`rok` 또는 `rok repl`)을 좀 더 자세히 봅니다. 각 셀은 한 번 실행됩니다. 최상위 `let` 바인딩과 `fn` / `struct` 정의는 이후 셀로 이어집니다. 식 셀은 값을 `Display`로 출력합니다. 끝에 `?`를 붙이면 대신 `Debug` 형태로 출력하고, `Display`가 없는 값은 타입만 출력합니다. `let` 셀은 아무것도 보여 주지 않습니다. 문장이 여럿인 셀은 마지막이 식으로 끝나면 그 식의 값을 보여 줍니다. 일반 소스와 달리 같은 이름을 `let`으로 다시 선언할 수 있고, `fn`을 다시 정의하면 새 정의로 바뀝니다. 에러가 난 셀은 흔적을 남기지 않습니다.

```text
$ rok repl
> let xs = [3, 1, 2];
> xs.len()
3
> xs.sort(); xs?
[1, 2, 3]
> :type xs
List<i32>
> fn twice(x: i32) -> i32 { x * 2 }
> twice(21)
42
> :quit
```

| 명령 | 하는 일 |
|------|---------|
| `expr?` | `Display` 대신 `Debug` 형태를 출력 |
| `:type expr` | 실행하지 않고 타입을 출력 |
| `:reset` | 모든 셀을 잊고 다시 시작 |
| `:add name`, `:add name --path path` | 현재 세션에 의존성을 추가. 환경에는 설치하고 패키지에는 추가하며, 스크립트에는 추가하지 않음 |
| `:help`, `:quit` | 도움말을 출력, 종료 |

변수를 캡처하는 클로저는 셀의 최상위 `let`에 남겨 둘 수 없습니다. 클로저마다 이름 없는 자기 타입이 있기 때문입니다. REPL은 그런 셀을 거부합니다.

```text
> let x = 3;
> let f = |n: i32| n * x;
error: `f: closure(i32) -> i32` cannot be kept for later cells (its type has no name)
```

바인딩에 함수 타입을 주면(`let f: fn(i32) -> i32 = |n: i32| n * x;`) 이후 셀로 이어집니다. `fn`도 이어지지만 `x` 같은 셀 변수는 보지 못하므로 인자로 넘기세요. `xs.iter().map(|n| n * x)`처럼 셀 안에서 쓰는 클로저는 평소대로 동작합니다.

`rok repl --package path`는 라이브러리 패키지를 붙이고 `use name::*;`를 미리 실행한 상태로 시작합니다. 라이브러리를 만들면서 함수를 시험해 보기에 편리합니다.

같은 셀 엔진을 Jupyter 노트북에서도 쓸 수 있습니다. 커널을 한 번 등록하면 Jupyter, VS Code 노트북 등에서 `rok` 커널을 고를 수 있습니다.

```text
$ rok jupyter install
   Installed kernel `rok` (/Users/me/Library/Jupyter/kernels/rok/kernel.json)
```

macOS에서는 `~/Library/Jupyter` 아래에, 그 밖에서는 `~/.local/share/jupyter`에 설치됩니다. `JUPYTER_DATA_DIR`로 위치를 바꿀 수 있습니다.

## 컴파일러 안 들여다보기

빌드 단계마다 걸리는 시간을 보려면 `-v`를 붙입니다.

```text
$ rok build -v
   Compiling hello (.)
   Stage: front end (250ms)
    Building ./target/debug/hello
   Stage: codegen + link (160ms)
    Finished in 0.41s
```

`--emit`은 중간 산출물을 `target/debug/build/`에 씁니다. 종류는 `ast`, `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`(메타 전개 결과)입니다.

```text
$ rok emit --emit=llvmir
   Compiling hello (.)
       Wrote ./target/debug/build/hello.ll
    Finished in 0.32s
$ grep define target/debug/build/hello.ll | head -2
define i32 @_Y2cfee941__square_(i32 %0) {
define void @_Y2cfee941__main_() {
```

`_Y`로 시작하는 이름은 Rok의 맹글링된 심볼입니다. 디버거나 프로파일러에서 만나면 `rokc demangle`로 푸세요.

```text
$ rokc demangle _Y2cfee941__square_
package:  2cfee941
path:     (root)
name:     square
generics: (none)
```

`-g`로 빌드하면 DWARF 디버그 정보가 들어가므로 `lldb` / `gdb`에서 소스 줄 단위로 따라갈 수 있습니다. gdb는 합 타입이나 enum의 활성 변형만 스스로 보여 줍니다. lldb용으로는 toolchain에 같은 일을 하는 포매터가 들어 있습니다. `rokc --print lldbinit`이 그 포매터를 로드하는 명령을 출력합니다.

```text
$ lldb -O "$(rokc --print lldbinit)" ./target/debug/hello
$ rokc --print lldbinit >> ~/.lldbinit      # or load them every time
```

## 요약

- `rok test` / `rok bench`는 `#[test]` / `#[bench]` 함수를 따로 빌드해 실행합니다. 테스트는 각자 자기 프로세스에서 돕니다.
- `rok fmt`는 들여쓰기와 빈 줄만 고칩니다. CI에서는 `rok fmt --check`를 씁니다.
- `rok lint`는 린트를 실행하고, `rok check`는 코드 생성까지 갑니다. 린트 수준은 `rok.rokon` < `-A/-W/-D` < 소스 속성 순으로 정해집니다.
- `rok doc`은 `///`와 `//!` 문서 주석으로 `target/doc/`에 HTML을 생성합니다.
- 에디터에 `rokls`를 LSP 서버로 등록합니다.
- REPL에는 `:type` / `:reset` / `:add` 명령이 있고, 노트북에는 `rok jupyter install`을 씁니다.
- 문제를 파고들 때는 `-v`, `--emit`, `rokc demangle`, `-g`를 씁니다. lldb에서는 `rokc --print lldbinit`과 함께 씁니다.

## 더 읽을거리

- 사양: [rokc CLI](../spec/refs-lang/009-tools/901-rokc-cli.html), [표준 포맷](../spec/refs-lang/009-tools/901-rokc-cli.html#9111-표준-형식rokc-fmt), [린트 제어](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#629-allow--deny--warn-린트-제어), [Rok 프런트엔드](../spec/refs-lang/009-tools/910-rok-frontend.html), [JIT 셀과 REPL](../spec/refs-lang/009-tools/907-jit-cells.html), [Jupyter](../spec/refs-lang/009-tools/908-jupyter-integration.html), [디버그 도구](../spec/refs-lang/009-tools/909-debug-tools.html), [심볼 맹글링](../spec/refs-lang/009-tools/905-symbol-mangling.html)
- API: [rok::core의 assert 함수](../../std/core/panic.html)
- 다음 장: [17. 조건부 컴파일](17-conditional-compilation.html)
