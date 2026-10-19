# JSON parser

JSON values as an enum, a character-level recursive-descent parser that reports error positions with `throws`, and an indenting pretty printer.

## Code

```rok
// A small JSON parser: recursive descent over chars, errors with positions, pretty printer.

enum Json { Null, Bool(bool), Num(f64), Str(str), Arr(List<Json>), Obj(OrderedDict<str, Json>) }

struct ParseError { msg: str; pos: usize; }

struct Parser { cs: List<char>; pos: usize = 0; }

impl Parser {
    fn peek(self) -> char | none { self.cs.get(self.pos) }

    fn fail(self, msg: str) -> ParseError { ParseError { msg: msg, pos: self.pos } }

    fn skip_ws(ref self) {
        while self.peek()?.is_whitespace() ?? false { self.pos += 1; }
    }

    // Skip whitespace, then consume `c` if it is next.
    fn eat(ref self, c: char) -> bool {
        self.skip_ws();
        if self.peek() != c { return false; }
        self.pos += 1;
        true
    }

    fn expect(ref self, c: char) throws (ParseError) {
        if !self.eat(c) { throw self.fail(f"expected '{c}'"); }
    }

    fn value(ref self) throws (ParseError) -> Json {
        self.skip_ws();
        let c: char = self.peek() else { throw self.fail("unexpected end of input"); };
        match c {
            'n' => try self.word("null", Json::Null),
            't' => try self.word("true", Json::Bool(true)),
            'f' => try self.word("false", Json::Bool(false)),
            '"' => Json::Str(try self.string()),
            '[' => try self.array(),
            '{' => try self.object(),
            _ => if c == '-' || c.is_ascii_digit() { try self.number() } else { throw self.fail(f"unexpected '{c}'"); },
        }
    }

    fn word(ref self, w: str, v: Json) throws (ParseError) -> Json {
        for c in w.chars() {
            if self.peek() != c { throw self.fail(f"invalid literal, expected {w}"); }
            self.pos += 1;
        }
        v
    }

    fn number(ref self) throws (ParseError) -> Json {
        let start = self.pos;
        let sb = StrBuf::new();
        while let c: char = self.peek() {
            if !(c.is_ascii_digit() || "+-.eE".contains(f"{c}")) { break; }
            sb.push(c);
            self.pos += 1;
        }
        let text = sb.to_str();
        let n = try text.parse::<f64>() catch _ { throw ParseError { msg: f"bad number '{text}'", pos: start }; };
        Json::Num(n)
    }

    fn string(ref self) throws (ParseError) -> str {
        self.pos += 1; // opening quote
        let sb = StrBuf::new();
        loop {
            let c: char = self.peek() else { throw self.fail("unterminated string"); };
            self.pos += 1;
            if c == '"' { break; }
            if c != '\\' { sb.push(c); continue; }
            let e: char = self.peek() else { throw self.fail("unterminated string"); };
            self.pos += 1;
            match e {
                'n' => sb.push('\n'),
                't' => sb.push('\t'),
                '"' | '\\' | '/' => sb.push(e),
                _ => throw ParseError { msg: f"unknown escape '\\{e}'", pos: self.pos - 2 },
            }
        }
        sb.to_str()
    }

    fn array(ref self) throws (ParseError) -> Json {
        self.pos += 1;
        let items: List<Json> = [];
        if self.eat(']') { return Json::Arr(items); }
        loop {
            items.push(try self.value());
            if self.eat(']') { return Json::Arr(items); }
            try self.expect(',');
        }
    }

    fn object(ref self) throws (ParseError) -> Json {
        self.pos += 1;
        let fields: OrderedDict<str, Json> = OrderedDict::new();
        if self.eat('}') { return Json::Obj(fields); }
        loop {
            self.skip_ws();
            if self.peek() != '"' { throw self.fail("expected a string key"); }
            let key = try self.string();
            try self.expect(':');
            fields[key] = try self.value();
            if self.eat('}') { return Json::Obj(fields); }
            try self.expect(',');
        }
    }
}

fn parse(text: str) throws (ParseError) -> Json {
    let cs: List<char> = [];
    for c in text.chars() { cs.push(c); }
    let p = Parser { cs: cs };
    let v = try p.value();
    p.skip_ws();
    if p.pos < cs.len() { throw p.fail("trailing characters"); }
    v
}

// Render `parts` one per line between `open` and `close`.
fn block(open: str, close: str, parts: List<str>, depth: usize) -> str {
    if parts.is_empty() { return f"{open}{close}"; }
    let sb = StrBuf::new();
    sb.push_str(open);
    for (i, p) in parts.iter().enumerate() {
        let comma = if i + 1 < parts.len() { "," } else { "" };
        sb.push_str(f"\n{"  ".repeat(depth + 1)}{p}{comma}");
    }
    sb.push_str(f"\n{"  ".repeat(depth)}{close}");
    sb.to_str()
}

fn pretty(v: Json, depth: usize) -> str {
    match v {
        Json::Null => "null",
        Json::Bool(b) => f"{b}",
        Json::Num(n) => f"{n}",
        Json::Str(s) => f"{s:?}",
        Json::Arr(items) => {
            let parts: List<str> = [];
            for x in items { parts.push(pretty(x, depth + 1)); }
            block("[", "]", parts, depth)
        },
        Json::Obj(fields) => {
            let parts: List<str> = [];
            for (k, x) in fields.entries() { parts.push(f"{k:?}: {pretty(x, depth + 1)}"); }
            block("{", "}", parts, depth)
        },
    }
}

fn main() {
    let doc = "{\"name\": \"rok\", \"tags\": [\"fast\", \"safe\"], \"version\": 0.1, \"stars\": 1.2e3,
               \"stable\": false, \"deps\": {}, \"notes\": null, \"quote\": \"say \\\"hi\\\"\"}";
    let v = try parse(doc) catch e { println(f"error: {e.msg} at {e.pos}"); return; };
    println(pretty(v, 0));

    for bad in ["[1, 2,]", "{\"a\" 1}", "[tru]", "\"open", "[1] x", "-"] {
        try parse(bad) catch e { println(f"{bad:<9} -> {e.msg} at offset {e.pos}"); };
    }
}
```

```output
{
  "name": "rok",
  "tags": [
    "fast",
    "safe"
  ],
  "version": 0.1,
  "stars": 1200,
  "stable": false,
  "deps": {},
  "notes": null,
  "quote": "say \"hi\""
}
[1, 2,]   -> unexpected ']' at offset 6
{"a" 1}   -> expected ':' at offset 5
[tru]     -> invalid literal, expected true at offset 4
"open     -> unterminated string at offset 5
[1] x     -> trailing characters at offset 4
-         -> bad number '-' at offset 0
```

## Walkthrough

- A JSON value is the six-variant enum `Json`. Arrays and objects hold themselves recursively as `List<Json>` and `OrderedDict<str, Json>`; objects use `OrderedDict` to keep key order ([Sum types](../learn/05-sum-types.html), [Structs, enums and traits](../learn/06-structs-enums-traits.html)).
- The parser is a struct holding the position `pos`, and the methods that advance it take `ref self`. When such a method returns (or throws), the updated `self` is written back to the caller ([4.7.5 method receivers](../spec/refs-lang/004-functions/407-ref-keyword.html#475-exception-for-method-receivers)).
- End of input is checked with `let c: char = self.peek() else { throw ...; };`. `peek()` returns `char | none`, so `let-else` leaves on `none`, and below it `c` is a `char` ([3.16 `let-else`](../spec/refs-lang/003-expressions/316-let-else.html#3161-form)).
- Every error is a single `ParseError` carrying a message and a position. Errors thrown by the standard library, such as a failed number parse, are turned into positioned errors with `catch _ { throw ParseError { ... }; }` ([Errors](../learn/10-errors.html)).
- Output is built by the recursive function `pretty`, which returns a string. For arrays and objects each element is rendered first, then `block` adds indentation and commas; string values use `{s:?}` (Debug) for quotes and escapes ([Strings](../learn/09-strings.html)).
