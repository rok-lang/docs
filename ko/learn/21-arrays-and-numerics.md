# 21. 배열과 수치 계산

Rok의 수치 계산은 세 층으로 되어 있습니다. 보통의 수에는 흔한 연산자와 거듭제곱 `**`가 있습니다. `List<T>`와 고정 배열은 [8장](08-collections-iterators.html)의 원소별 점 연산자(`.+`, `.*`, ...)를 받습니다. 축이 둘 이상인 데이터에는 `rok::core::array::Array<T, N>`이 있습니다. NumPy식 인덱싱, 슬라이싱, 브로드캐스트를 갖춘 n차원 배열이며, 선형대수·통계·그래프 라이브러리가 모두 한 타입을 함께 쓰도록 core에 들어 있습니다.

## 거듭제곱

`a ** b`는 `a`의 `b` 제곱입니다. 수학이나 Python처럼 전위 `-`보다 강하게 묶이고 오른쪽부터 묶입니다. `-x ** 2`는 `-(x ** 2)`이고, `2 ** 3 ** 2`는 `2 ** 9`입니다. 음수를 거듭제곱하려면 괄호로 묶습니다.

```rok
let x = 3.0;
println(f"{x ** 2} {-x ** 2} {(-x) ** 2}");
println(f"{2 ** 3 ** 2} {2 ** 10}");
println(f"{2.0 ** -1} {2.0 ** 0.5}");
```

```output
9 -9 9
512 1024
0.5 1.4142135623730951
```

정수 밑은 `u32` 지수를 받아 자기 타입의 정수를 줍니다. 오버플로는 `*`처럼 panic합니다. 부동소수 밑은 부동소수 지수나 `i32` 지수를 받습니다(`i32`면 반복 곱셈을 씁니다). 지수가 밑과 같은 타입일 필요는 없습니다. `i64`인 `n`에 대한 `n ** 2`도 되고, 리터럴은 `u32`가 됩니다. `2 ** -1`은 답이 분수이므로 컴파일 에러이며, 에러는 `2.0 ** -1.0`으로 쓰라고 알려 줍니다. `**=`는 제자리 거듭제곱이고 `.**`는 원소별 형태입니다.

```rok
let n = 7;
let k: u32 = 3;
let y = 1.5;
y **= 2;
println(f"{n ** 2} {n ** k} {y} {[1, 2, 3] .** 2:?}");
```

```output
49 343 2.25 [1, 4, 9]
```

`^`는 C나 Rust처럼 그대로 비트 XOR입니다.

## 배열 만들기

`Array<T, N>`은 `T` 타입의 원소를 `N`개 축을 따라 담습니다. `N`은 랭크이며 타입의 일부입니다. `Vector<T>`와 `Matrix<T>`는 랭크 1과 2의 이름입니다. `Array`가 올 자리에서는 중첩 리스트 리터럴이 배열을 만듭니다. 중첩 깊이가 랭크이고, 행들이 모양을 정합니다.

```rok
use rok::core::array::{Array, Matrix};

let m: Matrix<f64> = [[1, 2, 3],
                      [4, 5, 6]];
println(f"{m:?}");
println(f"shape {m.shape():?}, {m.len()} elements, rank {m.ndim()}");
```

```output
[[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]
shape [2, 3], 6 elements, rank 2
```

슬롯이 `f64`라고 하므로 정수 리터럴이 `f64`가 되었습니다. 행의 길이가 다르면 컴파일 에러(E0143)입니다. 배열은 직사각형이므로 들쭉날쭉한 데이터는 `List<List<T>>`에 둡니다.

생성자는 모양을 길이들의 배열로 받습니다.

```rok
use rok::core::array::Array;

let z: Array<i64, 2> = Array::zeros([2, 3]);
let o: Array<f64, 1> = Array::ones([3]);
let g = Array::from_fn([2, 3], |i: [usize; 2]| -> usize { 10 * i[0] + i[1] });
let r = Array::from_list([1, 2, 3, 4, 5, 6], [3, 2]);
println(f"{z:?} {o:?}");
println(f"{g:?} {r:?}");
```

```output
[[0, 0, 0], [0, 0, 0]] [1.0, 1.0, 1.0]
[[0, 1, 2], [10, 11, 12]] [[1, 2], [3, 4], [5, 6]]
```

`Array::full(shape, v)`는 한 값을 되풀이하고, `Array::from(xs)`는 리스트를 랭크 1 배열로 바꿉니다.

고정 크기 `[T; N]`에는 따로 반복 형태 `[v; N]`이 있습니다. `v`를 한 번 평가하고 원소마다 따로 복사본을 줍니다. `0.0` 같은 평범한 값은 그대로 복사하고, 컬렉션을 가진 원소는 복제하므로 `[List::<f64>::new(); 4]`는 서로 다른 리스트 네 개, `[[0.0; 3]; 3]`은 서로 다른 행 세 개입니다([2장](02-values-and-types.html)). `array::from_fn(f)`는 `i`번째 원소가 `f(i)`인 `[T; N]`을 만들며, `f`는 순서대로 불립니다. `N`은 기대 타입이나 `array::from_fn::<T, N>`에서 옵니다. 원소마다 따로 값을 만들므로 `[v; N]`으로는 만들 수 없는 배열, 곧 `Clone`이 아닌 원소나 소유자가 하나인 자원의 배열도 만듭니다.

```rok
let xs = [3, 1, 2];
let tens: [i64; 3] = array::from_fn(|i| xs[i] * 10);
let lists = array::from_fn::<List<i64>, 2>(|_| []);
lists[0].push(1);
println(f"{tens:?} {lists:?}");
```

```output
[30, 10, 20] [[1], []]
```

## 인덱싱과 슬라이싱

괄호 안에서 콤마로 나뉜 부분 하나가 축 하나입니다. 위치는 그 축에서 인덱스 하나를 고르고 그 축을 없앱니다. 슬라이스 `a:b:k`는 고른 원소들로 축을 남기며, 규칙은 리스트 슬라이스와 같습니다(음수 경계는 끝에서, 경계는 잘라 맞추고, 음수 스텝은 거꾸로 걷습니다). 랭크만큼의 위치는 원소 하나를 줍니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[0, 1, 2, 3],
                        [10, 11, 12, 13],
                        [20, 21, 22, 23]];
println(f"{a[1, 2]} {a[-1, -1]}");
println(f"{a[1]:?} {a[:, 0]:?}");
println(f"{a[1:, ::2]:?}");
println(f"{a[::-1, 3]:?}");
```

```output
12 23
[10, 11, 12, 13] [0, 10, 20]
[[10, 12], [20, 22]]
[23, 13, 3]
```

모양을 다룰 때 쓰는 부분이 둘 더 있습니다. `..`는 다른 부분이 남긴 축 전부를 뜻하므로, `a[.., 0]`은 랭크와 상관없이 첫 열입니다. `none`은 길이 1인 축을 끼워 넣습니다. 벡터를 행이나 열로 만들 때 씁니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let v: Array<i64, 1> = [7, 8, 9];
println(f"{a[.., 0]:?} {v[none, :].shape():?} {v[:, none].shape():?}");
```

```output
[1, 4] [1, 3] [3, 1]
```

위치는 어떤 정수 타입이어도 됩니다. `usize` 루프 카운터도 리터럴만큼 잘 됩니다.

슬라이스, 축을 남기는 위치, `..`, `none`, `t()`, `reshape`는 모두 *뷰*를 돌려줍니다. 같은 원소 위에 새 모양과 보폭을 얹은 것이며 복사하지 않습니다. 뷰를 거쳐 쓰면 배열에 씁니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let col = a[:, 1];
col[0] = 20;
let t = a.t();
println(f"{a:?} {t[2, 1]} {t.shape():?}");
println(f"{a.reshape([3, 2]):?}");
```

```output
[[1, 20, 3], [4, 5, 6]] 6 [3, 2]
[[1, 20], [3, 4], [5, 6]]
```

따로 떨어진 배열이 필요하면 `copy()`를 씁니다. 모양이 같은 불리언 배열은 *마스크*입니다. `a[mask]`는 `true`인 자리의 원소를 순서대로 모아 새 랭크 1 배열로 줍니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
println(f"{a[a .% 2 .== 0]:?}");
```

```output
[2, 4, 6]
```

## 선택한 곳에 대입하기

인덱싱할 수 있는 곳에는 대입할 수도 있습니다. 값 하나는 선택한 곳을 채웁니다. 배열은 선택한 곳의 모양으로 늘어나 복사됩니다(아래 브로드캐스트 참고). 마스크 대입은 고른 원소만 바꿉니다.

```rok
use rok::core::array::Array;

let a: Array<f64, 2> = Array::zeros([3, 4]);
a[0, :] = 1.0;
a[1:, 0] = 2.0;
let row: Array<f64, 1> = [5, 6, 7, 8];
a[2] = row;
a[a .> 6.0] = -1.0;
println(f"{a:?}");
```

```output
[[1.0, 1.0, 1.0, 1.0], [2.0, 0.0, 0.0, 0.0], [5.0, 6.0, -1.0, -1.0]]
```

`List`도 같은 길이의 구간에 대한 슬라이스 대입을 받습니다: `xs[1:3] = [8, 9];`. 스텝이 있는 리스트 슬라이스에는 대입할 수 없습니다(컴파일러가 그렇게 알리고 원소를 하나씩 저장하라고 제안합니다).

## 원소별 연산과 브로드캐스트

점 연산자는 배열에서 원소마다 동작합니다. 산술은 원소 타입의 배열을, 비교는 `bool` 배열을 줍니다. 어느 쪽에 있든 스칼라는 모든 원소에 적용됩니다.

```rok
use rok::core::array::Array;

let a: Array<f64, 2> = [[1, 2], [3, 4]];
println(f"{a .* a .+ 1.0:?}");
println(f"{2.0 .* a:?} {a .** 2.0:?}");
println(f"{a .>= 2.0:?}");
```

```output
[[2.0, 5.0], [10.0, 17.0]]
[[2.0, 4.0], [6.0, 8.0]] [[1.0, 4.0], [9.0, 16.0]]
[[false, true], [true, true]]
```

랭크가 같은 두 배열은 축마다 길이가 맞아야 합니다. 다만 길이 1인 축은 다른 쪽 길이로 늘어납니다. NumPy의 규칙이며 규칙은 이것 하나뿐입니다. 모든 배열 타입이 같도록 core(`broadcast_shape`)에 있습니다. 그 밖의 경우는 두 모양을 밝히며 panic합니다.

```rok
use rok::core::array::Array;

let m: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let row: Array<i64, 2> = [[10, 20, 30]];
let col: Array<i64, 2> = [[100], [200]];
println(f"{m .+ row:?}");
println(f"{m .+ col:?}");
```

```output
[[11, 22, 33], [14, 25, 36]]
[[101, 102, 103], [204, 205, 206]]
```

랭크가 낮은 배열은 앞에 길이 1인 축이 있는 것처럼 다른 쪽의 마지막 축들과 맞춰집니다. 그래서 벡터를 행렬의 모든 행에 더할 수 있습니다. 결과는 높은 쪽 랭크입니다.

```rok
use rok::core::array::Array;

let m: Array<f64, 2> = [[1, 2], [3, 4]];
let v: Array<f64, 1> = [10, 20];
println(f"{m .+ v:?}");
```

```output
[[11.0, 22.0], [13.0, 24.0]]
```

점 연산자 사슬 전체는 `List`에서처럼 `Array`에서도 중간 배열 없이 루프 하나로 돕니다. `a .* x .+ b .* 2.0`은 `a`, `x`, `b`의 원소를 한 번씩 읽고 결과를 한 번 씁니다.

## 제자리에서 갱신하기

`y .= expr`는 `expr`의 원소를 `y`에 쓰고, `y .+= x`, `.-=`, `.*=`, `./=`, `.%=`, `.**=`는 `y`와 `x`를 원소별로 합칩니다. 할당은 없습니다. 루프가 `y`에 바로 씁니다. 대상이 뷰여도 되므로 배열의 슬라이스나 열을 그 자리에서 갱신합니다.

```rok
use rok::core::array::Array;

let y: Array<f64, 1> = Array::zeros([4]);
let x: Array<f64, 1> = [1, 2, 3, 4];
y .= 0.5 .* x .+ 1.0;
y .*= 2.0;
let m: Array<f64, 2> = [[1, 2], [3, 4]];
m[:, 0] .= 0.0;
let xs = [1, 2, 3];
xs .+= [10, 10, 10];
println(f"{y:?} {m:?} {xs:?}");
```

```output
[3.0, 4.0, 5.0, 6.0] [[0.0, 2.0], [0.0, 4.0]] [11, 12, 13]
```

대상은 모양을 유지합니다. 오른쪽이 대상에 맞춰 늘어나고, 반대는 없습니다. 오른쪽이 대상 자신의 원소를 다른 위치에서 읽으면(`v .= v[::-1]`) 먼저 떠 둔 복사본을 읽으므로, 결과는 모든 원소를 계산한 뒤에 저장한 것과 같습니다. 그 밖의 연산자는 `.=`로 씁니다: `xs .= xs .%% 3`.

## 빠진 값

원소 타입 `T | none`은 빠진 값을 나타냅니다. `a .?? b`는 `none`마다 `b`의 대응 원소로, `b`가 값 하나이면 그 값으로 채웁니다. NaN은 `none`이 아닙니다. 부동소수 값이므로 그대로 지나갑니다. NaN을 빠진 값으로 다루려면 `f64::is_nan.(a)`로 골라 대입합니다.

```rok
use rok::core::array::Array;

let raw: Array<f64 | none, 1> = Array::from_list([1.0, none, f64::NAN], [3]);
let filled = raw .?? 0.0;
println(f"{filled:?}");
filled[f64::is_nan.(filled)] = -1.0;
println(f"{filled:?}");
```

```output
[1.0, 0.0, NaN]
[1.0, 0.0, -1.0]
```

## 줄이기와 반복

`sum()`은 모든 원소를 더하고, `map(f)`는 같은 모양의 새 배열을 만들며, `to_list()`는 원소를 행 우선 순서(마지막 축이 가장 빨리 바뀜)로 주고, `for x in a`도 그 순서로 돕니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2], [3, 4]];
let total = a.sum();
let sq = a.map(|x: i64| -> i64 { x * x });
let parts = [];
for x in a.t() {
    parts.push(f"{x}");
}
println(f"{total} {sq:?} {parts.join(" ")}");
```

```output
10 [[1, 4], [9, 16]] 1 3 2 4
```

## 부동소수, 복소수, 난수

`num::FloatingPoint`는 `f32`와 `f64`를 함께 다루는 코드의 바운드입니다. 상수(`T::NAN`, `T::PI`)와 수학 함수(`sqrt`, `exp`, `sin`, ...)를 한 트레이트로 묶습니다. `num::Num`은 모든 정수·부동소수 타입이 만족하는 더 넓은 바운드로, `+ - * /`, `0`, `1`, 비교를 줍니다. 부동소수 리스트와 반복자에는 `min`, `max`, `sum`이 있습니다. 항목 가운데 NaN이 있으면 `min`과 `max`는 NaN이 되고, `sum`은 정해진 순서로 둘씩 더하므로 결과가 플랫폼에 따라 달라지지 않습니다. 부동소수는 일부만 순서가 있으니 정렬은 `total_cmp`로 합니다. `num::linspace(a, b, n)`은 `a`부터 `b`까지 고르게 놓인 값 `n`개를, `num::arange(a, b, step)`은 `a`부터 `b` 직전까지 `step`씩 늘린 값을 줍니다.

```rok
fn hypot<T: num::FloatingPoint>(a: T, b: T) -> T { (a * a + b * b).sqrt() }
println(f"{hypot(3.0, 4.0)} {hypot(5.0f32, 12.0f32)}");

let xs = [2.5, -1.0, 7.0, 0.5];
println(f"{xs.max() ?? 0.0} {xs.sum()} {xs.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{num::linspace(0.0, 1.0, 5):?}");
```

```output
5 13
7 9 [-1.0, 0.5, 2.5, 7.0]
[0.0, 0.25, 0.5, 0.75, 1.0]
```

`total_cmp`는 IEEE 754 전순서입니다. 부호 비트가 1인 NaN이 맨 앞, 0인 NaN이 맨 뒤에 옵니다. `f64::NAN`과 `f32::NAN`은 모든 타깃에서 부호 비트가 0인 quiet NaN이므로 `inf` 뒤로 정렬됩니다. 연산이 만든 NaN(`0.0 / 0.0`)은 부호가 어느 쪽이든 될 수 있고, x86은 1로 둡니다.

```rok
let ys = [1.0, f64::NAN, f64::NEG_INFINITY, -0.0, f64::INFINITY];
println(f"{ys.sorted_by(|a, b| a.total_cmp(b)):?}");
println(f"{f64::NAN.is_sign_negative()} {f64::NAN.to_bits():x}");
```

```output
[-inf, -0.0, 1.0, inf, NaN]
false 7ff8000000000000
```

`num::Complex<T>`는 `f32`나 `f64` 위의 복소수이고, `i` 접미사로 허수부를 씁니다: `3.0 + 4.0i`. 산술 연산자를 양쪽 어디에 실수가 와도 쓸 수 있고, `abs`, `arg`, `conj`, `exp`, `ln`, `sqrt`, `pow`가 있습니다.

```rok
let z = 3.0 + 4.0i;
println(f"{z} {z.abs()} {z * 2i} {z.conj()}");
```

```output
3+4i 5 -8+6i 3-4i
```

`rok::std::random`에는 운영체제가 시드를 준 생성기에서 뽑는 한 번 호출 형태(`random::range(1..7)`, `random::f64()`, `random::choice(xs)`, `random::shuffle(xs)`)가 있습니다. 다시 돌려도 같은 결과가 필요하면 `seed`로 `Xoshiro256PlusPlus`를 만듭니다. 같은 시드는 모든 타깃과 릴리스에서 같은 수를 줍니다. 메서드는 `Rng` 트레이트에서 옵니다(`range`, `uniform`, `normal`, `choice`, `sample`, `shuffle`). `split()`은 다른 스레드에 줄 독립된 생성기를 만듭니다. 키나 토큰에는 모든 수를 운영체제에서 뽑는 `OsRng`를 씁니다.

```rok
use rok::std::random;
use rok::std::random::{Rng, Xoshiro256PlusPlus};

let a = Xoshiro256PlusPlus::seed(7);
let b = Xoshiro256PlusPlus::seed(7);
println(a.range(1..7) == b.range(1..7));
println(0.0 <= a.uniform(0.0, 1.0) < 1.0);
println(random::range(1..7) in 1..7);
```

```output
true
true
true
```

## 메모리와 속도

배열의 원소는 빈틈없는 버퍼 하나에 있습니다. 원소 타입에 참조가 없으면(수, `bool`, `char`, `#[derive(Pod)]` 구조체) 가비지 컬렉터는 그 안을 보지 않으므로, 아주 큰 배열도 수집 비용이 들지 않습니다. 8 KiB 이상인 버퍼는 64바이트에 정렬되고 배열에 도달할 수 있는 동안 옮겨지지 않습니다. 벡터 코드와 BLAS 같은 C 라이브러리가 바라는 조건입니다. `unsafe { a.as_ptr() }`는 첫 원소를 주고, `strides()`는 나머지를 걷는 법을 알려 줍니다. `is_contiguous()`는 원소가 행 우선 순서로 붙어 있는지 알려 주고, `copy()`는 붙어 있게 만듭니다.

## 나만의 배열 타입

인덱싱 문법은 core의 `Array`만의 것이 아닙니다. 부분이 여러 개인 괄호는 튜플을 들고 `Index`(읽기)나 `IndexSet`(쓰기)을 호출하고, 슬라이스 부분은 `rok::core::index::SliceArg`로 옵니다. 타입이 표시 trait `AxisIndexed`를 구현하면 정수 부분은 모두 `isize`로, `none`은 `NewAxis`로, `..`는 `Rest`로 오므로, impl 하나가 인덱스를 쓰는 모든 방식을 받습니다.

```rok
use rok::core::index::{AxisIndexed, SliceArg};

struct Grid { rows: usize; cols: usize; }

impl AxisIndexed for Grid {}

impl index::Index<(SliceArg, isize)> for Grid {
    type Output = str;
    fn index(self, at: (SliceArg, isize)) -> str {
        let (first, count, step) = at.0.indices(self.rows);
        f"{count} rows from {first} by {step}, column {at.1}"
    }
}

let g = Grid { rows: 10, cols: 4 };
let c: usize = 2;
println(g[2:, c]);
println(g[::-3, -1]);
```

```output
8 rows from 2 by 1, column 2
4 rows from 9 by -3, column -1
```

수치 타입을 정의한 패키지는 그 타입에 스칼라가 왼쪽에 오는 연산자도 줄 수 있습니다. orphan 규칙([6장](06-structs-enums-traits.html))은 트레잇의 타입 인자 가운데 하나가 내 타입이면 core의 트레잇을 core의 타입에 구현하도록 허용하므로, `2.0 * v`가 됩니다.

```rok
#[derive(Clone, Copy)]
struct V2 { x: f64; y: f64; }

impl ops::Mul<V2> for f64 {
    type Output = V2;
    fn mul(self, v: V2) -> V2 { V2 { x: self * v.x, y: self * v.y } }
}

let v = 2.0 * V2 { x: 1.5, y: -1.0 };
println(f"{v.x} {v.y}");
```

```output
3 -2
```

맨 타입 매개변수가 내 타입보다 먼저 나오는 `impl<T> Mul<V2> for T`는 여전히 거부됩니다. 다른 패키지가 자기 타입에 대해 같은 impl을 쓸 수 있기 때문입니다.

내 타입의 점 식이 `Array`처럼 융합되게 하려면 원소를 `Array`로 보여 주는 `Broadcastable`과, 내 타입이 앞설 때 결과를 감싸는 `Materialize`를 구현합니다. 데이터를 `Array`에 두는 타입은 그것을 돌려주면 됩니다.

```rok
use rok::core::array::Array;
use rok::core::ops::{Broadcastable, Materialize};

struct Signal { data: Array<f64, 1>; }

impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}

impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(self, out: Array<f64, 1>) -> Signal { Signal { data: out } }
}

let s = Signal { data: [1, 2, 3] };
let t = s .* 2.0 .+ 0.5;
t .-= 0.5;
println(f"{t.data:?}");
```

```output
[2.0, 4.0, 6.0]
```

## 리터럴 접미사

패키지는 핸들러 함수에 `#[literal(number "suffix")]`를 붙여 수에 자기 접미사를 줄 수 있습니다. 핸들러가 `pub`이면 그 패키지에 의존하는 모든 패키지도 `use` 없이 그 접미사를 씁니다.

```rok norun
// `units` 스톤 안
pub struct Meters { pub v: f64; }

#[literal(number "m")]
pub fn meters(n: f64) -> Meters { Meters { v: n } }
```

```rok norun
// `"units": Dep::path("../units")`가 있는 패키지 안
let d = 9.81m;          // units::meters(9.81)
println(f"{d.v}");
```

`pub`이 아닌 핸들러는 자기 패키지 안에서만 동작합니다. 두 의존성이 같은 접미사를 등록하면 그 사용이 모호해지므로 한쪽에 `priority`를 줍니다.
