# 9. 문자열과 f-string

문자열은 거의 모든 프로그램에 등장합니다. 이 장에서는 `str`과 `StrBuf`, 자주 쓰는 메서드, 여러 종류의 리터럴, 그리고 지금까지 써 온 f-string의 형식 지정자를 다룹니다.

## `str`: 불변 UTF-8

`str`은 UTF-8 바이트의 나열이고, 한 번 만들면 바뀌지 않습니다. 그래서 `len()`은 문자 수가 아니라 **바이트** 수를 반환합니다. 문자(코드 포인트) 수를 세려면 `char_count()`를 씁니다.

```rok
let s = "hello, 세계";
println(f"{s.len()} bytes, {s.char_count()} chars");
println(s[0..5]);

for c in "가나".chars() {
    println(f"{c} = U+{c as u32:X}");
}
```

```output
13 bytes, 9 chars
hello
가 = U+AC00
나 = U+B098
```

여기서는 일부러 한국어 텍스트를 넣었습니다. 한글 음절은 UTF-8에서 한 글자에 3바이트를 차지하므로 `"hello, 세계"`는 13바이트이지만 문자는 9개뿐입니다.

슬라이스 `s[a..b]`도 바이트 위치에서 자릅니다. 한글 한 글자는 3바이트이므로 문자 경계가 아닌 곳에서 자르면 안 됩니다. 문자 중간을 자르면 panic하고, 문제가 된 바이트를 알려 줍니다.

```rok build
let s = "hello, 세계";
println(s[0..8]);
```

```text
thread 'main' panicked at main.rok:2:9:
string slice is not on a char boundary: byte 8 is inside `세` (bytes 7..10)
string offsets count bytes; to work by character, use `chars()` or `char_indices()`
```

문자 단위로 다루려면 `chars()`나 `char_indices()`를 씁니다.

문자 하나는 작은따옴표로 쓰는 `char`입니다. 유니코드 스칼라 값 하나를 담고, 문자를 분류하고 변환하는 메서드가 있습니다.

```rok
let c = 'A';
println(f"{c} {c.is_ascii_uppercase()} {c.to_ascii_lowercase()}");
println(f"{'7'.is_ascii_digit()} {'세'.len_utf8()} {'\u{1F600}'}");
```

```output
A true a
true 3 😀
```

## 자주 쓰는 메서드

`str`의 메서드는 원본을 바꾸지 않고 새 값을 반환합니다. 아무것도 찾지 못할 수 있는 연산은 `T | none`을 반환하므로 5장의 `??`, `if let`과 함께 쓰기 좋습니다.

```rok
let line = "  name = rok  ";
let t = line.trim();
println(f"[{t}]");
println(f"{t.starts_with("name")} {t.contains("=")} {t.find("rok") ?? 99}");

if let (k, v): (str, str) = t.split_once(" = ") {
    println(f"key={k} value={v}");
}

let parts: List<str> = "a,b,,c".split(",").collect();
println(f"{parts:?}");
println(f"{"ab".repeat(3)} {t.to_uppercase()} {t.replace("rok", "ROK")}");

let n: i32 = try! "42".parse();
println(f"{n + 1}");
```

```output
[name = rok]
true true 7
key=name value=rok
["a", "b", "", "c"]
ababab NAME = ROK name = ROK
43
```

`parse`는 실패하면 에러를 던지는 함수이므로 예제에서는 `try!`를 붙였습니다. 에러는 [다음 장](10-errors.html)에서 다룹니다. 그 밖에 `lines()`, `split_whitespace()`, `strip_prefix()`, `trim_start()` 같은 메서드가 있습니다([`str` API](../../std/core/str.html#impl.str)).

## `StrBuf`: 조금씩 만들기

`str`은 불변이므로 루프 안에서 `+`로 이어 붙이면 매번 새 문자열이 생깁니다. 문자열을 조금씩 만들 때는 `StrBuf` 버퍼에 쓰고, 다 쓰면 `to_str()`로 결과를 꺼냅니다.

```rok
let b = StrBuf::new();
for i in 1..=5 {
    if !b.is_empty() { b.push_str(", "); }
    b.push_str(f"{i * i}");
}
b.push('!');
let s = b.to_str();
println(f"{s} ({s.len()} bytes)");
```

```output
1, 4, 9, 16, 25! (16 bytes)
```

두 문자열을 한 번 잇는 데는 `+`로 충분합니다. 다만 C나 Python과 달리 나란히 쓴 문자열 리터럴(`"abc" "def"`)은 자동으로 이어지지 않고, 컴파일 에러가 납니다. 이으려면 `+`를 씁니다([1.7.4](../spec/refs-lang/001-lexical/107-string-literals.html#174-인접-문자열의-자동-결합)).

```rok
let greeting = "hello, " + "world";
println(greeting);
```

```output
hello, world
```

## 리터럴: 이스케이프, 원시 문자열, 줄바꿈 잇기

일반 문자열은 `\n`, `\t`, `\"`, `\\`, `\u{..}` 같은 이스케이프를 처리합니다. 백슬래시가 많은 경로나 정규식에는 `r"..."` 형태의 원시 문자열(raw string)이 더 편합니다. 안에 `"`가 필요하면 양쪽을 같은 개수의 `#`으로 감쌉니다.

```rok
println("tab:\t| quote:\" | snow: \u{2603}");
println(r"C:\path\to\file\n");
println(r#"she said "hi""#);
println("one long \
         line");
```

```output
tab:	| quote:" | snow: ☃
C:\path\to\file\n
she said "hi"
one long line
```

마지막 예처럼 줄 끝의 `\`는 줄바꿈과 다음 줄의 앞 공백을 함께 없앱니다. 이 규칙도 원시 문자열에는 적용되지 않습니다([1.7.2](../spec/refs-lang/001-lexical/107-string-literals.html#172-원시-문자열)).

## 바이트 문자열 `b"..."`

프로토콜과 파일 형식은 텍스트가 아니라 바이트로 정의됩니다. 바이트 문자열 `b"..."`는 그 바이트들로 이루어진 고정 배열 `[u8; N]`입니다. ASCII는 그대로 쓰고, 나머지는 `\xHH`(`\x00`-`\xFF`의 어떤 값이든)로 씁니다. `Slice<u8>`이 기대되는 자리에서는 같은 리터럴이 그 배열의 뷰가 되므로 바이트를 받는 함수에 바로 넘길 수 있습니다.

```rok
let magic = b"\x7FELF";                  // [u8; 4]
println(f"{magic.len()} {magic:?}");

fn starts_with(data: Slice<u8>, prefix: Slice<u8>) -> bool {
    data.len() >= prefix.len() && data[:prefix.len()] == prefix
}
println(f"{starts_with(b"\x7FELF\x02\x01", b"\x7FELF")}");

let req: Slice<u8> = b"GET / HTTP/1.1\r\n";
println(f"{req[:3]:?} {try! str::from_utf8(req[:3])}");
```

```output
4 [127, 69, 76, 70]
true
[71, 69, 84] GET
```

바이트 문자열 안에 ASCII가 아닌 문자를 직접 쓰거나 `\xFF`보다 큰 이스케이프를 쓰면 컴파일 에러입니다. 줄바꿈 잇기는 지원하지 않습니다.

## 파일 포함: `include_str!` / `include_bytes!`

템플릿, 테스트 픽스처, 아이콘을 실행 파일 안에 넣어 배포하려면 컴파일 시점에 파일을 포함합니다. `include_str!("path")`는 파일 내용을 `str`로, `include_bytes!("path")`는 바이트 문자열처럼 `[u8; N]`으로 돌려줍니다. 경로는 호출을 쓴 소스 파일을 기준으로 한 상대 경로입니다. 결과는 상수이므로 `const`나 `static`을 초기화하는 데 쓸 수 있습니다.

```text
app/
└── src/
    ├── main.rok
    └── page.html        // <h1>{title}</h1>
```

```rok norun
// app/src/main.rok
const PAGE: str = include_str!("page.html");
static LOGO: [u8; 1406] = include_bytes!("logo.png");

print(PAGE.replace("{title}", "Hello"));
```

파일이 없으면 컴파일 에러입니다. `include_str!`는 파일이 유효한 UTF-8이 아니어도 컴파일 에러입니다. 포함한 파일을 고치면 그 파일을 쓰는 코드가 다시 빌드됩니다.

## f-string 의 구멍

`f"..."`의 `{ }` 안에는 어떤 식이든 들어갈 수 있습니다. 규칙은 Python 3.12(PEP 701)와 같습니다. 구멍 안에서 바깥과 같은 따옴표를 다시 쓸 수 있고, f-string을 중첩할 수도 있습니다. 중괄호 자체를 쓰려면 `{{`와 `}}`를 씁니다.

```rok
let d = {"lang": "rok"};
let words = ["a", "b"];
println(f"{d["lang"]} has {words.len()} words: {words[0] + words[1]}");
println(f"nested: {f"[{1 + 1}]"}");
println(f"{{literal braces}}");
```

```output
rok has 2 words: ab
nested: [2]
{literal braces}
```

구멍 안에서는 `\"` 같은 이스케이프를 쓰지 않고 따옴표를 그대로 씁니다. 여러 줄에 걸친 텍스트나 구멍에는 `f"""..."""`를 씁니다.

## 형식 지정자

구멍의 식 뒤에 `:`을 붙여 형식을 지정합니다. 문법은 `[fill][align][sign][#][0][width][.precision][type]`입니다.

```rok
let pi = 3.14159;
println(f"[{pi:.2}] [{pi:8.3}] [{pi:e}]");
println(f"[{42:>6}] [{42:<6}] [{42:^6}] [{42:*^6}]");
println(f"[{"ab":5}] [{"ab":>5}] [{7:03}] [{5:+}]");
println(f"{255:x} {255:#X} {5:b} {-1i8:x}");
println(f"{"hi":?} {[1, 2]:?}");
```

```output
[3.14] [   3.142] [3.14159e0]
[    42] [42    ] [  42  ] [**42**]
[ab   ] [   ab] [007] [+5]
ff 0xFF 101 ff
"hi" [1, 2]
```

정렬을 지정하지 않으면 숫자는 오른쪽, 그 밖의 값은 왼쪽으로 정렬됩니다. `?`는 `Debug` 형식을 고르고, 문자열을 따옴표와 이스케이프까지 포함해 보여 줍니다. 지정자는 컴파일 시점에 구멍의 타입과 맞는지 검사하므로 `{s:x}`(`s: str`) 같은 실수는 바로 에러가 됩니다.

너비나 정밀도를 실행 시점에 정하려면 지정자 안에 구멍을 하나 더 넣습니다. `{..}`와 `${..}` 모두 쓸 수 있습니다([동적 형식 지정자](../spec/refs-lang/001-lexical/107-string-literals.html#동적-형식-지정자)).

```rok
let x = 2.0 / 3.0;
for prec in 1..=3 {
    let width = prec + 4;
    println(f"[{x:>${width}.${prec}f}] [{x:.{prec}f}]");
}
```

```output
[  0.7] [0.7]
[  0.67] [0.67]
[  0.667] [0.667]
```

## 내 타입 출력하기: `Display` 와 `Debug`

`f"{x}"`는 `Display` 트레잇을, `f"{x:?}"`는 `Debug`를 호출합니다. `Debug`는 `#[derive(Debug)]`로 자동 구현할 수 있고, 사용자에게 보여 줄 `Display`는 직접 구현합니다. `fmt`는 받은 `Formatter`에 텍스트를 씁니다.

```rok
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Display for Point {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(f"({self.x}, {self.y})");
    }
}

#[derive(Debug)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let p = Point { x: 1, y: -2 };
println(f"{p} | {p:?} | [{p:>10}]");
println(f"{[p, Point { x: 3, y: 4 }]:?}");
println(f"{Shape::Circle(1.5):?} {Shape::Rect { w: 2.5, h: 3.0 }:?}");
```

```output
(1, -2) | Point { x: 1, y: -2 } | [   (1, -2)]
[Point { x: 1, y: -2 }, Point { x: 3, y: 4 }]
Circle(1.5) Rect { w: 2.5, h: 3 }
```

너비와 정렬(`{p:>10}`)은 `fmt`가 쓴 출력 전체에 적용됩니다. `Display`를 구현하면 그 타입을 f-string 어디에든 바로 넣을 수 있고, `Debug`가 있으면 그 타입을 담은 `List`나 `Dict`도 `{:?}`로 출력할 수 있습니다. `3.0`이 `3`으로 출력되는 이유는 부동소수의 기본 출력이 "다시 읽었을 때 같은 값이 되는 가장 짧은 표현"이기 때문입니다.

## 정리

- `str`은 불변 UTF-8입니다. `len()`과 슬라이싱은 바이트 단위로 동작하고, 문자는 `chars()`로 다룹니다.
- 문자열은 `StrBuf`에서 조금씩 만들고 `to_str()`로 꺼냅니다. 리터럴은 자동으로 이어지지 않으니 `+`로 잇습니다.
- 원시 문자열 `r"..."`, `r#"..."#`는 이스케이프를 처리하지 않습니다.
- `b"..."`는 `[u8; N]`(기대되는 자리에서는 `Slice<u8>`)입니다. `include_str!` / `include_bytes!`는 컴파일 시점에 파일을 포함합니다.
- f-string의 구멍에는 바깥과 같은 따옴표를 포함해 어떤 식이든 들어갑니다. `{{`와 `}}`는 중괄호 자체입니다.
- `:.2`, `:>8`, `:x`, `:?` 같은 지정자로 형식을 정하고, `${..}`로 동적으로 만듭니다.
- 내 타입에는 `Display`를 구현하고 `Debug`는 derive합니다.

## 더 읽을거리

- 사양: [문자열 리터럴과 f-string](../spec/refs-lang/001-lexical/107-string-literals.html), [문자 리터럴](../spec/refs-lang/001-lexical/108-character-literals.html), [바이트 문자열](../spec/refs-lang/001-lexical/107-string-literals.html#175-바이트-문자열), [파일 포함](../spec/refs-lang/006-meta-system/611-include.html)
- 표준 라이브러리 사양: [fmt](../spec/refs-standard-libraries/200-core/210-fmt.html), [문자열](../spec/refs-standard-libraries/200-core/215-string.html)
- API: [`str` 메서드](../../std/core/str.html#impl.str), [`StrBuf`](../../std/core/str.html#struct.StrBuf), [`Display`](../../std/core/fmt.html#trait.Display), [`Formatter`](../../std/core/fmt.html#struct.Formatter)
- 다음 장: [10. 에러 처리](10-errors.html)
