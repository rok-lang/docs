# 1. 시작하기

이 장에서는 rok toolchain을 설치하고 첫 패키지를 만들어 실행합니다. 단일 파일 스크립트와 REPL도 씁니다. 이 장을 마치면 가이드의 나머지 예제를 직접 실행할 수 있습니다.

## 설치

### 준비물

`rokc`는 필요한 것을 모두 갖추고 있습니다. LLVM, MLIR, lld 링커가 들어 있으므로 LLVM을 따로 설치할 필요가 없습니다([9.2.18](../spec/refs-lang/009-tools/902-aot-pipeline.html#9218-내장-toolchain)).

- macOS: Xcode Command Line Tools (`xcode-select --install`). 마지막 링크 단계에 SDK가 필요합니다.
- Linux: 필요 없습니다. toolchain과 toolchain으로 빌드한 프로그램에는 glibc 2.34 이상(Ubuntu 22.04, Debian 12, RHEL 9 이후)이 필요합니다.
- Windows: 필요 없습니다.

릴리스 아카이브(archive)는 macOS arm64, Linux x86_64와 arm64(glibc와 정적 musl 둘 다), Windows x86_64용으로 빌드합니다. macOS x86_64, Windows arm64 같은 다른 호스트용 릴리스 아카이브는 아직 없습니다. 이런 호스트에서는 소스에서 Rok을 빌드하세요.

### 설치 스크립트

설치 스크립트는 [GitHub releases](https://github.com/rok-lang/rok/releases)에서 플랫폼에 맞는 toolchain을 내려받아 SHA-256으로 검증하고 `~/.rok`에 풉니다.

```bash
curl -fsSL https://rok-lang.org/install.sh | sh
```

Windows에서는 PowerShell에서 다음을 실행합니다. `%USERPROFILE%\.rok`에 설치하고, 그 아래 `bin`을 사용자 `PATH`에 추가합니다.

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://rok-lang.org/install.ps1 | iex"
```

`install.sh --version <v>`와 `install.ps1 -Version <v>`로 릴리스를 고릅니다.

설치 위치는 `ROK_HOME` 환경 변수로 바꿀 수 있습니다(기본값 `~/.rok`). 스크립트가 끝나면 다음 구조가 생깁니다.

```text
~/.rok/
├── bin/                        # rok, rokc, rokls proxy
└── toolchains/
    └── 0.1.1-aarch64-apple-darwin/
        ├── bin/{rok, rokc, rokls}
        └── lib/                # runtime and standard library sources
```

방금 설치한 toolchain이 기본값이 됩니다. `~/.rok/bin`의 세 실행 파일은 **프록시**입니다. 실제 일은 활성 toolchain에 있는 같은 이름의 실행 파일이 합니다. 그래서 여러 toolchain을 나란히 설치해 두고 하나를 골라 쓸 수 있습니다. rustup과 같은 방식입니다. 이 디렉터리를 `PATH`에 추가하세요. Windows에서는 설치 프로그램이 이미 추가해 두었습니다.

```bash
# bash / zsh
echo 'export PATH="$HOME/.rok/bin:$PATH"' >> ~/.zshrc
# fish
fish_add_path ~/.rok/bin
```

새 셸에서 버전을 확인합니다.

```bash
rok --version
rok toolchain list
```

실행 파일 세 개가 설치됩니다.

| 실행 파일 | 하는 일 |
|------------|--------------|
| `rokc`     | 컴파일러. 빌드, 실행, 테스트, 포맷([9.1](../spec/refs-lang/009-tools/901-rokc-cli.html)) |
| `rok`      | 패키지 / 빌드 프런트엔드. `rok.rokon`, 의존성, 환경, REPL, Jupyter([9.10](../spec/refs-lang/009-tools/910-rok-frontend.html)) |
| `rokls`    | 언어 서버(LSP) |

### toolchain 관리

`rok toolchain`은 설치된 toolchain을 관리합니다. 활성 toolchain은 명령줄의 `+version`(`rok +0.1 build`), `ROK_TOOLCHAIN` 환경 변수, 가장 가까운 `rok.rokon`의 `toolchain` 필드, 기본값 순서로 정해집니다.

```bash
rok toolchain list                     # installed toolchains
rok toolchain default 0.1.1            # change the default
rok toolchain install 0.2.0 --from rok-0.2.0-aarch64-apple-darwin.tar.gz
rok toolchain update                   # download the latest release and make it the default
```

`rok toolchain update 0.2.0`은 최신 릴리스 대신 그 버전을 설치합니다. 내려받은 파일은 공개된 SHA-256으로 검증합니다. 어느 단계에서든 실패하면 이전 기본값이 그대로 남습니다. 이전 toolchain은 지우지 않습니다. 패키지의 `rok.lock`에 고정된 버전을 올리는 `rok update`와 헷갈리지 마세요([11장](11-modules-packages.html)).

### 소스에서 빌드하기(선택)

toolchain 아카이브를 직접 빌드해 같은 스크립트로 설치할 수도 있습니다. Rust toolchain(`rustup`)이 필요합니다.

```bash
git clone https://github.com/rok-lang/rok && cd rok
./dist.sh                                  # dist/rok-<version>-<triple>.tar.gz
sh install.sh --archive dist/rok-*.tar.gz  # into ~/.rok
```

처음 실행할 때는 고정된 버전의 LLVM도 `~/.cache/rok`에 빌드합니다(cmake와 ninja 필요, 약 한 시간). Linux에서는 `./dist-linux.sh`(podman 또는 docker)가 같은 아카이브를 컨테이너 안에서 빌드합니다. 그래서 결과물은 glibc 2.34 이상에서 돌아갑니다. 릴리스 아카이브도 이 방식으로 빌드합니다. 자세한 내용은 저장소의 README에 있습니다.

컴파일러를 개발하고 있다면 설치하지 않고 빌드 결과를 바로 써도 됩니다. 고정된 LLVM이 없으면 이렇게 빌드한 `rokc`는 `-O0`으로만 컴파일합니다.

```bash
cargo build --release
export PATH="$PWD/target/release:$PATH"
```

## 첫 패키지

`rok new`는 패키지 디렉터리를 만들고, `target/`을 제외하는 `.gitignore`와 함께 git 저장소를 초기화합니다. 패키지 이름은 소문자, 숫자, `-`, `_`로 이루어지며 문자로 시작합니다. `rok`은 표준 라이브러리가 쓰는 이름이므로 패키지 이름으로 쓸 수 없습니다. 코드에서는 `-` 자리에 `_`를 씁니다(`rok new my-lib`으로 만든 패키지는 `use my_lib::..`로 가져옵니다).

```bash
rok new hello
cd hello
```

```text
     Created executable package (hello)
```

파일은 두 개입니다. 매니페스트인 `rok.rokon`에는 패키지 이름, 버전, 의존성이 들어 있습니다.

```rokon
Package { name: "hello", version: "0.1.0", deps: {} }
```

소스는 `src/`에 있습니다. `src/main.rok`은 인사를 출력하는 한 줄짜리 코드입니다.

```rok
println("Hello, world!");
```

```output
Hello, world!
```

`rok run`은 빌드하고 실행합니다. 결과물은 `target/debug/`에 생깁니다.

```bash
rok run
```

```text
   Compiling hello (.)
    Finished in 1.55s
     Running `./target/debug/hello`
Hello, world!
```

`src/main.rok`의 최상위 문은 위에서 아래로 실행되고, 이 문이 곧 프로그램입니다. 함수와 타입 선언은 최상위 문 사이 어디에 두어도 됩니다. 그 밖에 자주 쓰는 명령은 `rok build`(빌드만), `rok check`(타입 검사만), `rok test`, `rok fmt`입니다.

## 조금 더 큰 프로그램

`src/main.rok`을 다음과 같이 바꾸고 `rok run`을 다시 실행합니다.

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

`f"..."`는 f-string입니다. `{}` 안의 식을 평가해 그 자리에 넣습니다. `greet`는 호출하는 곳보다 아래에 선언되어 있지만 호출할 수 있습니다. 함수 같은 선언은 순서와 관계없이 보이기 때문입니다.

최상위 문 대신 `fn main()`을 진입점으로 쓸 수도 있습니다. 둘 중 하나만 써야 합니다.

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

## 단일 파일 스크립트

간단히 실험할 때는 패키지가 필요 없습니다. `rok run`에 파일 경로를 주면 그 파일 하나만 실행합니다.

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

스크립트에서도 의존성을 쓸 수 있습니다. `rok add json hi.rok`을 실행하면 파일 맨 위의 `//!` 주석에 의존성이 적힙니다([인라인 매니페스트](../spec/refs-lang/009-tools/910-rok-frontend.html#9107-단일-파일-스크립트의-인라인-매니페스트), [11장](11-modules-packages.html#단일-파일-스크립트와-환경) 참고).

## REPL

인자 없이 `rok`을 실행하면 REPL이 열립니다. 식을 입력하면 값을 출력합니다. 식 끝에 `?`를 붙이면 `Debug` 형태로 출력합니다. `Display`가 없는 값은 타입만 출력하고, `let`은 출력 없이 이름만 정의합니다.

```text
$ rok
> let xs = [1, 2, 3];
> xs.len() * 2
6
> xs?
[1, 2, 3]
> :type xs
List<i32>
> :quit
```

`:type expr`는 식을 실행하지 않고 타입만 알려 줍니다. 가이드를 읽다가 타입이 궁금하면 REPL에서 확인해 보세요. 일반 소스와 달리 REPL에서는 같은 이름을 `let`으로 다시 선언할 수 있습니다([JIT 셀](../spec/refs-lang/009-tools/907-jit-cells.html)).

## 정리

- `curl -fsSL https://rok-lang.org/install.sh | sh`(Windows는 `install.ps1`)로 `~/.rok`에 설치하고 `~/.rok/bin`을 `PATH`에 추가합니다.
- `rok new name`은 `rok.rokon`과 `src/main.rok`을 만들고, `rok run`은 빌드한 뒤 실행합니다.
- `src/main.rok`의 최상위 문이 프로그램입니다. `fn main()`을 쓸 수도 있습니다.
- `rok run file.rok`은 스크립트를 실행하고, 인자 없는 `rok`은 REPL을 시작합니다.

## 더 읽을거리

- [설치 레이아웃](../spec/refs-lang/009-tools/902-aot-pipeline.html#9219-설치-레이아웃), [toolchain 고정](../spec/refs-lang/009-tools/910-rok-frontend.html#9106-toolchain)
- [`rok` 프런트엔드](../spec/refs-lang/009-tools/910-rok-frontend.html), [`rokc` CLI](../spec/refs-lang/009-tools/901-rokc-cli.html)
- [진입점](../spec/refs-lang/004-functions/416-entry-point.html)
- [JIT 셀과 REPL](../spec/refs-lang/009-tools/907-jit-cells.html)
- 다음 장: [2. 값과 타입](02-values-and-types.html)
