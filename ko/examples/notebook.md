# Jupyter 노트북으로 데이터 탐색

Jupyter 에 Rok 커널을 설치하고 온도 데이터를 셀 단위로 탐색합니다. 같은 작업을 하는 실행 가능한 프로그램 하나도 함께 실었습니다.

노트북에서는 한 단계씩 작업합니다. 한 셀은 데이터를 읽고, 다른 셀은 함수를 정의하고, 또 다른 셀은 결과를 확인합니다. 아래 프로그램은 그 셀들을 위에서 아래로 순서대로 이어 붙인 것입니다. 4주 동안의 일 최고 기온에서 요약 통계, 주별 평균, 텍스트 히스토그램을 계산합니다.

## 코드

```rok
// Daily high temperatures (°C) for four weeks.
let temps: List<f64> = [
    12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
    19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
    20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
    22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8,
];

fn mean(xs: List<f64>) -> f64 {
    let sum = 0.0;
    for x in xs { sum += x; }
    sum / xs.len() as f64
}

fn std_dev(xs: List<f64>) -> f64 {
    let m = mean(xs);
    let sq = 0.0;
    for x in xs { sq += (x - m) * (x - m); }
    (sq / xs.len() as f64).sqrt()
}

fn median(xs: List<f64>) -> f64 {
    let sorted: List<f64> = [];
    for x in xs { sorted.push(x); }
    sorted.sort_by(|a, b| a.partial_cmp(b) ?? Ordering::Equal);
    let n = sorted.len();
    if n % 2 == 1 { sorted[n / 2] } else { (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0 }
}

// Summary statistics.
let lo = temps.iter().fold(f64::INFINITY, |a, x| if x < a { x } else { a });
let hi = temps.iter().fold(f64::NEG_INFINITY, |a, x| if x > a { x } else { a });
println(f"n      = {temps.len()}");
println(f"mean   = {mean(temps):.2}");
println(f"median = {median(temps):.2}");
println(f"stddev = {std_dev(temps):.2}");
println(f"range  = {lo:.1} .. {hi:.1}");

// Weekly averages.
for week in 0..4 {
    let days = temps[week * 7:(week + 1) * 7];
    let w: List<f64> = days.iter().collect();
    println(f"week {week + 1}: {mean(w):5.2}");
}

// An ASCII histogram with 2-degree buckets.
let start = 12.0;
let width = 2.0;
let counts: List<usize> = [];
for _ in 0..8 { counts.push(0); }
for t in temps {
    let b = ((t - start) / width) as usize;
    counts[b] = counts[b] + 1;
}
println("histogram:");
for (i, c) in counts.iter().enumerate() {
    let from = start + width * i as f64;
    println(f"  {from:4.1}-{from + width:4.1} | {"#".repeat(c)} {c}");
}
```

```output
n      = 28
mean   = 20.16
median = 20.65
stddev = 3.49
range  = 12.5 .. 26.0
week 1: 15.41
week 2: 21.83
week 3: 23.04
week 4: 20.36
histogram:
  12.0-14.0 | ## 2
  14.0-16.0 | ## 2
  16.0-18.0 | ### 3
  18.0-20.0 | ##### 5
  20.0-22.0 | ###### 6
  22.0-24.0 | ###### 6
  24.0-26.0 | ### 3
  26.0-28.0 | # 1
```

## 설명

- 노트북 셀의 최상위 `let` 과 `fn` 은 세션 내내 유지됩니다. 일반 소스 파일과 달리 셀은 최상위에서 같은 이름을 다시 `let` 할 수 있으므로, 다른 데이터로 셀을 다시 실행할 수 있습니다 ([5.1.4 같은 스코프 내 재선언 금지](../spec/refs-lang/005-modules/501-variable-declaration.html#514-같은-스코프-내-재선언-금지)).
- `f64` 는 전순서가 아니므로 `sort_by` 에 `partial_cmp` 를 넘기고, 비교할 수 없는 NaN 은 `?? Ordering::Equal` 로 처리합니다 ([컬렉션과 반복자](../learn/08-collections-iterators.html)).
- `fold` 하나로 최솟값과 최댓값을 함께 찾습니다. `f64::INFINITY` / `f64::NEG_INFINITY` 에서 시작하므로, 리스트가 비어 있지 않은 한 첫 원소가 그 값을 대체합니다 ([`Iterator`](../../std/core/iter.html#trait.Iterator)).
- `temps[week * 7:(week + 1) * 7]` 은 리스트의 한 구간을 슬라이스로 가져옵니다 ([3.6 인덱싱과 슬라이싱](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)).
- 히스토그램 막대는 `"#".repeat(c)` 로 만들고, 구간 라벨은 `{from:4.1}` 같은 포맷 지정자로 정렬합니다. `{}` 로 출력한 `f64` 는 다시 읽었을 때 같은 값이 되는 가장 짧은 자릿수를 쓰므로 `26.0` 은 `26` 으로 출력됩니다. 자릿수를 고정하려면 `{hi:.1}` 처럼 정밀도를 적습니다 ([문자열](../learn/09-strings.html)).

## 실행하기

먼저 Rok 커널을 Jupyter 에 등록합니다. `rok jupyter install` 은 kernelspec (`kernels/rok/kernel.json`) 을 Jupyter 데이터 디렉토리 (macOS 는 `~/Library/Jupyter`, 그 밖에는 `~/.local/share/jupyter`) 에 씁니다 ([9.8.3 kernelspec 설치](../spec/refs-lang/009-tools/908-jupyter-integration.html#983-kernelspec-설치)).

```bash
rok jupyter install
jupyter lab            # pick the "rok" kernel in a new notebook
```

환경마다 커널을 따로 두려면 `rok jupyter install --env <name>` 으로 설치합니다. 그러면 `rok (<name>)` 으로 표시되는 커널이 하나 더 생깁니다.

노트북에서는 위 프로그램을 몇 개의 셀로 나눠 실행합니다. 셀의 마지막 표현식이 그 셀의 결과이며 `type: value` 형태로 표시됩니다.

```text
In [1]:  let temps: List<f64> = [12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
                                 19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
                                 20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
                                 22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8];
         temps.len()
Out[1]:  usize: 28

In [2]:  fn mean(xs: List<f64>) -> f64 {
             let sum = 0.0;
             for x in xs { sum += x; }
             sum / xs.len() as f64
         }
         mean(temps)
Out[2]:  f64: 20.16071428571428

In [3]:  temps[0:7]
Out[3]:  Slice<f64>: [12.5, 14, 13.2, 15.8, 17.1, 16.4, 18.9]

In [4]:  let week1: List<f64> = temps[0:7].iter().collect();
         mean(week1)
Out[4]:  f64: 15.414285714285715
```

`let temps` 와 `fn mean` 은 이후 셀에서도 정의된 채로 남으므로, 셀 `[3]` 과 `[4]` 는 앞에서 만든 값과 함수를 씁니다. 함수를 고치고 그 셀을 다시 실행하면 뒤따르는 셀은 새 정의를 씁니다 ([9.7 JIT 셀 실행](../spec/refs-lang/009-tools/907-jit-cells.html)). 터미널에서 같은 방식의 셀 세션을 쓰려면 `rok repl` 을 실행합니다 ([도구](../learn/16-tools.html)).
