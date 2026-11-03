# 21. Arrays and numerics

Numeric work in Rok has three layers. Plain numbers have the usual operators plus `**` for powers. `List<T>` and fixed arrays take the element-wise dot operators (`.+`, `.*`, ...) from [chapter 8](08-collections-iterators.html). For data with more than one axis, `rok::core::array::Array<T, N>` is an n-dimensional array with NumPy-style indexing, slicing and broadcasting, built into core so that every library that does linear algebra, statistics or plotting can share one type.

## Powers

`a ** b` raises `a` to `b`. It binds tighter than a prefix `-` and groups to the right, as in mathematics and Python: `-x ** 2` is `-(x ** 2)`, and `2 ** 3 ** 2` is `2 ** 9`. Parenthesize to raise a negative number.

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

An integer base takes a `u32` exponent and gives an integer of its own type; overflow panics like `*` does. A float base takes a float exponent, or an `i32` one, which uses repeated multiplication. The exponent does not have to be the base's type: `n ** 2` with `n: i64` is fine, and the literal becomes a `u32`. `2 ** -1` is a compile error, since the answer is a fraction; the error says to write `2.0 ** -1.0`. `**=` raises in place, and `.**` is the element-wise form.

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

`^` stays the bitwise XOR it is in C and Rust.

## Making arrays

`Array<T, N>` holds elements of type `T` along `N` axes; `N` is the rank, part of the type. `Vector<T>` and `Matrix<T>` name ranks 1 and 2. Where an `Array` is expected, a nested list literal builds one: the nesting depth is the rank and the rows give the shape.

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

The integer literals became `f64`s because the slot says `f64`. Rows of different lengths are a compile error (E0143): an array is a rectangle, so keep ragged data in a `List<List<T>>`.

The constructors take the shape as an array of lengths:

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

`Array::full(shape, v)` repeats one value, and `Array::from(xs)` turns a list into a rank-1 array.

## Indexing and slicing

Inside the brackets, each comma-separated part is one axis. A position picks one index along its axis and removes that axis; a slice `a:b:k` keeps the axis with the elements it selects, by the same rules as a list slice (negative bounds from the end, clamping, a negative step walks backwards). As many positions as the rank give one element.

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

Two more parts help with shapes. `..` stands for all the axes the other parts leave, so `a[.., 0]` is the first column of an array of any rank. `none` inserts an axis of length 1, which is how a vector becomes a row or a column.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
let v: Array<i64, 1> = [7, 8, 9];
println(f"{a[.., 0]:?} {v[none, :].shape():?} {v[:, none].shape():?}");
```

```output
[1, 4] [1, 3] [3, 1]
```

Positions can be any integer type; a `usize` loop counter works as well as a literal.

A slice, a position that leaves axes, `..`, `none`, `t()` and `reshape` all return *views*: new shapes and strides over the same elements, with no copy. Writing through a view writes the array.

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

Use `copy()` when you want an independent array. A boolean array of the same shape is a *mask*: `a[mask]` collects the elements where it is `true`, in order, as a new rank-1 array.

```rok
use rok::core::array::Array;

let a: Array<i64, 2> = [[1, 2, 3], [4, 5, 6]];
println(f"{a[a .% 2 .== 0]:?}");
```

```output
[2, 4, 6]
```

## Assigning to a selection

Anything you can index, you can assign to. A single value fills the selection; an array copies in, stretched to the selection's shape (see broadcasting below); a mask assignment changes only the selected elements.

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

A `List` takes slice assignment too, for a run of the same length: `xs[1:3] = [8, 9];`. A list slice with a step cannot be assigned (the compiler says so and suggests storing elements one at a time).

## Element-wise operations and broadcasting

The dot operators work on arrays element by element: arithmetic gives an array of the element type, comparisons an array of `bool`. A scalar on either side applies to every element.

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

Two arrays of the same rank must agree axis by axis, except that an axis of length 1 stretches to the other side's length. That is NumPy's rule, and it is the only one: it lives in core (`broadcast_shape`) so every array type agrees. Anything else panics and names both shapes.

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

An array of lower rank lines up with the last axes of the other, as if it had leading axes of length 1, so a vector adds to every row of a matrix. The result has the higher rank.

```rok
use rok::core::array::Array;

let m: Array<f64, 2> = [[1, 2], [3, 4]];
let v: Array<f64, 1> = [10, 20];
println(f"{m .+ v:?}");
```

```output
[[11.0, 22.0], [13.0, 24.0]]
```

A whole chain of dot operators runs as one loop with no intermediate arrays, on an `Array` as on a `List`: `a .* x .+ b .* 2.0` reads each element of `a`, `x` and `b` once and writes the result once.

## Updating in place

`y .= expr` writes the elements of `expr` into `y`, and `y .+= x`, `.-=`, `.*=`, `./=`, `.%=`, `.**=` combine `y` with `x` element by element. Nothing is allocated: the loop writes straight into `y`. The target can be a view, so a slice or a column of an array is updated where it sits.

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

The target keeps its shape: the right side stretches to it, never the other way. When the right side reads the target's own elements at other positions (`v .= v[::-1]`), it reads a copy taken first, so the result is what you would get by computing every element before storing any. Other operators are written with `.=`: `xs .= xs .%% 3`.

## Missing values

An element type `T | none` marks missing values. `a .?? b` fills each `none` with the matching element of `b`, or with `b` itself when it is a single value. NaN is not `none`: it is a float value and passes through. To treat NaN as missing, select it with `f64::is_nan.(a)` and assign.

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

## Reductions and loops

`sum()` adds every element, `map(f)` builds a new array of the same shape, `to_list()` gives the elements in row-major order (the last axis fastest), and `for x in a` visits them in that order.

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

## Memory and speed

An array's elements sit in one packed buffer. When the element type holds no references (numbers, `bool`, `char`, `#[derive(Pod)]` structs), the garbage collector never looks inside it, so even a huge array costs a collection nothing. A buffer of 8 KiB or more is aligned to 64 bytes and never moves while the array is reachable, which is what vector code and C libraries such as BLAS want: `unsafe { a.as_ptr() }` gives the first element, and `strides()` says how to walk the rest. `is_contiguous()` tells whether the elements are adjacent in row-major order; `copy()` makes them so.

## Your own array type

The indexing syntax is not reserved for core's `Array`. A bracket with several parts calls `Index` (reads) or `IndexSet` (writes) with a tuple, and a slice part arrives as a `rok::core::index::SliceArg`. If your type implements the marker trait `AxisIndexed`, every integer part arrives as an `isize`, `none` as `NewAxis` and `..` as `Rest`, so one impl covers every way of writing the index.

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

A package that defines a numeric type can also give it scalar-left operators. The orphan rule ([chapter 6](06-structs-enums-traits.html)) lets you implement a trait from core for a type from core when one of the trait's type arguments is yours, so `2.0 * v` works:

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

The rule still refuses `impl<T> Mul<V2> for T`, where a bare type parameter comes before your type: another package could write the same impl for its own type.

To make dot expressions over your type fuse like `Array`'s, implement `Broadcastable`, which shows its elements as an `Array`, and `Materialize`, which wraps a result when your type comes first. A type that keeps its data in an `Array` returns it:

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

## Literal suffixes

A package can give numbers a suffix of its own with `#[literal(number "suffix")]` on a handler function. When the handler is `pub`, every package that depends on yours gets the suffix too, without a `use`:

```rok norun
// in the `units` stone
pub struct Meters { pub v: f64; }

#[literal(number "m")]
pub fn meters(n: f64) -> Meters { Meters { v: n } }
```

```rok norun
// in a package with `"units": Dep::path("../units")`
let d = 9.81m;          // units::meters(9.81)
println(f"{d.v}");
```

A handler that is not `pub` works only inside its own package. Two dependencies that register the same suffix make its uses ambiguous; give one a `priority`.
