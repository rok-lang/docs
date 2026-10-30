# 12. 동시성

여러 일을 한꺼번에 하는 올바른 방법은 목적에 따라 다릅니다. 여러 CPU를 쓰는 계산에는 스레드와 `par`가 맞고, 기다림이 많은 I/O에는 `async` task가 맞습니다. 이 장에서는 짧은 예제로 각 도구를 보여 줍니다. 데이터를 안전하게 주고받는 채널, `Mutex`, `Atomic`도 다룹니다.

> [!TIP]
> 동시에 실행되는 코드의 출력 순서는 실행할 때마다 달라질 수 있습니다. 이 장의 예제는 결과를 `join`으로 모으거나 정렬한 뒤 출력하므로 언제나 같은 출력을 냅니다.

이 가운데 `use`가 필요한 것은 거의 없습니다. `rok::core`의 모든 것은 언제나 스코프에 있습니다. `task`, `par`, `channel`, `Mutex`, `Atomic`, `Duration`이 그렇습니다. 운영체제를 다루는 `rok::std`의 모듈만 직접 들여옵니다. 파일을 `use rok::std::fs;`로 시작하듯 스레드는 `use rok::std::thread;`로 시작합니다.

## 스레드

`rok::std::thread::spawn`은 새 OS 스레드에서 클로저를 실행하고 `JoinHandle`을 반환합니다. `join()`은 그 스레드가 끝나기를 기다려 클로저의 반환값을 꺼냅니다. 스레드가 panic으로 끝났으면 `JoinError`를 던지므로 예제에서는 `try`를 붙였습니다.

```rok
use rok::std::thread;

let handles: List<thread::JoinHandle<i32>> = [];
for i in 1..=4 {
    handles.push(thread::spawn(|| i * i));
}

let results: List<i32> = [];
for h in handles {
    results.push(try h.join());
}
println(f"{results:?}");

let named = try thread::Builder::new().name("worker").spawn(|| {
    thread::current().name() ?? "?"
});
println(try named.join());
```

```output
[1, 4, 9, 16]
worker
```

예제는 모든 핸들을 만든 뒤에야 각 스레드를 기다리므로 네 스레드가 동시에 실행됩니다. 결과는 핸들 순서대로 모이므로 출력 순서는 고정됩니다.

## 채널

스레드 사이에 값을 전달하는 가장 간단한 방법은 채널입니다. `channel()`은 보내는 `Sender`와 받는 `Receiver`를 반환합니다. `Sender`는 `clone()`으로 복제할 수 있습니다. 모든 `Sender`를 `close()`로 닫으면 받는 쪽의 `for` 루프가 끝납니다.

```rok
use rok::std::thread;

let (tx, rx) = channel::<str>();
let workers: List<thread::JoinHandle<()>> = [];
for id in 0..3 {
    let tx = tx.clone();
    workers.push(thread::spawn(|| {
        for j in 0..2 {
            try! tx.send(f"w{id}-{j}");
        }
        tx.close();
    }));
}
tx.close();

let got: List<str> = [];
for msg in rx {
    got.push(msg);
}
for w in workers { try w.join(); }

got.sort();
println(f"{got:?}");
```

```output
["w0-0", "w0-1", "w1-0", "w1-1", "w2-0", "w2-1"]
```

Rok의 값은 GC가 관리하므로 `Sender`는 스코프를 벗어나도 자동으로 닫히지 않습니다. 받는 쪽이 끝났음을 알 수 있도록 보내기를 마친 쪽이 직접 `close()`를 호출해야 합니다. 채널의 용량을 제한하려면 `channel_bounded(n)`을 씁니다.

## `Mutex`와 `Atomic`

여러 스레드가 같은 값을 바꾸면 동기화가 필요합니다. 정수 카운터 하나에는 `Atomic`을 쓰고, 그 밖의 값은 `Mutex`로 감쌉니다. `Mutex::lock()`이 반환하는 가드(guard)는 `using`으로 바인딩해야 합니다. 그래야 블록이 끝날 때 잠금이 풀립니다. [10장](10-errors.html)의 `using`과 같은 규칙입니다.

```rok
use rok::std::thread;

let counter = Atomic::new(0);
let log = Mutex::new([]);

let hs: List<thread::JoinHandle<()>> = [];
for i in 1..=5 {
    hs.push(thread::spawn(|| {
        counter.fetch_add(i);
        using g = log.lock() {
            g.get().push(i * 10);
        }
    }));
}
for h in hs { try h.join(); }

let items = using g = log.lock() { g.get().clone() };   // a copy: the list itself stays behind the lock
items.sort();
println(f"sum = {counter.load()}, items = {items:?}");
```

```output
sum = 15, items = [10, 20, 30, 40, 50]
```

가드를 푸는 것은 `using`뿐이므로 컴파일러는 아무것도 풀지 않는 가드를 거부합니다. 일반 `let`(이나 `if let`)으로 바인딩하거나 `lock()`의 결과를 그냥 버리는 경우가 그렇습니다. `RwLock`의 `read()` / `write()` 가드도 마찬가지입니다.

```rok norun
let g = m.lock();
```

```text check=checks/12-guard-let.rok
error[E200]: `g` holds a `MutexGuard` that is never released: only `using` releases a lock guard, so bind it with `using g = .. { .. }`
```

### 읽기 전용 데이터: `Frozen`

처음 읽을 때는 이 절을 건너뛰어도 됩니다.

설정, 조회 테이블, 신뢰 앵커처럼 한 번 만들고 나서 읽기만 하는 데이터에는 잠금이 필요 없습니다. `Frozen<T>`로 감싸면 어느 스레드든 언제든 읽을 수 있습니다. `Frozen::new`는 자기 복사본을 보관하므로 밖에서는 그 내용을 바꿀 수 없습니다. `get()`은 읽는 쪽마다 따로 복사본을 줍니다. 읽는 쪽이 자기 복사본을 바꿔도 다른 쪽이 보는 값은 바뀌지 않습니다.

```rok
use rok::std::thread;

struct Settings {
    name: str;
    limits: List<i32>;
}

static SETTINGS: Frozen<Settings> = Frozen::new(Settings { name: "demo", limits: [10, 20, 30] });

let hs: List<thread::JoinHandle<i32>> = [];
for i in 0..3 {
    hs.push(thread::spawn(|| {
        let s = SETTINGS.get();      // this thread's own copy
        s.limits.push(i);            // changes the copy only
        s.limits[0] + s.limits.len() as i32
    }));
}
let total = 0;
for h in hs { total += try h.join(); }
let s = SETTINGS.get();
println(f"{s.name} {s.limits:?} {total}");
```

```output
demo [10, 20, 30] 42
```

`get`은 호출할 때마다 값 안의 컬렉션을 복사합니다. 그러니 큰 읽기 전용 부분은 그 복사에서 빼 두세요. `table: Frozen<List<Entry>>`처럼 그 부분에 자기 `Frozen` 필드를 주고, 필요한 곳에서만 읽습니다. `Mutex`나 스스로 잠그는 캐시처럼 이미 `Sync`인 부분은 복사되지 않고 공유됩니다.

## 다른 스레드로 값 넘기기

이 절은 값이 스레드 사이를 옮겨 갈 때 컴파일러가 강제하는 규칙을 다룹니다. 처음 읽을 때는 훑어보고, 컴파일러가 여기를 가리킬 때 다시 돌아와도 됩니다.

`Mutex::new(xs)`, `RwLock::new(xs)`, `tx.send(xs)`는 `xs`를 '넘깁니다'. 그때부터 그 값은 잠금이나 받는 쪽의 것입니다. Rok에는 move 의미론이 없으므로 컴파일러가 대신 넘김을 추적합니다. 넘긴 뒤에 `xs`를 쓰면 지금의 소유자와 경쟁하게 되므로 컴파일 에러입니다. 검사는 복사(`let ys = xs`), struct 필드, 그 값을 담는 객체, 컬렉션의 원소, 클로저 캡처, 함수 호출을 거쳐 값을 따라갑니다.

```rok norun
let xs = [1, 2, 3];
let ys = xs;                  // the same list
let m = Mutex::new(xs);
ys.push(4);
```

```text check=checks/12-handoff-copy.rok
error[E202]: `ys` shares `xs`, which was moved into the `Mutex` at line 3
  --> main.rok:4:1
   |
 3 | let m = Mutex::new(xs);
   |         -------------- `xs` moved into the `Mutex` here
 4 | ys.push(4);
   | ^^
   |
   = help: it belongs to the `Mutex` now; use it through the lock: `using g = m.lock() { g.get().push(4) }`, or hand over `xs.clone()` to keep a separate copy
```

값은 잠금을 거쳐 쓰세요. 내 복사본이 계속 필요하다면 `clone()`을 넘기세요.

원소도 셉니다. 컬렉션에서 읽어 낸 것과 컬렉션 자체는 한 값입니다. 그래서 `rows`를 보낸 뒤 `first = rows[0]`에 원소를 넣는 것도 같은 경쟁입니다.

```text check=checks/12-handoff-element.rok
error[E202]: `first` shares `rows`, which was sent through a channel at line 4
  --> main.rok:5:1
   |
 4 | try! tx.send(rows);
   |      ------------- `rows` sent here
 5 | first.push(9);
   | ^^^^^
   |
   = help: the receiver owns it now; hand over `rows.clone()` to keep a separate copy, or give `first` a new value before using it again
```

원소를 하나씩 내주려면 먼저 하나씩 꺼냅니다. `pop`과 `remove`는 컬렉션에서 원소를 빼내므로 컬렉션은 계속 쓸 수 있습니다. 흔히 쓰는 작업 큐 방식입니다.

```rok
use rok::std::thread;

let jobs = [[3, 1, 2], [9, 7], [5, 4, 6]];
let (tx, rx) = channel::<List<i32>>();
let worker = thread::spawn(|| {
    let total = 0;
    for job in rx {
        job.sort();
        total += job[0];
    }
    total
});
while let job: List<i32> = jobs.pop() {
    try! tx.send(job);         // `pop` took it out of `jobs`, so `jobs` stays usable
}
tx.close();
println(f"smallest items add up to {try worker.join()}, jobs left: {jobs.len()}");
```

```output
smallest items add up to 12, jobs left: 0
```

잠금 가드의 값도 같은 방식으로 동작합니다. `g.get()`이 반환하는 것은 `Mutex`가 보호하는 객체이므로 `using` 블록을 떠날 수 없습니다. 블록이 끝나면 잠금이 더는 그 객체를 지키지 않기 때문입니다.

```text check=checks/12-lock-value.rok
error[E202]: the value of `g`'s `using` block is what the `Mutex` protects, and the block releases the lock before it is used
  --> main.rok:2:10
   |
 2 | let xs = using g = m.lock() { g.get() };
   |          ^^^^^
   |          ----- taken out of the lock here
   |
   = help: use it inside the `using` block, or make the block's value `using.clone()`
```

### 받는 쪽이 받는 것

넘김은 실행 중에도 복사합니다. 새 소유자는 값이 닿는 모든 컬렉션의 자기 복사본을 받습니다. 리스트, 딕셔너리, deque, 문자열 버퍼와 그 위에 만든 컬렉션이 깊이에 상관없이 모두 복사됩니다. 그래서 컴파일러가 보지 못하는 경로라도 두 스레드가 한 리스트를 쥐는 일은 생기지 않습니다. 이를 위해 따로 코드를 쓸 필요는 없습니다. 컴파일러가 받아들이는 프로그램이라면 다음 두 가지 말고는 알아차릴 일이 없습니다.

- **복사하는 양에 비례해 시간이 듭니다.** 숫자, `str`, 컬렉션이 없는 struct는 추가 비용이 없습니다. 평범한 숫자의 리스트는 메모리 복사 한 번이면 되므로 숫자 백만 개의 리스트도 1밀리초가 한참 안 걸립니다. 큰 데이터는 자주 도는 루프에서 조금씩 보내지 말고 한 번에 보내세요.
- **복사본은 자기 공유 관계를 유지합니다.** 보낸 값 안에서 한 리스트를 가리키던 두 핸들은 건너편에서도 한 리스트를 가리킵니다. 슬라이스는 자기가 나온 리스트를 계속 봅니다.

```rok
let data = [1, 2, 3, 4];
let (tx, rx) = channel::<(List<i32>, Slice<i32>)>();
try! tx.send((data, data[0:2]));

if let (list, head): (List<i32>, Slice<i32>) = rx.recv() {
    list[0] = 10;           // the view still sees the list it came from
    println(f"{list:?} {head:?}");
}
```

```output
[10, 2, 3, 4] [10, 2]
```

필드의 기본값(`log: Mutex<List<str>> = Mutex::new([])`)처럼 컴파일 시점에 만든 값은 복사하지 않습니다. 아직 다른 어떤 것도 그 값에 닿을 수 없기 때문입니다.

두 종류의 값은 아예 스레드를 건널 수 없습니다. 복사로도 안전하게 만들 수 없기 때문입니다.

- **`any Tr`에는 `Sync` 바운드가 필요합니다.** `any`의 사본은 그 박스를 공유합니다. 그래서 `any Tr`은 트레잇의 슈퍼트레잇에 `Send`와 `Sync`가 모두 있을 때에만 `Send`입니다. 에러 메시지가 무엇을 바꿀지 알려 줍니다. `an any crosses threads only when its bound has both Send and Sync, since copies share its box: give the trait Sync as a supertrait, or send the concrete type`.

```rok
use rok::std::thread;

trait Shape: Send + Sync { fn area(self) -> f64; }
struct Sq { s: f64; }
impl Shape for Sq { fn area(self) -> f64 { self.s * self.s } }

let (tx, rx) = channel::<any Shape>();
let h = thread::spawn(|| {
    if let s: any Shape = rx.recv() { s.area() } else { 0.0 }
});
try! tx.send(Sq { s: 3.0 });
println(f"{try h.join()}");
```

```output
9
```

- **`Weak`는 자기 스레드에 머뭅니다.** 넘김은 `Weak`(`rok::bare::gc`)가 가리키는 것을 복사할 수 없으므로 `Weak<T>`는 `Send`가 아닙니다. 대신 `GcPtr`를 보내세요.

`unsafe` 안에서는 컴파일러가 넘김 검사를 멈춥니다. 보낸 뒤에 값을 쓰지 않도록 지키는 것은 작성자의 책임입니다. 실행 중 복사는 여전히 일어납니다. `Handoff::new`와 `Mutex::new_unshared`는 값을 복사 '없이' 넘기는 두 가지 `unsafe` 방법입니다. 방금 만들어 다른 어떤 것도 닿지 않는 값에 씁니다. 가드에서 꺼내 `using` 블록 너머까지 쥐고 있는 핸들과 한 리스트를 공유하는 `par` 항목은 런타임이 대신 복사해 주지 않습니다. 그곳에서는 컴파일러의 검사가 유일한 보호이므로 `unsafe`로 검사를 끄지 마세요.

## `async`와 task

I/O처럼 기다림이 긴 작업에는 작업마다 스레드를 붙잡지 않는 `async`가 낫습니다. `async fn`을 호출하면 바로 future가 반환되고, `await`가 그 결과를 기다립니다. `task::spawn`으로 future를 별도 task로 띄우면 여러 task가 함께 실행됩니다. `task::join`은 모든 결과를 **띄운 순서대로** 모읍니다.

```rok
async fn fetch(id: i32) -> str {
    await task::sleep(Duration::ms((40 - id * 10) as i64));
    f"item-{id}"
}

let tasks = [task::spawn(fetch(1)), task::spawn(fetch(2)), task::spawn(fetch(3))];
let items = await task::join(tasks);
println(f"{items:?}");

let f = fetch(3);
println("not started waiting yet");
println(await f);
```

```output
["item-1", "item-2", "item-3"]
not started waiting yet
item-3
```

`await`는 `-`나 `!`만큼 강하게 결합하는 전위 연산자입니다. 바로 뒤의 호출에 적용되고, 식의 나머지는 그 결과에 적용됩니다. `await answer() == 42`는 기다려 얻은 값을 비교합니다. 비교 결과를 기다리는 것이 아닙니다.

```rok
async fn answer() -> i32 { 42 }

println(f"{await answer() == 42} {await answer() + 1}");
let r = try await task::timeout(Duration::ms(50), task::spawn(answer()));
println(f"{r}");
```

```output
true 43
42
```

마지막 줄에서 보듯 `try await f()`는 왼쪽에서 오른쪽으로 읽습니다. future를 기다린 다음 그 에러를 전파합니다.

최상위 문장에 `await`가 있으면 프로그램 전체가 `async fn main`으로 실행되므로 스크립트에서 `await`를 바로 쓸 수 있습니다. 일반 `fn`의 본체에서는 `await`를 쓸 수 없습니다.

시간 제한은 `task::timeout`으로 둡니다. 시간이 다 되면 `TimeoutError`를 던집니다.

```rok
async fn slow() -> i32 {
    await task::sleep(Duration::ms(200));
    1
}

let r = try await task::timeout(Duration::ms(10), task::spawn(slow()))
    catch e: TimeoutError { -1 };
println(f"{r}");
```

```output
-1
```

채널은 async 코드에서도 동작합니다. `for await`는 task를 막지 않고 값을 기다립니다.

```rok
let (tx, rx) = channel::<i32>();
let producer = task::spawn(async {
    for i in 1..=3 {
        try! tx.send(i);
        await task::sleep(Duration::ms(5));
    }
    tx.close();
    "producer done"
});

let sum = 0;
for await v in rx {
    println(f"got {v}");
    sum += v;
}
println(f"{await producer}, sum = {sum}");
```

```output
got 1
got 2
got 3
producer done, sum = 6
```

`task::spawn`은 채널에 값을 보낼 때와 같은 방식으로 future를 task에 넘깁니다. task는 future가 캡처한 리스트와 다른 컬렉션의 자기 복사본으로 작업합니다. 그 뒤에 task를 띄운 쪽 코드에서 그중 하나를 쓰면 에러입니다. task가 한 변경이 그쪽에는 보이지 않기 때문입니다.

```rok norun
async fn total(xs: List<i32>) -> i32 {
    let t = 0;
    for x in xs { t += x; }
    t
}

let data = [1, 2, 3];
let t = task::spawn(total(data));
data.push(4);
```

```text check=checks/12-task-handoff.rok
error[E202]: `data` was spawned as a task at line 8
 9 | data.push(4);
   | ^^^^
   = help: the task works on its own copy, so changes it makes are not seen here: return them from the task (`await`), hand over `data.clone()` to keep a separate copy, or share read-only data through `Frozen`
```

task가 바꾼 것은 반환하게 하고 `await`로 돌려받습니다.

```rok
async fn grow(xs: List<i32>) -> List<i32> {
    xs.push(4);
    xs
}

let data = [1, 2, 3];
let t = task::spawn(grow(data));
data = await t;
println(f"{data:?}");
```

```output
[1, 2, 3, 4]
```

## `par`: 구조적 병렬성

많은 데이터 조각에 같은 계산을 적용할 때는 스레드를 직접 관리할 필요가 없습니다. `rok::core::par`는 작업을 워커 풀에 나누어 주고, 함수가 반환될 때에는 모든 작업이 끝나 있습니다. `map`과 `filter`는 입력 순서를 유지합니다.

```rok
fn is_prime(n: i32) -> bool {
    if n < 2 { return false; }
    let d = 2;
    while d * d <= n {
        if n % d == 0 { return false; }
        d += 1;
    }
    true
}

let squares = par::map(1..=8, |x| x * x);
println(f"{squares:?}");

let count = Atomic::new(0);
par::for_each(1..=10000, |n| {
    if is_prime(n) { count.fetch_add(1); }
});
println(f"primes below 10000: {count.load()}");

let evens = par::filter([1, 2, 3, 4, 5, 6], |x| x % 2 == 0);
let total = par::reduce([1, 2, 3, 4], 0, |a, b| a + b);
println(f"{evens:?} {total}");
```

```output
[1, 4, 9, 16, 25, 36, 49, 64]
primes below 10000: 1229
[2, 4, 6] 10
```

`reduce`는 조각마다 따로 접은 뒤 결과를 합칩니다. 그래서 결과가 결정적이려면 결합 법칙을 만족하는 함수(`+`, `max` 등)를 넘겨야 합니다.

작업이 정확히 두 개면 `par::join`을, 개수가 실행 시점에 정해지면 `par::scope`를 씁니다. scope 안에서 `spawn`으로 띄운 모든 작업은 scope가 끝나기 전에 끝납니다.

```rok
let (a, b) = par::join(|| (1..=100).sum(), || "done");
println(f"{a} {b}");

par::scope(|s| {
    let left = s.spawn(|| (1..=50).sum());
    let right = s.spawn(|| (51..=100).sum());
    println(f"{left.join() + right.join()}");
});
```

```output
5050 done
5050
```

`par`는 항목을 하나씩 워커에 주므로 두 항목이 리스트나 다른 컬렉션을 공유하면 안 됩니다. 두 워커가 그 컬렉션을 동시에 바꿀 수 있기 때문입니다. `par`는 시작하기 전에 항목이 서로 다른 컬렉션에 닿는지 확인합니다. 두 항목이 하나를 공유하면 그 항목은 호출한 스레드에서 하나씩 차례로 실행됩니다. 결과는 올바르지만 더는 병렬이 아닙니다. 이때 `par`는 그 호출에서 한 번 경고합니다.

```rok norun
let shared = [1, 2, 3];
par::for_each([shared, shared], |xs| { xs.sort(); });
```

```text
warning: par::for_each at main.rok:2:1: two items reach one collection or future, so they run one at a time on this thread; give each item its own copy, or share read-only data through `Frozen` (stdlib 2.19.1)
```

한 리스트의 서로 다른 부분을 덮는 슬라이스는 공유로 치지 않습니다. 그래서 큰 리스트를 조각으로 나누어도 병렬로 실행됩니다. 이 검사는 항목이 닿는 모든 컬렉션을 방문하므로 호출마다 컬렉션당 약간의 비용이 듭니다.

## 스트림

`async yield fn`은 값을 하나씩 비동기로 내놓는 스트림을 만듭니다. 제너레이터의 비동기 판이라고 생각하면 됩니다. `for await`로 소비합니다.

```rok
async yield fn ticks(n: i32) -> i32 {
    for i in 1..=n {
        await task::sleep(Duration::ms(5));
        yield i * 10;
    }
}

for await t in ticks(3) {
    println(f"tick {t}");
}
```

```output
tick 10
tick 20
tick 30
```

`Stream` 트레잇에는 `map`, `filter`, `take`, `throttle` 같은 결합자가 있습니다([`Stream` API](../../std/core/stream.html#trait.Stream)).

## 함정

**`[ref x]` 캡처는 스레드를 건널 수 없습니다.** `[ref x]` 셀([3장](03-functions.html))에는 동기화가 없습니다. 그래서 `thread::spawn`, `task::spawn`, `par::*`에 주는 클로저에서는 컴파일러가 이 캡처를 거부합니다. 이런 클로저에서는 두 스레드가 셀을 동시에 바꿀 수 있기 때문입니다.

```text check=checks/12-ref-thread.rok
error[E200]: `[ref count]` can't be used in a closure that may run on another thread: both threads could change `count` at once
   = help: write `let count = Atomic::new(count);` before the closure, capture `count` without `ref`, and use `count.load()` / `count.store(..)`; or return the value from the closure instead
```

대신 `Atomic`(숫자)이나 `Mutex`(그 밖의 것)를 공유하세요. 스레드에서 값을 반환하게 하고 `join` 뒤에 결과를 합쳐도 됩니다.

```rok
use rok::std::thread;

let count = Atomic::new(0);
let hs: List<thread::JoinHandle<()>> = [];
for _ in 0..4 {
    hs.push(thread::spawn(|| {
        for _ in 0..10000 { count.fetch_add(1); }
    }));
}
for h in hs { try h.join(); }
println(f"{count.load()}");
```

```output
40000
```

**`static`은 `Sync`여야 합니다.** `static`은 모든 스레드에서 보이므로 그 타입은 공유해도 안전해야 합니다. `static XS: List<i32> = [];`는 에러입니다. 리스트를 `Mutex`로 감싸세요.

```rok
static SEEN: Mutex<List<i32>> = Mutex::new([]);

fn record(x: i32) {
    using g = SEEN.lock() { g.get().push(x); }
}

record(1);
record(2);
println(f"{using g = SEEN.lock() { g.get().clone() }:?}");
```

```output
[1, 2]
```

**기다리지 않은 스레드는 `main`이 반환될 때 죽습니다.** 프로그램은 `main`이 끝날 때 끝나고, 아직 실행 중인 스레드는 도중에 함께 멈춥니다. `JoinHandle`을 보관해 두고 `join`으로 기다리세요. `rok check`와 `rok lint`는 한 번도 기다리지 않는 핸들을 경고합니다.

```text check=checks/12-unjoined.rok
warning[E503]: this thread is never joined, so the program may end before it finishes
  --> main.rok:3:1
   |
 3 | thread::spawn(|| println("hi"));
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = help: join it with `try thread::spawn(|| println("hi")).join()`, or write `thread::spawn(|| println("hi")).detach()` to let it run on its own
```

스레드의 결과에 더는 관심이 없고 스레드가 계속 살아 있어야 한다면 핸들에 `detach()`를 호출합니다.

```rok
use rok::std::thread;

let t = thread::spawn(|| {
    thread::sleep(Duration::ms(10));
    println("thread done");
});
println("main done");
try t.join();               // without this line, "thread done" may never print
```

```output
main done
thread done
```

**future는 기다려야 실행됩니다.** `async fn`을 호출하면 future가 만들어질 뿐입니다. 본체는 `await`로 기다리거나 `task::spawn`에 넘길 때 실행됩니다. 한 번도 기다리지 않은 future는 실행되지 않습니다. 에러는 아니지만 `rok check`와 `rok lint`가 `the future is never awaited, so its body never runs`라고 경고하고 `await f`를 쓰라고 제안합니다. `task::spawn`이 준 `Task`는 버려도 경고가 없습니다. 띄운 task는 스스로 실행됩니다(fire-and-forget).

```rok
async fn work() -> i32 {
    println("work ran");
    1
}

let f = work();             // nothing runs yet
println("before await");
println(f"{await f}");
```

```output
before await
work ran
1
```

**`task::spawn`에는 async `main`이 필요합니다.** task는 executor 위에서 실행되고, executor는 `async fn main`이 시작합니다. 스크립트는 최상위에 `await`가 하나라도 있으면 그렇게 됩니다. 그렇지 않으면 `task::spawn`에서 panic이 일어납니다.

```text
thread 'main' panicked at .../library/core/src/task.rok:703:18:
no task executor is running: await inside `async fn main` or `task::block_on`
```

그러니 위의 예제처럼 띄운 task는 `await`로 기다리세요. 일반 `fn`에서 async 코드를 부르려면 future를 끝까지 실행하고 그 값을 반환하는 `task::block_on`을 씁니다.

```rok
async fn work() -> i32 { 7 }

fn compute() -> i32 {
    task::block_on(work())
}

println(f"{compute()}");
```

```output
7
```

## 정리

- `thread::spawn`은 OS 스레드를 시작합니다. 결과는 `try h.join()`으로 얻습니다.
- 채널은 `channel()`로 만들고, 각 `Sender`는 보내기를 마치면 직접 `close()`로 닫습니다.
- `rok::core`(`task`, `par`, `channel`, `Mutex`, `Atomic`)는 언제나 스코프에 있고, `rok::std::thread`에는 `use`가 필요합니다.
- 공유하는 값은 `Atomic`이나 `Mutex`로 보호합니다. 잠금 가드는 `using`으로 바인딩해야 하고, 다른 방법은 컴파일 에러입니다. `[ref x]` 캡처는 스레드를 건널 수 없고, `static`은 `Sync`여야 합니다.
- `Mutex::new`나 `send`에 넘긴 값은 새 소유자의 것입니다. 그 뒤에 그 값이나 그 원소를 쓰면 컴파일 에러(`E202`)입니다. 잠금을 거쳐 쓰거나, `clone()`을 넘기거나, 보내기 전에 `pop` / `remove`로 원소를 꺼냅니다. `g.get()`이 반환한 것은 `using` 블록 안에 머뭅니다.
- 받는 쪽은 값에 든 컬렉션을 복사해서 받습니다. 복사본은 공유 관계를 유지하고, 복사에는 크기에 비례해 시간이 듭니다. `any Tr`은 트레잇에 `Send + Sync`가 있을 때에만 스레드를 건너고, `Weak`는 결코 건너지 않습니다.
- 읽기만 하는 데이터는 `Frozen<T>`로 공유합니다. `get()`은 매번 읽는 쪽에 따로 복사본을 주므로 큰 읽기 전용 부분은 자기 `Frozen` 필드에 둡니다.
- `await`는 뒤따르는 호출에 붙는 전위 연산자입니다(`await f() == x`는 결과를 비교합니다). 기다림이 많은 작업을 동시에 실행하려면 `async fn` / `await`와 `task::spawn` / `task::join`을 씁니다.
- `task::spawn`은 send처럼 future를 넘깁니다. task는 future가 캡처한 것의 자기 복사본으로 작업하므로 변경 결과는 `await`로 돌려받습니다.
- 스레드는 join으로, future는 await로 기다립니다. 기다리지 않은 스레드는 `main`과 함께 죽고, future는 기다릴 때에만 실행됩니다. 두 경우 모두 린트가 경고합니다. 버린 `task::spawn` task는 그래도 실행됩니다. `task::spawn`에는 async `main`(또는 `task::block_on`)이 필요합니다.
- 데이터 병렬 계산에는 `par::map`, `par::for_each`, `par::join`, `par::scope`가 간단합니다. 컬렉션을 공유하는 항목은 경고와 함께 하나씩 실행됩니다.
- 스트림은 `async yield fn`으로 만들고 `for await`로 소비합니다.

## 더 읽을거리

- 사양: [비동기(`async` / `await`)](../spec/refs-lang/004-functions/412-async.html), [연산자 우선순위](../spec/refs-lang/003-expressions/302-precedence-and-associativity.html), [스트림](../spec/refs-lang/004-functions/413-stream.html), [`using`](../spec/refs-lang/003-expressions/317-using-expression.html), [스레드를 건너는 캡처](../spec/refs-lang/004-functions/406-capture-semantics.html), [`static`](../spec/refs-lang/005-modules/503-static.html)
- 표준 라이브러리 사양: [sync](../spec/refs-standard-libraries/200-core/216-sync.html), [channel](../spec/refs-standard-libraries/200-core/217-channel.html), [task](../spec/refs-standard-libraries/200-core/218-task.html), [par](../spec/refs-standard-libraries/200-core/219-par.html)
- API: [`thread::spawn`](../../std/std/thread.html#fn.spawn), [`task::spawn`](../../std/core/task.html#fn.spawn), [`task::join`](../../std/core/task.html#fn.join), [`channel`](../../std/core/channel.html#fn.channel), [`Mutex`](../../std/core/sync.html#struct.Mutex), [`Atomic`](../../std/core/sync.html#struct.Atomic), [`par::map`](../../std/core/par.html#fn.map), [`par::scope`](../../std/core/par.html#fn.scope)
- 다음 장: [13. 파일과 네트워크 IO](13-io.html)
