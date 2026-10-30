# 18. Feature 와 stone

[11장](11-modules-packages.html)에서는 패키지를 만들고 의존성을 추가했습니다. 이 장은 거기서 더 나아갑니다. 라이브러리 사용자가 그 일부를 켜고 끌 수 있게 하는 **feature**, Rok 프로젝트가 관리하는 stone, 그리고 자기 라이브러리를 [rokstones.io](https://rokstones.io)에 publish하는 법입니다.

## feature 정의하기

feature는 `rok.rokon`의 `features` dict에 있는 이름이며, 그 feature가 켜는 것들의 리스트에 대응합니다. 코드에서 켜진 feature는 `feature = "<name>"` 조건입니다([17장](17-conditional-compilation.html)). 여기서 `geom` 라이브러리에는 기본으로 켜진 `circle` feature와, optional 의존을 끌어들이는 `pretty` feature가 있습니다.

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
| `"dep:fancy"` | optional 의존 `fancy`                                             |
| `"http/tls"` | 의존 `http`의 feature `tls`(그리고 `http`가 optional이면 `http` 자체) |
| `"http?/tls"` | `http`의 feature `tls`. 다른 무언가가 이미 `http`를 켰을 때만   |

- `default`는 사용자가 끄지 않는 한 켜져 있는 보통의 feature입니다.
- optional 의존(`optional: true`)은 켜진 feature가 `dep:`이나 `<dep>/<f>`로 가리킬 때만 빌드에 들어갑니다. 자기 이름의 feature가 저절로 생기지는 않으며, 어떤 feature도 켜지 않는 optional 의존은 에러입니다. `rok add fancy --path ../fancy --optional`은 의존과 `"fancy": ["dep:fancy"]` feature를 함께 씁니다.
- 없는 feature나 의존을 가리키는 것, optional이 아닌 의존에 `dep:`을 쓰는 것, 순환은 에러이며, `rok.rokon`의 줄과 열로 보고됩니다.

feature 이름은 무엇을 켜는지 말해 주는 소문자 단어(`http1`, `tls`, `pretty`)로 하며, `with-x`나 `no-x`는 쓰지 않습니다. feature는 언제나 더하기만 합니다.

## feature 켜기

빌드하는 패키지는 자기 `default` feature에서 시작합니다. `--features`는 자기 것(`fancy`)이나 직접 의존의 것(`geom/pretty`)을 더합니다. `--no-default-features`와 `--all-features`는 이름 그대로 동작합니다. `run`, `build`, `test`, `check`와 그 밖의 빌드 명령에서 쓸 수 있습니다.

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

마지막 것도 `circle`를 출력합니다. `--no-default-features`는 `geom`이 아니라 빌드하는 패키지(`app`)의 기본값을 끄기 때문입니다. 의존의 feature는 그 의존을 적은 자리에서 고릅니다.

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

feature는 **합쳐집니다**(unified). 각 패키지는 빌드 전체에서 feature 집합을 하나만 가지며, 이 집합은 모든 의존자가 요구한 것의 합집합입니다. 그래서 두 라이브러리가 모두 `geom`을 쓰는데 하나는 `pretty`를 켜고 하나는 켜지 않으면, 둘 다 `pretty`를 받습니다. feature가 API를 더하기만 하고 기존 코드의 동작을 결코 바꾸지 않아야 하는 이유가 이것입니다. `rok deps`는 결과를 `--extern-feature` 줄로 보여 줍니다.

lockfile는 optional인 것까지 그래프의 모든 패키지를 고정하므로, feature를 바꿔도 `rok.lock`은 바뀌지 않습니다.

## Rok 프로젝트의 stone

Rok 프로젝트는 몇몇 라이브러리를 표준 라이브러리 밖의 독립된 stone으로 관리합니다. 모두 순수 Rok이며 `rok::std` 밖의 의존이 없습니다.

| Stone      | 무엇인가                                                                      |
|------------|-------------------------------------------------------------------------------|
| `json`     | JSON (RFC 8259): `Value` 트리, `line:col` 에러를 내는 엄격한 parser, 스트리밍 tokenizer, `#[derive(Json)]`으로 하는 타입 있는 encode / decode, JSON Pointer |
| `toml`     | TOML 1.1.0: 타입 있는 날짜-시각을 갖는 `Value` 트리, `line:col` 에러를 내는 parser, writer, `#[derive(Toml)]`로 하는 타입 있는 encode / decode |
| `http`     | HTTP/1.1과 HTTP/2 클라이언트와 서버: 엄격한 파싱, HPACK, 스트리밍 body, 연결 풀링, std의 TLS 위의 `https`. feature는 `http1`, `http2`, `tls`이며 모두 기본으로 켜짐 |
| `markdown` | CommonMark와 GitHub 확장(표, callout, 각주, 제목 anchor)을 HTML로. 이 사이트를 렌더링함 |
| `rok-syntax` | Rok과 ROKON 소스의 tokenizer와 HTML 강조기. 이 사이트의 코드를 강조함 |

`json`과 `toml`은 std의 [`rokon`](15-rokon.html) 모듈과 같은 모양을 따릅니다. `parse` / `to_string` / `from_str` / `encode` / `decode`, 그리고 자기 타입을 위한 derive입니다.

```rok norun
#[derive(Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_string(s));      // {"host":"localhost","port":80}
```

다른 stone과 똑같이 추가합니다. 요구를 적지 않으면 `rok add`는 `^`과 최신 버전을 씁니다.

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

스크립트에서는 파일 이름을 마지막에 적습니다. `rok add json app.rok`은 의존을 스크립트의 `//!` 블록에 씁니다([11장](11-modules-packages.html#단일-파일-스크립트와-환경)).

`http`를 쓰면 HTTPS 요청은 몇 줄이면 됩니다. stone이 TCP 연결을 열고, OS trust store를 상대로 [`rok::std::net::tls`](19-crypto-and-tls.html)의 TLS 1.3 핸드셰이크를 실행하고, ALPN으로 HTTP/2를 제안하며, 안 되면 HTTP/1.1로 물러섭니다.

```rok norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## stone publish 하기

publish는 레지스트리 index에 git 저장소의 커밋 하나를 가리키는 줄 하나를 더하는 일입니다. 소스 archive를 올리지는 않습니다. 패키지는 라이브러리(`src/lib.rok`)여야 하고, remote `origin`이 있는 깨끗한 git work tree의 루트에 있어야 하며, 모든 normal 의존은 그 자체가 stone이어야 합니다(path와 git 의존은 거부되고, path와 git의 `dev_deps`는 빠집니다). `description`, `license`, `repository`를 채우세요. 이 값들이 `rok search`와 웹사이트에 나타납니다.

> [!IMPORTANT]
> 지금은 rokstones.io maintainer만 publish할 수 있습니다. 누구나 로그인해 stone을 쓸 수 있지만, `rok login`은 아직 publish할 수 없는 계정을 거절합니다.

### `rok login`

[rokstones.io/me](https://rokstones.io/me)에서 GitHub로 로그인하고 거기서 계정 토큰을 만듭니다. 그런 다음 `rok`에 넘겨줍니다.

```text
$ rok login
a token from https://rokstones.io/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://rokstones.io (token in ~/.rok/credentials.rokon)
```

인자가 없으면 토큰은 화면에 보이지 않게 표준 입력에서 읽습니다. `rok login`은 레지스트리에 토큰이 누구의 것인지 물으며, 레지스트리가 거부하는 토큰은 아무것도 저장하지 않고 거절합니다. 올바른 토큰은 `~/.rok/credentials.rokon`(본인만 읽을 수 있음)에 저장되며, `rok logout`은 이 파일을 지웁니다. CI에서는 대신 `ROK_REGISTRY_TOKEN`을 설정합니다.

### `rok publish`

`rok publish`만 쓰면 dry run입니다. 더할 index 항목을 출력하며, 아무것도 기계 밖으로 나가지 않습니다.

```text
$ rok publish
Stone { name: "textwrap", vers: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), cksum: "823a3592..", yanked: false, description: "Word wrapping for terminals", license: "MIT OR Apache-2.0", repository: "https://github.com/you/textwrap" }
note: dry run; --commit adds it to https://github.com/rok-lang/rokstones.io-index as te/xt/textwrap
```

항목에는 HEAD 커밋과 그 tree hash가 기록되므로, 그 커밋을 먼저 push하세요. `rok publish --commit`은 토큰과 함께 항목을 레지스트리에 보내며(`Published textwrap 0.1.0 to https://rokstones.io`), index에 들어가면 그 버전이 공개됩니다. 한 버전은 두 번 publish할 수 없으며, 기존 stone과 대소문자나 `-` / `_`만 다른 이름은 이미 쓰인 것으로 봅니다.

### `rok yank`

심각한 버그가 있는 버전은 yank할 수 있습니다. 새로운 해소는 그 버전을 건너뛰지만, `rok.lock`이 이미 그 버전을 고정한 프로젝트는 계속 빌드되므로, yank가 누군가를 깨뜨리는 일은 없습니다. `--undo`로 되돌립니다.

```bash
rok yank --vers 0.1.0          # in the package directory
rok yank --vers 0.1.0 --undo
```

## 요약

- `rok.rokon`의 `features`는 이름을 항목에 대응시킵니다. 항목은 다른 feature, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`입니다. `default`는 끄지 않는 한 켜져 있습니다.
- optional 의존(`optional: true`)은 feature가 켤 때만 빌드됩니다.
- feature는 `--features`, `--no-default-features`, `--all-features`로, 의존마다는 `features:` / `default_features: false`로 고릅니다. feature는 빌드 전체에서 합쳐지므로 더하기만 해야 합니다.
- 코드는 `#[cfg(feature = "..")]`와 `cfg!(feature = "..")`로 feature를 검사합니다.
- `json`, `toml`, `http`, `markdown`, `rok-syntax`가 Rok 프로젝트의 stone입니다.
- publish는 `rok login`(rokstones.io 토큰), `rok publish`(dry run), `rok publish --commit`으로 하고, 철회는 `rok yank`로 합니다. 지금은 maintainer만 publish할 수 있습니다.

## 더 읽을거리

- 사양: [9.10.4c Features](../spec/refs-lang/009-tools/910-rok-frontend.html#9104c-features), [9.10.4a stone과 rokstones.io](../spec/refs-lang/009-tools/910-rok-frontend.html#9104a-stone-과-rokstonesio), [`rok search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-rok-frontend.html#rok-search--publish--yank--login--logout), [`rok.rokon`](../spec/refs-lang/009-tools/910-rok-frontend.html#9103-rokrokon--roklock)
- [rokstones.io](https://rokstones.io)
- 다음 장: [19. 암호와 TLS](19-crypto-and-tls.html)
