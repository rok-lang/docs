# TCP 에코 서버

프로그램 하나가 비어 있는 `127.0.0.1` 포트에 에코 서버를 띄우고, 클라이언트가 줄을 보내 에코를 다시 읽게 한 다음, 모든 것을 깔끔하게 종료합니다.

## 코드

```rok
// A line-based TCP echo server and its clients in one program.
use rok::std::net::{TcpListener, TcpStream, SocketAddr, Shutdown};
use rok::std::io::{IoError, BufReader, Reader};
use rok::std::thread;

// Echo every line of one connection back until the client closes its side.
fn serve(conn: TcpStream) throws (IoError) -> i32 {
    let lines = 0;
    let r = BufReader::new(try conn.try_clone());
    for item in r.lines() {
        let line = try item.get();
        try conn.write_str(f"echo: {line}\n");
        lines += 1;
    }
    // Close both handles (the clone shares the socket) so the client sees EOF.
    r.into_inner().close();
    conn.close();
    lines
}

// Send each message, wait for its echo, then close the write side.
fn client(id: i32, addr: SocketAddr, msgs: List<str>) throws (IoError) {
    let conn = try TcpStream::connect(addr);
    let r = BufReader::new(try conn.try_clone());
    for m in msgs {
        try conn.write_str(f"{m}\n");
        let reply = (try r.read_line()).trim_end();
        println(f"client {id}: sent {m:?}, got {reply:?}");
    }
    try conn.shutdown(Shutdown::Write);
    // An empty read means the server closed the connection.
    if (try r.read_line()).is_empty() { println(f"client {id}: closed"); }
}

fn main() throws (IoError) {
    // Port 0: the OS picks a free port; local_addr tells us which.
    let listener = try TcpListener::bind(try! "127.0.0.1:0".parse::<SocketAddr>());
    let addr = try listener.local_addr();
    let clients = 3;

    // The server thread accepts `clients` connections, one handler thread each.
    let server = thread::spawn(|| {
        let handlers: List<thread::JoinHandle<i32>> = [];
        for _ in 0..clients {
            let (conn, _peer) = try! listener.accept();
            handlers.push(thread::spawn(|| try serve(conn) catch e { println(f"server: {e}"); 0 }));
        }
        let total = 0;
        for h in handlers { total += try! h.join(); }
        listener.close();
        total
    });

    try client(1, addr, ["hello", "rok"]);
    try client(2, addr, ["one line"]);
    try client(3, addr, ["a", "b", "c"]);

    let served = try! server.join();
    println(f"server echoed {served} lines and stopped");
}
```

```output
client 1: sent "hello", got "echo: hello"
client 1: sent "rok", got "echo: rok"
client 1: closed
client 2: sent "one line", got "echo: one line"
client 2: closed
client 3: sent "a", got "echo: a"
client 3: sent "b", got "echo: b"
client 3: sent "c", got "echo: c"
client 3: closed
server echoed 6 lines and stopped
```

## 설명

- `TcpListener` 를 포트 0 에 바인딩하면 OS 가 빈 포트를 고르고, `local_addr()` 가 실제 주소를 알려 줍니다. 그래서 다른 프로그램과 충돌하지 않습니다 ([3.2 net](../spec/refs-standard-libraries/300-std/302-net.html), [`TcpListener`](../../std/std/net.html#struct.TcpListener)).
- 서버 스레드는 연결마다 처리 스레드를 하나씩 띄운 뒤, 모두 `join` 해서 처리한 줄 수를 더합니다. 클로저는 `listener` 와 `conn` 을 캡처해서 다른 스레드로 가져갑니다 ([동시성](../learn/12-concurrency.html), [`thread::spawn`](../../std/std/thread.html#fn.spawn)).
- 줄 단위로 읽기 위해 소켓을 `try_clone()` 하고 `BufReader` 로 감쌉니다. 읽기는 복제본을, 쓰기는 원본을 거칩니다 ([`BufReader`](../../std/std/io.html#struct.BufReader)).
- 종료 순서가 중요합니다. 클라이언트는 모두 보낸 뒤 `shutdown(Shutdown::Write)` 로 "더 보낼 것 없음" 을 알리고, 서버의 `lines()` 는 그 신호로 루프를 끝냅니다. 서버가 두 핸들을 모두 닫으면 클라이언트의 `read_line()` 은 빈 문자열을 돌려줍니다.
- 같은 서버를 블로킹 API 대신 `accept_async` / `read_async` 와 태스크로 작성할 수도 있습니다 ([3.2.0 공통 규칙](../spec/refs-standard-libraries/300-std/302-net.html#320-공통-규칙)).
