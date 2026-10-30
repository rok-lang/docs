# C 라이브러리 호출

libc 의 `strlen`, `getpid`, `getenv`, `snprintf`, `qsort` 를 `extern "C"` 블록에 선언하고, `rok::bare` 의 포인터 함수와 할당 함수로 C 메모리를 직접 관리합니다.

## 코드

```rok
use rok::bare::alloc::{Layout, LayoutError, alloc, dealloc};
use rok::bare::ptr::{CPtr, ConstPtr, ManyPtr, Ptr};

extern "C" {
    fn strlen(s: str) -> usize;
    fn getpid() -> i32;
    fn setenv(name: str, value: str, overwrite: i32) -> i32;
    fn getenv(name: str) -> CPtr<c_char>;   // NULL when unset
    fn snprintf(buf: Ptr<u8>, size: usize, fmt: str, ...) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: extern "C" fn(ConstPtr<i32>, ConstPtr<i32>) -> i32);
}

// A C string (NUL-terminated chars) copied into a rok `str`.
unsafe fn from_c(p: ConstPtr<c_char>) -> str {
    let bytes: List<u8> = [];
    let i = 0usize;
    loop {
        let b = p.add(i).read();
        if b == c'\0' { break; }
        bytes.push(b as u8);
        i += 1;
    }
    try str::from_utf8(bytes) catch _ { "<invalid utf-8>" }
}

// getenv with the NULL case turned into `none`.
fn c_getenv(name: str) -> str | none {
    let p = unsafe { getenv(name) }.as_const()?;
    unsafe { from_c(p) }
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
    println(f"snprintf wrote {n} bytes: {unsafe { from_c(buf.cast::<c_char>().as_const()) }}");
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

## 설명

- `extern "C" { ... }` 블록에는 본체 없는 시그니처만 들어갑니다. 여기에 선언한 함수는 묵시적으로 `unsafe` 이므로, 모든 호출은 `unsafe { ... }` 안에 둡니다 ([5.9 `extern` 블록](../spec/refs-lang/005-modules/509-extern-block.html), [4.15 unsafe](../spec/refs-lang/004-functions/415-unsafe.html)).
- `str` 매개변수는 NUL 로 끝나는 `char*` 로 C 에 전달됩니다. 그래서 `strlen("naïve")` 는 문자 수 5 가 아니라 UTF-8 바이트 수 6 을 돌려줍니다 ([5.9.4 FFI 호환 타입](../spec/refs-lang/005-modules/509-extern-block.html#594-ffi-호환-타입)).
- NULL 일 수 있는 C 포인터는 [`CPtr`](../../std/bare/ptr.html#struct.CPtr) 로 받습니다. `as_const()` 는 null 이 아닌 [`ConstPtr`](../../std/bare/ptr.html#struct.ConstPtr) 또는 `none` 을 주므로, `?` 하나로 `getenv` 를 `str | none` 을 돌려주는 안전한 함수로 감쌉니다 ([합 타입](../learn/05-sum-types.html)).
- 메모리는 [`Layout`](../../std/bare/alloc.html#struct.Layout) 과 `alloc` / `dealloc` 으로 직접 할당하고, [`ManyPtr`](../../std/bare/ptr.html#struct.ManyPtr) 의 `read(i)` / `write(i, v)` 로 C 배열처럼 접근합니다. GC 가 관리하지 않으므로 직접 해제해야 합니다.
- `extern "C" fn descending` 은 C ABI 로 컴파일되는 Rok 함수입니다. 함수 포인터로 `qsort` 에 넘기면, C 가 정렬하는 동안 Rok 코드를 다시 호출할 수 있습니다 ([5.9.10 `extern fn` 정의](../spec/refs-lang/005-modules/509-extern-block.html#5910-extern-fn-정의-외부에-노출)).
- `snprintf` 같은 가변 인자 함수는 시그니처 끝에 `...` 를 붙여 선언합니다. 자세한 내용은 [FFI 장](../learn/14-ffi.html)을 보십시오.
