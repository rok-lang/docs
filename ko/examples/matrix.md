# 행렬 연산

연산자 트레잇과 인덱스 트레잇을 구현해서 `a * b`, `a .* b`, `m[i, j]` 로 쓸 수 있는 작은 행렬 타입입니다.

## 코드

```rok
// A dense row-major matrix of f64.
struct Matrix {
    rows: usize;
    cols: usize;
    data: List<f64>;
}

impl Matrix {
    fn zeros(rows: usize, cols: usize) -> Matrix {
        let data: List<f64> = List::new();
        for _ in 0..rows * cols { data.push(0.0); }
        Matrix { rows: rows, cols: cols, data: data }
    }

    fn from_rows(rows: List<List<f64>>) -> Matrix {
        let m = Matrix::zeros(rows.len(), rows[0].len());
        for i in 0..m.rows {
            for j in 0..m.cols { m[i, j] = rows[i][j]; }
        }
        m
    }

    fn identity(n: usize) -> Matrix {
        let m = Matrix::zeros(n, n);
        for i in 0..n { m[i, i] = 1.0; }
        m
    }

    fn transpose(self) -> Matrix {
        let t = Matrix::zeros(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols { t[j, i] = self[i, j]; }
        }
        t
    }
}

// m[i, j] reads an element ...
impl Index<(usize, usize)> for Matrix {
    type Output = f64;
    fn index(self, idx: (usize, usize)) -> f64 {
        self.data[idx.0 * self.cols + idx.1]
    }
}

// ... and m[i, j] = v writes one.
impl IndexSet<(usize, usize)> for Matrix {
    type Value = f64;
    fn index_set(ref self, idx: (usize, usize), value: f64) {
        self.data[idx.0 * self.cols + idx.1] = value;
    }
}

impl Add for Matrix {
    type Output = Matrix;
    fn add(self, rhs: Matrix) -> Matrix {
        let out = Matrix::zeros(self.rows, self.cols);
        for k in 0..self.data.len() { out.data[k] = self.data[k] + rhs.data[k]; }
        out
    }
}

impl Mul for Matrix {
    type Output = Matrix;
    fn mul(self, rhs: Matrix) -> Matrix {
        let out = Matrix::zeros(self.rows, rhs.cols);
        for i in 0..self.rows {
            for j in 0..rhs.cols {
                let sum = 0.0;
                for k in 0..self.cols { sum += self[i, k] * rhs[k, j]; }
                out[i, j] = sum;
            }
        }
        out
    }
}

// Scalar scaling: Matrix * f64.
impl Mul<f64> for Matrix {
    type Output = Matrix;
    fn mul(self, k: f64) -> Matrix {
        let out = Matrix::zeros(self.rows, self.cols);
        for i in 0..self.data.len() { out.data[i] = self.data[i] * k; }
        out
    }
}

// Element-wise (Hadamard) product: a .* b.
impl DotMul for Matrix {
    type Output = Matrix;
    fn dot_mul(self, rhs: Matrix) -> Matrix {
        let out = Matrix::zeros(self.rows, self.cols);
        for k in 0..self.data.len() { out.data[k] = self.data[k] * rhs.data[k]; }
        out
    }
}

impl Display for Matrix {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for i in 0..self.rows {
            try f.write_str("[");
            for j in 0..self.cols {
                try f.write_str(f"{self[i, j]:6.1}");
            }
            try f.write_str(" ]\n");
        }
    }
}

let a = Matrix::from_rows([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]]);
let b = Matrix::from_rows([[7.0, 8.0], [9.0, 10.0], [11.0, 12.0]]);

println(f"A =\n{a}");
println(f"A^T =\n{a.transpose()}");
println(f"A * B =\n{a * b}");
println(f"A + A =\n{a + a}");
println(f"A .* A =\n{a .* a}");
println(f"A * 0.5 =\n{a * 0.5}");

let i3 = Matrix::identity(3);
let same = a * i3;
println(f"A * I == A: {same.data == a.data}");
```

```output
A =
[   1.0   2.0   3.0 ]
[   4.0   5.0   6.0 ]

A^T =
[   1.0   4.0 ]
[   2.0   5.0 ]
[   3.0   6.0 ]

A * B =
[  58.0  64.0 ]
[ 139.0 154.0 ]

A + A =
[   2.0   4.0   6.0 ]
[   8.0  10.0  12.0 ]

A .* A =
[   1.0   4.0   9.0 ]
[  16.0  25.0  36.0 ]

A * 0.5 =
[   0.5   1.0   1.5 ]
[   2.0   2.5   3.0 ]

A * I == A: true
```

## 설명

- Rok의 모든 연산자는 트레잇 메서드 호출입니다. `impl Add for Matrix` 가 있으면 `a + a` 는 `Add::add(a, a)` 로 풀립니다 ([3.1 연산자 디스패치](../spec/refs-lang/003-expressions/301-operator-dispatch.html), [`Add`](../../std/core/ops.html#trait.Add), [`Mul`](../../std/core/ops.html#trait.Mul)).
- 같은 연산자를 오른쪽 타입을 달리해서 여러 번 구현할 수 있습니다. `impl Mul for Matrix` 는 행렬 곱이고 `impl Mul<f64> for Matrix` 는 스칼라 곱입니다. 컴파일러는 피연산자 타입으로 impl 을 고릅니다.
- `.*` 처럼 점이 앞에 붙은 연산자는 원소별 연산을 위한 별도의 트레잇 계열로 풀립니다 ([`DotMul`](../../std/core/ops.html#trait.DotMul)). 그래서 행렬 곱 `*` 와 아다마르 곱 `.*` 가 나란히 존재할 수 있습니다 ([3.3 브로드캐스트 연산자](../spec/refs-lang/003-expressions/303-broadcast-operators.html)).
- 쉼표로 구분한 인덱스는 튜플로 묶입니다. 그래서 `m[i, j]` 읽기는 [`Index<(usize, usize)>`](../../std/core/index.html#trait.Index) 를, `m[i, j] = v` 쓰기는 [`IndexSet`](../../std/core/index.html#trait.IndexSet) 을 거칩니다. 쓰기 메서드는 행렬을 제자리에서 고치려고 `ref self` 를 받습니다 ([3.6 인덱싱과 슬라이싱](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)).
- `impl Display for Matrix` 덕분에 f-문자열의 `{a}` 가 행렬 전체를 출력합니다. 각 원소는 폭과 정밀도를 위해 `{x:6.1}` 을 씁니다 ([구조체, enum, 트레잇](../learn/06-structs-enums-traits.html)).
