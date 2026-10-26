# 18. Features and Stones

[Chapter 11](11-modules-packages.html) built a package and added dependencies. This chapter goes further: **features**, which let the users of a library turn parts of it on and off; the stones the Rok project maintains; and publishing your own library to [rokstones.io](https://rokstones.io).

## Defining features

A feature is a name in the `features` dict of `rok.rokon`, mapped to the list of what it turns on. In code, an enabled feature is the condition `feature = "<name>"` ([chapter 17](17-conditional-compilation.html)). Here a `geom` library has a `circle` feature, on by default, and a `pretty` feature that pulls in an optional dependency.

```rokon
Package {
    name: "geom",
    version: "0.1.0",
    deps: {
        "fancy": Dep::path("../fancy", optional: true),
    },
    features: {
        "default": ["circle"],
        "circle": [],
        "pretty": ["dep:fancy"],
    },
}
```

```rok norun
// geom/src/lib.rok
pub fn shapes() -> List<str> {
    let out: List<str> = ["square"];
    #[cfg(feature = "circle")]
    out.push("circle");
    out
}

#[cfg(feature = "pretty")]
pub fn show(s: str) -> str { fancy::frame(s) }

#[cfg(not(feature = "pretty"))]
pub fn show(s: str) -> str { s }
```

Each entry of a feature's list is one of:

| Entry       | Turns on                                                            |
|-------------|---------------------------------------------------------------------|
| `"circle"`  | Another feature of this package                                     |
| `"dep:fancy"` | The optional dependency `fancy`                                   |
| `"http/tls"` | Feature `tls` of the dependency `http` (and `http` itself, if it is optional) |
| `"http?/tls"` | Feature `tls` of `http`, only if something else already turned `http` on |

- `default` is an ordinary feature that is on unless the user turns it off.
- An optional dependency (`optional: true`) is part of the build only when an enabled feature names it with `dep:` or `<dep>/<f>`. It does not get a feature of its own name automatically, and an optional dependency no feature enables is an error. `rok add fancy --path ../fancy --optional` writes both the dependency and a `"fancy": ["dep:fancy"]` feature.
- Naming a feature or dependency that does not exist, using `dep:` on a non-optional dependency, and cycles are errors, reported with the line and column in `rok.rokon`.

Feature names are lowercase words that say what they enable (`http1`, `tls`, `pretty`), not `with-x` or `no-x`: features only ever add.

## Enabling features

The package being built starts from its `default` feature. `--features` adds more, either its own (`fancy`) or a direct dependency's (`geom/pretty`); `--no-default-features` and `--all-features` do what they say. They work with `run`, `build`, `test`, `check` and the other build commands.

```text
$ rok run
square
circle
$ rok run --features geom/pretty
*** square ***
*** circle ***
$ rok run --no-default-features
square
circle
```

The last one still prints `circle`: `--no-default-features` turns off the defaults of the package being built (`app`), not those of `geom`. A dependency's features are chosen where you depend on it.

```rokon
Package {
    name: "app",
    version: "0.1.0",
    deps: {
        "geom": Dep::path("../geom", default_features: false, features: ["pretty"]),
    },
    features: {
        "fancy": ["geom/pretty"],
    },
}
```

Features are **unified**: each package has one feature set for the whole build, the union of everything every dependent asks of it. So if two libraries both use `geom`, one with `pretty` and one without, both get `pretty`. That is why a feature must only add API and never change what existing code does. `rok deps` shows the result as `--extern-feature` lines.

The lockfile pins every package of the graph, optional ones included, so switching features never changes `rok.lock`.

## The Rok stones

The Rok project maintains a few libraries as stones of their own, outside the standard library. Each is pure Rok with no dependencies beyond `rok::std`.

| Stone      | What it is                                                                    |
|------------|-------------------------------------------------------------------------------|
| `json`     | JSON (RFC 8259): a `Value` tree, a strict parser with `line:col` errors, a streaming tokenizer, typed encode / decode with `#[derive(Json)]`, JSON Pointer |
| `toml`     | TOML 1.1.0: a `Value` tree with typed date-times, a parser with `line:col` errors, a writer, typed encode / decode with `#[derive(Toml)]` |
| `http`     | HTTP/1.1 and HTTP/2 client and server: strict parsing, HPACK, streaming bodies, connection pooling, `https` over std's TLS. Features `http1`, `http2`, `tls`, all on by default |
| `markdown` | CommonMark plus GitHub extensions (tables, callouts, footnotes, heading anchors) to HTML. It renders this site |
| `rok-syntax` | A tokenizer and HTML highlighter for Rok and ROKON source. It highlights the code on this site |

`json` and `toml` follow the shape of std's [`rokon`](15-rokon.html) module: `parse` / `to_string` / `from_str` / `encode` / `decode`, and a derive for your own types.

```rok norun
#[derive(Json)]
struct Server { host: str; port: u16 = 80; }

let s: Server = try json::from_str("{\"host\": \"localhost\"}");
println(json::to_string(s));      // {"host":"localhost","port":80}
```

They are added like any other stone. Without a requirement, `rok add` writes `^` and the newest version:

```bash
rok add json
rok add http --no-default-features --features http1,tls
```

```rokon
Package { name: "app", version: "0.1.0", deps: {
    "json": Dep::version("^0.1.0"),
    "http": Dep::version("^0.1.0", default_features: false, features: ["http1", "tls"]),
} }
```

In a script, name the file last: `rok add json app.rok` writes the dependency into the script's `//!` block ([chapter 11](11-modules-packages.html#single-file-scripts-and-environments)).

With `http`, an HTTPS request is a few lines. The stone opens the TCP connection, runs the TLS 1.3 handshake from [`rok::std::net::tls`](19-crypto-and-tls.html) against the OS trust store, offers HTTP/2 through ALPN and falls back to HTTP/1.1:

```rok norun
let r = try await http::get("https://example.com/");
println(f"{r.status} {r.version:?} {r.headers.get("content-type") ?? "-"}");
println(try await r.body.text());
```

## Publishing a stone

Publishing adds one line to the registry index, pointing at a commit of your git repository; there is no upload of source archives. The package must be a library (`src/lib.rok`) at the root of a clean git work tree with a remote `origin`, and every normal dependency must itself be a stone (path and git dependencies are rejected; path and git `dev_deps` are dropped). Fill in `description`, `license` and `repository`: they show up in `rok search` and on the website.

> [!IMPORTANT]
> For now only the rokstones.io maintainers can publish. Anyone can sign in and use stones, but `rok login` refuses an account that cannot publish yet.

### `rok login`

Sign in at [rokstones.io/me](https://rokstones.io/me) with GitHub and create an account token there. Then hand it to `rok`:

```text
$ rok login
a token from https://rokstones.io/me (not shown; Ctrl-C to cancel):
   Logged in as your-name at https://rokstones.io (token in ~/.rok/credentials.rokon)
```

Without an argument, the token is read from standard input without echoing. `rok login` asks the registry who the token belongs to and refuses a token it rejects, saving nothing. A good token is kept in `~/.rok/credentials.rokon` (readable only by you); `rok logout` deletes it. In CI, set `ROK_REGISTRY_TOKEN` instead.

### `rok publish`

`rok publish` alone is a dry run: it prints the index entry it would add, and nothing leaves your machine.

```text
$ rok publish
Stone { name: "textwrap", vers: "0.1.0", deps: [], source: Dep::git("https://github.com/you/textwrap.git", rev: "8e2afb20.."), cksum: "823a3592..", yanked: false, description: "Word wrapping for terminals", license: "MIT OR Apache-2.0", repository: "https://github.com/you/textwrap" }
note: dry run; --commit adds it to https://github.com/rok-lang/rokstones.io-index as te/xt/textwrap
```

The entry records the HEAD commit and its tree hash, so push that commit first. `rok publish --commit` sends the entry to the registry with your token (`Published textwrap 0.1.0 to https://rokstones.io`), and the version is live once the index has it. A version cannot be published twice, and a name differing only in case or `-` / `_` from an existing stone is taken.

### `rok yank`

A version with a serious bug can be yanked. New resolutions skip it, but projects whose `rok.lock` already pins it keep building, so yanking never breaks anyone. `--undo` reverses it.

```bash
rok yank --vers 0.1.0          # in the package directory
rok yank --vers 0.1.0 --undo
```

## Summary

- `features` in `rok.rokon` maps names to entries: other features, `dep:<d>`, `<d>/<f>`, `<d>?/<f>`. `default` is on unless turned off.
- Optional dependencies (`optional: true`) are built only when a feature enables them.
- Choose features with `--features`, `--no-default-features`, `--all-features`, and per dependency with `features:` / `default_features: false`. Features are unified across the build, so they must only add.
- Code tests them with `#[cfg(feature = "..")]` and `cfg!(feature = "..")`.
- `json`, `toml`, `http`, `markdown` and `rok-syntax` are the Rok project's stones.
- Publish with `rok login` (a rokstones.io token), `rok publish` (dry run) and `rok publish --commit`; withdraw with `rok yank`. Publishing is open to maintainers only for now.

## Further reading

- Spec: [9.10.4c Features](../spec/refs-lang/009-tools/910-rok-frontend.html#9104c-features), [9.10.4a Stones and rokstones.io](../spec/refs-lang/009-tools/910-rok-frontend.html#9104a-stones-and-rokstonesio), [`rok search` / `publish` / `yank` / `login`](../spec/refs-lang/009-tools/910-rok-frontend.html#rok-search--publish--yank--login--logout), [`rok.rokon`](../spec/refs-lang/009-tools/910-rok-frontend.html#9103-rokrokon--roklock)
- [rokstones.io](https://rokstones.io)
- Next chapter: [19. Cryptography and TLS](19-crypto-and-tls.html)
