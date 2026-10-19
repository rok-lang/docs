# Text adventure

A small text adventure with rooms, exits, items and a locked door, played to the end from a prewritten list of commands.

## Code

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

## Walkthrough

- Player input is first parsed into a `Command` enum, and the game logic only deals with that enum. String handling and state changes are kept apart, which makes new commands easy to add, and the compiler checks that `match` covers every variant ([Sum types](../learn/05-sum-types.html), [3.12 match](../spec/refs-lang/003-expressions/312-match-expression.html)).
- Or-patterns such as `"n" | "north" => ...` group aliases, and `"go"` calls `parse` again on the remaining words so `go west` and `west` become the same command ([3.11 Patterns](../spec/refs-lang/003-expressions/311-patterns.html#or-patterns)).
- `Dict::get` returns `usize | none`. Splitting into `next: usize => ...` and `none => ...` handles "no exit that way" naturally ([`Dict`](../std/core/collections/dict.html#struct.Dict)).
- The game state `Game` changes in place through `step(ref self, ...)`. A room's `items` is a `List`, so taking it out with `let items = self.rooms[self.here].items;` and calling `remove` also changes the room's own list ([Collections and iterators](../learn/08-collections-iterators.html)).
- The `script` list is fed in one line at a time instead of standard input. To make it a real game, replace this loop with one that reads from `io::stdin()` ([I/O](../learn/13-io.html)).
