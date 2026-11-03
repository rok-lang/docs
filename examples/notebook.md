# Exploring data in a Jupyter notebook

Install the Rok kernel in Jupyter and explore temperature data cell by cell, alongside a single runnable program that does the same work.

In a notebook you work step by step: one cell loads the data, another defines functions, another checks the results. The program below is those cells joined in order from top to bottom. From four weeks of daily highs it computes summary statistics, weekly averages and a text histogram.

## Code

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

## Walkthrough

- Top-level `let`s and `fn`s in a notebook cell persist for the whole session. A cell may `let` the same name again, as a source file may, so you can re-run a cell with different data ([5.1.4 Redeclaration in the same scope](../spec/refs-lang/005-modules/501-variable-declaration.html#514-redeclaration-in-the-same-scope)).
- `f64` has no `Ord` (NaN is unordered), so the median sorts by the IEEE total order, `sorted_by(f64::total_cmp)` ([Collections and iterators](../learn/08-collections-iterators.html)).
- `temps.min()` / `temps.max()` work on floats (a NaN would win, as in NumPy); `?? f64::NAN` covers an empty list ([`Iterator`](../std/core/iter.html#trait.Iterator)).
- `temps[week * 7:(week + 1) * 7]` takes one range of the list as a slice, and `to_list()` copies it into the `List` that `mean` takes. `sum()` adds a whole list, and `(x - m) ** 2` squares ([3.6 Indexing and slicing](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)).
- Histogram bars come from `"#".repeat(c)` and the bucket labels are aligned with format specs such as `{from:4.1}`. An `f64` printed with `{}` uses the shortest digits that read back as the same value, so `26.0` prints as `26`; write a precision such as `{hi:.1}` when you want fixed digits ([Strings](../learn/09-strings.html)).

## Running it

First register the Rok kernel with Jupyter. `rok jupyter install` writes the kernelspec (`kernels/rok/kernel.json`) into the Jupyter data directory (`~/Library/Jupyter` on macOS, `%APPDATA%\jupyter` on Windows, `~/.local/share/jupyter` elsewhere) ([9.8.3 Installing the kernelspec](../spec/refs-lang/009-tools/908-jupyter-integration.html#983-kernelspec-installation)).

```bash
rok jupyter install
jupyter lab            # pick the "rok" kernel in a new notebook
```

To keep a separate kernel per environment, install with `rok jupyter install --env <name>`. That creates another kernel shown as `rok (<name>)`.

In the notebook, the program above runs split into a few cells. The last expression of a cell is its result and is shown with `Display`; end it with `?` to see the `Debug` form.

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

`let temps` and `fn mean` stay defined in later cells, so cells `[3]` and `[4]` use the values and functions created earlier. If you fix a function and re-run its cell, the following cells use the new definition ([9.7 JIT cell execution](../spec/refs-lang/009-tools/907-jit-cells.html)). To use the same kind of cell session in a terminal, run `rok repl` ([Tools](../learn/16-tools.html)).
