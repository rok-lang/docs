# Parallel map / reduce

Split prime counting across cores with `map`, `reduce`, `filter` and `join` from `rok::core::par`, and check the result against a sequential count.

## Code

```rok
// Count primes in parallel with rok::core::par and check against a sequential count.

fn is_prime(n: i64) -> bool {
    if n < 2 { return false; }
    if n % 2 == 0 { return n == 2; }
    let d: i64 = 3;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 2;
    }
    true
}

fn count_primes(lo: i64, hi: i64) -> i64 {
    let c: i64 = 0;
    for n in lo..hi {
        if is_prime(n) { c += 1; }
    }
    c
}

fn main() {
    let limit: i64 = 2000000;
    let block: i64 = 250000;

    // 1. map: one block per item; results come back in input order.
    let starts: List<i64> = [];
    let s: i64 = 0;
    while s < limit { starts.push(s); s += block; }
    let counts: List<i64> = par::map(starts, |lo: i64| count_primes(lo, lo + block));
    for (i, c) in counts.iter().enumerate() {
        let lo = starts[i];
        println(f"[{lo:>7}, {lo + block:>7})  {c:>6} primes");
    }

    // 2. reduce: sum the per-block counts (the function must be associative).
    let total = par::reduce(counts, 0i64, |a: i64, b: i64| a + b);
    println(f"total below {limit}: {total}");

    // 3. filter: keep the primes of a smaller range, in order.
    let nums: List<i64> = [];
    for n in 999900i64..1000100i64 { nums.push(n); }
    let near: List<i64> = par::filter(nums, |n: i64| is_prime(n));
    println(f"primes around one million: {near:?}");

    // 4. join: two independent computations at once.
    let (twins, max_gap) = par::join(|| twin_primes(100000), || largest_gap(100000));
    println(f"twin prime pairs below 100000: {twins}, largest gap: {max_gap}");

    // The sequential answer must match the parallel one.
    let seq = count_primes(0, limit);
    println(f"sequential: {seq}, parallel: {total}, match: {seq == total}");
}

fn twin_primes(limit: i64) -> i64 {
    let c: i64 = 0;
    for n in 3i64..limit - 2 {
        if is_prime(n) && is_prime(n + 2) { c += 1; }
    }
    c
}

fn largest_gap(limit: i64) -> i64 {
    let prev: i64 = 2;
    let best: i64 = 0;
    for n in 3i64..limit {
        if is_prime(n) {
            if n - prev > best { best = n - prev; }
            prev = n;
        }
    }
    best
}
```

```output
[      0,  250000)   22044 primes
[ 250000,  500000)   19494 primes
[ 500000,  750000)   18700 primes
[ 750000, 1000000)   18260 primes
[1000000, 1250000)   17971 primes
[1250000, 1500000)   17686 primes
[1500000, 1750000)   17453 primes
[1750000, 2000000)   17325 primes
total below 2000000: 148933
primes around one million: [999907, 999917, 999931, 999953, 999959, 999961, 999979, 999983, 1000003, 1000033, 1000037, 1000039, 1000081, 1000099]
twin prime pairs below 100000: 1224, largest gap: 72
sequential: 148933, parallel: 148933, match: true
```

## Walkthrough

- `par::map` splits the items into chunks processed at the same time on the thread pool, but the result list keeps the input order, so `counts[i]` is the answer for the block starting at `starts[i]` ([2.19.1 core API](../spec/refs-standard-libraries/200-core/219-par.html#2191-core-api), [`par::map`](../std/core/par.html#fn.map)).
- `par::reduce` folds each chunk first and then folds the chunk results. The function must be associative so that regrouping gives the same answer: addition is, subtraction is not ([`par::reduce`](../std/core/par.html#fn.reduce)).
- `par::filter` keeps the order as well, so the primes around one million come out ascending.
- `par::join` runs two unrelated computations at once and returns both results as a tuple ([Concurrency](../learn/12-concurrency.html)).
- Comparing the parallel and sequential answers is the simplest way to check parallelized code. Timings vary from machine to machine, so they are not part of the output.

## Running it

To see the speedup, time an optimized build.

```bash
rokc build --release par.rok -o par
time ./par
```
