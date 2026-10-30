# 7. 제네릭

같은 로직을 여러 타입에 쓰고 싶을 때 제네릭을 씁니다. Rok의 제네릭은 호출에 쓰인 구체 타입마다 따로 컴파일되므로(단형화, monomorphization) 타입별로 직접 쓴 함수만큼 빠릅니다.

## 제네릭 함수

함수 이름 뒤 `<T>`에 타입 매개변수를 선언하고, 매개변수 타입과 반환 타입에 씁니다. 호출하는 곳에서는 보통 타입을 적지 않고, 컴파일러가 인자에서 추론합니다.

```rok
fn first_or<T>(xs: List<T>, fallback: T) -> T {
    xs.first() ?? fallback
}

println(f"{first_or([10, 20], 0)}");
println(first_or(["a", "b"], "-"));
let empty: List<f64> = [];
println(f"{first_or(empty, 1.5)}");
```

```output
10
a
1.5
```

한 호출 안에서 `T`는 하나의 타입입니다. `first_or([1, 2], "x")`처럼 서로 다른 타입을 넘기면 컴파일 에러입니다.

## 트레잇 바운드

바운드가 없는 `T`로는 값을 옮기는 일밖에 할 수 없습니다. 비교하거나 더하려면 `T: Ord` 같은 **바운드**로 필요한 트레잇을 지정합니다. 바운드 여러 개는 `+`로 잇습니다. 컴파일러는 제네릭 본체를 바운드만 보고 검사하므로 어떤 타입으로 호출하든 본체가 깨지지 않습니다.

```rok
fn largest<T: Ord>(xs: List<T>) -> T? {
    let best = xs.first()?;
    for x in xs {
        if x > best { best = x; }
    }
    best
}

println(f"{largest([3, 9, 2]):?}");
println(f"{largest(["pear", "apple"]):?}");
let none_here: List<i32> = [];
println(f"{largest(none_here):?}");
```

```output
9
"pear"
none
```

`T: Zero`가 있으면 `T::zero()`처럼 타입 매개변수의 정적 메서드도 호출할 수 있습니다.

```rok
fn sum<T: Zero + Add<Output = T>>(xs: List<T>) -> T {
    let acc = T::zero();
    for x in xs { acc = acc + x; }
    acc
}

println(f"{sum([1, 2, 3])} {sum([0.5, 0.25])}");
```

```output
6 0.75
```

### 본체는 바운드로 검사합니다

제네릭 본체는 호출과 상관없이 본체만으로 한 번 타입 검사를 받습니다. 본체 안에서 `T`로 할 수 있는 일은 바운드가 약속한 것과 모든 타입이 지원하는 것(옮기기, `any`에 담기, `T | none` 만들기, `is`)뿐입니다. 바운드를 빠뜨리면, 모든 호출이 문제없이 동작할 타입을 넘기더라도 본체에서 에러가 납니다.

```rok norun
fn double<T>(x: T) -> T { x + x }
```

```text check=checks/07-missing-bound.rok
error[E200]: operator `+` on `T` needs a `Add` bound on `T` (spec 2.10.9)
  --> main.rok:1:27
   |
 1 | fn double<T>(x: T) -> T { x + x }
   |                           ^^^^^
```

바운드를 추가하면 같은 본체가 `Add`를 구현한 모든 타입으로 컴파일됩니다. 바운드가 필요 없는 연산은 바운드 없이 동작합니다.

```rok
fn double<T: Add<T, Output = T>>(x: T) -> T { x + x }
fn keep<T>(x: T, yes: bool) -> T | none { if yes { x } else { none } }
fn boxed<T>(x: T) -> any { x }

println(f"{double(21)} {double(1.5)}");
println(f"{keep("a", true):?} {keep(1, false):?}");
println(f"{boxed(3) is i32}");
```

```output
42 3
"a" none
true
```

이 규칙 덕분에 라이브러리의 제네릭 함수가 내 환경에서 컴파일되면, 그 함수를 호출하는 누구에게서나 컴파일됩니다. 멀리 떨어진 호출 지점에서 에러가 드러나는 일이 없습니다. 트레잇의 기본 메서드도 같은 방식으로 검사하고, 이때 `Self`는 트레잇이 알려 주는 정보로만 다룰 수 있습니다. 타입 없는 매개변수가 있는 함수(아래의 암묵 제네릭)는 예외입니다.

바운드에 같은 트레잇을 매개변수만 달리해 두 번 적을 수 있습니다. 본체에서 트레잇을 쓸 때마다 인자 타입과 매개변수가 맞는 바운드가 선택됩니다. 연산자가 타입의 impl 가운데 하나를 고르는 것과 같은 방식입니다.

```rok
struct Meters { v: f64; }
impl Add<i32> for Meters { type Output = Meters; fn add(self, rhs: i32) -> Meters { Meters { v: self.v + rhs as f64 } } }
impl Add<f64> for Meters { type Output = Meters; fn add(self, rhs: f64) -> Meters { Meters { v: self.v + rhs } } }

fn nudge<T: Add<i32, Output = T> + Add<f64, Output = T>>(x: T) -> T {
    (x + 1) + 0.5          // Add<i32>, then Add<f64>
}

println(f"{nudge(Meters { v: 2.0 }).v}");
```

```output
3.5
```

## `where` 절

바운드가 길어지면 시그니처 끝의 `where` 절로 옮기세요. 읽기 쉬워지고 의미는 같습니다.

```rok
fn join_all<T>(xs: List<T>, sep: str) -> str
where
    T: Display,
{
    let out = StrBuf::new();
    for i in 0..xs.len() {
        if i > 0 { out.push_str(sep); }
        out.push_str(f"{xs[i]}");
    }
    out.to_str()
}

println(join_all([1, 2, 3], ", "));
println(join_all([true, false], " / "));
```

```output
1, 2, 3
true / false
```

## 제네릭 구조체와 impl

구조체와 enum도 타입 매개변수를 받을 수 있습니다. 메서드를 추가할 때는 `impl<A, B> Pair<A, B>`처럼 `impl`에도 매개변수를 선언합니다.

```rok
struct Pair<A, B> { first: A; second: B; }

impl<A, B> Pair<A, B> {
    fn swap(self) -> Pair<B, A> {
        Pair { first: self.second, second: self.first }
    }
}

let p = Pair { first: 1, second: "one" };
let q = p.swap();
println(f"{q.first} {q.second}");
```

```output
one 1
```

## 연관 타입

트레잇의 구현마다 타입 하나를 고르게 하려면 **연관 타입** `type Item;`을 선언합니다. 트레잇 안에서는 그냥 `Item`으로, 바깥의 제네릭 코드에서는 `C::Item`으로 씁니다. 표준 `Iterator`가 이렇게 동작합니다(다음 장).

```rok
trait Container {
    type Item;
    fn get(self, i: usize) -> Item | none;
    fn first(self) -> Item | none { self.get(0) }
}

struct Shelf { books: List<str>; }

impl Container for Shelf {
    type Item = str;
    fn get(self, i: usize) -> str? { self.books.get(i) }
}

fn first_of<C: Container>(c: C) -> C::Item | none { c.first() }

let s = Shelf { books: ["Dune", "Emma"] };
println(f"{first_of(s):?} {s.get(5):?}");
```

```output
"Dune" none
```

타입 매개변수 대신 연관 타입을 쓰는 이유는 구현하는 타입 하나에 `Item`이 정확히 한 타입으로 정해지기 때문입니다. 쓰는 쪽에서는 `Shelf`의 원소 타입을 적을 필요가 없습니다.

## 심화: const 제네릭

처음 읽을 때는 이 절을 건너뛰어도 됩니다. 매개변수는 타입 대신 컴파일 타임 **값**일 수도 있습니다. 고정 크기 배열 `[T; N]`의 길이처럼 값이 타입의 일부일 때 씁니다.

```rok
fn total<const N: usize>(a: [i32; N]) -> i32 {
    let s = 0;
    for i in 0..N { s += a[i]; }
    s
}

let a: [i32; 4] = [1, 2, 3, 4];
let z: [f64; 3] = [0.5; 3];
println(f"{total(a)} {a.len()} {z:?}");
```

```output
10 4 [0.5, 0.5, 0.5]
```

`const` 매개변수의 타입은 정수, `bool`, `char`만 될 수 있습니다.

## 제네릭과 `any`

제네릭은 정적입니다. 반대로 런타임에 값의 타입이 바뀔 수 있는 슬롯이 필요하면 `any`를 **직접** 적습니다. `any` 값은 쓰기 전에 `is`로 좁힙니다. 동적 디스패치 비용은 `any`를 쓴 곳에서만 생깁니다.

```rok
let v: any = 1;
println(f"{v is i32}");
v = "text";
if v is str {
    println(f"now a string, length {v.len()}");
}
```

```output
true
now a string, length 4
```

명세는 매개변수 타입을 생략한 함수 `fn show_sum(x, y)`를 `any`가 아니라 암묵 제네릭으로 정의합니다. 바운드는 본체가 매개변수를 쓰는 방식에서 추론하고, 함수는 호출 지점마다 단형화됩니다.

```rok
fn show_sum(x, y) { println(f"{x + y}"); }   // implicit generic: T: Add<U>

show_sum(1, 2);       // 3
show_sum(1.5, 2.0);   // 3.5
```

```output
3
3.5
```

## 정리

- 제네릭은 `fn f<T>(..)`, `struct S<A, B>`, `impl<A, B> S<A, B>`로 선언합니다. 타입 인자는 보통 추론됩니다.
- `T: Ord + Clone` 같은 바운드가 본체에서 쓸 수 있는 연산을 정합니다. 긴 바운드는 `where` 절로 옮깁니다.
- 연관 타입 `type Item;`은 구현마다 한 타입으로 정해지고, 바깥에서는 `C::Item`으로 씁니다.
- `const N: usize`는 값을 타입 매개변수로 받습니다.
- 제네릭은 정적이고(단형화), `any`는 직접 적은 곳에만 생기는 동적 슬롯입니다.

## 더 읽을거리

- [2.10. 제네릭](../spec/refs-lang/002-types/210-generics.html): [2.10.3. 트레잇 바운드](../spec/refs-lang/002-types/210-generics.html#2103-트레잇-바운드), [2.10.4. where 절](../spec/refs-lang/002-types/210-generics.html#2104-where-절), [2.10.9. 본체 검사](../spec/refs-lang/002-types/210-generics.html#2109-본체-검사), [2.10.7. 점진과의 결합](../spec/refs-lang/002-types/210-generics.html#2107-점진과의-결합)
- [2.8.3. 연관 타입](../spec/refs-lang/002-types/208-trait.html#283-연관-타입-associated-type)
- [2.11. `const` 제네릭](../spec/refs-lang/002-types/211-const-generics.html)
- [2.2. `any` 슬롯](../spec/refs-lang/002-types/202-any-slot.html)
- 다음 장: [8. 컬렉션과 반복자](08-collections-iterators.html)
