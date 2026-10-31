# 동시 다운로더

워커 task 풀이 가상의 자원을 나눠 내려받습니다. 네트워크 대신 `task::sleep`으로 지연을 흉내 냅니다. 결과는 채널로 모으고 정렬한 보고서를 출력합니다.

## 코드

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

## 설명

- 채널은 송신자도 수신자도 여럿일 수 있습니다(MPMC). `Receiver`를 복사하면 같은 큐를 함께 쓰므로, 워커마다 작업 수신자 하나의 복사본을 받고, URL은 쉬고 있다가 먼저 가져가는 워커에게 갑니다([2.17.4 다중 생산자와 다중 소비자](../spec/refs-standard-libraries/200-core/217-channel.html#2174-다중-생산자와-다중-소비자), [`channel`](../../std/core/channel.html#fn.channel)).
- 워커마다 `clone()`으로 자기 송신자를 받고 일이 끝나면 `close()`로 닫습니다. `for await r in done_rx`는 모든 송신자가 닫혀야 끝나므로 `main`은 원래의 `done_tx`를 일찍 닫습니다([동시성](../learn/12-concurrency.html)).
- `await jobs.recv_async()`는 값이 올 때까지 스레드가 아니라 task를 멈춥니다. `main`이 `job_tx`를 닫고 큐가 비면 `none`을 내고, `let-else` 안의 `break`가 워커 루프를 빠져나갑니다.
- `task::timeout`은 task가 제때 끝나지 않으면 `TimeoutError`를 던집니다. 타임아웃이 task를 멈추지는 않으므로 `t.cancel()`로 명시적으로 취소합니다([`task::timeout`](../../std/core/task.html#fn.timeout), [2.18.1 핵심 API](../spec/refs-standard-libraries/200-core/218-task.html#2181-핵심-api)).
- 결과가 도착하는 순서는 실행할 때마다 다를 수 있습니다. 그래서 출력하기 전에 URL로 정렬합니다. 보고서는 언제나 같습니다.
