# Bank transfers

Several threads move money between accounts guarded by `Mutex`, collect statistics in `Atomic` counters, and finally check that the total never changed.

## Code

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

## Walkthrough

- Each account is its own `Mutex<i64>`, kept in a fixed array. A `List` would not compile here: lists are shared mutable handles and are not `Sync`, so a thread closure may not capture one ([4.6.9](../spec/refs-lang/004-functions/406-capture-semantics.html)). The lock is taken with `using g = m.lock() { ... }` and released when the block ends; the guarded value is read and written with `get()` / `set()` ([Concurrency](../learn/12-concurrency.html), [3.17 `using`](../spec/refs-lang/003-expressions/317-using-expression.html), [`Mutex`](../std/core/sync.html#struct.Mutex)).
- A transfer must lock both accounts. If two threads ran A→B and B→A at the same time and locked in opposite order, they could wait for each other forever, so the lower-numbered account is always locked first.
- Independent counters, such as the amount moved and the number of rejections, are counted without a lock using `fetch_add` on `Atomic<i64>` ([`Atomic`](../std/core/sync.html#struct.Atomic), [2.16 sync](../spec/refs-standard-libraries/200-core/216-sync.html)).
- Each thread uses a tiny pseudo-random generator with its own seed; `wrapping_mul` / `wrapping_add` keep multiplication overflow from panicking.
- The balance of each account depends on how the threads interleave, but if transfers are atomic the total is always the same. The output contains only facts that do not depend on that order.
