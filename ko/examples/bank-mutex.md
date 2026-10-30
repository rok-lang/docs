# 은행 송금

여러 스레드가 `Mutex`로 보호되는 계좌 사이에서 돈을 옮기고, `Atomic` 카운터에 통계를 모은 뒤, 마지막에 총액이 한 번도 바뀌지 않았는지 확인합니다.

## 코드

```rok
// Threads move money between accounts; locks keep the total constant.
use rok::std::thread;

const ACCOUNTS: usize = 5;
const THREADS: i32 = 4;
const TRANSFERS: i32 = 20000;
const START: i64 = 1000;

// A tiny deterministic pseudo-random generator (one per thread).
struct Rng { state: u64; }

impl Rng {
    fn next(ref self, bound: u64) -> u64 {
        self.state = self.state.wrapping_mul(6364136223846793005u64).wrapping_add(1442695040888963407u64);
        (self.state >> 33u64) % bound
    }
}

struct Bank {
    // A fixed array, not a List: a List is a shared mutable handle and is
    // not `Sync`, so threads may not share one. The array's elements are
    // `Mutex` handles, which are.
    accounts: [Mutex<i64>; 5];
    moved: Atomic<i64>;
    rejected: Atomic<i64>;
}

impl Bank {
    fn new() -> Bank {
        let accounts: [Mutex<i64>; 5] = [
            Mutex::new(START), Mutex::new(START), Mutex::new(START), Mutex::new(START),
            Mutex::new(START),
        ];
        Bank { accounts: accounts, moved: Atomic::new(0i64), rejected: Atomic::new(0i64) }
    }

    // Lock both accounts, always the lower index first, so two opposite
    // transfers can never wait for each other forever.
    fn transfer(self, from: usize, to: usize, amount: i64) {
        if from == to { return; }
        let first = if from < to { from } else { to };
        let second = if from < to { to } else { from };
        using a = self.accounts[first].lock() {
            using b = self.accounts[second].lock() {
                let (src, dst) = if from == first { (a, b) } else { (b, a) };
                if src.get() < amount {
                    self.rejected.fetch_add(1i64);
                } else {
                    src.set(src.get() - amount);
                    dst.set(dst.get() + amount);
                    self.moved.fetch_add(amount);
                }
            }
        }
    }

    // Sum of all balances (called while no transfers are running).
    fn total(self) -> i64 {
        let sum: i64 = 0;
        for m in self.accounts {
            using g = m.lock() { sum += g.get(); }
        }
        sum
    }
}

fn main() {
    let bank = Bank::new();
    let before = bank.total();

    let handles: List<thread::JoinHandle<i32>> = [];
    for t in 0..THREADS {
        handles.push(thread::spawn(|| {
            let rng = Rng { state: t as u64 + 1u64 };
            for _ in 0..TRANSFERS {
                let from = rng.next(ACCOUNTS as u64) as usize;
                let to = rng.next(ACCOUNTS as u64) as usize;
                bank.transfer(from, to, rng.next(300u64) as i64 + 1);
            }
            TRANSFERS
        }));
    }
    let attempted = 0;
    for h in handles { attempted += try! h.join(); }

    let after = bank.total();
    println(f"{THREADS} threads attempted {attempted} transfers");
    println(f"money moved: {bank.moved.load() > 0}, some rejected for lack of funds: {bank.rejected.load() > 0}");
    for i in 0..ACCOUNTS {
        using g = bank.accounts[i].lock() {
            let ok = if g.get() >= 0 { "ok" } else { "NEGATIVE" };
            println(f"account {i}: balance {ok}");
        }
    }
    println(f"total before {before}, after {after}, invariant holds: {before == after}");
}
```

```output
4 threads attempted 80000 transfers
money moved: true, some rejected for lack of funds: true
account 0: balance ok
account 1: balance ok
account 2: balance ok
account 3: balance ok
account 4: balance ok
total before 5000, after 5000, invariant holds: true
```

## 설명

- 계좌마다 `Mutex<i64>`를 하나씩 두고 고정 배열에 담습니다. 여기서 `List`를 쓰면 컴파일되지 않습니다. 리스트는 공유되는 가변 핸들이고 `Sync`가 아니므로, 스레드 클로저가 캡처할 수 없습니다([4.6.9](../spec/refs-lang/004-functions/406-capture-semantics.html)). 잠금은 `using g = m.lock() { ... }`로 얻고 블록이 끝날 때 풀립니다. 보호되는 값은 `get()` / `set()`으로 읽고 씁니다([동시성](../learn/12-concurrency.html), [3.17 `using`](../spec/refs-lang/003-expressions/317-using-expression.html), [`Mutex`](../../std/core/sync.html#struct.Mutex)).
- 송금은 두 계좌를 모두 잠가야 합니다. 두 스레드가 A→B와 B→A를 동시에 실행하면서 서로 반대 순서로 잠그면 서로를 영원히 기다릴 수 있습니다. 그래서 번호가 작은 계좌를 항상 먼저 잠급니다.
- 옮긴 금액이나 거절 횟수처럼 서로 독립된 카운터는 잠금 없이 `Atomic<i64>`의 `fetch_add`로 셉니다([`Atomic`](../../std/core/sync.html#struct.Atomic), [2.16 sync](../spec/refs-standard-libraries/200-core/216-sync.html)).
- 스레드마다 자기 시드를 가진 작은 의사 난수 생성기를 씁니다. `wrapping_mul` / `wrapping_add`를 쓰면 곱셈 오버플로가 panic을 일으키지 않습니다.
- 계좌별 잔액은 스레드가 어떻게 교차 실행되는지에 따라 달라집니다. 하지만 송금이 원자적이라면 총액은 언제나 같습니다. 출력에는 그 순서와 무관한 사실만 담습니다.
