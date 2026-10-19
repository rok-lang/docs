# 14. FFI Basics

This chapter shows how to call functions written in C from rok, and how to hand rok functions to C. Operating systems and countless libraries expose their functionality through the C ABI, so knowing FFI (foreign function interface) lets you use features the standard library does not provide.

All examples really run against libc on macOS / Linux.

## `extern "C"` blocks

To use a C function, first declare its signature in an `extern "C" { ... }` block. There is no body; the declaration ends with `;`. The name is the C symbol name as is.

```rok
extern "C" {
    fn strlen(s: str) -> usize;
    fn abs(x: i32) -> i32;
    fn getpid() -> i32;
}

println(f"strlen: {unsafe { strlen("hello, C") }}");
println(f"abs: {unsafe { abs(-42) }}");
println(f"pid positive: {unsafe { getpid() } > 0}");
```

```output
strlen: 8
abs: 42
pid positive: true
```

Write C types as primitive types of the same size: `int` is `i32`, `size_t` is `usize`, `double` is `f64`. A `str` parameter is passed as a NUL-terminated `char*`; rok makes a NUL-terminated copy when needed.

`extern` blocks may only appear at module top level, and must name the ABI string (`"C"`).

## Why `unsafe` is required

The compiler has no way to check that a C function behaves as its signature says. Passing a bad pointer or getting the declaration wrong can corrupt memory. So every function in an `extern "C"` block is implicitly `unsafe` and can only be called inside `unsafe { ... }`.

```text
error[E200]: call to unsafe function `abs` outside an `unsafe` block or `unsafe fn` (spec 4.15; C functions are implicitly unsafe)
  --> main.rok:4:12
   |
 4 | println(f"{abs(-1)}");
   |            ^^^^^^^
```

An `unsafe` block says "I guarantee safety here." Good practice is to keep these blocks small and wrap dangerous calls in safe rok functions, so only a safe API is visible from outside.

## C characters and nullable pointers

A C `char` is one byte; in rok it is the `c_char` type with `c'A'` literals. It is a different type from the Unicode `char` (`'A'`), so code touching the FFI boundary stands out.

```rok
extern "C" {
    fn toupper(c: i32) -> i32;
}

let c: c_char = c'a';
let up = unsafe { toupper(c as i32) };
println(f"toupper: {up} = {up as u8 as char}");
```

```output
toupper: 65 = A
```

If a C function can return NULL, declare its return type as `CPtr<T>`. A `CPtr` is a pointer that can hold null, just like in C; `as_const()` extracts a non-null `ConstPtr<T> | none`. The example below reads the result of `getenv` byte by byte and copies it into a `str`. The dangerous part is confined to `from_c`, and only the safe `env_var` is exposed.

```rok
use rok::bare::ptr::{CPtr, ConstPtr};

extern "C" {
    fn getenv(name: str) -> CPtr<c_char>;
    fn setenv(name: str, value: str, overwrite: i32) -> i32;
    fn strerror(code: i32) -> str;
}

// Copy a NUL-terminated C string into a rok str.
unsafe fn from_c(p: ConstPtr<c_char>) -> str {
    let sb = StrBuf::new();
    let i: usize = 0;
    loop {
        let b = p.add(i).read();
        if b == c'\0' { break; }
        sb.push_byte(b as u8);
        i += 1;
    }
    sb.to_str()
}

fn env_var(name: str) -> str | none {
    let p = unsafe { getenv(name) }.as_const()?;
    unsafe { from_c(p) }
}

unsafe { setenv("ROK_FFI_DEMO", "hello", 1) };
println(env_var("ROK_FFI_DEMO") ?? "(unset)");
println(env_var("ROK_FFI_MISSING") ?? "(unset)");
println(unsafe { strerror(2) });
```

```output
hello
(unset)
No such file or directory
```

For a function that never returns NULL, such as `strerror`, declare the return type as `str` and rok measures the length with `strlen` and converts it to a string.

## Pointers and memory: `rok::bare::ptr`

Passing a buffer to C requires raw pointers. The `rok::bare` layer provides these types:

| Type | C equivalent | null |
|------|--------|------|
| `Ptr<T>`      | `T*`        | no (null is expressed as `none`) |
| `ConstPtr<T>` | `const T*`  | no |
| `CPtr<T>`     | `T*`        | yes |

The basic operations are `read()` / `write()` through the pointer, `add(n)` (advance by n elements), and `cast::<U>()`. Memory comes from the C heap via `rok::bare::alloc` and you return it yourself. The GC does not manage this memory, so forgetting `dealloc` leaks it.

```rok
use rok::bare::ptr::{Ptr, ConstPtr, size_of};
use rok::bare::alloc;

let xs: List<i32> = [10, 20, 30];
let p = unsafe { alloc::alloc_array::<i32>(xs.len()) }.as_ptr() ?? panic("out of memory");
for i in 0..xs.len() { unsafe { p.add(i).write(xs[i] * 2) }; }
let second = unsafe { p.add(1).read() };
println(f"second: {second}, i32 is {size_of::<i32>()} bytes");
unsafe { alloc::dealloc_array(p, xs.len()) };
```

```output
second: 40, i32 is 4 bytes
```

When exchanging structs with C, add `#[repr(C)]` so the field layout matches C.

```rok
#[repr(C)]
struct DivT { quot: i32; rem: i32; }

extern "C" {
    fn div(num: i32, den: i32) -> DivT;
}

let r = unsafe { div(17, 5) };
println(f"17 / 5 = {r.quot} rem {r.rem}");
```

```output
17 / 5 = 3 rem 2
```

## Callbacks: `extern "C" fn`

For C functions that take a function pointer, such as `qsort`, compile a rok function with the C ABI and pass it. Just put `extern "C"` before the function definition. Such a function is not a closure, so it cannot capture surrounding variables, and its parameters and return type must be FFI-compatible (integers, floats, pointers, `#[repr(C)]` structs).

```rok
use rok::bare::ptr::{Ptr, ConstPtr, size_of};
use rok::bare::alloc;

extern "C" {
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}

extern "C" fn by_value(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32 {
    let x = unsafe { a.cast::<i32>().read() };
    let y = unsafe { b.cast::<i32>().read() };
    if x < y { -1 } else if x > y { 1 } else { 0 }
}

let xs: List<i32> = [42, 7, 19, 3, 25];
let p = unsafe { alloc::alloc_array::<i32>(xs.len()) }.as_ptr() ?? panic("out of memory");
for i in 0..xs.len() { unsafe { p.add(i).write(xs[i]) }; }

unsafe { qsort(p.cast::<u8>(), xs.len(), size_of::<i32>(), by_value) };

let sorted: List<i32> = [];
for i in 0..xs.len() { sorted.push(unsafe { p.add(i).read() }); }
unsafe { alloc::dealloc_array(p, xs.len()) };
println(f"{sorted:?}");
```

```output
[3, 7, 19, 25, 42]
```

`str` is not an FFI type, so it cannot be a parameter of an `extern "C" fn`. The compiler tells you to use a `Ptr<u8>` and a length instead.

To let C find a rok function by name, set its symbol name with `#[export_name]`. The example below declares that symbol again as a C function in the same program and calls it.

```rok
#[export_name("rok_add")]
pub extern "C" fn add(a: i32, b: i32) -> i32 { a + b }

extern "C" {
    fn rok_add(a: i32, b: i32) -> i32;
}

println(f"{unsafe { rok_add(2, 3) }}");
println(f"{add(4, 5)}");
```

```output
5
9
```

## Linking libraries: `#[link]`

libc functions are linked without anything extra. For other libraries, put `#[link("name")]` on the `extern` block. This is the same as passing `-lname` to the linker.

```rok
#[link("m")]
extern "C" {
    fn sqrt(x: f64) -> f64;
    fn pow(x: f64, y: f64) -> f64;
}

println(f"{unsafe { sqrt(2.0) }:.4}");
println(f"{unsafe { pow(2.0, 10.0) }}");
```

```output
1.4142
1024
```

To select a static library, write `#[link(name = "foo", kind = "static")]`.

libc and libm are always linked. Any other library needs `#[link]`, like zlib below.

```rok
#[link("z")]
extern "C" {
    fn zlibVersion() -> str;
}
println(f"zlib {unsafe { zlibVersion() }}");
```

## Summary

- Declare C functions with `extern "C" { fn name(..) -> T; }` and call them inside `unsafe { }`.
- A C `char` is `c_char` / `c'A'`; a pointer that may be NULL is `CPtr<T>`.
- Handle C memory with `read` / `write` / `add` / `cast` on `Ptr` / `ConstPtr`, and with `rok::bare::alloc`.
- Structs use `#[repr(C)]`, callbacks use `extern "C" fn`, exported names use `#[export_name]`.
- Confine dangerous calls to small `unsafe` functions and expose safe functions to the outside.

## Further reading

- Spec: [5.9 extern blocks](../spec/refs-lang/005-modules/509-extern-block.html), [4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html), [6.2.3 the link attribute](../spec/refs-lang/006-meta-system/602-built-in-attributes.html#623-link-naming-an-external-library), [C-compatible characters](../spec/refs-lang/001-lexical/108-character-literals.html)
- Standard library spec: [FFI](../spec/refs-standard-libraries/100-bare/107-ffi.html), [Pointers](../spec/refs-standard-libraries/100-bare/101-ptr.html)
- API: [rok::bare::ptr](../std/bare/ptr.html), [rok::bare::alloc](../std/bare/alloc.html), [rok::bare::mem](../std/bare/mem.html)
