# Concurrent downloader

A pool of worker tasks shares out simulated resources (latency faked with `task::sleep` instead of a network), results come back over a channel, and a sorted report is printed.

## Code

```rok
// A "concurrent downloader": a pool of worker tasks fetches simulated resources.

enum FetchError {
    NotFound,
    Timeout,
}

enum Status {
    Done(usize),
    Failed(FetchError),
}

struct Outcome {
    url: str;
    status: Status;
}

const WORKERS: i32 = 3;
const DEADLINE_MS: i64 = 250;

// Pretend to download `url`: latency and size are derived from the name.
async fn fetch(url: str) throws (FetchError) -> usize {
    let latency = if url.contains("slow") { 5000 } else { (url.len() * 7 % 40) as i64 };
    await task::sleep(Duration::ms(latency));
    if url.ends_with(".missing") { throw FetchError::NotFound; }
    url.len() * 1024
}

// Fetch with a deadline; a fetch that takes too long is cancelled.
async fn fetch_with_deadline(url: str) -> Status {
    let t = task::spawn(async {
        Status::Done(try await fetch(url) catch e { return Status::Failed(e); })
    });
    try await task::timeout(Duration::ms(DEADLINE_MS), t) catch _ {
        t.cancel();
        Status::Failed(FetchError::Timeout)
    }
}

async fn main() {
    let urls = [
        "https://example.org/index.html", "https://example.org/logo.png",
        "https://example.org/old.missing", "https://example.org/style.css",
        "https://example.org/slow/video.mp4", "https://example.org/app.js",
        "https://example.org/data.json", "https://example.org/about.html",
    ];

    // A channel is multi-producer, multi-consumer: every worker holds a copy
    // of one job receiver, and each URL goes to whichever worker takes it first.
    let (job_tx, job_rx) = channel::<str>();
    let (done_tx, done_rx) = channel::<Outcome>();
    for _ in 0..WORKERS {
        let jobs = job_rx;
        let done = done_tx.clone();
        task::spawn(async {
            loop {
                let url: str = await jobs.recv_async() else { break; };
                try! done.send(Outcome { url: url, status: await fetch_with_deadline(url) });
            }
            done.close();
        });
    }
    done_tx.close();

    // Queue every URL, then close the channel so idle workers stop.
    for url in urls { try! job_tx.send(url); }
    job_tx.close();

    // Collect until every worker has closed its `done` sender.
    let results: List<Outcome> = [];
    for await r in done_rx { results.push(r); }
    results.sort_by(|a: Outcome, b: Outcome| a.url.cmp(b.url));

    let ok = 0;
    let bytes: usize = 0;
    for r in results {
        let name = r.url.strip_prefix("https://example.org/") ?? r.url;
        match r.status {
            Status::Done(n) => { ok += 1; bytes += n; println(f"{name:<16} {n:>7} B"); },
            Status::Failed(FetchError::NotFound) => println(f"{name:<16}   404 not found"),
            Status::Failed(FetchError::Timeout) => println(f"{name:<16}   timed out"),
        }
    }
    println(f"{ok}/{results.len()} downloaded, {bytes} bytes total");
}
```

```output
about.html         30720 B
app.js             26624 B
data.json          29696 B
index.html         30720 B
logo.png           28672 B
old.missing        404 not found
slow/video.mp4     timed out
style.css          29696 B
6/8 downloaded, 176128 bytes total
```

## Walkthrough

- A channel has many senders and many receivers (MPMC). A copy of a `Receiver` shares its queue, so every worker takes a copy of the one job receiver, and each URL goes to whichever idle worker takes it first ([2.17.4 multiple producers and consumers](../spec/refs-standard-libraries/200-core/217-channel.html#2174-multiple-producers-and-consumers), [`channel`](../std/core/channel.html#fn.channel)).
- Each worker gets its own sender via `clone()` and `close()`s it when done. `for await r in done_rx` ends only once every sender is closed, so `main` closes its original `done_tx` early ([Concurrency](../learn/12-concurrency.html)).
- `await jobs.recv_async()` suspends the task, not the thread, until a value arrives. Once `main` closes `job_tx` and the queue is empty it yields `none`, and the `break` in the `let-else` leaves the worker loop.
- `task::timeout` throws `TimeoutError` if the task does not finish in time. A timeout does not stop the task, so it is cancelled explicitly with `t.cancel()` ([`task::timeout`](../std/core/task.html#fn.timeout), [2.18.1 core API](../spec/refs-standard-libraries/200-core/218-task.html#2181-core-api)).
- Results may arrive in a different order on every run, so they are sorted by URL before printing and the report is always the same.
