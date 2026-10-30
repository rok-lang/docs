# 17. 조건부 컴파일

소스 트리 하나를 여러 모양으로 빌드해야 할 때가 많습니다. 운영체제마다, 디버그와 릴리스로, 선택 기능이 있을 때와 없을 때로 말입니다. Rok은 이것을 `#[cfg(..)]` 로 컴파일 타임에 정합니다. 조건이 거짓인 코드는 파싱 직후 처음부터 없었던 것처럼 제거되므로, 실행 시간에 비용이 없고 그 빌드에 없는 이름을 가리켜도 됩니다.

## 조건

조건은 플래그, `key = "value"` 쌍, 또는 이것들을 `all(..)`, `any(..)`, `not(..)` 로 묶은 것입니다.

| 조건                            | 참일 때                                                |
|---------------------------------|--------------------------------------------------------|
| `target_os = "linux"`           | Linux 용으로 빌드할 때 (`"macos"`, `"windows"`, `"wasi"`, ..) |
| `target_arch = "aarch64"`       | 그 CPU 용으로 빌드할 때 (`"x86_64"`, ..)               |
| `target_family = "unix"`        | Unix 계열 target (`"windows"`, `"wasm"`)               |
| `target_pointer_width = "64"`   | 64 비트 target                                         |
| `feature = "tls"`               | 패키지의 feature `tls` 가 켜져 있을 때 ([18장](18-features-and-stones.html)) |
| `debug_assertions`              | `-O0` 으로 빌드할 때 (`rok run` 의 기본값)             |
| `test`                          | `rok test` 용으로 빌드할 때                            |

`target_*` 는 `--target` 을 따르므로, 교차 빌드에서는 호스트가 아니라 target 의 값을 봅니다.

## `#[cfg]` 가 붙는 자리

`#[cfg]` 는 항목 (함수, 타입, `mod`, `use`, `const`, `static`, `impl` 블록과 그 멤버) 에 붙고, 코드 안의 더 작은 조각에도 붙습니다. 문장, `match` 갈래, struct 필드, struct 리터럴의 필드, 함수 매개변수입니다.

```rok
enum Op { Add, Trace, Quit }

struct Build {
    name: str;
    #[cfg(debug_assertions)]
    checks: bool;                         // a field only debug builds have
}

fn describe(op: Op) -> str {
    match op {
        Op::Add => "add",
        #[cfg(debug_assertions)]
        Op::Trace => "trace (debug build)", // an arm
        _ => "unsupported",
    }
}

let b = Build {
    name: "demo",
    #[cfg(debug_assertions)]
    checks: true,                         // a struct literal field
};

#[cfg(debug_assertions)]
println(f"debug build of {b.name}");      // a statement
#[cfg(not(debug_assertions))]
println(f"release build of {b.name}");

println(describe(Op::Trace));
```

```output
debug build of demo
trace (debug build)
```

같은 파일을 `-O2` 로 빌드하면 `release build of demo` 와 `unsupported` 를 출력합니다. `checks` 필드, 그 초기화, `Trace` 갈래가 모두 사라지기 때문입니다.

조건이 거짓인 매개변수는 시그니처에서 제거되며, 호출은 남은 인자만 넘깁니다. 그래서 선택 기능이 함수를 두 벌 만들지 않고도 매개변수를 더할 수 있습니다.

```rok
fn connect(host: str, #[cfg(feature = "tls")] verify: bool) -> str {
    #[cfg(feature = "tls")]
    if verify { return f"{host} (verified)"; }
    host
}

// Without the `tls` feature, `connect` takes one argument.
println(connect("example.com"));
```

```output
example.com
```

클로저 매개변수에는 속성을 붙일 수 없습니다.

## 정의 둘, 빌드 하나

함수의 본문을 플랫폼마다 다르게 하려면, 서로 배제하는 조건으로 두 번 씁니다. 어느 빌드에서든 하나만 남습니다. 둘 다 남으면 보통의 중복 정의 에러입니다.

```rok
#[cfg(target_family = "windows")]
fn path_sep() -> str { "\\" }

#[cfg(not(target_family = "windows"))]
fn path_sep() -> str { "/" }

println(["usr", "local", "bin"].join(path_sep()));
```

Linux 와 macOS 에서는 `usr/local/bin` 을 출력합니다.

모듈 전체도 조건부로 만들 수 있습니다. `#[cfg(target_os = "linux")] mod linux;` 는 다른 빌드에서 `linux.rok` 를 읽지조차 않습니다.

## `cfg!`: 컴파일 타임 `bool`

양쪽 갈래가 어디서나 컴파일되고 선택만 다르다면 `cfg!(..)` 가 더 짧습니다. 상수 `bool` 이므로 컴파일러가 죽은 갈래를 버립니다. 다만 `#[cfg]` 와 달리 양쪽 모두 타입 검사는 통과해야 합니다.

```rok
const DEBUG: bool = cfg!(debug_assertions);

let level = if cfg!(feature = "verbose") { 2 } else { 1 };
println(f"debug={DEBUG} level={level}");
```

```output
debug=true level=1
```

`#[cfg_attr(cond, attr)]` 는 조건이 참일 때만 다른 속성을 붙입니다. 예를 들어 `#[cfg_attr(test, derive(Debug))]` 입니다.

## 요약

- `#[cfg(..)]` 는 컴파일 타임에 코드를 남기거나 제거합니다. 조건은 `key = "value"`, 플래그 (`test`, `debug_assertions`), 그리고 `all` / `any` / `not` 입니다.
- 항목, 필드, 변형, 문장, match 갈래, struct 리터럴 필드, 매개변수에 붙습니다. 제거된 매개변수는 시그니처에서 사라집니다.
- 플랫폼마다 다른 본문은 서로 배제하는 정의로 씁니다.
- `cfg!(..)` 는 보통의 `if` 에 쓰는 상수 `bool` 입니다.
- `feature = ".."` 조건은 패키지의 feature 가 정하며, 다음 장에서 다룹니다.

## 더 읽을거리

- 사양: [6.3 `#[cfg]`](../spec/refs-lang/006-meta-system/603-cfg.html), [6.3.3 부착 위치](../spec/refs-lang/006-meta-system/603-cfg.html#633-부착-위치), [6.3.5 `cfg!`](../spec/refs-lang/006-meta-system/603-cfg.html#635-cfg-컴파일-타임-bool-값), [플랫폼 target](../spec/refs-lang/000-meta/006-platform-targets.html)
- 다음 장: [18. Feature 와 stone](18-features-and-stones.html)
