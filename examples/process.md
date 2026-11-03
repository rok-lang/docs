# Child processes

Run external programs such as `echo`, a shell pipeline and `tr` with `process::run` and `rok::std::process::Command`, and work with their output, exit codes, environment variables and standard input.

## Code

```rok
use rok::std::process;
use rok::std::process::{Command, Stdio, Output};

// Captured bytes as trimmed text.
fn text(bytes: List<u8>) -> str {
    (try str::from_utf8(bytes) catch _ { "<not utf-8>" }).trim()
}

fn show(label: str, out: Output) {
    println(f"{label}: [{text(out.stdout)}] ({out.status})");
}

fn main() {
    // 1. Run a program and take what it prints as text.
    let said = try process::run("echo", ["hello", "from", "a child"]);
    println(f"echo: [{said.trim()}]");

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
    let input = child.take_stdin() ?? panic("stdin not piped");
    try input.write_all("shout this\n");
    try input.close();   // EOF lets `tr` finish
    let output = child.take_stdout() ?? panic("stdout not piped");
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
echo: [hello from a child]
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

- `process::run(program, args)` runs a program to the end and returns its stdout as text; a failing exit status throws an `ExitStatusError` that carries stderr ([`process::run`](../std/std/process.html#fn.run)). `main` declares no `throws`: an error it lets through is printed and the program exits with status 1.
- [`Command`](../std/std/process.html#struct.Command) is a builder. Methods such as `args`, `env`, `current_dir` and `stdin` return the updated command, so calls chain on one line, ending with one of `output()`, `status()` or `spawn()` ([3.7.1 `Command` builder](../spec/refs-standard-libraries/300-std/307-process.html#371-command-builder)).
- `output()` waits for the child to finish and collects both stdout and stderr into an [`Output`](../std/std/process.html#struct.Output). It drains both pipes together, so the child never stalls on a full pipe. `str::from_utf8` turns the bytes into a string.
- The pipe (`|`) is handled by `sh -c`, not by rok. It is the simplest option when you need shell syntax.
- `env` adds a variable for the child only, and `env_clear` empties the inherited environment. The parent's environment does not change.
- `spawn()` returns a [`Child`](../std/std/process.html#struct.Child) without waiting. The `Child` holds the pipes opened with `Stdio::Piped` until you take them: `child.take_stdin()` hands over the stdin pipe and leaves `none` behind, so the pipe has one owner, you. Pipes you never take close when the `Child` is released. Write to `stdin`, send EOF with `close()`, then read `stdout`, close it too, and `wait()`. A pipe's close can fail, so the compiler asks for it rather than closing it for you.
- Copies of a `Command` are independent: a builder method returns a changed copy, so `base.arg("hi")` leaves `base` as it was, and one base command can start several variants. A builder call whose result is dropped warns, since it changes nothing.
- `signal()` comes from `ExitStatusExt` in `rok::std::os::unix::process`. On Linux and macOS it is in the prelude, so it needs no `use`; on Windows it does not exist ([3.11 `os::`](../spec/refs-standard-libraries/300-std/311-os.html)).
- A missing executable throws an `IoError`. `try ... catch e { ... }` catches it on the spot and substitutes a default ([3.14 Error control flow](../spec/refs-lang/003-expressions/314-error-control-flow.html), [Errors](../learn/10-errors.html)).
