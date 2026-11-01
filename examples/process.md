# Child processes

Run external programs such as `echo`, a shell pipeline and `tr` with `rok::std::process::Command`, and work with their output, exit codes, environment variables and standard input.

## Code

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

## Walkthrough

- [`Command`](../std/std/process.html#struct.Command) is a builder. Methods such as `args`, `env`, `current_dir` and `stdin` return the updated command, so calls chain on one line, ending with one of `output()`, `status()` or `spawn()` ([3.7.1 `Command` builder](../spec/refs-standard-libraries/300-std/307-process.html#371-command-builder)).
- `output()` waits for the child to finish and collects both stdout and stderr into an [`Output`](../std/std/process.html#struct.Output). It drains both pipes together, so the child never stalls on a full pipe. `str::from_utf8` turns the bytes into a string.
- The pipe (`|`) is handled by `sh -c`, not by rok. It is the simplest option when you need shell syntax.
- `env` adds a variable for the child only, and `env_clear` empties the inherited environment. The parent's environment does not change.
- `spawn()` returns a [`Child`](../std/std/process.html#struct.Child) without waiting. The `Child` owns the pipes opened with `Stdio::Piped`, and a pipe has one owner, so `replace(ref child.stdin, none)` takes it out and leaves `none` behind. Write to `stdin`, send EOF with `close()`, then read `stdout`, close it too, and `wait()`. A pipe's close can fail, so the compiler asks for it rather than closing it for you.
- Copies of a `Command` are independent: a builder method returns a changed copy, so `base.arg("hi")` leaves `base` as it was, and one base command can start several variants. A builder call whose result is dropped warns, since it changes nothing.
- `signal()` comes from `ExitStatusExt` in `rok::std::os::unix::process`. On Linux and macOS it is in the prelude, so it needs no `use`; on Windows it does not exist ([3.11 `os::`](../spec/refs-standard-libraries/300-std/311-os.html)).
- A missing executable throws an `IoError`. `try ... catch e { ... }` catches it on the spot and substitutes a default ([3.14 Error control flow](../spec/refs-lang/003-expressions/314-error-control-flow.html), [Errors](../learn/10-errors.html)).
