# 2. 값과 타입

이 장에서는 변수를 선언하는 `let`, 숫자와 문자열 같은 기본 타입, 값을 묶는 튜플과 리스트를 다룹니다. Rok은 정적 타입 언어이지만 타입을 대부분 추론하므로 직접 적을 일은 드뭅니다.

## `let`과 재할당

`let`은 새 변수를 만듭니다. 이미 있는 변수에 값을 쓰려면 `let` 없이 `=`를 씁니다. 변수는 기본적으로 바꿀 수 있습니다(`mut` 표시가 없습니다).

```rok
let name = "rok";
let count = 0;
count = count + 1;
count += 1;
println(f"{name}: {count}");
```

```output
rok: 2
```

선언과 재할당은 문법이 다르므로 오타 때문에 새 변수가 조용히 생기는 일이 없습니다. 선언한 적 없는 이름에 대입하면 컴파일 에러입니다.

```text check=checks/02-undeclared.rok
error[E200]: `coutn` is not declared; a variable is created only by `let`
  --> main.rok:2:1
   |
 2 | coutn = count + 1;
   | ^^^^^
   |
   = help: write `let coutn = count + 1;`
```

규칙이 몇 가지 더 있습니다.

- 초기값이 필요합니다. `let x;`는 에러입니다. 아직 값이 없으면 `let x: i32 | none = none;`처럼 값이 없다는 것을 직접 적습니다.
- 같은 스코프에서 같은 이름을 `let`으로 다시 선언할 수 없습니다. Rust 식 섀도잉(shadowing)이 없습니다. 안쪽 블록의 `let`은 바깥 변수와 별개의 변수를 만듭니다.
- 변수의 타입은 선언할 때 정해지고 바뀌지 않습니다. `i32` 변수에 문자열을 대입하면 에러입니다.
- `macro`, `defer`, `dyn`, `move`, `gen`, `do` 여섯 단어는 Rok 0.2.0에서 키워드가 됩니다. 지금 이 단어로 이름을 지으면 경고가 나옵니다. 다른 이름을 고르거나 앞에 `r#`을 붙이세요(`r#move`). `r#`을 붙이면 키워드(`r#type`)를 포함해 어떤 단어든 평범한 이름이 됩니다.

```rok
let a = 1;
{
    let a = 42;
    a = 10;
    println(f"inner a = {a}");
}
println(f"outer a = {a}");
```

```output
inner a = 10
outer a = 1
```

## 숫자 타입과 리터럴

정수 타입은 `i8` `i16` `i32` `i64` `isize`와 부호 없는 `u8` `u16` `u32` `u64` `usize`입니다. 부동소수점 수 타입은 `f32` `f64`입니다. 타입 이름 접미사가 붙은 리터럴은 그 타입이고, 접미사가 없으면 **기대 타입**을 따릅니다. 기대 타입이 없으면 정수는 `i32`, 실수는 `f64`가 기본입니다.

```rok
let a = 42;            // i32
let b: u8 = 200;       // u8, from the annotation
let c = 7i64;          // suffix
let d = 2.5f32;
let e = 3.14;          // f64
let big = 1_000_000u64;
println(f"{a} {b} {c} {d} {e} {big}");
println(f"{0xFF} {0b1010} {0o17} {1e3}");
println(f"{7 / 2} {-7 / 2} {7.0 / 2.0}");
```

```output
42 200 7 2.5 3.14 1000000
255 10 15 1000
3 -3 3.5
```

`_`는 자릿수 구분자이고, `0x` / `0b` / `0o`는 16진수 / 2진수 / 8진수입니다. 앞에 0만 붙인다고 8진수가 되지는 않습니다. `0755`는 십진수 755이고, rokc는 경고와 함께 `0o755`를 제안합니다. 정수 나눗셈은 0 쪽으로 버립니다. 타입의 범위를 벗어난 리터럴(`let x: u8 = 256;`)은 컴파일 에러입니다.

## 암묵 변환 없음: `as`

Rok은 서로 다른 숫자 타입을 자동으로 섞지 않습니다. `i32`와 `i64`를 더하면 에러입니다.

```text check=checks/02-mixed-int.rok
error[E200]: operator `+` cannot be applied to `i32` and `i64`
  --> main.rok:3:12
   |
 3 | println(f"{a + b}");
   |            ^^^^^
   |
   = help: rok never converts numbers implicitly; convert one side: `(a as i64) + b`
```

정밀도가 모르는 사이에 사라지지 않도록 일부러 이렇게 설계했습니다. 변환은 직접 적습니다. `as`는 항상 성공하지만 값을 자르거나 바꿀 수 있고, `.into()`는 손실 없는 변환에만 쓸 수 있습니다.

```rok
let x: i32 = 10;
let y: i64 = 3;
println(f"{(x as i64) + y}");
let w: i64 = x.into();
println(f"{w}");
println(f"{2.9 as i32} {300 as u8} {x as f64 / 4.0}");
```

```output
13
10
2 44 2.5
```

`2.9 as i32`는 소수 부분을 버리고, `300 as u8`은 하위 8비트만 남깁니다. 값을 조금도 잃으면 안 되는 변환에는 `.try_into::<T>()`를 씁니다. 이 메서드는 범위를 검사하고, 범위를 벗어나면 에러를 던집니다. 에러를 처리하는 방법은 [10장](10-errors.html)에서 다룹니다.

## 정수의 함정

Python, Go, JavaScript를 쓰던 사람이 놀랄 만한 정수 규칙이 몇 가지 있습니다.

**`len()`은 `usize`입니다.** 길이와 인덱스는 `usize`이고, 접미사 없는 리터럴은 `i32`입니다. 암묵 변환이 없으므로 `total`이 `i32`이면 `total / xs.len()`은 에러입니다. 한쪽을 `as`로 변환하세요.

```text check=checks/02-len-usize.rok
error[E200]: operator `/` cannot be applied to `i32` and `usize`
```

```rok
let xs = [10, 20, 30];
let total = 0;
for x in xs { total += x; }
let avg = total / (xs.len() as i32);
println(f"average {avg}");

for i in 0..xs.len() {       // i is a usize, so xs[i] just works
    print(f"{i}:{xs[i]} ");
}
println("");
```

```output
average 20
0:10 1:20 2:30 
```

**`/`와 `%`는 0 쪽으로 버립니다.** `-7 / 2`는 `-3`, `-7 % 2`는 `-1`입니다. C, Go, Rust처럼 나머지는 왼쪽 피연산자의 부호를 따릅니다. 결과가 제수의 부호를 따르는 Python의 `%`가 필요하면 `%%`(내림 나머지, floored modulo)를 씁니다. 인덱스가 끝에서 처음으로 돌아가게 할 때 편합니다.

```rok
println(f"{-7 / 2} {-7 % 2} {-7 %% 2}");
let len = 5;
let i = -1;
println(f"{i % len} {i %% len}");
```

```output
-3 -1 1
-1 4
```

**오버플로는 빌드에 따라 다릅니다.** `rok run`과 `rok build`는 최적화 없이(`-O0`) 오버플로 검사를 켠 채 컴파일합니다. 그래서 범위를 넘으면 panic이 일어납니다. 최적화 빌드(`-O1` 이상, `rok build --release`는 `-O2`)는 검사를 끕니다. 그래서 같은 덧셈이 조용히 한 바퀴 돌아 작은 값이 됩니다.

```rok build
let x: i32 = 2147483647;
println(f"{x + 1}");
```

```text
$ rok run
thread 'main' panicked at src/main.rok:2:12:
attempt to add with overflow
$ rok run --release
-2147483648
```

최적화 빌드에서도 검사를 유지하려면 `-C overflow-checks=yes`를 추가합니다. 오버플로가 날 수 있는 곳에서는 `checked_add`(`none`을 반환), `wrapping_add`, `saturating_add`로 원하는 동작을 밝히세요. 오버플로가 없는 `int`(아래 참고)를 써도 됩니다.

```rok
let x: i32 = 2147483647;
println(f"{x.checked_add(1):?} {x.wrapping_add(1)} {x.saturating_add(1)}");
```

```output
none -2147483648 2147483647
```

## bool, char, str

`bool`은 `true` / `false`이고 `&&`, `||`, `!`로 조합합니다. `char`는 Unicode 문자 하나이고 작은따옴표로 씁니다. `str`은 UTF-8 문자열이고 큰따옴표로 씁니다.

```rok
let ok = 3 < 5 && !false;
let ch = '가';
println(f"{ok} {ch} {ch as u32} {65u8 as char}");

let s = "héllo";
println(f"{s.len()} bytes, {s.char_count()} chars");
let t = s + ", rok";
println(f"{t} {t.to_uppercase()} {t.contains("rok")}");
println("tab:\tend");
println(r"raw: C:\path\n");
```

```output
true 가 44032 A
6 bytes, 5 chars
héllo, rok HÉLLO, ROK true
tab:	end
raw: C:\path\n
```

한글 음절 `'가'`(U+AC00, 코드 포인트 44032)에서 보듯이 `char`에는 ASCII뿐 아니라 어떤 Unicode 문자든 담을 수 있습니다. `len()`은 바이트 수를 셉니다. 문자 수가 필요하면 `char_count()`를 씁니다. `r"..."`는 이스케이프를 처리하지 않는 원시 문자열입니다. 그 밖의 `str` 메서드(`trim`, `split`, `replace`, `parse` 등)는 [표준 라이브러리 문서](../../std/core/str.html)에 있습니다.

## 튜플, 배열, 리스트

**튜플**은 타입이 서로 다른 값 몇 개를 묶습니다. `.0`, `.1`로 접근하거나 `let`으로 분해합니다.

```rok
let p = (1, "one", true);
println(f"{p.0} {p.1} {p:?}");
let (n, word, _) = p;
println(f"{word} = {n}");
```

```output
1 one (1, "one", true)
one = 1
```

`[...]` 리터럴은 기본적으로 **`List`**입니다. 길이가 늘어나는 리스트입니다. 리스트를 바꾸면 같은 리스트를 가리키는 모든 변수에서 바뀐 내용이 보입니다. `List`는 `{}`로 출력할 수 없으므로 Debug 형식 `{:?}`를 씁니다.

```rok
let names = ["ada", "grace"];
names.push("linus");
println(f"{names:?} {names.len()} {names[0]}");

let empty: List<i32> = [];
println(f"{empty.is_empty()}");
```

```output
["ada", "grace", "linus"] 3 ada
true
```

변수의 타입이 `[T; N]`이면 같은 리터럴이 길이가 고정된 **배열**이 됩니다. 크기를 컴파일 타임에 알 수 있으므로 행렬이나 버퍼처럼 크기가 고정된 데이터에 씁니다.

```rok
let grid: [[i32; 2]; 2] = [[1, 2], [3, 4]];
println(f"{grid:?} {grid[1][0]} {grid.len()}");
```

```output
[[1, 2], [3, 4]] 3 2
```

빈 리스트처럼 원소에서 원소 타입을 추론할 수 없으면 `let xs: List<i32> = [];`처럼 타입을 적습니다.

## 큰 정수: `int`

`i32` 같은 고정 폭 타입은 빠르지만 범위가 있습니다. 범위를 신경 쓰지 않고 정확한 정수가 필요하면 `int`를 씁니다. 작은 값은 고정 폭 정수처럼 저장되고, 커지면 자동으로 큰 정수로 바뀝니다. `int` 값은 타입을 적은 곳에서만 생깁니다. 고정 폭 타입과 섞을 때도 변환을 직접 적어야 합니다.

```rok
let big: int = 1;
for _ in 0..100 {
    big = big * 2;
}
println(f"2^100 = {big}");
```

```output
2^100 = 1267650600228229401496703205376
```

> [!NOTE]
> 심화: `int`에는 `isqrt`, `modpow`, `gcd`, `lcm` 같은 정수론 도우미도 있습니다. [`int` API](../../std/core/num/bigint.html)에 정리되어 있습니다.

## 상수

`const`는 컴파일 타임에 정해지는 값입니다. `let`과 달리 타입을 생략할 수 없고, 모듈의 최상위에 둘 수 있습니다. 컴파일러는 쓰는 곳마다 값을 그 자리에 넣습니다.

```rok
const MAX_USERS: i32 = 100;
const GREETING: str = "welcome";

fn capacity_left(used: i32) -> i32 {
    MAX_USERS - used
}

println(f"{GREETING}: {capacity_left(42)} seats left");
```

```output
welcome: 58 seats left
```

## 정리

- `let`은 선언하고, `=`는 재할당합니다. 같은 스코프에서 다시 선언할 수 없고 초기값이 필요합니다.
- 접미사 없는 리터럴은 기대 타입을 따르고, 기본은 `i32` / `f64`입니다.
- 숫자 타입 사이에 자동 변환은 없습니다. `as`, `.into()`, `.try_into::<T>()`를 씁니다.
- `len()`과 인덱스는 `usize`입니다. `i32`와 섞으려면 `as`로 변환합니다.
- `/`와 `%`는 0 쪽으로 버리고, `%%`는 내림 나머지입니다. 오버플로가 나면 debug 빌드에서는 panic이 일어나고 최적화 빌드에서는 값이 한 바퀴 돕니다.
- `[...]`는 `List`이고, `[T; N]` 타입이면 고정 배열입니다. `List`는 `{:?}`로 출력합니다.
- `int`는 범위 제한이 없는 정수이고, `const`는 컴파일 타임 상수입니다.

## 더 읽을거리

- [변수 선언](../spec/refs-lang/005-modules/501-variable-declaration.html), [`const`](../spec/refs-lang/005-modules/502-const.html)
- [숫자 리터럴](../spec/refs-lang/001-lexical/106-numeric-literals.html), [문자열 리터럴](../spec/refs-lang/001-lexical/107-string-literals.html), [문자 리터럴](../spec/refs-lang/001-lexical/108-character-literals.html)
- [형 변환](../spec/refs-lang/002-types/214-cast-and-conversion.html), [타입 표현식](../spec/refs-lang/002-types/201-type-expressions.html)
- [숫자 타입과 `int`](../spec/refs-standard-libraries/200-core/201-numbers.html), [형식화된 출력](../spec/refs-standard-libraries/200-core/210-fmt.html)
- 표준 라이브러리 API: [`str`](../../std/core/str.html), [`List`](../../std/core/collections/list.html)
- 다음 장: [3. 함수](03-functions.html)
