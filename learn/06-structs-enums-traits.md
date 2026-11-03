# 6. Structs, Enums, and Traits

Now it's time to define your own types. This chapter covers `struct`, which groups data; `enum`, which lists named cases; and `trait`, which defines behavior shared by several types.

## Structs and field defaults

A `struct` is a group of named fields. Fields are written `name: Type;` and can be given a default with `= value`. Fields with defaults may be omitted at construction.

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

Defaults may only be literals or `const` expressions. Initial values that must be computed at runtime go in a constructor function such as `fn new() -> Config`. There are also tuple structs, whose fields are identified by position (`struct Rgb(u8, u8, u8);`), and field-less unit structs (`struct Marker;`). Their constructors are values, so a function, `const` or `static` of the same name in the module is a duplicate definition.

## Methods: `impl` and `self`

Functions inside an `impl Type { ... }` block become methods of that type. If the first parameter is `self`, it is an instance method (`p.dist2(q)`); otherwise it is a static method (`Point::origin()`). Methods that modify fields take `ref self`.

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

Changes made by a `ref self` method are reflected in the caller's variable. You don't need to write `ref` at the call site. Why `ref` is needed at all is the subject of the next section.

## Structs are values

A struct is a **value**, like an `i32`. Assigning it, passing it to a function, and reading it out of a list (`ps[i]`) or dict all make a copy, and writing to the copy leaves the original alone.

```rok
struct Point { x: i32; y: i32; }

let a = Point { x: 1, y: 2 };
let b = a;              // a copy
b.x = 100;
println(f"{a.x} {b.x}");

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
let first = ps[0];      // also a copy
first.x = 99;
println(f"{ps[0].x} {first.x}");
ps[0].x = 99;           // writes into the list
println(f"{ps[0].x}");
```

```output
1 100
1 99
99
```

A `List`, `Dict`, or `Set` is the opposite: the variable is a handle to one shared object, so every copy of the handle sees the change.

|                       | `struct`, `enum`, tuple, number | `List`, `Dict`, `Set`, `Mutex`, ... |
|-----------------------|---------------------------------|-------------------------------------|
| `let b = a;`          | independent copy                | the same object                     |
| writing a field / element | changes only the copy       | seen through every handle           |
| making a separate copy | the assignment itself          | `.clone()`                          |

The copy is shallow. A struct field that holds a `List` copies the handle, so two copies of the struct share that list. `#[derive(Clone)]` (see below) gives a `clone()` that copies the list too.

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

### Changing the caller's value: `ref`

A function gets a copy of its struct arguments. To change the caller's variable, declare the parameter `ref` and write `ref` at the call site as well, so the reader can see which arguments may change.

```rok
struct Counter { n: i32; }

fn bumped(c: Counter) -> i32 { c.n += 1; c.n }    // changes its own copy
fn bump(ref c: Counter) { c.n += 1; }             // changes the caller's variable

let c = Counter { n: 0 };
println(f"{bumped(c)} {c.n}");
bump(ref c);
println(f"{c.n}");
```

```output
1 0
1
```

A method works the same way: `self` is a copy, and `ref self` is the caller's value (for methods, the call site needs no `ref`). A field set on a copy that nothing reads afterwards would be lost, so forgetting `ref self` is an error:

```rok norun
struct Counter { n: i32; }

impl Counter {
    fn inc(self) { self.n += 1; }
}
```

```text check=checks/06-write-to-copy.rok
error[E0057]: `self.n` is set on a copy of the receiver, so the change is lost when `inc` returns
  --> main.rok:4:20
   |
 4 |     fn inc(self) { self.n += 1; }
   |                    ^^^^^^
   |
   = help: take `ref self` to change the caller's value: `fn inc(ref self)`; to change a local copy, write `let c = self;` and change `c`
```

Take `ref self`, as the help says:

```rok
struct Counter { n: i32; }

impl Counter {
    fn inc(ref self) { self.n += 1; }
}

let c = Counter { n: 0 };
c.inc();
c.inc();
println(f"{c.n}");
```

```output
2
```

The same goes for loops. `for p in ps` binds a copy of each element, so `p.x *= 10` would change copies only:

```rok norun
for p in ps { p.x *= 10; }
```

```text check=checks/06-loop-copy.rok
error[E0057]: `p.x` is set on a copy of the element, so `ps` does not change
  --> main.rok:4:15
   |
 4 | for p in ps { p.x *= 10; }
   |               ^^^
   |
   = help: bind each element as a place with `for ref`: `for ref p in ps { p.x *= 10; }`
```

`for ref p in ps` binds each element as a place: reading `p` reads the element, and writing `p` or its fields writes the element. A `ref self` method called on `p` (or on `ps[i]`, or a `Dict` entry) changes it in place too.

```rok
struct Point { x: i32; y: i32; }

let ps = [Point { x: 1, y: 0 }, Point { x: 2, y: 0 }];
for ref p in ps {
    p.x *= 10;
}
println(f"{ps[0].x} {ps[1].x}");
```

```output
10 20
```

A `for ref` loop cannot resize the list it walks: `ps.push(..)` in its body is an error ([E0138](../errors/E0138.html)). Change the length after the loop.

## enum: named cases

An `enum` defines a value that is one of several variants. A variant may carry no data (`Empty`), positional fields (`Circle(f64)`), or named fields (`Rect { w: f64; h: f64; }`). Construct values like `Shape::Circle(1.0)` and destructure them with patterns of the same shape in `match`.

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

`s is Shape::Circle` tests only the variant, regardless of the payload.

Where the expected type is the enum, the variant can be written bare: the arms of a `match` on a `Shape`, an argument for a `Shape` parameter, a `let` with the type `Shape`, a `return` value, a field in a struct literal, and the right side of `s == ..`. This works for every enum, the standard library's too (`let o: Ordering = Less;`).

```rok
enum Shape {
    Circle(f64),
    Rect { w: f64; h: f64; },
    Empty,
}

fn sides(s: Shape) -> i32 {
    match s {
        Circle(_) => 0,
        Rect { .. } => 4,
        Empty => 0,
    }
}

println(f"{sides(Rect { w: 1.0, h: 2.0 })} {sides(Empty)}");
```

```output
4 0
```

With no expected type, as in `let s = Empty;`, write the full path, `Shape::Empty`; the error says so. A name you define yourself always wins over a variant.

How is this different from a sum type? A sum type groups existing types without names, while an `enum` gives each case a meaningful name. If you need to tell a "radius" from a "side length" even though both are `f64`, an `enum` is the right choice.

## Traits and default methods

A `trait` is a list of methods a type must provide. Methods written as a signature without a body must be supplied by the implementor; methods with a body serve as default implementations. Implement a trait with `impl Trait for Type`.

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

A trait's methods are found only where the trait is in scope: defined in the same module, brought in with `use`, or one of the standard library's `rok::` traits. A method of a trait from another module needs a `use` of the trait; without it the compiler says which one to add:

```rok norun
mod zoo {
    pub trait Animal { fn name(self) -> str; }
    pub struct Dog;
    impl Animal for Dog { fn name(self) -> str { "dog" } }
}

use zoo::Animal;              // without this, `zoo::Dog.name()` is an error

println(zoo::Dog.name());
```

```text
error[E0136]: method `name` comes from trait `zoo::Animal`, which is not in scope
   = help: add `use zoo::Animal;`
```

A trait whose impls promise something the compiler cannot check is declared `unsafe trait`, and each impl of it is written `unsafe impl`. The standard library's `Pod` is one; [chapter 14](14-ffi.html) shows both.

## Automatic implementations with derive

Mechanical implementations such as comparison, hashing, and debug output are generated with `#[derive(...)]`. The built-in derives are `Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, and `Hash`; the standard library adds `Rokon` for ROKON serialization ([chapter 15](15-rokon.html)).

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

`Ord` compares fields in declaration order, so `sort` orders by version. `Display` cannot be derived: the type's author decides its human-readable format (see the example in the operators section below).

When equality is hand-written, `Hash` must be too, and it must feed the hasher exactly what equality looks at, so that equal values hash alike. A `Hash` impl does not compute a number; it writes its parts to the `Hasher` it is given. `Hasher` is not in the prelude, so the impl imports it from `rok::core::hash`:

```rok
use rok::core::hash::Hasher;

struct Tag { name: str; }

impl PartialEq for Tag {
    fn eq(self, o: Tag) -> bool { self.name.to_lowercase() == o.name.to_lowercase() }
    fn ne(self, o: Tag) -> bool { !self.eq(o) }
}
impl Eq for Tag {}
impl Hash for Tag {
    fn hash<H: Hasher>(self, ref h: H) { h.write_str(self.name.to_lowercase()); }
}

let tags: Set<Tag> = Set::new();
tags.insert(Tag { name: "Rok" });
println(f"{tags.contains(Tag { name: "ROK" })}");
```

```output
true
```

The hasher `Dict` and `Set` use is keyed per process, so hash values differ from run to run; never store one.

The standard library's own types follow the same rules. Every public type implements `Debug`, so `{x:?}` works on a range (`0..3`), a `File` (`File { path: "a.txt" }`) or an iterator; a type holding a key prints `{ .. }` instead of the secret. Value types such as `Path`, `Ipv4Addr` and `Ordering` implement `Eq` and `Hash`, so they work as `Dict` keys and `Set` elements. A std enum that may gain cases later, such as `IoErrorKind` or `HashAlgorithm`, is `#[non_exhaustive]`: a `match` on it needs a `_` arm.

## Traits are not types

In Rok, a trait name cannot be used directly in type position. `fn f(a: Animal)` is a compile error. This prevents hidden dynamic dispatch costs.

```text check=checks/06-trait-as-type.rok
error[E0071]: trait `Animal` cannot be used as a type directly
  --> main.rok:2:9
   |
 2 | fn f(a: Animal) {}
   |         ^^^^^^
   |
   = help: use `any Animal` for a value of any type implementing it, or a generic parameter `<T: Animal>`
```

Instead, use a generic or a sum type.

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

If the candidates are a fixed few, use a sum type; if the set is open, use generics (next chapter).

> [!NOTE]
> Advanced: when one collection must mix several implementations chosen at run time, write `any Trait`, as in `let zoo: List<any Animal> = [Dog, Cat];`. Calls through it use dynamic dispatch, and the `any` in the type makes that cost visible. A method with type parameters of its own, such as `fn write<B: AsBytes>(..)`, can be called through it too ([`any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-constrained-any-any-trait)).

## Operator traits

Operators such as `+`, `==`, and `<` are all trait method calls. `+` is `Add`, and string interpolation `{x}` is `Display`. So implementing these traits gives your own types operators and printing. The operator traits live in the module `ops`, so an impl names `ops::Add`; the operators themselves work everywhere without an import.

```rok
struct V2 { x: i32; y: i32; }

impl ops::Add for V2 {
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

`type Output = V2;` sets the trait's **associated type**. You will see these again in the next chapter.

The `Display` impl has a fixed shape you can copy: `ref f` lets `fmt` write into the caller's `Formatter`, and writing can fail, so `fmt` is `throws (FmtError)` and the call is marked with `try`. Errors are the subject of [chapter 10](10-errors.html).

## Summary

- `struct` fields can have defaults via `= value`, and methods live in `impl` blocks.
- A struct is a value: assignment, arguments, and `list[i]` copy it (shallowly; a `List` field stays shared). To change the caller's value, take a `ref` parameter (`f(ref x)`) or `ref self`.
- `enum` variants come in three shapes, unit, tuple, and named, and are destructured with `match`.
- A `trait` has required and default methods and is implemented with `impl Trait for Type`.
- `#[derive(...)]` automatically implements `Debug`, `Clone`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, and `Default` (and `Rokon`, from the standard library).
- Traits are not types. Use a generic or a sum type (or, advanced, `any Trait`).
- Operators and `{x}` printing also come from implementing traits (`Add`, `Display`, and so on).

## Further reading

- [2.3. `struct`](../spec/refs-lang/002-types/203-struct.html), [2.3.8. Value semantics](../spec/refs-lang/002-types/203-struct.html#238-value-semantics), [4.7. `ref`](../spec/refs-lang/004-functions/407-ref-keyword.html), [2.4. `enum`](../spec/refs-lang/002-types/204-enum.html), [2.5. `union`](../spec/refs-lang/002-types/205-union.html)
- [2.8. `trait`](../spec/refs-lang/002-types/208-trait.html), [2.9. `impl`](../spec/refs-lang/002-types/209-impl.html), [4.9. Methods](../spec/refs-lang/004-functions/409-method.html)
- [6.2.1. `#[derive(...)]`](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#621-derive-automatic-trait-implementation)
- [2.12. Trait-as-type ban](../spec/refs-lang/002-types/212-trait-as-type-banned.html), [2.2.10. `any Trait`](../spec/refs-lang/002-types/202-any-slot.html#2210-constrained-any-any-trait)
- [3.1. Operator dispatch](../spec/refs-lang/003-expressions/301-operator-dispatch.html), API: [`Add`](../std/core/ops.html#trait.Add), [`Display`](../std/core/fmt.html#trait.Display), [ROKON](../std/std/rokon.html)
- Next chapter: [7. Generics](07-generics.html)
