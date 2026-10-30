# 병렬 map / reduce

`rok::core::par`의 `map`, `reduce`, `filter`, `join`으로 소수 세기를 여러 코어에 나누고, 그 결과를 순차 계산과 맞춰 봅니다.

## 코드

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

## 설명

- `par::map`은 항목을 여러 덩어리로 나눠 스레드 풀에서 동시에 처리하지만, 결과 리스트는 입력 순서를 유지합니다. 그래서 `counts[i]`는 `starts[i]`에서 시작하는 구간의 답입니다([2.19.1 핵심 API](../spec/refs-standard-libraries/200-core/219-par.html#2191-핵심-api), [`par::map`](../../std/core/par.html#fn.map)).
- `par::reduce`는 먼저 덩어리마다 접고, 그다음 덩어리의 결과들을 접습니다. 묶는 방식을 바꿔도 같은 답이 나오도록 함수는 결합 법칙을 만족해야 합니다. 덧셈은 만족하고 뺄셈은 만족하지 않습니다([`par::reduce`](../../std/core/par.html#fn.reduce)).
- `par::filter`도 순서를 유지하므로, 백만 근처의 소수가 오름차순으로 나옵니다.
- `par::join`은 서로 관계없는 두 계산을 동시에 실행하고 두 결과를 튜플로 돌려줍니다([동시성](../learn/12-concurrency.html)).
- 병렬로 구한 답과 순차로 구한 답을 비교하는 것이 병렬화한 코드를 확인하는 가장 간단한 방법입니다. 걸린 시간은 기계마다 다르므로 출력에 넣지 않습니다.

## 실행하기

빨라진 정도를 보려면 최적화 빌드의 실행 시간을 잽니다.

```bash
rokc build --release par.rok -o par
time ./par
```
