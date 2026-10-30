# 작은 Lisp 인터프리터

토크나이저, 재귀 하강 파서, 환경을 들고 다니는 평가기로 작은 Lisp 를 만듭니다. `define`, `if`, `lambda`, 클로저를 지원하고, 런타임 에러는 `throws` 로 알립니다.

## 코드

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

## 설명

- 프로그램과 값은 enum `Value` 하나를 같이 씁니다. Lisp 는 코드와 데이터의 모양이 같으므로, 파서는 `Value` 를 만들고 평가기는 `Value` 를 받아 `Value` 를 돌려줍니다 ([합 타입](../learn/05-sum-types.html), [구조체, enum, 트레잇](../learn/06-structs-enums-traits.html)).
- 파서는 `ref pos: usize` 로 읽는 위치를 호출한 쪽과 공유합니다. 괄호 안을 읽을 때 자기 자신을 호출하는 전형적인 재귀 하강 파서입니다 ([4.7 `ref` 키워드](../spec/refs-lang/004-functions/407-ref-keyword.html)).
- 정수가 아닌 토큰은 모두 심볼입니다. `try? tok.parse::<i64>()` 가 파싱 에러를 `none` 으로 바꾸므로, 가지가 둘인 `match` 로 경우를 나눕니다 ([3.14.3 `try?` 와 `try!`](../spec/refs-lang/003-expressions/314-error-control-flow.html#3143-try-와-try)). 변형 하나만 받아들이는 자리에서는 `let Value::Sym(name) = items[1] else { throw ...; };` 같은 [let-else](../spec/refs-lang/003-expressions/316-let-else.html) 가 값을 바인딩하거나 함수를 빠져나갑니다.
- 런타임 에러는 `LispError` 를 `throw` 하고, 각 함수는 시그니처에 `throws (LispError)` 로 이를 밝힙니다. 아래쪽의 `try { ... } catch e: LispError { ... }` 가 한 줄의 에러를 메시지로 바꾸므로, 다음 줄은 계속 실행됩니다 ([에러 처리](../learn/10-errors.html)).
- `lambda` 는 자기가 만들어진 `Env` 를 캡처합니다. `Dict` 는 공유되는 참조이므로, `make-adder` 가 돌려준 함수는 `k` 를 기억하고, `fact` 는 나중에 정의되는데도 자기 자신을 찾을 수 있습니다.

> [!NOTE]
> 자기 자신을 직접 담는 enum (`Lambda(..., Value, ...)`) 은 크기가 무한해지므로 컴파일러가 거부합니다. 그래서 lambda 본체는 `List<Value>` 에 담습니다.
