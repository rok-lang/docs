# Big integers and factorials

Compute 100!, the 300th Fibonacci number and powers beyond 128 bits with the arbitrary-precision `int`, and find where `i64` overflows.

## Code

```rok
// n! as an arbitrary-precision `int`.
fn factorial(n: i64) -> int {
    let acc: int = 1;
    for k in 2..=n { acc = acc * int::from(k); }
    acc
}

// n! in i64: none once it overflows.
fn factorial_i64(n: i64) -> i64 | none {
    let acc = 1i64;
    for k in 2..=n {
        acc = acc.checked_mul(k)?;
    }
    acc
}

fn fib(n: i64) -> int {
    let a: int = 0;
    let b: int = 1;
    for _ in 0..n {
        let next = a + b;
        a = b;
        b = next;
    }
    a
}

fn digit_sum(x: int) -> i64 {
    let sum = 0i64;
    for c in f"{x}".chars() { sum += (c as i64) - ('0' as i64); }
    sum
}

let f100 = factorial(100);
let text = f"{f100}";
println(f"100! = {text[:20]}...{text[-10:]}");
println(f"digits: {text.len()}, digit sum: {digit_sum(f100)}");
println(f"bits: {f100.bit_length()}");

// Where does i64 give up?
for n in 19i64..=22 {
    match factorial_i64(n) {
        v: i64 => println(f"{n}! fits in i64: {v}"),
        none => println(f"{n}! overflows i64, int says {factorial(n)}"),
    }
}

println(f"fib(300) = {fib(300)}");
println(f"fib(300) % 1_000_000_007 = {fib(300) % int::from(1_000_000_007)}");

// Literals larger than i64 work directly in an int slot.
let big: int = 123_456_789_012_345_678_901_234_567_890;
println(f"big * big = {big * big}");
println(f"2^128 = {int::from(2).pow(128)}");
println(f"gcd(100!, 2^200) = 2^{int::from(2).pow(200).gcd(f100).bit_length() - 1}");
println(f"100! / 98! = {f100 / factorial(98)}");
```

```output
100! = 93326215443944152681...0000000000
digits: 158, digit sum: 648
bits: 525
19! fits in i64: 121645100408832000
20! fits in i64: 2432902008176640000
21! overflows i64, int says 51090942171709440000
22! overflows i64, int says 1124000727777607680000
fib(300) = 222232244629420445529739893461909967206666939096499764990979600
fib(300) % 1_000_000_007 = 644264086
big * big = 15241578753238836750495351562536198787501905199875019052100
2^128 = 340282366920938463463374607431768211456
gcd(100!, 2^200) = 2^97
100! / 98! = 9900
```

## Walkthrough

- `int` is an integer with no size limit. Small values are held in a single `i64` and switch to a digit array only when they overflow, so for small numbers it is about as fast as a fixed-width integer ([2.1.2 `int`](../spec/refs-standard-libraries/200-core/201-numbers.html#212-int---arbitrary-precision-integer), [`Int`](../std/core/num/bigint.html#struct.Int)).
- `int` and `i64` don't mix in arithmetic. You convert explicitly, as in `int::from(k)`, so the code shows at a glance which operations can overflow ([2.1.6 arithmetic result types](../spec/refs-standard-libraries/200-core/201-numbers.html#216-arithmetic-result-types)).
- On the `i64` side, `checked_mul` returns `none` on overflow. The trailing `?` returns that `none` straight out of the function, so `factorial_i64` reports its result as `i64 | none` ([Sum types](../learn/05-sum-types.html)).
- Integer literals beyond the `i64` range can be written directly in an `int` slot (`let big: int = 123_456_...`).
- `int` implements `Display`, so `f"{x}"` gives its decimal string, and the string slices `text[:20]` and `text[-10:]` show just the beginning and end ([Strings](../learn/09-strings.html)).
