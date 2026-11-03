# Jupyter 노트북으로 데이터 탐색

Jupyter에 Rok 커널을 설치하고 온도 데이터를 셀 단위로 탐색합니다. 같은 작업을 하는 실행 가능한 프로그램 하나도 함께 실었습니다.

노트북에서는 한 단계씩 작업합니다. 한 셀은 데이터를 읽고, 다른 셀은 함수를 정의하고, 또 다른 셀은 결과를 확인합니다. 아래 프로그램은 그 셀을 위에서 아래로 차례로 이어 붙인 것입니다. 4주 동안의 일 최고 기온에서 요약 통계, 주별 평균, 텍스트 히스토그램을 계산합니다.

## 코드

```rok
// Daily high temperatures (°C) for four weeks.
let temps = [
    12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
    19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
    20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
    22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8,
];

fn mean(xs: List<f64>) -> f64 { xs.sum() / xs.len() as f64 }

fn std_dev(xs: List<f64>) -> f64 {
    let m = mean(xs);
    (xs.map(|x| (x - m) ** 2).sum() / xs.len() as f64).sqrt()
}

fn median(xs: List<f64>) -> f64 {
    let sorted = xs.sorted_by(f64::total_cmp);
    let n = sorted.len();
    if n % 2 == 1 { sorted[n / 2] } else { (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0 }
}

// Summary statistics.
let lo = temps.min() ?? f64::NAN;
let hi = temps.max() ?? f64::NAN;
println(f"n      = {temps.len()}");
println(f"mean   = {mean(temps):.2}");
println(f"median = {median(temps):.2}");
println(f"stddev = {std_dev(temps):.2}");
println(f"range  = {lo:.1} .. {hi:.1}");

// Weekly averages.
for week in 0..4 {
    let days = temps[week * 7:(week + 1) * 7].to_list();
    println(f"week {week + 1}: {mean(days):5.2}");
}

// An ASCII histogram with 2-degree buckets.
let start = 12.0;
let width = 2.0;
let counts = List::filled(8, 0);
for t in temps {
    counts[((t - start) / width) as usize] += 1;
}
println("histogram:");
for (i, c) in counts.enumerate() {
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

- 노트북 셀의 최상위 `let`과 `fn`은 세션 내내 유지됩니다. 소스 파일과 같이 셀에서도 같은 이름을 `let`으로 다시 선언할 수 있습니다. 그래서 다른 데이터로 셀을 다시 실행할 수 있습니다([5.1.4 같은 스코프 안의 재선언](../spec/refs-lang/005-modules/501-variable-declaration.html#514-같은-스코프-안의-재선언)).
- `f64`에는 `Ord`가 없으므로(NaN은 순서가 없음) 중앙값은 IEEE 전순서 `sorted_by(f64::total_cmp)`로 정렬합니다([컬렉션과 반복자](../learn/08-collections-iterators.html)).
- `temps.min()` / `temps.max()`는 부동소수점에도 됩니다(NumPy처럼 NaN이 있으면 NaN). `?? f64::NAN`은 빈 리스트를 처리합니다([`Iterator`](../../std/core/iter.html#trait.Iterator)).
- `temps[week * 7:(week + 1) * 7]`은 리스트의 한 구간을 슬라이스로 가져오고, `to_list()`는 그것을 `mean`이 받는 `List`로 복사합니다. `sum()`은 리스트 전체를 더하고 `(x - m) ** 2`는 제곱입니다([3.6 인덱싱과 슬라이싱](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)).
- 히스토그램 막대는 `"#".repeat(c)`로 만들고, 구간 라벨은 `{from:4.1}` 같은 포맷 지정자로 정렬합니다. `{}`로 출력한 `f64`는 다시 읽었을 때 같은 값이 되는 가장 짧은 자릿수를 쓰므로 `26.0`은 `26`으로 출력됩니다. 자릿수를 고정하려면 `{hi:.1}`처럼 정밀도를 적습니다([문자열](../learn/09-strings.html)).

## 실행하기

먼저 Rok 커널을 Jupyter에 등록합니다. `rok jupyter install`은 kernelspec(`kernels/rok/kernel.json`)을 Jupyter 데이터 디렉터리(macOS는 `~/Library/Jupyter`, Windows는 `%APPDATA%\jupyter`, 그 밖에는 `~/.local/share/jupyter`)에 씁니다([9.8.3 kernelspec 설치](../spec/refs-lang/009-tools/908-jupyter-integration.html#983-kernelspec-설치)).

```bash
rok jupyter install
jupyter lab            # pick the "rok" kernel in a new notebook
```

환경마다 커널을 따로 두려면 `rok jupyter install --env <name>`으로 설치합니다. 그러면 `rok (<name>)`으로 표시되는 커널이 하나 더 생깁니다.

노트북에서는 위 프로그램을 여러 셀로 나눠 실행합니다. 셀의 마지막 표현식이 그 셀의 결과이고 `Display`로 표시됩니다. 끝에 `?`를 붙이면 `Debug` 형태를 볼 수 있습니다.

```text
In [1]:  let temps = [12.5, 14.0, 13.2, 15.8, 17.1, 16.4, 18.9,
                      19.5, 21.0, 20.2, 22.7, 24.1, 23.3, 22.0,
                      20.8, 19.4, 21.6, 23.9, 25.2, 26.0, 24.4,
                      22.1, 20.5, 18.7, 17.9, 19.2, 21.3, 22.8];
         temps.len()
Out[1]:  28

In [2]:  fn mean(xs: List<f64>) -> f64 { xs.sum() / xs.len() as f64 }
         mean(temps)
Out[2]:  20.16071428571428

In [3]:  temps[0:7]?
Out[3]:  [12.5, 14, 13.2, 15.8, 17.1, 16.4, 18.9]

In [4]:  mean(temps[0:7].to_list())
Out[4]:  15.414285714285715
```

`let temps`와 `fn mean`은 이후 셀에서도 정의된 채로 남으므로 셀 `[3]`과 `[4]`는 앞에서 만든 값과 함수를 씁니다. 함수를 고치고 그 셀을 다시 실행하면 뒤따르는 셀은 새 정의를 씁니다([9.7 JIT 셀 실행](../spec/refs-lang/009-tools/907-jit-cells.html)). 터미널에서 같은 방식의 셀 세션을 쓰려면 `rok repl`을 실행합니다([도구](../learn/16-tools.html)).
