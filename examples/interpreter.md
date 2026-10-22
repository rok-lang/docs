# A small Lisp interpreter

Build a small Lisp with a tokenizer, a recursive-descent parser and an evaluator that carries an environment, supporting `define`, `if`, `lambda` and closures, with runtime errors reported through `throws`.

## Code

```rok
// A tiny Lisp: integers, define, if, lambda with closures.
enum Value {
    Num(i64),
    Bool(bool),
    Sym(str),
    List(List<Value>),
    Lambda(List<str>, List<Value>, Env),   // params, body, captured scope
}

// Scopes from outermost to innermost. Dicts are shared, so a closure
// sees definitions made after it was created (needed for recursion).
struct Env { frames: List<Dict<str, Value>>; }

struct LispError { msg: str; }
impl Display for LispError { fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.msg) } }
impl Debug for LispError { fn fmt(self, ref f: Formatter) throws (FmtError) { try f.write_str(self.msg) } }
impl Error for LispError {}
fn err(msg: str) -> LispError { LispError { msg: msg } }
fn tokenize(src: str) -> List<str> {
    let tokens: List<str> = [];
    for t in src.replace("(", " ( ").replace(")", " ) ").split_whitespace() { tokens.push(t); }
    tokens
}

fn parse(tokens: List<str>, ref pos: usize) throws (LispError) -> Value {
    if pos >= tokens.len() { throw err("unexpected end of input"); }
    let tok = tokens[pos];
    pos += 1;
    if tok == ")" { throw err("unexpected `)`"); }
    if tok != "(" {
        return match try? tok.parse::<i64>() {
            n: i64 => Value::Num(n),
            none => Value::Sym(tok),
        };
    }
    let items: List<Value> = [];
    while pos < tokens.len() && tokens[pos] != ")" {
        items.push(try parse(tokens, ref pos));
    }
    if pos >= tokens.len() { throw err("missing `)`"); }
    pos += 1;
    Value::List(items)
}

fn eval(x: Value, env: Env) throws (LispError) -> Value {
    let items = match x {
        Value::Sym(name) => {
            for i in (0..env.frames.len()).rev() {
                if let v: Value = env.frames[i].get(name) { return v; }
            }
            throw err(f"undefined symbol `{name}`");
        },
        Value::List(items) => items,
        _ => return x,    // numbers and booleans evaluate to themselves
    };
    if items.is_empty() { return x; }
    let head = match items[0] { Value::Sym(s) => s, _ => "" };
    match head {
        "define" => {
            let Value::Sym(name) = items[1] else { throw err("define needs a name"); };
            let v = try eval(items[2], env);
            env.frames[env.frames.len() - 1].insert(name, v);
            return v;
        },
        "if" => {
            let truthy = match try eval(items[1], env) { Value::Bool(b) => b, _ => true };
            return try eval(items[if truthy { 2 } else { 3 }], env);
        },
        "lambda" => {
            let params: List<str> = [];
            if let Value::List(ps) = items[1] {
                for p in ps {
                    match p { Value::Sym(s) => params.push(s), _ => { throw err("parameters must be symbols"); } }
                }
            }
            return Value::Lambda(params, [items[2]], env);
        },
        _ => {},
    }
    let args: List<Value> = [];
    for i in 1..items.len() { args.push(try eval(items[i], env)); }
    if head == "+" || head == "-" || head == "*" || head == "/" || head == "<" || head == "=" {
        return try arith(head, args);
    }
    try apply(try eval(items[0], env), args)
}

fn arith(op: str, args: List<Value>) throws (LispError) -> Value {
    let (a, b) = match (args[0], args[1]) {
        (Value::Num(a), Value::Num(b)) => (a, b),
        _ => { throw err(f"`{op}` needs two numbers"); },
    };
    match op {
        "+" => Value::Num(a + b),
        "-" => Value::Num(a - b),
        "*" => Value::Num(a * b),
        "/" => if b == 0 { throw err("division by zero"); } else { Value::Num(a / b) },
        "<" => Value::Bool(a < b),
        _ => Value::Bool(a == b),
    }
}

fn apply(f: Value, args: List<Value>) throws (LispError) -> Value {
    let Value::Lambda(params, body, captured) = f else { throw err(f"{show(f)} is not a function"); };
    if params.len() != args.len() { throw err(f"expected {params.len()} arguments, got {args.len()}"); }
    let frame: Dict<str, Value> = Dict::new();   // on top of the captured scopes
    for i in 0..params.len() { frame.insert(params[i], args[i]); }
    let frames: List<Dict<str, Value>> = [];
    for fr in captured.frames { frames.push(fr); }
    frames.push(frame);
    try eval(body[0], Env { frames: frames })
}

fn show(v: Value) -> str {
    match v {
        Value::Num(n) => f"{n}",
        Value::Bool(b) => if b { "#t" } else { "#f" },
        Value::Sym(s) => s,
        Value::List(items) => f"<list of {items.len()}>",
        Value::Lambda(params, _, _) => f"<lambda/{params.len()}>",
    }
}

let global = Env { frames: [Dict::new()] };
let program = [
    "(define square (lambda (x) (* x x)))",
    "(square 12)",
    "(define fact (lambda (n) (if (< n 2) 1 (* n (fact (- n 1))))))",
    "(fact 20)",
    "(define make-adder (lambda (k) (lambda (x) (+ x k))))",
    "(define add5 (make-adder 5))",
    "(add5 37)",
    "(/ 10 0)",
    "(undefined-thing 1)",
    "(5 3)",
    "(+ 1 (* 2 3)",
];

for src in program {
    let pos = 0usize;
    let result = try {
        show(try eval(try parse(tokenize(src), ref pos), global))
    } catch e: LispError {
        f"error: {e}"
    };
    println(f"{src}\n  => {result}");
}
```

```output
(define square (lambda (x) (* x x)))
  => <lambda/1>
(square 12)
  => 144
(define fact (lambda (n) (if (< n 2) 1 (* n (fact (- n 1))))))
  => <lambda/1>
(fact 20)
  => 2432902008176640000
(define make-adder (lambda (k) (lambda (x) (+ x k))))
  => <lambda/1>
(define add5 (make-adder 5))
  => <lambda/1>
(add5 37)
  => 42
(/ 10 0)
  => error: division by zero
(undefined-thing 1)
  => error: undefined symbol `undefined-thing`
(5 3)
  => error: 5 is not a function
(+ 1 (* 2 3)
  => error: missing `)`
```

## Walkthrough

- Programs and values share one enum, `Value`. Because Lisp code and data have the same shape, the parser produces `Value`s and the evaluator takes a `Value` and returns a `Value` ([Sum types](../learn/05-sum-types.html), [Structs, enums and traits](../learn/06-structs-enums-traits.html)).
- The parser shares its read position with the caller through `ref pos: usize`. It is a classic recursive-descent parser that calls itself to read inside parentheses ([4.7 the `ref` keyword](../spec/refs-lang/004-functions/407-ref-keyword.html)).
- Any token that isn't an integer is a symbol. `try? tok.parse::<i64>()` turns the parse error into `none`, so a two-armed `match` separates the cases ([3.14.3 `try?` and `try!`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3143-try-and-try)). Where only one variant is acceptable, a [let-else](../spec/refs-lang/003-expressions/316-let-else.html) such as `let Value::Sym(name) = items[1] else { throw ...; };` binds it or leaves the function.
- Runtime errors `throw` a `LispError`, and each function states this in its signature with `throws (LispError)`. The `try { ... } catch e: LispError { ... }` at the bottom turns one line's error into a message, so the next line still runs ([Errors](../learn/10-errors.html)).
- A `lambda` captures the `Env` it was created in. `Dict` is a shared reference, so the function returned by `make-adder` remembers `k`, and `fact` can find itself even though it is defined afterwards.

> [!NOTE]
> An enum that contains itself directly (`Lambda(..., Value, ...)`) would have infinite size, so the compiler rejects it; lambda bodies are kept in a `List<Value>` instead.
