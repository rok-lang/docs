# 자식 프로세스

`rok::std::process::Command`로 `echo`, 셸 파이프라인, `tr` 같은 외부 프로그램을 실행합니다. 출력, 종료 코드, 환경 변수, 표준 입력을 다루는 방법을 보여 줍니다.

## 코드

```rok
use rok::core::marker::replace;
use rok::std::process::{Command, Stdio, Output};
use rok::core::io::IoError;

// Captured bytes as trimmed text.
fn text(bytes: List<u8>) -> str {
    (try str::from_utf8(bytes) catch _ { "<not utf-8>" }).trim()
}

fn show(label: str, out: Output) {
    println(f"{label}: [{text(out.stdout)}] ({out.status})");
}

fn main() throws (IoError) {
    // 1. Run a program and capture what it prints.
    show("echo", try Command::new("echo").args(["hello", "from", "a child"]).output());

    // 2. A shell pipeline: the shell does the piping for us.
    let sorted = try Command::new("sh")
        .args(["-c", "printf 'pear\\napple\\nfig\\napple\\n' | sort | uniq -c | sort -rn"])
        .output();
    println("word counts:");
    for line in text(sorted.stdout).lines() { println(f"  {line.trim()}"); }

    // 3. Exit codes and stderr.
    let failed = try Command::new("sh").args(["-c", "echo oops >&2; exit 3"]).output();
    println(f"success={failed.status.success()} code={failed.status.code() ?? -1} stderr=[{text(failed.stderr)}]");

    // 4. Environment variables for the child only.
    let greet = try Command::new("sh")
        .args(["-c", "echo \"$GREETING, $NAME\""])
        .env("GREETING", "annyeong")
        .env("NAME", "rok")
        .output();
    show("env", greet);

    let clean = try Command::new("/usr/bin/env").env_clear().env("ONLY", "1").output();
    show("env_clear", clean);

    // 5. Feed the child's stdin ourselves and read its stdout.
    let child = try Command::new("tr").args(["a-z", "A-Z"])
        .stdin(Stdio::Piped)
        .stdout(Stdio::Piped)
        .spawn();
    let input = replace(ref child.stdin, none) ?? panic("stdin not piped");
    try input.write_all("shout this\n");
    try input.close();   // EOF lets `tr` finish
    let output = replace(ref child.stdout, none) ?? panic("stdout not piped");
    let shouted = try output.read_to_str();
    try output.close();
    println(f"tr: [{shouted.trim()}] ({try child.wait()})");

    // 6. A working directory for the child.
    show("pwd", try Command::new("pwd").current_dir("/").output());

    // 7. One base command, several variants: each builder call returns a
    //    changed copy and leaves `base` as it was.
    let base = Command::new("sh").args(["-c", "echo $0 $1", "greet"]);
    show("variant", try base.arg("hi").output());
    show("base", try base.output());

    // 8. A child ended by a signal has no exit code; `signal()` names it.
    let killed = try Command::new("sh").args(["-c", "kill -9 $$"]).status();
    println(f"killed: code={killed.code() ?? -1} signal={killed.signal():?}");

    // 9. A missing program is an IoError, not a crash.
    let ok = try Command::new("no-such-program").status().success() catch e {
        println(f"spawn failed: {e.kind():?}");
        false
    };
    println(f"ok={ok}");
}
```

```output
echo: [hello from a child] (exit status: 0)
word counts:
  2 apple
  1 pear
  1 fig
success=false code=3 stderr=[oops]
env: [annyeong, rok] (exit status: 0)
env_clear: [ONLY=1] (exit status: 0)
tr: [SHOUT THIS] (exit status: 0)
pwd: [/] (exit status: 0)
variant: [greet hi] (exit status: 0)
base: [greet] (exit status: 0)
killed: code=-1 signal=Kill
spawn failed: NotFound
ok=false
```

## 설명

- [`Command`](../../std/std/process.html#struct.Command)는 빌더입니다. `args`, `env`, `current_dir`, `stdin` 같은 메서드는 갱신된 명령을 돌려주므로 호출을 한 줄로 이어 갑니다. 마지막에는 `output()`, `status()`, `spawn()` 중 하나로 끝냅니다([3.7.1 `Command` 빌더](../spec/refs-standard-libraries/300-std/307-process.html#371-command-빌더)).
- `output()`는 자식이 끝나기를 기다리고 stdout과 stderr를 모두 [`Output`](../../std/std/process.html#struct.Output)에 모읍니다. 두 파이프를 함께 비우므로 자식이 가득 찬 파이프에 막혀 멈추는 일이 없습니다. `str::from_utf8`은 바이트를 문자열로 바꿉니다.
- 파이프(`|`)는 Rok이 아니라 `sh -c`가 처리합니다. 셸 문법이 필요할 때 가장 간단한 방법입니다.
- `env`는 자식에게만 변수를 더하고, `env_clear`는 물려받은 환경을 비웁니다. 부모의 환경은 바뀌지 않습니다.
- `spawn()`은 기다리지 않고 [`Child`](../../std/std/process.html#struct.Child)를 돌려줍니다. `Stdio::Piped`로 연 파이프는 `Child`가 소유하고 파이프의 주인은 하나뿐이므로, `replace(ref child.stdin, none)`으로 꺼내고 그 자리에는 `none`을 남깁니다. `stdin`에 쓰고 `close()`로 EOF를 보낸 다음, `stdout`을 읽고 그것도 닫은 뒤 `wait()`로 기다립니다. 파이프 닫기는 실패할 수 있으므로 컴파일러가 대신 닫지 않고 직접 닫도록 요구합니다.
- `Command`의 복사본은 서로 독립입니다. 빌더 메서드는 바뀐 복사본을 돌려주므로 `base.arg("hi")` 뒤에도 `base`는 그대로이고, 기본 명령 하나로 여러 변형을 띄울 수 있습니다. 결과를 버린 빌더 호출은 아무것도 바꾸지 않으므로 경고가 납니다.
- `signal()`은 `rok::std::os::unix::process`의 `ExitStatusExt`에서 옵니다. Linux와 macOS에서는 prelude에 있어 `use`가 필요 없고, Windows에는 없습니다([3.11 `os::`](../spec/refs-standard-libraries/300-std/311-os.html)).
- 실행 파일이 없으면 `IoError`를 던집니다. `try ... catch e { ... }`가 그 자리에서 잡아 기본값으로 대신합니다([3.14 에러 제어 흐름](../spec/refs-lang/003-expressions/314-error-control-flow.html), [에러 처리](../learn/10-errors.html)).
