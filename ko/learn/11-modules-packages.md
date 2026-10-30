# 11. 모듈과 패키지

프로그램이 커지면 코드를 나누고 다른 사람이 작성한 라이브러리를 써야 합니다. Rok에서 코드를 나누는 단위는 **모듈**(module)이고, 빌드하고 배포하는 단위는 **패키지**(package)입니다. 레지스트리에 공개된 패키지는 **stone**이라고 부릅니다.

## `mod`와 경로

`mod name { .. }`는 그 자리에 모듈을 정의합니다. 모듈 안의 항목은 `module::item` 경로로 가리킵니다. 모듈 안에 다시 모듈을 둘 수 있습니다.

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

경로는 정해진 단어로만 시작할 수 있습니다.

| 시작     | 뜻                                                           |
|----------|--------------------------------------------------------------|
| `pkg::`  | 현재 패키지의 루트부터                                       |
| `sup::`  | 부모 모듈부터(더 위로 가려면 `sup::sup::`)                  |
| `rok::`  | 표준 라이브러리(`rok::core`, `rok::std`)                    |
| 이름     | 현재 모듈의 항목이나 자식 모듈, 또는 `use`로 들여온 이름    |

이름 찾기는 부모 모듈을 자동으로 뒤지지 않습니다. 그래서 `dist` 안에서 `Point`를 쓰려면 `use sup::Point;`로 명시적으로 들여와야 합니다. 덕분에 이름이 어디서 왔는지 언제나 파일 안에서 알 수 있습니다.

## `use`로 들여오기

긴 경로를 매번 쓰는 대신 `use`로 이름을 들여옵니다. `{..}`로 여러 개를 들여오고, `as`로 이름을 바꾸고, `*`로 모듈의 공개 항목을 모두 들여올 수 있습니다.

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

이름을 지정해 들여온 이름은 같은 모듈에서 다시 정의할 수 없습니다. 정의가 아무 말 없이 가져온 이름을 가리기 때문입니다. glob(`use shapes::*;`)으로 들여온 이름은 모듈 자신의 항목에 그냥 자리를 내줍니다.

```rok norun
use shapes::square;
fn square(x: f64) -> f64 { x * x }
```

```text check=checks/11-import-clash.rok
error[E102]: `square` is both imported from `shapes::square` and defined in this module
  --> main.rok:1:5
   |
 1 | use shapes::square;
   |     ^^^^^^ imported here
 2 | fn square(x: f64) -> f64 { x * x }
   | -- defined here, which would hide the import
   |
   = help: import it under another name, `use shapes::square as shapes_square;`, or remove this `use` if the module does not need it
```

지금까지 `use` 없이 써 온 `List`, `Dict`, `println` 같은 이름은 모든 파일에 암묵적으로 들어 있는 `use rok::core::**;` 덕분에 쓸 수 있습니다. 이 줄은 `*`에 하위 모듈까지 더한 재귀 glob입니다. 파일, 네트워크, 스레드 같은 `rok::std`의 모듈은 `use rok::std::fs;`처럼 명시적으로 들여옵니다.

## 가시성

기본적으로 항목은 자기 모듈 안에서만 보입니다. 밖에서 보이게 하려면 가시성 키워드를 붙입니다.

| 키워드     | 보이는 곳                               |
|------------|-----------------------------------------|
| (없음)     | 같은 모듈                               |
| `pub(sub)` | 같은 모듈과 그 하위 모듈                |
| `pub(pkg)` | 같은 패키지 어디서나                    |
| `pub`      | 다른 패키지에서도                       |

struct 필드도 하나씩 공개해야 합니다. 그래서 위의 `Point`는 `pub x: f64`처럼 필드마다 `pub`을 붙였습니다. 비공개 항목을 밖에서 쓰면 컴파일러가 어디에 `pub`을 붙여야 하는지 알려 줍니다.

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

`balance` 필드와 `check` 함수는 `bank` 밖에서 보이지 않으므로 잔액은 `bank`의 함수로만 바꿀 수 있습니다.

## 파일로 나누기

실제 패키지에서는 모듈을 파일로 나눕니다. `mod shapes;`처럼 본체 없이 모듈을 선언하면 같은 디렉터리의 `shapes.rok`이 그 모듈의 본체가 됩니다. 하위 모듈이 있으면 `shapes.rok` 옆에 같은 이름의 `shapes/` 디렉터리를 둡니다. `mod.rok` 같은 특별한 파일 이름은 없습니다.

```text
geom/
├── rok.rokon
└── src/
    ├── lib.rok          // mod shapes;
    ├── shapes.rok       // shapes module body, declares mod circle;
    └── shapes/
        └── circle.rok   // shapes::circle
```

패키지의 소스는 `src/`에 있습니다. 패키지 루트 모듈은 실행 파일이면 `src/main.rok`, 라이브러리면 `src/lib.rok`입니다. 실행되는 최상위 문장은 `src/main.rok`에만 올 수 있고, 다른 모듈에는 항목만 들어갑니다.

## 패키지와 `rok.rokon`

`rok new`는 패키지를 만듭니다. 실행 파일이면 `src/main.rok`을, `--lib`을 주면 예제 함수와 `#[test]` 하나가 든 `src/lib.rok`을 만듭니다. 매니페스트 `rok.rokon`은 [ROKON](../spec/refs-standard-libraries/300-std/309-rokon.html) 형식입니다.

```bash
rok new --lib geom
rok new app
```

```text
     Created library package (geom)
     Created executable package (app)
```

`geom`에 모듈을 두 개 만듭니다. 하위 모듈의 항목을 패키지 루트에서 `pub use`로 다시 내보내면(re-export) 라이브러리 사용자는 내부 구조를 알 필요가 없습니다.

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

`app`에서 `rok add`로 의존성을 추가하면 `rok.rokon`의 `deps`에 들어갑니다. 의존 패키지의 이름이 최상위 경로가 됩니다(`-`는 `_`로 바뀝니다).

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

`PI`는 `pub(pkg)`이므로 `geom` 안에서는 어디서나 쓸 수 있지만 `app`에서는 보이지 않습니다. git 저장소는 `rok add parser --git <url> --tag v1.2`처럼 추가합니다. git과 stone 의존성은 정확한 커밋이 `rok.lock`에 고정됩니다. 그러니 `rok.lock`도 버전 관리에 커밋하세요. 고정된 버전을 앞으로 옮기는 것은 `rok update`뿐입니다.

## stone과 rokstones.io

레지스트리에 있는 패키지를 stone이라고 부릅니다. 기본 레지스트리는 [rokstones.io](https://rokstones.io)입니다. 이름으로 검색하고 버전 요구 사항과 함께 추가합니다. 버전 요구 사항은 Cargo와 같이 동작합니다. `^1.2`는 `>=1.2.0, <2.0.0`을 뜻합니다. 요구 사항을 생략하면 최신 버전이 `^`과 함께 기록됩니다.

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

그냥 문자열은 `Dep::version`의 축약이고, 연산자가 없는 요구 사항은 `^`을 뜻합니다. 그래서 다음 세 줄은 같은 것을 요청합니다. stone의 README에는 짧은 형태인 `"json": "0.1"`이 자주 나옵니다. 이 형태는 `>=0.1.0, <0.2.0`을 뜻합니다.

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "textwrap": "1.2",
    // "textwrap": "^1.2",
    // "textwrap": Dep::version("^1.2"),
} }
```

경로는 버전이 아닙니다. `"../geom"`이 아니라 `Dep::path("../geom")`로 씁니다.

라이브러리에는 사용자가 켜고 끌 수 있는 **feature**를 둘 수 있습니다. `rok login`과 `rok publish`로 내 라이브러리를 stone으로 공개할 수도 있습니다. 둘 다 Rok 프로젝트가 관리하는 stone(`json`, `toml`, `http`, `markdown`, `rok-syntax`)과 함께 [18장](18-features-and-stones.html)에서 다룹니다.

## 단일 파일 스크립트와 환경

패키지까지 만들 필요가 없는 작은 스크립트는 파일 맨 위의 `//!` 주석 안에 있는 ```` ```rokon ```` 블록에 의존성을 둡니다. 스크립트를 마지막 인자로 주고 `rok add`를 실행하면 이 블록을 대신 써 줍니다. 경로는 스크립트 기준의 상대 경로로 기록되고, lockfile은 캐시에 보관됩니다.

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

스크립트에도 파일 모듈을 둘 수 있습니다. 패키지의 `src/`에서와 마찬가지로 `mod helper;`는 옆에 있는 `helper.rok`을 불러옵니다. 모듈 파일에도 자기 `//!` 블록을 둘 수 있고(`rok add <name> helper.rok`), 모든 파일의 블록이 합쳐집니다. 스크립트는 `rok install`로 설치한 라이브러리를 보지 않고, 블록에 적힌 것만 셉니다. 스크립트가 한 파일로 부족해지면 `rok init --script tool.rok`으로 패키지로 바꿉니다. 블록은 `rok.rokon`이 되고, `tool.rok`과 그 모듈 파일은 `src/`로 옮겨집니다.

이름 붙은 환경(environment)은 REPL과 Jupyter에서 씁니다. `rok install`로 환경에 라이브러리를 설치합니다. 환경은 `--env`로 고르거나 `rok env use`로 기본값을 바꿉니다. 따로 활성화하는 단계는 없습니다.

```text
$ rok env create sci
     Created environment `sci` (~/.rok/envs/sci)
$ rok install --env sci geom --path ~/work/geom
$ rok env use sci
$ rok env list
* sci
```

패키지는 환경을 쓰지 않습니다. 패키지 안에서는 자기 `rok.rokon`만이 기준입니다.

## 프로그램 설치

`rok install`은 `cargo install`처럼 프로그램도 설치합니다. `src/main.rok`이 있는 패키지(stone, git, 경로에서)나 스크립트 파일을 `--release`로 빌드해 `~/.rok/bin`에 넣습니다. 같은 stone 버전이나 git 커밋을 다시 설치하려 하면 아무것도 하지 않고, 다시 빌드하는 방법을 알려 줍니다. 새 버전은 이전 버전을 대체합니다. 경로나 스크립트 파일은 언제나 다시 빌드합니다.

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

`rok install --list`는 설치된 것을 보여 주고, `rok uninstall tool`은 설치한 것을 제거합니다.

## 정리

- `mod`로 모듈을 만들고, `pkg::` / `sup::` / `rok::` 경로와 `use`로 이름을 가리킵니다.
- 항목은 기본적으로 비공개입니다. `pub(sub)`, `pub(pkg)`, `pub`으로 범위를 넓힙니다.
- `mod x;`는 `x.rok`(과 `x/`)에 대응합니다. 루트는 `main.rok` 또는 `lib.rok`입니다.
- `rok new [--lib]`으로 패키지를 만들고 `rok add`로 의존성(경로, git, stone)을 추가합니다. `rok.lock`은 커밋합니다.
- rokstones.io의 stone은 `rok search` / `rok add name@^1.2`로 다룹니다. feature와 공개는 [18장](18-features-and-stones.html)에 있습니다.
- 스크립트는 `//! ```rokon` 블록을 쓰고(`rok add <name> <file.rok>`), `rok init --script`로 패키지로 바꿀 수 있습니다. `rok env`는 REPL / Jupyter용입니다.

## 더 읽을거리

- 사양: [모듈](../spec/refs-lang/005-modules/504-modules.html), [`use`](../spec/refs-lang/005-modules/505-use.html), [가시성](../spec/refs-lang/005-modules/506-visibility.html), [경로](../spec/refs-lang/005-modules/507-paths.html), [re-export](../spec/refs-lang/005-modules/508-re-export.html)
- 도구: [`rok` 프런트엔드](../spec/refs-lang/009-tools/910-rok-frontend.html), [`rok.rokon` / `rok.lock`](../spec/refs-lang/009-tools/910-rok-frontend.html#9103-rokrokon--roklock), [stone과 rokstones.io](../spec/refs-lang/009-tools/910-rok-frontend.html#9104a-stone과-rokstonesio), [환경](../spec/refs-lang/009-tools/910-rok-frontend.html#9105-환경), [인라인 매니페스트](../spec/refs-lang/009-tools/910-rok-frontend.html#9107-단일-파일-스크립트의-인라인-매니페스트)
- [1. 시작하기](01-getting-started.html)
- 다음 장: [12. 동시성](12-concurrency.html)
