# 텍스트 어드벤처

방, 출구, 아이템, 잠긴 문이 있는 작은 텍스트 어드벤처를 미리 적어 둔 명령 목록으로 끝까지 플레이합니다.

## 코드

```rok
enum Dir { North, South, East, West }

enum Command {
    Go(Dir),
    Take(str),
    Use(str),
    Look,
    Inventory,
    Unknown(str),
}

struct Room {
    name: str;
    desc: str;
    exits: Dict<str, usize>;    // direction name -> room index
    items: List<str>;
}

struct Game {
    rooms: List<Room>;
    here: usize;
    bag: List<str>;
    door_open: bool;
    won: bool;
}

fn dir_name(d: Dir) -> str {
    match d { Dir::North => "north", Dir::South => "south", Dir::East => "east", Dir::West => "west" }
}

fn parse(line: str) -> Command {
    let words: List<str> = line.trim().to_lowercase().split_whitespace().collect();
    if words.is_empty() { return Command::Unknown(""); }
    let arg = if words.len() > 1 { words[1] } else { "" };
    match words[0] {
        "n" | "north" => Command::Go(Dir::North),
        "s" | "south" => Command::Go(Dir::South),
        "e" | "east" => Command::Go(Dir::East),
        "w" | "west" => Command::Go(Dir::West),
        "go" => parse(arg),
        "take" | "get" => Command::Take(arg),
        "use" => Command::Use(arg),
        "look" | "l" => Command::Look,
        "i" | "inventory" => Command::Inventory,
        _ => Command::Unknown(line),
    }
}

impl Game {
    fn describe(self) {
        let room = self.rooms[self.here];
        println(f"== {room.name} ==\n{room.desc}");
        if !room.items.is_empty() { println(f"You see: {room.items:?}"); }
    }

    fn step(ref self, cmd: Command) {
        match cmd {
            Command::Go(d) => {
                let way = dir_name(d);
                match self.rooms[self.here].exits.get(way) {
                    next: usize => {
                        if self.rooms[next].name == "Vault" && !self.door_open {
                            println("The vault door is locked.");
                        } else {
                            self.here = next;
                            self.describe();
                        }
                    },
                    none => println(f"You can't go {way}."),
                }
            },
            Command::Take(item) => {
                let items = self.rooms[self.here].items;
                match items.iter().position(|x| x == item) {
                    i: usize => {
                        self.bag.push(items.remove(i));
                        println(f"Taken: {item}.");
                    },
                    none => println(f"There is no {item} here."),
                }
            },
            Command::Use(item) => {
                if !self.bag.iter().any(|x| x == item) {
                    println(f"You don't have a {item}.");
                } else if item == "key" && self.rooms[self.here].name == "Hall" {
                    self.door_open = true;
                    println("The key turns. The vault door swings open.");
                } else if item == "lamp" && self.rooms[self.here].name == "Vault" {
                    self.won = true;
                    println("The lamp reveals a pile of gold. You win!");
                } else {
                    println(f"Nothing happens.");
                }
            },
            Command::Look => self.describe(),
            Command::Inventory => println(f"You carry: {self.bag:?}"),
            Command::Unknown(text) => println(f"I don't understand \"{text}\"."),
        }
    }
}

fn room(name: str, desc: str, exits: Dict<str, usize>, items: List<str>) -> Room {
    Room { name: name, desc: desc, exits: exits, items: items }
}

let game = Game {
    rooms: [
        room("Cellar", "Damp stone walls. Stairs lead north.", {"north": 1usize}, ["lamp"]),
        room("Hall", "A long hall with a heavy door to the east.", {"south": 0usize, "west": 2usize, "east": 3usize}, []),
        room("Library", "Dusty shelves everywhere.", {"east": 1usize}, ["book", "key"]),
        room("Vault", "It is pitch dark.", {"west": 1usize}, []),
    ],
    here: 0,
    bag: [],
    door_open: false,
    won: false,
};

// Scripted input instead of stdin.
let script = ["look", "take lamp", "north", "east", "go west", "take key", "take sword",
              "dance", "e", "use key", "e", "i", "use lamp"];

game.describe();
for line in script {
    if game.won { break; }
    println(f"\n> {line}");
    game.step(parse(line));
}
```

```output
== Cellar ==
Damp stone walls. Stairs lead north.
You see: ["lamp"]

> look
== Cellar ==
Damp stone walls. Stairs lead north.
You see: ["lamp"]

> take lamp
Taken: lamp.

> north
== Hall ==
A long hall with a heavy door to the east.

> east
The vault door is locked.

> go west
== Library ==
Dusty shelves everywhere.
You see: ["book", "key"]

> take key
Taken: key.

> take sword
There is no sword here.

> dance
I don't understand "dance".

> e
== Hall ==
A long hall with a heavy door to the east.

> use key
The key turns. The vault door swings open.

> e
== Vault ==
It is pitch dark.

> i
You carry: ["lamp", "key"]

> use lamp
The lamp reveals a pile of gold. You win!
```

## 설명

- 플레이어 입력은 먼저 `Command` enum으로 파싱되고, 게임 로직은 그 enum만 다룹니다. 문자열 처리와 상태 변경이 분리되어 새 명령을 더하기 쉽고, `match`가 모든 변형을 다루는지 컴파일러가 검사합니다([합 타입](../learn/05-sum-types.html), [3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)).
- `"n" | "north" => ...` 같은 or 패턴은 별칭을 묶습니다. `"go"`는 남은 단어로 `parse`를 다시 호출하므로 `go west`와 `west`는 같은 명령이 됩니다([3.11 패턴](../spec/refs-lang/003-expressions/311-patterns.html#or-패턴)).
- `Dict::get`은 `usize | none`을 돌려줍니다. `next: usize => ...`와 `none => ...`로 나누면 "그쪽으로는 출구가 없음"을 자연스럽게 처리합니다([`Dict`](../../std/core/collections/dict.html#struct.Dict)).
- 게임 상태 `Game`은 `step(ref self, ...)` 메서드가 제자리에서 바꿉니다. 방의 `items`는 `List`이므로, `let items = self.rooms[self.here].items;`로 꺼내서 `remove`를 호출하면 방이 가진 리스트도 함께 바뀝니다([컬렉션과 반복자](../learn/08-collections-iterators.html)).
- 표준 입력 대신 `script` 리스트를 한 줄씩 넣습니다. 실제 게임으로 만들려면 이 루프를 `io::stdin()`에서 읽는 루프로 바꿉니다([I/O](../learn/13-io.html)).
