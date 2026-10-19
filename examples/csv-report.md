# CSV report

Write a small sales CSV to a temporary file, read it back line by line, and print a per-region summary table with aligned columns.

## Code

```rok
// Write a small sales CSV, read it back line by line, and print a per-region report.
use rok::std::env;
use rok::std::fs;
use rok::std::fs::File;
use rok::std::io::{IoError, BufReader, BufWriter, Reader, Writer};

const ROWS: str = "date,region,product,units,unit_price
2026-01-03,north,widget,12,2.50
2026-01-03,south,gadget,3,19.99
2026-01-04,north,gadget,1,19.99
2026-01-05,east,widget,40,2.25
2026-01-05,south,widget,7,2.50
2026-01-06,west,gizmo,two,7.00
2026-01-07,east,gizmo,5,7.00
2026-01-08,north,gizmo,2,7.50
2026-01-09,east;widget,3,2.25";

struct Sale {
    region: str;
    units: i64;
    price: f64;
}

struct Group {
    orders: i64 = 0;
    units: i64 = 0;
    revenue: f64 = 0.0;
}

enum RowError {
    Fields(usize),
    Number(str),
}

fn parse_row(line: str) throws (RowError) -> Sale {
    let f: List<str> = [];
    for part in line.split(",") { f.push(part.trim()); }
    if f.len() != 5 { throw RowError::Fields(f.len()); }
    let units = try f[3].parse::<i64>() catch _ { throw RowError::Number(f[3]); };
    let price = try f[4].parse::<f64>() catch _ { throw RowError::Number(f[4]); };
    Sale { region: f[1], units: units, price: price }
}

fn write_csv(path: str) throws (IoError) {
    let w = BufWriter::new(try File::create(path));
    for line in ROWS.lines() {
        try w.write_fmt(f"{line}\n");
    }
    try w.flush();
}

fn summarize(path: str) throws (IoError) -> Dict<str, Group> {
    let groups: Dict<str, Group> = Dict::new();
    let r = BufReader::new(try File::open(path));
    let n: usize = 0;
    for item in r.lines() {
        let line = try item.get();
        n += 1;
        if n == 1 || line.is_empty() { continue; } // header
        let sale = try parse_row(line) catch e: RowError {
            match e {
                RowError::Fields(k) => println(f"line {n}: expected 5 fields, got {k}, skipped"),
                RowError::Number(s) => println(f"line {n}: bad number '{s}', skipped"),
            }
            continue;
        };
        let g = groups.get(sale.region) ?? Group {};
        g.orders += 1;
        g.units += sale.units;
        g.revenue += sale.units as f64 * sale.price;
        groups[sale.region] = g;
    }
    groups
}

fn print_report(groups: Dict<str, Group>) {
    let regions: List<str> = [];
    for k in groups.keys() { regions.push(k); }
    regions.sort();

    println(f"{"region":<8}|{"orders":>7}|{"units":>6}|{"revenue":>9}|{"avg":>7}");
    println(f"{"":-<8}+{"":-<7}+{"":-<6}+{"":-<9}+{"":-<7}");
    let total = Group {};
    for name in regions {
        let g = groups[name];
        let avg = g.revenue / g.units as f64;
        println(f"{name:<8}|{g.orders:>7}|{g.units:>6}|{g.revenue:>9.2}|{avg:>7.2}");
        total.orders += g.orders;
        total.units += g.units;
        total.revenue += g.revenue;
    }
    println(f"{"total":<8}|{total.orders:>7}|{total.units:>6}|{total.revenue:>9.2}|{"":>7}");
}

fn main() throws (IoError) {
    let path = (env::temp_dir() / "rok-sales-demo.csv").as_str();
    try write_csv(path);
    let groups = try summarize(path) finally { try! fs::remove_file(path); };
    print_report(groups);
}
```

```output
line 7: bad number 'two', skipped
line 10: expected 5 fields, got 4, skipped
region  | orders| units|  revenue|    avg
--------+-------+------+---------+-------
east    |      2|    45|   125.00|   2.78
north   |      3|    15|    64.99|   4.33
south   |      2|    10|    77.47|   7.75
total   |      7|    70|   267.46|
```

## Walkthrough

- The file is created under `env::temp_dir()` and removed in the `finally` of `main`, so no temporary file is left behind even if summarizing fails ([Files and I/O](../learn/13-io.html), [`env::temp_dir`](../std/std/env.html#fn.temp_dir)).
- Writing goes through a `BufWriter` and reading through a `BufReader`. Each item of `r.lines()` is a `Result<str, IoError>`, so `try item.get()` takes the line out and propagates read errors ([`BufReader`](../std/std/io.html#struct.BufReader), [`fs`](../std/std/fs.html)).
- `parse_row` throws a `RowError` for a malformed row; the caller prints a warning in its `catch` block and moves on with `continue`. One bad line does not sink the whole report ([Errors](../learn/10-errors.html)).
- The totals are a `Dict<str, Group>`. `groups.get(region) ?? Group {}` fetches an entry or starts a new one, and `groups[region] = g` stores the updated value back. Every field of `Group` has a default, so `Group {}` alone starts at zero ([Collections and iterators](../learn/08-collections-iterators.html)).
- Format specs line up the table: `{name:<8}` left-aligns, `{g.revenue:>9.2}` uses width 9 and two decimals, and `{"":-<8}` fills an empty string with `-` to draw the rule ([2.10.8 align and width](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)).
