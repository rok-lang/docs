# Matrix operations

A small matrix type that implements the operator and index traits, so you can write `a * b`, `a .* b` and `m[i, j]`.

## Code

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

## Walkthrough

- Every operator in rok is a trait method call. With `impl Add for Matrix`, `a + a` desugars to `Add::add(a, a)` ([3.1 Operator dispatch](../spec/refs-lang/003-expressions/301-operator-dispatch.html), [`Add`](../std/core/ops.html#trait.Add), [`Mul`](../std/core/ops.html#trait.Mul)).
- The same operator can be implemented several times with different right-hand types. `impl Mul for Matrix` is the matrix product and `impl Mul<f64> for Matrix` is scalar multiplication; the compiler picks the impl from the operand types.
- Dot-prefixed operators such as `.*` desugar to a separate trait family for element-wise operations ([`DotMul`](../std/core/ops.html#trait.DotMul)), so the matrix product `*` and the Hadamard product `.*` can live side by side ([3.3 Broadcast operators](../spec/refs-lang/003-expressions/303-broadcast-operators.html)).
- Comma indices are grouped into a tuple, so reading `m[i, j]` goes through [`Index<(usize, usize)>`](../std/core/index.html#trait.Index) and writing `m[i, j] = v` through [`IndexSet`](../std/core/index.html#trait.IndexSet). The write method takes `ref self` to modify the matrix in place ([3.6 Indexing and slicing](../spec/refs-lang/003-expressions/306-indexing-and-slicing.html)).
- Thanks to `impl Display for Matrix`, `{a}` in an f-string prints the whole matrix. Each element uses `{x:6.1}` for width and precision ([Structs, enums and traits](../learn/06-structs-enums-traits.html)).
