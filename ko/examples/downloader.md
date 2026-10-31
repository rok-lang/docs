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

    // Channels are multi-producer, single-consumer: idle workers announce
    // themselves on `ready`, the dispatcher answers on that worker's own channel.
    let (ready_tx, ready_rx) = channel::<i32>();
    let (done_tx, done_rx) = channel::<Outcome>();
    let jobs: List<Sender<str>> = [];
    for w in 0..WORKERS {
        let (job_tx, job_rx) = channel::<str>();
        jobs.push(job_tx);
        let ready = ready_tx.clone();
        let done = done_tx.clone();
        task::spawn(async {
            loop {
                try! ready.send(w);
                let url: str = await job_rx.recv_async() else { break; };
                try! done.send(Outcome { url: url, status: await fetch_with_deadline(url) });
            }
            ready.close();
            done.close();
        });
    }
    ready_tx.close();
    done_tx.close();

    // Dispatch every URL to the next idle worker, then tell the workers to stop.
    for url in urls {
        let w: i32 = await ready_rx.recv_async() else { break; };
        try! jobs[w as usize].send(url);
    }
    for j in jobs { j.close(); }

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

- 채널은 송신자가 여럿이고 수신자는 하나입니다(mpsc). `Receiver`는 복제할 수 없으므로 워커가 작업 채널 하나를 함께 쓰지 않습니다. 대신 쉬고 있는 워커가 `ready` 채널로 자기 번호를 알리면 분배자가 그 워커 전용 채널로 URL을 보냅니다([2.17.4 다중 생산자와 다중 소비자](../spec/refs-standard-libraries/200-core/217-channel.html#2174-다중-생산자와-다중-소비자), [`channel`](../../std/core/channel.html#fn.channel)).
- 워커마다 `clone()`으로 자기 송신자를 받고 일이 끝나면 `close()`로 닫습니다. `for await r in done_rx`는 모든 송신자가 닫혀야 끝나므로 `main`은 원래의 `done_tx`를 일찍 닫습니다([동시성](../learn/12-concurrency.html)).
- `await job_rx.recv_async()`는 값이 올 때까지 스레드가 아니라 task를 멈춥니다. 채널이 닫히면 `none`을 내고, `let-else` 안의 `break`가 워커 루프를 빠져나갑니다.
- `task::timeout`은 task가 제때 끝나지 않으면 `TimeoutError`를 던집니다. 타임아웃이 task를 멈추지는 않으므로 `t.cancel()`로 명시적으로 취소합니다([`task::timeout`](../../std/core/task.html#fn.timeout), [2.18.1 핵심 API](../spec/refs-standard-libraries/200-core/218-task.html#2181-핵심-api)).
- 결과가 도착하는 순서는 실행할 때마다 다를 수 있습니다. 그래서 출력하기 전에 URL로 정렬합니다. 보고서는 언제나 같습니다.
