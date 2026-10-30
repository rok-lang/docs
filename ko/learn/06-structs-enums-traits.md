# 6. 구조체, enum, 트레잇

이제 직접 타입을 정의할 차례입니다. 이 장에서는 데이터를 묶는 `struct`, 이름 붙은 경우를 나열하는 `enum`, 여러 타입이 공유하는 동작을 정의하는 `trait`을 다룹니다.

## 구조체와 필드 기본값

`struct`는 이름 붙은 필드의 묶음입니다. 필드는 `name: Type;`으로 쓰고, `= value`로 기본값을 줄 수 있습니다. 기본값이 있는 필드는 생성할 때 생략할 수 있습니다.

```rok
struct Config {
    name: str;
    retry: i32 = 3;
    verbose: bool = false;
}

let a = Config { name: "api" };
let b = Config { name: "db", retry: 5 };
println(f"{a.name} retry={a.retry} verbose={a.verbose}");
println(f"{b.name} retry={b.retry}");
```

```output
api retry=3 verbose=false
db retry=5
```

기본값으로는 리터럴이나 `const` 식만 쓸 수 있습니다. 런타임에 계산해야 하는 초기값은 `fn new() -> Config` 같은 생성자 함수에서 정합니다. 필드를 위치로 구별하는 tuple 구조체(`struct Rgb(u8, u8, u8);`)와 필드 없는 unit 구조체(`struct Marker;`)도 있습니다. 이 구조체들의 생성자는 값이므로 모듈에 같은 이름의 함수, `const`, `static`이 있으면 중복 정의가 됩니다.

## 메서드: `impl` 과 `self`

`impl Type { ... }` 블록 안의 함수는 그 타입의 메서드가 됩니다. 첫 매개변수가 `self`이면 인스턴스 메서드(`p.dist2(q)`)이고, 아니면 정적 메서드(`Point::origin()`)입니다. 필드를 바꾸는 메서드는 `ref self`를 받습니다.

```rok
struct Point { x: f64; y: f64; }

impl Point {
    fn origin() -> Point { Point { x: 0.0, y: 0.0 } }

    fn dist2(self, o: Point) -> f64 {
        let dx = self.x - o.x;
        let dy = self.y - o.y;
        dx * dx + dy * dy
    }

    fn shift(ref self, dx: f64) { self.x += dx; }
}

let p = Point { x: 3.0, y: 4.0 };
println(f"{p.dist2(Point::origin())}");
p.shift(1.5);
println(f"{p.x}");
```

```output
25
4.5
```

`ref self` 메서드에서 바꾼 내용은 호출한 쪽의 변수에 반영됩니다. 호출하는 곳에는 `ref`를 쓰지 않아도 됩니다. `ref`가 왜 필요한지는 다음 절에서 다룹니다.

## 구조체는 값입니다

구조체는 `i32`처럼 **값**입니다. 대입하거나, 함수에 넘기거나, 리스트(`ps[i]`)나 dict에서 읽어 오면 언제나 사본이 생깁니다. 사본을 바꿔도 원본은 그대로입니다.

```rok
struct Point { x: i32; y: i32; }

let a = Point { x: 1, y: 2 };
let b = a;              // a copy
b.x = 100;
println(f"{a.x} {b.x}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
let first = ps[0];      // also a copy
first.x = 99;
println(f"{ps[0].x}");
ps[0].x = 99;           // writes into the list
println(f"{ps[0].x}");
```

```output
1 100
1
99
```

`List`, `Dict`, `Set`은 반대입니다. 변수에 담긴 것은 공유되는 객체 하나를 가리키는 핸들이므로, 객체를 바꾸면 그 핸들의 모든 사본에서 바뀐 내용이 보입니다.

|                       | `struct`, `enum`, 튜플, 숫자 | `List`, `Dict`, `Set`, `Mutex`, ... |
|-----------------------|---------------------------------|-------------------------------------|
| `let b = a;`          | 독립된 사본                | 같은 객체                     |
| 필드 / 원소 쓰기 | 사본만 바뀜       | 모든 핸들에서 보임           |
| 별도 사본 만들기 | 대입 자체          | `.clone()`                          |

복사는 얕은 복사입니다. 구조체 필드에 `List`가 있으면 핸들만 복사되므로 구조체의 두 사본이 그 리스트를 공유합니다. `#[derive(Clone)]`(아래 참고)으로 만든 `clone()`은 리스트까지 복사합니다.

```rok
#[derive(Clone)]
struct Team { name: str; members: List<str>; }

let a = Team { name: "core", members: ["ada"] };
let b = a;
b.name = "docs";            // only b
b.members.push("grace");    // the list both share
println(f"{a.name} {a.members:?}");

let c = a.clone();
c.members.push("linus");
println(f"{a.members:?} {c.members:?}");
```

```output
core ["ada", "grace"]
["ada", "grace"] ["ada", "grace", "linus"]
```

### 호출한 쪽의 값 바꾸기: `ref`

함수는 구조체 인자의 사본을 받습니다. 호출한 쪽의 변수를 바꾸려면 매개변수를 `ref`로 선언하고 호출하는 곳에도 `ref`를 씁니다. 그러면 코드를 읽는 사람이 어떤 인자가 바뀔 수 있는지 알 수 있습니다.

```rok
struct Counter { n: i32; }

fn bump_copy(c: Counter) { c.n += 1; }    // changes its own copy
fn bump(ref c: Counter) { c.n += 1; }     // changes the caller's variable

let c = Counter { n: 0 };
bump_copy(c);
println(f"{c.n}");
bump(ref c);
println(f"{c.n}");
```

```output
0
1
```

메서드도 같습니다. `self`는 사본이고, `ref self`는 호출한 쪽의 값입니다(메서드는 호출하는 곳에 `ref`가 필요 없습니다). `ref self`를 빠뜨려도 컴파일은 되지만, 메서드가 반환하면 바꾼 내용이 사라집니다.

```rok
struct Counter { n: i32; }

impl Counter {
    fn inc_wrong(self) { self.n += 1; }    // changes a copy
    fn inc(ref self) { self.n += 1; }
}

let c = Counter { n: 0 };
c.inc_wrong();
println(f"{c.n}");
c.inc();
println(f"{c.n}");
```

```output
0
1
```

`write_to_copy` lint가 이 실수를 잡습니다. `rok check`와 `rok lint`(그리고 에디터)는 사본에 값을 쓴 뒤 아무도 읽지 않는 필드를 경고합니다.

```text check=checks/06-write-to-copy.rok
warning[E503]: `self.n` is set on a copy of the receiver, so the change is lost when `inc_wrong` returns
  --> main.rok:4:26
   |
 4 |     fn inc_wrong(self) { self.n += 1; }    // changes a copy
   |                          ^^^^^^
   |
   = help: take `ref self` to change the caller's value: `fn inc_wrong(ref self)`
   = note: `#[warn(write_to_copy)]` on by default
```

같은 함정은 루프에도 있습니다. `for p in ps { p.x *= 10; }`는 원소의 사본을 바꾸고, lint가 이를 알려 줍니다.

```rok norun
for p in ps { p.x *= 10; }
```

```text check=checks/06-loop-copy.rok
warning[E503]: `p.x` is set on a copy of the element, so `ps` does not change
  --> main.rok:4:15
   |
 4 | for p in ps { p.x *= 10; }
   |               ^^^
   |
   = help: loop over the indexes to change the elements: `for i in 0..ps.len() { ps[i].x *= 10; }`
```

원소를 제자리에서 바꾸려면 리스트에 인덱스로 접근하거나 새 리스트를 만듭니다.

```rok
struct Point { x: i32; y: i32; }

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for i in 0..ps.len() {
    ps[i].x *= 10;
}
println(f"{ps[0].x} {ps[1].x}");
```

```output
10 20
```

## enum: 이름 붙은 경우

`enum`은 여러 변형(variant) 중 하나인 값을 정의합니다. 변형에는 데이터가 없을 수도 있고(`Empty`), 위치 필드가 있을 수도 있고(`Circle(f64)`), 이름 있는 필드가 있을 수도 있습니다(`Rect { w: f64; h: f64; }`). 값은 `Shape::Circle(1.0)`처럼 만들고, `match`에서 같은 모양의 패턴으로 분해합니다.

```rok
enum Shape {
    Circle(f64),
    Rect { w: f64; h: f64; },
    Empty,
}

fn area(s: Shape) -> f64 {
    match s {
        Shape::Circle(r) => 3.0 * r * r,
        Shape::Rect { w, h } => w * h,
        Shape::Empty => 0.0,
    }
}

let shapes = [Shape::Circle(1.0), Shape::Rect { w: 2.0, h: 3.0 }, Shape::Empty];
for s in shapes {
    println(f"{area(s)}");
}
println(f"{shapes[0] is Shape::Circle}");
```

```output
3
6
0
true
```

`s is Shape::Circle`은 데이터와 관계없이 변형만 검사합니다.

패턴은 `Shape::Empty`처럼 enum과 함께 변형 이름을 씁니다. `Empty` 같은 맨 이름은 모든 값에 맞는 새 변수가 되므로 컴파일러가 거부합니다.

```text check=checks/06-bare-variant.rok
error[E200]: `Empty` here is a new variable that matches every value, not the variant `Shape::Empty`
   --> main.rok:11:9
    |
 11 |         Empty => 0.0,
    |         ^^^^^
    |
    = help: did you mean `Shape::Empty`?
```

합 타입과 다른 점은 이름입니다. 합 타입은 이미 있는 타입들을 이름 없이 묶고, `enum`은 각 경우에 의미 있는 이름을 붙입니다. 둘 다 `f64`인 "반지름"과 "변의 길이"를 구별해야 한다면 `enum`이 맞습니다.

## 트레잇과 기본 메서드

`trait`은 타입이 제공해야 하는 메서드의 목록입니다. 본체 없이 시그니처만 적은 메서드는 구현하는 쪽이 제공해야 하고, 본체가 있는 메서드는 기본 구현이 됩니다. 트레잇은 `impl Trait for Type`으로 구현합니다.

```rok
trait Animal {
    fn name(self) -> str;
    fn sound(self) -> str;
    fn speak(self) -> str { f"{self.name()}: {self.sound()}!" }
}

struct Dog;
struct Cat { lives: i32; }

impl Animal for Dog {
    fn name(self) -> str { "dog" }
    fn sound(self) -> str { "woof" }
}

impl Animal for Cat {
    fn name(self) -> str { "cat" }
    fn sound(self) -> str { "meow" }
    fn speak(self) -> str { f"cat ({self.lives} lives): meow" }
}

println(Dog.speak());
println(Cat { lives: 9 }.speak());
```

```output
dog: woof!
cat (9 lives): meow
```

## derive 로 자동 구현

비교, 해시, debug 출력처럼 기계적으로 쓸 수 있는 구현은 `#[derive(...)]`로 생성합니다. 내장 derive는 `Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`이고, 표준 라이브러리가 ROKON 직렬화용 `Rokon`을 더합니다([15장](15-rokon.html)).

```rok
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
struct Version { major: i32; minor: i32; }

let vs = [Version { major: 1, minor: 2 }, Version { major: 0, minor: 9 }];
vs.sort();
println(f"{vs:?}");

let seen: Set<Version> = Set::new();
seen.insert(vs[0].clone());
println(f"{seen.contains(Version { major: 0, minor: 9 })}");

let zero: Version = Default::default();
println(f"{zero == vs[0]}");
```

```output
[Version { major: 0, minor: 9 }, Version { major: 1, minor: 2 }]
true
false
```

`Ord`는 필드를 선언 순서대로 비교하므로 `sort`는 버전 순으로 정렬합니다. `Display`는 derive할 수 없습니다. 사람이 읽는 형식은 타입을 만든 사람이 정합니다(아래 연산자 절의 예 참고).

## 트레잇은 타입이 아닙니다

Rok에서는 트레잇 이름을 타입 자리에 그대로 쓸 수 없습니다. `fn f(a: Animal)`은 컴파일 에러입니다. 눈에 보이지 않는 동적 디스패치 비용을 막기 위해서입니다.

```text check=checks/06-trait-as-type.rok
error[E200]: trait `Animal` cannot be used as a type directly
  --> main.rok:2:1
   |
 2 | fn f(a: Animal) {}
   | ^^
   |
   = help: use `any Animal` for a value of any type implementing it, or a generic parameter `<T: Animal>`
```

대신 제네릭이나 합 타입을 씁니다.

```rok
trait Animal { fn sound(self) -> str; }
struct Dog;
struct Cat;
impl Animal for Dog { fn sound(self) -> str { "woof" } }
impl Animal for Cat { fn sound(self) -> str { "meow" } }

// 1. Open set of types: a generic with a bound (static dispatch).
fn twice<T: Animal>(a: T) -> str { f"{a.sound()} {a.sound()}" }

// 2. Closed set of types: a sum type.
fn loud(a: Dog | Cat) -> str {
    match a {
        d: Dog => d.sound().to_uppercase(),
        c: Cat => c.sound().to_uppercase(),
    }
}

println(twice(Dog));
println(loud(Cat));
```

```output
woof woof
MEOW
```

후보가 정해진 몇 개라면 합 타입을, 후보가 열려 있다면 제네릭(다음 장)을 씁니다.

> [!NOTE]
> 심화: 런타임에 고른 여러 구현을 한 컬렉션에 섞어야 하면 `let zoo: List<any Animal> = [Dog, Cat];`처럼 `any Trait`을 씁니다. 이렇게 담은 값의 메서드 호출은 동적 디스패치를 쓰고, 타입에 적힌 `any`가 그 비용을 드러냅니다([`any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-제약-붙은-any-any-trait)).

## 연산자 트레잇

`+`, `==`, `<` 같은 연산자는 모두 트레잇 메서드 호출입니다. `+`는 `Add`이고, 문자열 보간 `{x}`는 `Display`입니다. 그래서 이 트레잇들을 구현하면 직접 만든 타입에도 연산자를 쓰고 값을 출력할 수 있습니다.

```rok
struct V2 { x: i32; y: i32; }

impl Add for V2 {
    type Output = V2;
    fn add(self, o: V2) -> V2 { V2 { x: self.x + o.x, y: self.y + o.y } }
}

impl Display for V2 {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(f"({self.x}, {self.y})");
    }
}

let v = V2 { x: 1, y: 2 } + V2 { x: 10, y: 20 };
println(f"v = {v}");
```

```output
v = (11, 22)
```

`type Output = V2;`는 트레잇의 **연관 타입**을 정합니다. 다음 장에서 다시 봅니다.

`Display` 구현은 모양이 정해져 있어 그대로 복사해 쓰면 됩니다. `ref f`이므로 `fmt`는 호출한 쪽의 `Formatter`에 씁니다. 쓰기는 실패할 수 있으므로 `fmt`는 `throws (FmtError)`이고, 호출에는 `try`를 붙입니다. 에러는 [10장](10-errors.html)에서 다룹니다.

## 정리

- `struct` 필드는 `= value`로 기본값을 가질 수 있고, 메서드는 `impl` 블록에 둡니다.
- 구조체는 값입니다. 대입, 인자 전달, `list[i]`는 사본을 만듭니다(얕은 복사라서 `List` 필드는 공유됩니다). 호출한 쪽의 값을 바꾸려면 `ref` 매개변수(`f(ref x)`)나 `ref self`를 받습니다.
- `enum` 변형은 unit, tuple, named 세 모양이 있고 `match`로 분해합니다.
- `trait`에는 필수 메서드와 기본 메서드가 있고, `impl Trait for Type`으로 구현합니다.
- `#[derive(...)]`는 `Debug`, `Clone`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Default`(그리고 표준 라이브러리의 `Rokon`)를 자동으로 구현합니다.
- 트레잇은 타입이 아닙니다. 제네릭이나 합 타입(또는 심화로 `any Trait`)을 씁니다.
- 연산자와 `{x}` 출력도 트레잇 구현(`Add`, `Display` 등)으로 동작합니다.

## 더 읽을거리

- [2.3. `struct`](../spec/refs-lang/002-types/203-struct.html), [2.3.8. 값 의미](../spec/refs-lang/002-types/203-struct.html#238-값-의미), [4.7. `ref`](../spec/refs-lang/004-functions/407-ref-keyword.html), [2.4. `enum`](../spec/refs-lang/002-types/204-enum.html), [2.5. `union`](../spec/refs-lang/002-types/205-union.html)
- [2.8. `trait`](../spec/refs-lang/002-types/208-trait.html), [2.9. `impl`](../spec/refs-lang/002-types/209-impl.html), [4.9. 메서드](../spec/refs-lang/004-functions/409-method.html)
- [6.2.1. `#[derive(...)]`](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#621-derive-자동-트레잇-구현)
- [2.12. 트레잇-as-타입 금지](../spec/refs-lang/002-types/212-trait-as-type-banned.html), [2.2.10. `any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-제약-붙은-any-any-trait)
- [3.1. 연산자 디스패치](../spec/refs-lang/003-expressions/301-operator-dispatch.html), API: [`Add`](../../std/core/ops.html#trait.Add), [`Display`](../../std/core/fmt.html#trait.Display), [ROKON](../../std/std/rokon.html)
- 다음 장: [7. 제네릭](07-generics.html)
