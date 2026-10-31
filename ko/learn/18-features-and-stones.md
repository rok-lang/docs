# 18. Feature와 stone

[11장](11-modules-packages.html)에서는 패키지를 만들고 의존성을 추가했습니다. 이 장은 거기서 더 나아갑니다. 라이브러리 사용자가 그 일부를 켜고 끌 수 있게 하는 **feature**, Rok 프로젝트가 관리하는 stone, 그리고 자기 라이브러리를 [rokstones.io](https://rokstones.io)에 공개하는 법을 다룹니다.

## feature 정의하기

feature는 `rok.rokon`의 `features` 딕셔너리에 있는 이름입니다. 이름마다 그 feature가 켜는 것의 리스트가 붙습니다. 코드에서 켜진 feature는 `feature = "<name>"` 조건입니다([17장](17-conditional-compilation.html)). 여기서 `geom` 라이브러리에는 기본으로 켜진 `circle` feature와 선택 의존성을 끌어들이는 `pretty` feature가 있습니다.

```rokon
Package {
    name: "geom",
    version: "0.1.0",
    deps: {
        "fancy": Dep::path("../fancy", optional: true),
    },
    features: {
        "default": ["circle"],
        "circle": [],
        "pretty": ["dep:fancy"],
    },
}
```

```rok norun
// geom/src/lib.rok
pub fn shapes() -> List<str> {
    let out: List<str> = ["square"];
    #[cfg(feature = "circle")]
    out.push("circle");
    out
}

#[cfg(feature = "pretty")]
pub fn show(s: str) -> str { fancy::frame(s) }

#[cfg(not(feature = "pretty"))]
pub fn show(s: str) -> str { s }
```

feature 리스트의 각 항목은 다음 중 하나입니다.

| 항목        | 켜는 것                                                             |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | 이 패키지의 다른 feature                                            |
| `"dep:fancy"` | 선택 의존성 `fancy`                                               |
| `"http/tls"` | 의존성 `http`의 feature `tls`(`http`가 선택 의존성이면 `http` 자체도) |
| `"http?/tls"` | `http`의 feature `tls`. 다른 무언가가 이미 `http`를 켰을 때만   |

- `default`는 사용자가 끄지 않는 한 켜져 있는 보통의 feature입니다.
- 선택 의존성(`optional: true`)은 켜진 feature가 `dep:`이나 `<dep>/<f>`로 가리킬 때만 빌드에 들어갑니다. 자기 이름의 feature가 저절로 생기지는 않습니다. 어떤 feature도 켜지 않는 선택 의존성은 에러입니다. `rok add fancy --path ../fancy --optional`은 의존성과 `"fancy": ["dep:fancy"]` feature를 함께 씁니다.
- 없는 feature나 의존성을 가리키는 것, 선택 의존성이 아닌 의존성에 `dep:`을 쓰는 것, 순환은 에러입니다. 이 에러는 `rok.rokon`의 줄과 열로 보고됩니다.

feature 이름은 무엇을 켜는지 말해 주는 소문자 단어(`http1`, `tls`, `pretty`)로 짓고, `with-x`나 `no-x`는 쓰지 않습니다. feature는 언제나 더하기만 하기 때문입니다.

## feature 켜기

빌드하는 패키지는 자기 `default` feature에서 시작합니다. `--features`는 자기 feature(`fancy`)나 직접 의존성의 feature(`geom/pretty`)를 더합니다. `--no-default-features`와 `--all-features`는 이름 그대로 동작합니다. `run`, `build`, `test`, `check`와 그 밖의 빌드 명령에서 쓸 수 있습니다.

```text
$ rok run
square
circle
$ rok run --features geom/pretty
*** square ***
*** circle ***
$ rok run --no-default-features
square
circle
```

마지막 명령도 `circle`을 출력합니다. `--no-default-features`는 `geom`이 아니라 빌드하는 패키지(`app`)의 기본값을 끄기 때문입니다. 의존성의 feature는 그 의존성을 적은 자리에서 고릅니다.

```rokon
Package {
    name: "app",
    version: "0.1.0",
    deps: {
        "geom": Dep::path("../geom", default_features: false, features: ["pretty"]),
    },
    features: {
        "fancy": ["geom/pretty"],
    },
}
```

feature는 **합쳐집니다**(unified). 빌드 전체에서 패키지마다 feature 집합은 하나뿐이고, 이 집합은 그 패키지에 의존하는 모든 패키지가 요구한 것의 합집합입니다. 그래서 두 라이브러리가 모두 `geom`을 쓰는데 하나는 `pretty`를 켜고 하나는 켜지 않으면 둘 다 `pretty`를 받습니다. feature가 API를 더하기만 하고 기존 코드의 동작을 바꾸면 안 되는 이유가 이것입니다. `rok deps`는 결과를 `--extern-feature` 줄로 보여 줍니다.

lockfile은 선택 의존성까지 그래프의 모든 패키지를 고정하므로 feature를 바꿔도 `rok.lock`은 바뀌지 않습니다.

## Rok 프로젝트의 stone

Rok 프로젝트는 몇몇 라이브러리를 표준 라이브러리 밖의 독립된 stone으로 관리합니다. 모두 순수 Rok이고 `rok::std` 밖의 의존성이 없습니다.

| Stone      | 무엇인가                                                                      |
|------------|-------------------------------------------------------------------------------|
| `json`     | JSON (RFC 8259): `Value` 트리, `line:col` 에러를 내는 엄격한 파서, 스트리밍 토크나이저, `#[derive(json::Json)]`으로 하는 타입 있는 encode / decode, JSON Pointer |
| `toml`     | TOML 1.1.0: 타입 있는 날짜-시각이 들어가는 `Value` 트리, `line:col` 에러를 내는 파서, 라이터, `#[derive(toml::Toml)]`로 하는 타입 있는 encode / decode |
| `http`     | HTTP/1.1과 HTTP/2 클라이언트와 서버: 엄격한 파싱, HPACK, 스트리밍 body, 연결 풀링, std의 TLS 위의 `https`. feature는 `http1`, `http2`, `tls`이며 모두 기본으로 켜짐 |
| `markdown` | CommonMark와 GitHub 확장(표, 콜아웃, 각주, 제목 앵커)을 HTML로 변환. 이 사이트를 렌더링함 |
| `rok-syntax` | Rok과 ROKON 소스의 토크나이저와 HTML 강조기. 이 사이트의 코드를 강조함 |

`json`과 `toml`은 std의 [`rokon`](15-rokon.html) 모듈과 같은 모양을 따릅니다. `parse` / `to_string` / `from_str` / `encode` / `decode`, 그리고 자기 타입을 위한 derive입니다.

```rok norun
#[derive(json::Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_string(s));      // {"host":"localhost","port":80}
```

다른 stone과 똑같이 추가합니다. 버전 요구 사항을 적지 않으면 `rok add`는 `^`과 최신 버전을 씁니다.

```bash
rok add json
rok add http --no-default-features --features http1,tls
```

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
    "http": Dep::version("^0.1.0", default_features: false, features: ["http1", "tls"]),
} }
```

스크립트에서는 파일 이름을 마지막에 적습니다. `rok add json app.rok`은 의존성을 스크립트의 `//!` 블록에 씁니다([11장](11-modules-packages.html#단일-파일-스크립트와-환경)).

`http`를 쓰면 HTTPS 요청은 몇 줄이면 됩니다. stone이 TCP 연결을 열고 OS 신뢰 저장소를 기준으로 [`rok::std::net::tls`](19-crypto-and-tls.html)의 TLS 1.3 핸드셰이크를 실행합니다. ALPN으로 HTTP/2를 제안하고, 안 되면 HTTP/1.1로 물러섭니다.

```rok norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## stone 공개하기

공개는 레지스트리 index에 줄 하나를 더하는 일입니다. 그 줄은 git 저장소의 커밋 하나를 가리킵니다. 소스 아카이브를 올리지는 않습니다. 패키지는 라이브러리(`src/lib.rok`)여야 하고, 원격 `origin`이 있는 깨끗한 git 작업 트리의 루트에 있어야 합니다. 일반 의존성은 모두 그 자체가 stone이어야 합니다. 경로 의존성과 git 의존성은 거부되고, 경로와 git의 `dev_deps`는 빠집니다. `description`, `license`, `repository`를 채우세요. 이 값이 `rok search`와 웹사이트에 나타납니다.

> [!IMPORTANT]
> 지금은 rokstones.io 관리자만 stone을 공개할 수 있습니다. 누구나 로그인해 stone을 쓸 수 있지만, `rok login`은 아직 공개 권한이 없는 계정을 거절합니다.

### `rok login`

[rokstones.io/me](https://rokstones.io/me)에서 GitHub로 로그인하고 거기서 계정 토큰을 만듭니다. 그런 다음 `rok`에 넘겨줍니다.

```text
$ rok login
a token from https://rokstones.io/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://rokstones.io (token in ~/.rok/tokens.rokon)
```

인자가 없으면 토큰을 화면에 보이지 않게 표준 입력에서 읽습니다. `rok login`은 레지스트리에 토큰이 누구의 것인지 묻고, 레지스트리가 거부하는 토큰은 아무것도 저장하지 않고 거절합니다. 올바른 토큰은 `~/.rok/tokens.rokon`(본인만 읽을 수 있음)에 그 토큰을 만든 index별로 저장되고, 그 index에만 보내집니다. 다른 index의 토큰은 `rok login --registry <url>`로 저장합니다. 기본 index의 토큰은 rok 0.1.5 이하가 찾는 `~/.rok/credentials.rokon`에도 저장됩니다. `rok logout`은 두 파일에서 토큰을 지우고, 파일의 다른 필드는 그대로 둡니다. `rok logout --all`은 모든 index의 토큰을 지웁니다. CI에서는 대신 `ROK_REGISTRY_TOKEN`을 설정합니다.

### `rok publish`

`rok publish --dry-run`은 시험 실행(dry run)입니다. 더할 index 항목을 출력할 뿐, 아무것도 기계 밖으로 보내지 않습니다. 0.1.6에서는 `--dry-run`도 `--commit`도 없이 `rok publish`만 써도 시험 실행이지만 경고가 나옵니다. 0.2.0부터는 `cargo publish`처럼 `rok publish`만 쓰면 바로 게시하므로, 미리 보려면 `--dry-run`을 적으세요.

```text
$ rok publish --dry-run
Stone { name: "textwrap", vers: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), cksum: "823a3592..", yanked: false, description: "Word wrapping for terminals", license: "MIT OR Apache-2.0", repository: "https://github.com/you/textwrap" }
note: dry run; --commit sends it to the registry API of https://github.com/rok-lang/rokstones.io-index (needs `rok login`), which adds it as te/xt/textwrap
```

항목에는 HEAD 커밋과 그 트리 해시가 기록되므로 그 커밋을 먼저 원격 저장소에 올리세요(push). `rok publish --commit`은 토큰과 함께 항목을 레지스트리에 보냅니다(`Published textwrap 0.1.0 to https://rokstones.io`). index에 항목이 들어가면 그 버전이 공개됩니다. 한 버전은 두 번 공개할 수 없습니다. 기존 stone과 대소문자나 `-` / `_`만 다른 이름은 이미 쓰인 것으로 봅니다.

### `rok yank`

심각한 버그가 있는 버전은 철회(yank)할 수 있습니다. 새로 의존성을 해소할 때는 그 버전을 건너뜁니다. 하지만 `rok.lock`이 이미 그 버전을 고정한 프로젝트는 계속 빌드되므로 철회가 누군가의 빌드를 깨뜨리는 일은 없습니다. `--undo`로 되돌립니다.

```bash
rok yank textwrap@0.1.0
rok yank textwrap@0.1.0 --undo
```

예전 형태인 `rok yank --vers 0.1.0`(패키지 디렉터리에서)도 동작하지만 경고가 나옵니다.

### 공개한 API 바꾸기

stone의 함수, 메서드, 필드, 타입을 없애고 싶다면 바로 지우지 말고 먼저 `#[deprecated]`를 붙이세요. 그것을 쓰는 코드는 계속 빌드되고, 대신 무엇을 쓰면 되는지 알려 주는 경고를 받습니다. 실제로 지우는 것은 다음 호환성이 깨지는 릴리스에서 합니다.

```rok norun
#[deprecated(since = "0.3.0", note = "use `wrap_width`")]
pub fn wrap(s: str) -> str { wrap_width(s, 80) }
```

```text
warning[E503]: use of deprecated function `wrap`
  --> src/main.rok:6:9
   |
 6 | println(textwrap::wrap("hi"));
   |         ^^^^^^^^^^^^^^^^^^^^
   |
   = note: deprecated since 0.3.0
   = note: use `wrap_width`
   = note: `#[warn(deprecated)]` on by default
```

## 요약

- `rok.rokon`의 `features`는 feature 이름마다 켤 항목을 적습니다. 항목은 다른 feature, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`입니다. `default`는 끄지 않는 한 켜져 있습니다.
- 선택 의존성(`optional: true`)은 feature가 켤 때만 빌드됩니다.
- feature는 `--features`, `--no-default-features`, `--all-features`로, 의존성마다는 `features:` / `default_features: false`로 고릅니다. feature는 빌드 전체에서 합쳐지므로 더하기만 해야 합니다.
- 코드는 `#[cfg(feature = "..")]`와 `cfg!(feature = "..")`로 feature를 검사합니다.
- `json`, `toml`, `http`, `markdown`, `rok-syntax`가 Rok 프로젝트의 stone입니다.
- 공개는 `rok login`(rokstones.io 토큰), `rok publish --dry-run`, `rok publish --commit` 순서로 하고, 철회는 `rok yank name@version`으로 합니다. API를 없앨 때는 먼저 `#[deprecated]`를 붙입니다. 지금은 관리자만 공개할 수 있습니다.

## 더 읽을거리

- 사양: [9.10.4c feature](../spec/refs-lang/009-tools/910-rok-frontend.html#9104c-feature), [9.10.4a stone과 rokstones.io](../spec/refs-lang/009-tools/910-rok-frontend.html#9104a-stone과-rokstonesio), [`rok search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-rok-frontend.html#rok-search--publish--yank--login--logout), [`rok.rokon`](../spec/refs-lang/009-tools/910-rok-frontend.html#9103-rokrokon--roklock)
- [rokstones.io](https://rokstones.io)
- 다음 장: [19. 암호와 TLS](19-crypto-and-tls.html)
