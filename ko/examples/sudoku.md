# 스도쿠 풀이기

고정 크기 배열로 된 판에서, 빈 칸마다 후보 숫자를 하나씩 넣어 보는 백트래킹으로 스도쿠를 풉니다.

## 코드

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

## 설명

- 판은 고정 크기 배열 `[[u8; 9]; 9]` 입니다. 크기가 타입의 일부이므로, 따로 할당하지 않고 `cells[r][c]` 로 바로 읽고 씁니다 ([값과 타입](../learn/02-values-and-types.html)).
- `solve` 는 `ref self` 로 판을 제자리에서 고치고, `ref steps` 로 호출한 쪽의 카운터를 올립니다. 호출하는 쪽도 `board.solve(ref steps)` 처럼 `ref` 를 적으므로, 값이 바뀐다는 사실이 드러납니다 ([4.7 `ref` 키워드](../spec/refs-lang/004-functions/407-ref-keyword.html), [함수](../learn/03-functions.html)).
- 백트래킹의 핵심은 세 줄입니다. 숫자를 놓고 (`self.cells[r][c] = d`), 나머지를 재귀적으로 풀어 보고, 실패하면 칸을 다시 비웁니다. 첫 빈 칸에서 1-9 가 모두 실패하면 `false` 가 반환되고, 한 단계 위에서 다른 숫자를 시도합니다.
- 리터럴 범위 `0..9` 는 기본적으로 `i32` 이므로, `usize` 인자로 넘길 반복 변수는 `0usize..9` 처럼 접미사로 타입을 정합니다 ([1.6 숫자 리터럴](../spec/refs-lang/001-lexical/106-numeric-literals.html)).
- 퍼즐 문자열을 읽는 코드는 숫자와 `.` 만 받고 공백과 줄바꿈은 건너뛰므로, 판을 읽기 좋게 배치할 수 있습니다 ([문자열](../learn/09-strings.html)).
