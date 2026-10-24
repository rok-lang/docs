# Game of Life

Run Conway's Game of Life on a 10×8 torus grid whose edges wrap around, printing a few generations as text.

## Code

```rok
const W: usize = 10;
const H: usize = 8;

// A grid of cells on a torus: the edges wrap around.
struct Grid {
    cells: [[bool; W]; H];
}

impl Grid {
    fn parse(rows: List<str>) -> Grid {
        let cells = [[false; W]; H];
        for (y, row) in rows.iter().enumerate() {
            for (x, c) in row.chars().enumerate() {
                cells[y][x] = c == '#';
            }
        }
        Grid { cells: cells }
    }

    fn alive(self, x: usize, y: usize) -> bool { self.cells[y][x] }

    fn neighbors(self, x: usize, y: usize) -> usize {
        let n = 0usize;
        for dy in [H - 1, 0, 1] {
            for dx in [W - 1, 0, 1] {
                if dx == 0 && dy == 0 { continue; }
                // Adding W - 1 and taking % W steps one cell left with wraparound.
                if self.alive((x + dx) % W, (y + dy) % H) { n += 1; }
            }
        }
        n
    }

    fn step(self) -> Grid {
        let next = [[false; W]; H];
        for y in 0..H {
            for x in 0..W {
                next[y][x] = match (self.alive(x, y), self.neighbors(x, y)) {
                    (true, 2) | (true, 3) => true,
                    (false, 3) => true,
                    _ => false,
                };
            }
        }
        Grid { cells: next }
    }

    fn population(self) -> usize {
        let n = 0usize;
        for row in self.cells {
            for c in row { if c { n += 1; } }
        }
        n
    }
}

impl Display for Grid {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        for row in self.cells {
            for c in row { try f.write_char(if c { '#' } else { '.' }); }
            try f.write_char('\n');
        }
    }
}

// A glider in the corner and a blinker on the right.
let grid = Grid::parse([
    ".#........",
    "..#.......",
    "###.......",
    "..........",
    "......###.",
    "..........",
    "..........",
    "..........",
]);

for gen in 0..4 {
    println(f"generation {gen} (population {grid.population()})");
    println(f"{grid}");
    grid = grid.step();
}

// A lone glider returns to its shape every 4 steps, shifted by (1, 1);
// on a 10x8 torus it needs lcm(10, 8) = 40 such moves to come home.
let glider = Grid::parse([".#", "..#", "###"]);
let g = glider.step();
let period = 1;
while g.cells != glider.cells {
    g = g.step();
    period += 1;
}
println(f"a lone glider comes back after {period} generations");
```

```output
generation 0 (population 8)
.#........
..#.......
###.......
..........
......###.
..........
..........
..........

generation 1 (population 8)
..........
#.#.......
.##.......
.#.....#..
.......#..
.......#..
..........
..........

generation 2 (population 8)
..........
..#.......
#.#.......
.##.......
......###.
..........
..........
..........

generation 3 (population 8)
..........
.#........
..##......
.##....#..
.......#..
.......#..
..........
..........

a lone glider comes back after 160 generations
```

## Walkthrough

- The grid is a fixed-size 2D array `[[bool; W]; H]` (10×8). Arrays carry their size in the type and need no separate allocation; they are built with the repeat form `[[false; W]; H]` and compared whole with `==` / `!=` ([Values and types](../learn/02-values-and-types.html)).
- Neighbors are counted with `(x + W - 1) % W` instead of `x - 1`. This avoids `usize` subtraction going below zero and makes the edges wrap to the opposite side.
- The survival rule is a single `match` on the `(alive, neighbors)` tuple. Patterns can be combined with `|`, as in `(true, 2) | (true, 3)` ([3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html), [Control flow](../learn/04-control-flow.html)).
- `impl Display for Grid` writes one character at a time with `Formatter::write_char`, so `f"{grid}"` draws the grid ([`Formatter`](../std/core/fmt.html#struct.Formatter), [Structs, enums and traits](../learn/06-structs-enums-traits.html)).
- A glider moves one cell diagonally every 4 generations, keeping its shape. On a 10×8 torus it needs lcm(10, 8) = 40 moves to return to its starting place, which takes 160 generations.
