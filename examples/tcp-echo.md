# TCP echo server

One program starts an echo server on a free `127.0.0.1` port, has clients send lines and read back the echoes, and then shuts everything down cleanly.

## Code

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

## Walkthrough

- Binding `TcpListener` to port 0 lets the OS choose a free port, and `local_addr()` reports the actual address, so there is no clash with other programs ([3.2 net](../spec/refs-standard-libraries/300-std/302-net.html), [`TcpListener`](../std/std/net.html#struct.TcpListener)).
- The server thread starts one handler thread per connection, then `join`s them all and adds up the lines served. The closures capture `listener` and `conn` and carry them to the other threads ([Concurrency](../learn/12-concurrency.html), [`thread::spawn`](../std/std/thread.html#fn.spawn)).
- To read line by line, the socket is `try_clone()`d and wrapped in a `BufReader`: reads go through the clone, writes through the original ([`BufReader`](../std/std/io.html#struct.BufReader)).
- Shutdown order matters. After sending everything, the client calls `shutdown(Shutdown::Write)` to say "nothing more", and the server's `lines()` ends its loop on that signal. Once the server closes both handles, the client's `read_line()` returns an empty string.
- The same server can be written with `accept_async` / `read_async` and tasks instead of the blocking API ([3.2.0 common rules](../spec/refs-standard-libraries/300-std/302-net.html#320-common-rules)).
