# 19. Cryptography and TLS

rok ships its own cryptography. `rok::core::crypto` has the algorithms (hashes, HMAC and HKDF, authenticated encryption, key exchange, signatures), `rok::std::crypto::random` reads the operating system's secure random source, and `rok::std::net::tls` is a TLS 1.3 client and server built on the two. No OpenSSL or other system library is involved, so the same code behaves the same on every target.

> [!WARNING]
> This code is new and has not been audited. It is tested against the published test vectors and written to run in constant time on secret data, but it has not had the independent review that cryptography needs before you trust it with real secrets. Use it to learn and to experiment, and keep an audited library for anything that matters for now.

## Hashes and MACs

The algorithms are items of `rok::core::crypto` (`HashAlgorithm`, `Hmac`, `Aead`, `X25519`, `Ed25519`, ...), each also in its submodule (`sha2`, `sha3`, `hmac`, `aead`, `x25519`, `ed25519`, `ecdsa`, `rsa`, `mlkem`). Every function that takes bytes accepts `List<u8>`, `Slice<u8>`, a `str` (its UTF-8 bytes) or a `Secret`, so byte strings and text can be passed directly. Results are `List<u8>`, except key material, which comes back as a `Secret` (below).

```rok
use rok::core::crypto::{HashAlgorithm, Digest, Hmac};

fn hex(bytes: List<u8>) -> str {
    let s = StrBuf::new();
    for b in bytes { s.push_str(f"{b:02x}"); }
    s.to_str()
}

println(hex(HashAlgorithm::Sha256.digest("abc")));

// The same hash, fed in pieces.
let d = Digest::new(HashAlgorithm::Sha256);
d.update("a");
d.update(b"bc");
println(f"{hex(d.finish()) == hex(HashAlgorithm::Sha256.digest("abc"))}");

let tag = Hmac::mac(HashAlgorithm::Sha256, "secret key", "message");
println(f"{tag.len()} {Hmac::verify(HashAlgorithm::Sha256, "secret key", "message", tag)}");
println(f"{Hmac::verify(HashAlgorithm::Sha256, "secret key", "massage", tag)}");
```

```output
ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
true
32 true
false
```

`Hmac::verify` compares in constant time. To compare two secrets yourself, use `crypto::constant_time_eq` rather than `==`, which stops at the first difference and so leaks how much matched. `Hkdf` derives keys from a shared secret, and `sha3` has SHA3-256 / 512 and SHAKE.

## Encryption and signatures

`Aead` encrypts and authenticates in one step (AES-128-GCM, AES-256-GCM or ChaCha20-Poly1305). `seal` returns the ciphertext with a 16-byte tag; `open` returns `none` when anything, the ciphertext, the tag or the associated data, was changed. Keys and nonces come from `random`: the algorithms take randomness as an argument and never fetch it themselves.

```rok
use rok::core::crypto::{Aead, AeadAlgorithm, Ed25519};
use rok::std::crypto::random;

let key = random::secret(32);
let nonce = random::bytes(12);           // never reuse a nonce with the same key
let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, key);
let sealed = try! aead.seal(nonce, "v1", "meet at noon");
println(f"{sealed.len()} bytes sealed");  // 12 bytes of text + a 16-byte tag

if let plain: List<u8> = try! aead.open(nonce, "v1", sealed) {
    println(try! str::from_utf8(plain));
}
sealed[0] = sealed[0] ^ 1u8;              // tamper with one bit
println(f"tampered: {(try! aead.open(nonce, "v1", sealed)) is none}");

let seed = random::secret(32);           // the private key
let public = try! Ed25519::public_key(seed);
let sig = try! Ed25519::sign(seed, "release 1.0");
println(f"{Ed25519::verify(public, "release 1.0", sig)} {Ed25519::verify(public, "release 1.1", sig)}");
```

```output
28 bytes sealed
meet at noon
tampered: true
true false
```

`X25519` and `MlKem768` agree on shared keys (the classic and the post-quantum way), and `Ecdsa` (P-256 / P-384) and `RsaPublicKey` / `RsaPrivateKey` sign and verify for the certificates the web uses.

## Key material: `Secret`

A `Secret` holds bytes in *secret storage*: a buffer the collector never moves or copies and overwrites with zeros before it frees it. `Hkdf::extract` / `expand`, `X25519::shared_secret` and the ML-KEM key functions return one, and `random::secret(n)` draws random bytes straight into one. It prints as `Secret(n bytes)`, never its contents, and has no `==`: compare with `constant_time_eq`. `wipe()` zeroes it now, and a `using` binding wipes it at the end of the block. `Aead` keeps its expanded key in secret storage too, and `Aead::wipe` clears it.

```rok
use rok::core::crypto::{Aead, AeadAlgorithm, HashAlgorithm, Hkdf, Secret, X25519, constant_time_eq};
use rok::std::crypto::random;

let a = random::secret(32);
let b = random::secret(32);
let ab = try! X25519::shared_secret(a, try! X25519::public_key(b)) ?? panic("bad key");
let ba = try! X25519::shared_secret(b, try! X25519::public_key(a)) ?? panic("bad key");
println(f"{ab:?} {constant_time_eq(ab, ba)}");

let key = try! Hkdf::expand(HashAlgorithm::Sha256, Hkdf::extract(HashAlgorithm::Sha256, "salt", ab), "key", 32);
let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, key);
key.wipe();                               // zero it now, not at some later collection
println(f"{key[0]}");
aead.wipe();                              // the cipher is done too

using s = Secret::new("temporary") {
    println(f"{s.len()} bytes");
}                                         // wiped here
```

```output
Secret(32 bytes) true
0
9 bytes
```

Copying a secret's bytes into an ordinary object (`as_bytes().to_list()`, a `str`, a formatted string) makes a copy the collector does not wipe. Secret storage is not locked in RAM either, so it can still reach swap or a core dump.

## TLS: `net::tls`

`tls::Client::new(stream, server_name)` runs a TLS 1.3 handshake over an open `TcpStream`, checks the server's certificate chain against the OS trust store and its name, and returns a connection with the same `read` / `write_all` / `read_to_end` methods as the socket. With the default config the handshake offers the X25519MLKEM768 post-quantum hybrid. TLS 1.2 and older are refused.

```rok norun
use rok::std::net::{tcp_connect, tls};

let conn = try tls::Client::new(try tcp_connect("example.com:443"), "example.com");
try conn.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n");
let resp = try conn.read_to_end();
try conn.close_notify();
println((try str::from_utf8(resp)).lines().next() ?? "");   // HTTP/1.1 200 OK
```

For real HTTP, use the [`http` stone](18-features-and-stones.html#the-rok-stones), which does this for you, with redirects, pooling and HTTP/2.

A server needs a certificate chain and its private key, usually read from PEM files. The program below runs both sides on the loopback interface: a server task with a self-signed certificate, and a client that trusts exactly that certificate through `ClientConfig.root_certs`. Every call has an `_async` form for tasks ([chapter 12](12-concurrency.html)).

```rok
use rok::std::net;
use rok::std::io::byte_buf;
use rok::std::net::{TcpStream, tls};

// A self-signed Ed25519 certificate for `localhost` and its key (a demo
// key: it is public, never use it for anything real).
const CERT: str = "
-----BEGIN CERTIFICATE-----
MIIBVTCCAQegAwIBAgIUOw8IA2ogBBGsxE/m7hJPQIZPLzYwBQYDK2VwMBQxEjAQ
BgNVBAMMCWxvY2FsaG9zdDAgFw0yNjEwMTYxNjEyMDRaGA8yMTI2MDkyMjE2MTIw
NFowFDESMBAGA1UEAwwJbG9jYWxob3N0MCowBQYDK2VwAyEAguLEN9y6fnek02iG
f4LullsxttwNkvHTYdhsjnFWsVajaTBnMB0GA1UdDgQWBBRUjvYk1Hhem6n+Hr1q
HZm6wzG3mzAfBgNVHSMEGDAWgBRUjvYk1Hhem6n+Hr1qHZm6wzG3mzAPBgNVHRMB
Af8EBTADAQH/MBQGA1UdEQQNMAuCCWxvY2FsaG9zdDAFBgMrZXADQQC2XwFaBIQ5
C138MCqnpjXays5b9C4QnQ+tBky5VqKY1ffbKN9O4IIWBeQqED+shdRM5Y4HGYcH
+Zr3PLdVn7QD
-----END CERTIFICATE-----
";
const KEY: str = "
-----BEGIN PRIVATE KEY-----
MC4CAQAwBQYDK2VwBCIEIMa20cYLozv3/41l6/owqHiube5cG+rd7cF4nHPZ6qd3
-----END PRIVATE KEY-----
";

let chain = try! tls::Certificate::from_pem(CERT);
let scfg = try! tls::ServerConfig::new(chain, try! tls::PrivateKey::from_pem(KEY));
let l = try! net::tcp_listen("127.0.0.1:0");
let addr = try! l.local_addr();
let server = task::spawn_local(async {
    let (conn, _) = try! await l.accept_async();
    let s = try! await tls::Server::new_async(conn, scfg);
    let buf = byte_buf(64);
    let n = try! await s.read_async(buf);
    try! await s.write_all_async(b"pong");
    try! await s.close_notify_async();
    f"server read {n} bytes, SNI {s.server_name() ?? "-"}"
});

let ccfg = tls::ClientConfig::new();
ccfg.root_certs = chain;               // trust only our certificate
let tcp = try! await TcpStream::connect_async(addr);
let c = try! await tls::Client::with_config_async(tcp, "localhost", ccfg);
try! await c.write_all_async(b"ping");
let reply = try! await c.read_to_end_async();
println(try! str::from_utf8(reply));
println(await server);
```

```output
pong
server read 4 bytes, SNI localhost
```

`tls::system_roots()` reads the OS trust store (`SSL_CERT_FILE` overrides it), `ClientConfig.alpn_protocols` negotiates protocols such as `h2`, and `negotiated_alpn()` / `peer_certificates()` tell you what was agreed. Client certificates, session resumption and revocation checks are not supported.

## Summary

- `rok::core::crypto` has SHA-2 / SHA-3, HMAC, HKDF, AEAD, X25519, ML-KEM, Ed25519, ECDSA and RSA. Byte arguments take `List<u8>`, `Slice<u8>` or `str`.
- Randomness comes only from `rok::std::crypto::random` (`bytes(n)`, `fill(buf)`, `secret(n)`) and is passed in.
- Keys come back as `Secret`, which is wiped by `wipe()`, by `using`, or when it is collected.
- Compare secrets with `constant_time_eq` or `Hmac::verify`, never `==`.
- `tls::Client` / `tls::Server` wrap a `TcpStream` in TLS 1.3 and then read and write like it.
- The implementation is new and unaudited.

## Further reading

- Standard library spec: [2.21 `crypto`](../spec/refs-standard-libraries/200-core/221-crypto.html), [3.10 `crypto::random`](../spec/refs-standard-libraries/300-std/310-crypto.html), [3.2.5 TLS 1.3](../spec/refs-standard-libraries/300-std/302-net.html#325-tls-13---nettls)
- API: [rok::core::crypto](../std/core/crypto.html), [rok::std::net::tls](../std/std/net/tls.html)
- [Examples](../examples/index.html)
- Next: [20. How Do I...](20-cookbook.html), short recipes for everyday tasks
