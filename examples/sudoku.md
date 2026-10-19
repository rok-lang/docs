# Sudoku solver

Solve a sudoku with backtracking over a fixed-size array board, trying each candidate digit in every empty cell.

## Code

```rok
// A 9x9 board; 0 marks an empty cell.
struct Board {
    cells: [[u8; 9]; 9];
}

impl Board {
    fn parse(text: str) -> Board {
        let cells = [[0u8; 9]; 9];
        let i = 0usize;
        for c in text.chars() {
            if c == '.' || c.is_ascii_digit() {
                cells[i / 9][i % 9] = if c == '.' { 0u8 } else { (c as u32 - '0' as u32) as u8 };
                i += 1;
            }
        }
        Board { cells: cells }
    }

    // Can digit `d` go at (r, c) without clashing in its row, column or box?
    fn allowed(self, r: usize, c: usize, d: u8) -> bool {
        for k in 0..9 {
            if self.cells[r][k] == d || self.cells[k][c] == d { return false; }
        }
        let (br, bc) = (r / 3 * 3, c / 3 * 3);
        for i in br..br + 3 {
            for j in bc..bc + 3 {
                if self.cells[i][j] == d { return false; }
            }
        }
        true
    }

    // Depth-first search; returns true once every cell is filled.
    fn solve(ref self, ref steps: u64) -> bool {
        for r in 0usize..9 {
            for c in 0usize..9 {
                if self.cells[r][c] != 0 { continue; }
                for d in 1u8..=9 {
                    if self.allowed(r, c, d) {
                        self.cells[r][c] = d;
                        steps += 1;
                        if self.solve(ref steps) { return true; }
                        self.cells[r][c] = 0;   // backtrack
                    }
                }
                return false;
            }
        }
        true
    }
}

impl Display for Board {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for r in 0..9 {
            if r > 0 && r % 3 == 0 { try f.write_str("------+-------+------\n"); }
            for c in 0..9 {
                if c > 0 && c % 3 == 0 { try f.write_str("| "); }
                let v = self.cells[r][c];
                try f.write_str(if v == 0 { ". " } else { f"{v} " });
            }
            try f.write_str("\n");
        }
    }
}

let board = Board::parse("
    53..7.... 6..195... .98....6.
    8...6...3 4..8.3..1 7...2...6
    .6....28. ...419..5 ....8..79
");
println(f"puzzle:\n{board}");

let steps = 0u64;
if board.solve(ref steps) {
    println(f"solved after {steps} placements:\n{board}");
} else {
    println("no solution");
}
```

```output
puzzle:
5 3 . | . 7 . | . . . 
6 . . | 1 9 5 | . . . 
. 9 8 | . . . | . 6 . 
------+-------+------
8 . . | . 6 . | . . 3 
4 . . | 8 . 3 | . . 1 
7 . . | . 2 . | . . 6 
------+-------+------
. 6 . | . . . | 2 8 . 
. . . | 4 1 9 | . . 5 
. . . | . 8 . | . 7 9 

solved after 4208 placements:
5 3 4 | 6 7 8 | 9 1 2 
6 7 2 | 1 9 5 | 3 4 8 
1 9 8 | 3 4 2 | 5 6 7 
------+-------+------
8 5 9 | 7 6 1 | 4 2 3 
4 2 6 | 8 5 3 | 7 9 1 
7 1 3 | 9 2 4 | 8 5 6 
------+-------+------
9 6 1 | 5 3 7 | 2 8 4 
2 8 7 | 4 1 9 | 6 3 5 
3 4 5 | 2 8 6 | 1 7 9
```

## Walkthrough

- The board is a fixed-size array `[[u8; 9]; 9]`. The size is part of the type, so `cells[r][c]` reads and writes directly with no separate allocation ([Values and types](../learn/02-values-and-types.html)).
- `solve` fixes the board in place through `ref self` and bumps the caller's counter through `ref steps`. The caller writes `ref` too, `board.solve(ref steps)`, so it is visible that the value changes ([4.7 The `ref` keyword](../spec/refs-lang/004-functions/407-ref-keyword.html), [Functions](../learn/03-functions.html)).
- The heart of backtracking is three lines: place a digit (`self.cells[r][c] = d`), recursively try to solve the rest, and clear the cell again on failure. If all of 1–9 fail in the first empty cell, `false` returns and the level above tries another digit.
- The literal range `0..9` defaults to `i32`, so a loop variable passed as a `usize` argument gets its type from a suffix: `0usize..9` ([1.6 Numeric literals](../spec/refs-lang/001-lexical/106-numeric-literals.html)).
- The puzzle string reader takes only digits and `.` and skips spaces and newlines, so the board can be laid out readably ([Strings](../learn/09-strings.html)).
