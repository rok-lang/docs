# 9. Strings and f-strings

Strings show up in almost every program. This chapter covers `str` and `StrBuf`, the commonly used methods, the different kinds of literals, and the format specifiers of the f-strings we have been using all along.

## `str`: immutable UTF-8

A `str` is a sequence of UTF-8 bytes and never changes once created. That is why `len()` returns the number of **bytes**, not characters. To count characters (code points), use `char_count()`.

```rok
let s = "hello, 세계";
println(f"{s.len()} bytes, {s.char_count()} chars");
println(s[0..5]);

for c in "가나".chars() {
    println(f"{c} = U+{c as u32:X}");
}
```

```output
13 bytes, 9 chars
hello
가 = U+AC00
나 = U+B098
```

Here the Korean text is deliberate: each Hangul syllable takes 3 bytes in UTF-8, so `"hello, 세계"` is 13 bytes but only 9 characters.

A slice `s[a..b]` also cuts at byte positions. Since a Hangul character is 3 bytes, you must not cut anywhere other than a character boundary; a cut inside a character panics and names the byte:

```rok build
let s = "hello, 세계";
println(s[0..8]);
```

```text
thread 'main' panicked at main.rok:2:9:
string slice is not on a char boundary: byte 8 is inside `세` (bytes 7..10)
string offsets count bytes; to work by character, use `chars()` or `char_indices()`
```

To work character by character, use `chars()` or `char_indices()`.

A single character is a `char`, written in single quotes. It holds one Unicode scalar value and has methods for classification and conversion.

```rok
let c = 'A';
println(f"{c} {c.is_ascii_uppercase()} {c.to_ascii_lowercase()}");
println(f"{'7'.is_ascii_digit()} {'세'.len_utf8()} {'\u{1F600}'}");
```

```output
A true a
true 3 😀
```

## Common methods

Methods on `str` do not modify the original; they return a new value. Operations that may find nothing return `T | none`, so they pair well with `??` and `if let` from chapter 5.

```rok
let line = "  name = rok  ";
let t = line.trim();
println(f"[{t}]");
println(f"{t.starts_with("name")} {t.contains("=")} {t.find("rok") ?? 99}");

if let (k, v): (str, str) = t.split_once(" = ") {
    println(f"key={k} value={v}");
}

let parts: List<str> = "a,b,,c".split(",").collect();
println(f"{parts:?}");
println(f"{"ab".repeat(3)} {t.to_uppercase()} {t.replace("rok", "ROK")}");

let n: i32 = try! "42".parse();
println(f"{n + 1}");
```

```output
[name = rok]
true true 7
key=name value=rok
["a", "b", "", "c"]
ababab NAME = ROK name = ROK
43
```

`parse` is a function that throws an error on failure, so we added `try!`. Errors are covered in the [next chapter](10-errors.html). Other methods include `lines()`, `split_whitespace()`, `strip_prefix()`, and `trim_start()` ([`str` API](../std/core/str.html#impl.str)).

## `StrBuf`: building piece by piece

Because `str` is immutable, concatenating with `+` in a loop creates a new string every time. When building a string piece by piece, write into a `StrBuf` buffer and take the result out with `to_str()` when done.

```rok
let b = StrBuf::new();
for i in 1..=5 {
    if !b.is_empty() { b.push_str(", "); }
    b.push_str(f"{i * i}");
}
b.push('!');
let s = b.to_str();
println(f"{s} ({s.len()} bytes)");
```

```output
1, 4, 9, 16, 25! (16 bytes)
```

For joining two strings once, `+` is enough. Unlike C or Python, however, adjacent string literals (`"abc" "def"`) are not joined automatically. Writing that is a compile error; if you want to join them, write `+` ([1.7.4](../spec/refs-lang/001-lexical/107-string-literals.html#174-automatic-concatenation-of-adjacent-strings)).

```rok
let greeting = "hello, " + "world";
println(greeting);
```

```output
hello, world
```

## Literals: escapes, raw strings, line continuation

Ordinary strings process escapes such as `\n`, `\t`, `\"`, `\\`, and `\u{..}`. For paths or regular expressions full of backslashes, raw strings `r"..."` are more convenient. If you need a `"` inside, surround the string with the same number of `#` on each side.

```rok
println("tab:\t| quote:\" | snow: \u{2603}");
println(r"C:\path\to\file\n");
println(r#"she said "hi""#);
println("one long \
         line");
```

```output
tab:	| quote:" | snow: ☃
C:\path\to\file\n
she said "hi"
one long line
```

As in the last example, a `\` at the end of a line removes both the newline and the leading whitespace of the next line. This rule does not apply in raw strings either ([1.7.2](../spec/refs-lang/001-lexical/107-string-literals.html#172-raw-strings)).

## Byte strings `b"..."`

Protocols and file formats speak bytes, not text. A byte string `b"..."` is a fixed array `[u8; N]` of its bytes. Write ASCII directly and anything else as `\xHH` (any value `\x00`-`\xFF`). Where a `Slice<u8>` is expected, the same literal is a view of that array, so it can be passed straight to functions that take bytes.

```rok
let magic = b"\x7FELF";                  // [u8; 4]
println(f"{magic.len()} {magic:?}");

fn starts_with(data: Slice<u8>, prefix: Slice<u8>) -> bool {
    data.len() >= prefix.len() && data[:prefix.len()] == prefix
}
println(f"{starts_with(b"\x7FELF\x02\x01", b"\x7FELF")}");

let req: Slice<u8> = b"GET / HTTP/1.1\r\n";
println(f"{req[:3]:?} {try! str::from_utf8(req[:3])}");
```

```output
4 [127, 69, 76, 70]
true
[71, 69, 84] GET
```

A non-ASCII character written directly in a byte string, or an escape above `\xFF`, is a compile error; line continuation is not supported.

## Embedding files: `include_str!` / `include_bytes!`

To ship a template, a test fixture or an icon inside the executable, embed the file at compile time. `include_str!("path")` is the file's contents as a `str`, and `include_bytes!("path")` its bytes as a `[u8; N]`, like a byte string. The path is relative to the source file that contains the call, and the result is a constant, so it can initialize a `const` or `static`.

```text
app/
└── src/
    ├── main.rok
    └── page.html        // <h1>{title}</h1>
```

```rok norun
// app/src/main.rok
const PAGE: str = include_str!("page.html");
static LOGO: [u8; 1406] = include_bytes!("logo.png");

print(PAGE.replace("{title}", "Hello"));
```

A missing file is a compile error, and so is a file that is not valid UTF-8 for `include_str!`. Editing the included file rebuilds what uses it.

## f-string holes

Any expression can go inside the `{ }` of an `f"..."`. The rules are the same as Python 3.12 (PEP 701): you can reuse the outer quote character inside a hole, and f-strings can be nested. To write a literal brace, use `{{` and `}}`.

```rok
let d = {"lang": "rok"};
let words = ["a", "b"];
println(f"{d["lang"]} has {words.len()} words: {words[0] + words[1]}");
println(f"nested: {f"[{1 + 1}]"}");
println(f"{{literal braces}}");
```

```output
rok has 2 words: ab
nested: [2]
{literal braces}
```

Inside a hole you do not use escapes like `\"`; just write the quotes as they are. For text or holes that span multiple lines, use `f"""..."""`.

## Format specifiers

Put a `:` after the expression in a hole to specify a format. The syntax is `[fill][align][sign][#][0][width][.precision][type]`.

```rok
let pi = 3.14159;
println(f"[{pi:.2}] [{pi:8.3}] [{pi:e}]");
println(f"[{42:>6}] [{42:<6}] [{42:^6}] [{42:*^6}]");
println(f"[{"ab":5}] [{"ab":>5}] [{7:03}] [{5:+}]");
println(f"{255:x} {255:#X} {5:b} {-1i8:x}");
println(f"{"hi":?} {[1, 2]:?}");
```

```output
[3.14] [   3.142] [3.14159e0]
[    42] [42    ] [  42  ] [**42**]
[ab   ] [   ab] [007] [+5]
ff 0xFF 101 ff
"hi" [1, 2]
```

Without an explicit alignment, numbers are right-aligned and everything else is left-aligned. `?` selects the `Debug` format, which shows strings with their quotes and escapes. Specifiers are checked against the hole's type at compile time, so a mistake like `{s:x}` (with `s: str`) is an immediate error.

To choose the width or precision at run time, put another hole inside the specifier. Both `{..}` and `${..}` work ([dynamic format specifiers](../spec/refs-lang/001-lexical/107-string-literals.html#dynamic-format-specifiers)).

```rok
let x = 2.0 / 3.0;
for prec in 1..=3 {
    let width = prec + 4;
    println(f"[{x:>${width}.${prec}f}] [{x:.{prec}f}]");
}
```

```output
[  0.7] [0.7]
[  0.67] [0.67]
[  0.667] [0.667]
```

## Printing your own types: `Display` and `Debug`

`f"{x}"` calls the `Display` trait and `f"{x:?}"` calls `Debug`. `Debug` can be implemented automatically with `#[derive(Debug)]`, while `Display`, which is meant for users, you implement yourself. `fmt` writes text into the `Formatter` it receives.

```rok
#[derive(Debug)]
struct Point { x: i32; y: i32; }

impl Display for Point {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(f"({self.x}, {self.y})");
    }
}

#[derive(Debug)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let p = Point { x: 1, y: -2 };
println(f"{p} | {p:?} | [{p:>10}]");
println(f"{[p, Point { x: 3, y: 4 }]:?}");
println(f"{Shape::Circle(1.5):?} {Shape::Rect { w: 2.5, h: 3.0 }:?}");
```

```output
(1, -2) | Point { x: 1, y: -2 } | [   (1, -2)]
[Point { x: 1, y: -2 }, Point { x: 3, y: 4 }]
Circle(1.5) Rect { w: 2.5, h: 3 }
```

Width and alignment (`{p:>10}`) apply to the whole output written by `fmt`. Once you implement `Display`, you can put the type directly anywhere in an f-string, and with `Debug`, a `List` or `Dict` holding that type also prints with `{:?}`. `3.0` prints as `3` because the default float output is "the shortest representation that reads back as the same value".

## Summary

- `str` is immutable UTF-8. `len()` and slicing work in bytes; handle characters with `chars()`.
- Build strings piece by piece in a `StrBuf` and take them out with `to_str()`. Literals are not joined automatically; join them with `+`.
- Raw strings `r"..."` and `r#"..."#` do not process escapes.
- `b"..."` is a `[u8; N]` (or a `Slice<u8>` where one is expected). `include_str!` / `include_bytes!` embed a file at compile time.
- An f-string hole takes any expression, including the same quote character as is. `{{` and `}}` are literal braces.
- Specifiers such as `:.2`, `:>8`, `:x`, and `:?` set the format, and `${..}` makes them dynamic.
- Implement `Display` for your own types and derive `Debug`.

## Further reading

- Spec: [String literals and f-strings](../spec/refs-lang/001-lexical/107-string-literals.html), [Character literals](../spec/refs-lang/001-lexical/108-character-literals.html), [Byte strings](../spec/refs-lang/001-lexical/107-string-literals.html#175-byte-strings), [File embedding](../spec/refs-lang/006-meta-system/611-include.html)
- Standard library spec: [fmt](../spec/refs-standard-libraries/200-core/210-fmt.html), [Strings](../spec/refs-standard-libraries/200-core/215-string.html)
- API: [`str` methods](../std/core/str.html#impl.str), [`StrBuf`](../std/core/str.html#struct.StrBuf), [`Display`](../std/core/fmt.html#trait.Display), [`Formatter`](../std/core/fmt.html#struct.Formatter)
