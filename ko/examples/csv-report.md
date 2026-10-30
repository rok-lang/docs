# CSV 보고서

작은 판매 CSV를 임시 파일에 쓰고 한 줄씩 다시 읽어서 열을 맞춘 지역별 요약 표를 출력합니다.

## 코드

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

## 설명

- 파일은 `env::temp_dir()` 아래에 만들고 `main`의 `finally`에서 지웁니다. 그래서 요약 도중 실패해도 임시 파일이 남지 않습니다([파일과 I/O](../learn/13-io.html), [`env::temp_dir`](../../std/std/env.html#fn.temp_dir)).
- 쓰기는 `BufWriter`를, 읽기는 `BufReader`를 거칩니다. `r.lines()`의 각 항목은 `Result<str, IoError>`이므로 `try item.get()`으로 줄을 꺼내고 읽기 에러는 전파합니다([`BufReader`](../../std/std/io.html#struct.BufReader), [`fs`](../../std/std/fs.html)).
- `parse_row`는 형식이 잘못된 행에서 `RowError`를 던집니다. 호출한 쪽은 `catch` 블록에서 경고를 출력하고 `continue`로 넘어갑니다. 잘못된 줄 하나 때문에 보고서 전체가 실패하지 않습니다([에러 처리](../learn/10-errors.html)).
- 합계는 `Dict<str, Group>`입니다. `groups.get(region) ?? Group {}`는 항목을 가져오거나 새로 시작하고, `groups[region] = g`는 갱신한 값을 다시 저장합니다. `Group`의 모든 필드에 기본값이 있으므로 `Group {}`만으로 0에서 시작합니다([컬렉션과 반복자](../learn/08-collections-iterators.html)).
- 포맷 지정자로 표를 정렬합니다. `{name:<8}`은 왼쪽 정렬, `{g.revenue:>9.2}`는 폭 9와 소수 둘째 자리, `{"":-<8}`은 빈 문자열을 `-`로 채워 구분선을 그립니다([2.10.8 정렬과 폭](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)).
