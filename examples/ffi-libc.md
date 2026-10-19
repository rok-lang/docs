# Calling a C library

Declare libc's `strlen`, `getpid`, `getenv`, `snprintf` and `qsort` in an `extern "C"` block, and manage C memory directly with the pointer and allocation functions of `rok::bare`.

## Code

```rok
use rok::bare::alloc::{Layout, LayoutError, alloc, dealloc};
use rok::bare::ptr::{CPtr, ConstPtr, ManyPtr, Ptr};

extern "C" {
    fn strlen(s: str) -> usize;
    fn getpid() -> i32;
    fn setenv(name: str, value: str, overwrite: i32) -> i32;
    fn getenv(name: str) -> CPtr<u8>;       // NULL when unset
    fn snprintf(buf: Ptr<u8>, size: usize, fmt: str, ...) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: extern "C" fn(ConstPtr<i32>, ConstPtr<i32>) -> i32);
}

// A C string (NUL-terminated bytes) copied into a rok `str`.
fn from_c(p: ManyPtr<u8>) -> str {
    let bytes: List<u8> = [];
    let i = 0usize;
    loop {
        let b = unsafe { p.read(i) };
        if b == 0u8 { break; }
        bytes.push(b);
        i += 1;
    }
    try str::from_utf8(bytes) catch _ { "<invalid utf-8>" }
}

// getenv with the NULL case turned into `none`.
fn c_getenv(name: str) -> str | none {
    let p = unsafe { getenv(name) };
    let raw: Ptr<u8> = p.as_ptr()?;
    from_c(raw.as_many())
}

// The comparator qsort calls back: a rok function with the C ABI.
extern "C" fn descending(a: ConstPtr<i32>, b: ConstPtr<i32>) -> i32 {
    let (x, y) = unsafe { (a.read(), b.read()) };
    if x < y { 1 } else if x > y { -1 } else { 0 }
}

fn main() throws (LayoutError) {
    // str arguments arrive in C as NUL-terminated char*.
    println(f"strlen(\"hello, rok\") = {unsafe { strlen("hello, rok") }}");
    println(f"strlen(\"naïve\") = {unsafe { strlen("naïve") }} bytes");

    let pid = unsafe { getpid() };
    println(f"getpid() > 0: {pid > 0}, matches process::id(): {pid as u32 == rok::std::process::id()}");

    unsafe { setenv("ROK_FFI_DEMO", "set from C", 1) };
    println(f"ROK_FFI_DEMO = {c_getenv("ROK_FFI_DEMO") ?? "(unset)"}");
    println(f"ROK_FFI_MISSING = {c_getenv("ROK_FFI_MISSING") ?? "(unset)"}");

    // A variadic call writing into a buffer we allocate.
    let text_layout = try Layout::array::<u8>(64);
    let buf: Ptr<u8> = unsafe { alloc(text_layout) }.as_ptr() ?? panic("out of memory");
    let n = unsafe { snprintf(buf, 64, "%-6s|%5d|%08.3f", "pi", 42, 3.14159) };
    println(f"snprintf wrote {n} bytes: {from_c(buf.as_many())}");
    unsafe { dealloc(buf, text_layout) };

    // qsort over a C array of i32.
    let values: List<i32> = [42, 7, 19, 88, 3, 56, 21];
    let layout = try Layout::array::<i32>(values.len());
    let base: Ptr<u8> = unsafe { alloc(layout) }.as_ptr() ?? panic("out of memory");
    let arr: ManyPtr<i32> = base.cast::<i32>().as_many();
    for (i, v) in values.iter().enumerate() { unsafe { arr.write(i, v) }; }
    unsafe { qsort(base, values.len(), 4, descending) };
    let sorted: List<i32> = [];
    for i in 0..values.len() { sorted.push(unsafe { arr.read(i) }); }
    println(f"qsort descending: {sorted:?}");
    unsafe { dealloc(base, layout) };
}
```

```output
strlen("hello, rok") = 10
strlen("naïve") = 6 bytes
getpid() > 0: true, matches process::id(): true
ROK_FFI_DEMO = set from C
ROK_FFI_MISSING = (unset)
snprintf wrote 21 bytes: pi    |   42|0003.142
qsort descending: [88, 56, 42, 21, 19, 7, 3]
```

## Walkthrough

- An `extern "C" { ... }` block holds only signatures without bodies. Functions declared there are implicitly `unsafe`, so every call goes inside `unsafe { ... }` ([5.9 `extern` blocks](../spec/refs-lang/005-modules/509-extern-block.html), [4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html)).
- A `str` parameter is passed to C as a NUL-terminated `char*`. That is why `strlen("naïve")` returns 6, the UTF-8 byte count, not the 5 characters ([5.9.4 FFI-compatible types](../spec/refs-lang/005-modules/509-extern-block.html#594-ffi-compatible-types)).
- A C pointer that may be NULL is received as a [`CPtr`](../std/bare/ptr.html#struct.CPtr). `as_ptr()` gives `none` for NULL, so a single `?` wraps `getenv` into a safe function returning `str | none` ([Sum types](../learn/05-sum-types.html)).
- Memory is allocated by hand with [`Layout`](../std/bare/alloc.html#struct.Layout) and `alloc` / `dealloc`, and accessed like a C array through [`ManyPtr`](../std/bare/ptr.html#struct.ManyPtr)'s `read(i)` / `write(i, v)`. The GC does not manage it, so you free it yourself.
- `extern "C" fn descending` is a rok function compiled with the C ABI. Passed to `qsort` as a function pointer, it lets C call back into rok code while sorting ([5.9.10 defining `extern fn`](../spec/refs-lang/005-modules/509-extern-block.html#5910-extern-fn-definitions-exposed-externally)).
- Variadic functions such as `snprintf` are declared with `...` at the end of the signature. See the [FFI chapter](../learn/14-ffi.html) for more.
