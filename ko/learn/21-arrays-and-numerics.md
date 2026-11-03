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

정수 밑은 `u32` 지수를 받아 자기 타입의 정수를 줍니다. 오버플로는 `*`처럼 panic합니다. 부동소수 밑은 부동소수 지수나 `i32` 지수를 받습니다(`i32`면 반복 곱셈을 씁니다). 지수가 밑과 같은 타입일 필요는 없습니다. `n: i64`에 대한 `n ** 2`도 되고, 리터럴은 `u32`가 됩니다. `2 ** -1`은 답이 분수이므로 컴파일 에러이며, 에러는 `2.0 ** -1.0`으로 쓰라고 알려 줍니다. `**=`는 제자리 거듭제곱이고 `.**`는 원소별 형태입니다.

```rok
let n: i64 = 7;
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

랭크가 다른 배열은 길이 1인 축을 직접 붙여 맞춥니다. 행이라면 `v[none, :]`입니다. 랭크가 타입에 들어 있으므로 결과의 랭크가 코드에 그대로 보입니다.

`List`에서는 점 연산자 사슬 전체가 중간 리스트 없이 루프 하나로 돕니다. `Array`에서는 지금은 연산자마다 결과를 만들므로, 긴 사슬은 연산자마다 한 번씩 할당합니다.

## 줄이기와 반복

`sum()`은 모든 원소를 더하고, `map(f)`는 같은 모양의 새 배열을 만들며, `to_list()`는 원소를 행 우선 순서(마지막 축이 가장 빨리 바뀜)로 주고, `for x in a`도 그 순서로 돕니다.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2], [3, 4]];
let total = a.sum();
let sq = a.map(|x: i64| -> i64 { x * x });
let parts: List<str> = [];
for x in a.t() {
    parts.push(f"{x}");
}
println(f"{total} {sq:?} {parts.join(" ")}");
```

```output
10 [[1, 4], [9, 16]] 1 3 2 4
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
