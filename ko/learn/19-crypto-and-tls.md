# 19. 암호와 TLS

Rok에는 자체 암호 구현이 들어 있습니다. `rok::core::crypto`에는 해시, HMAC과 HKDF, 인증 암호화, 키 교환, 서명 알고리듬이 있습니다. `rok::std::crypto::random`은 운영체제의 안전한 난수 원천을 읽습니다. `rok::std::net::tls`는 이 둘 위에 만든 TLS 1.3 클라이언트와 서버입니다. OpenSSL이나 다른 시스템 라이브러리가 끼어들지 않으므로 같은 코드는 모든 타깃에서 같게 동작합니다.

> [!WARNING]
>이 코드는 새것이고 감사(audit)를 받지 않았습니다. 공개된 테스트 벡터로 시험했고 비밀 데이터를 다룰 때 상수 시간(constant time)으로 실행되도록 작성했습니다. 하지만 실제 비밀을 맡기기 전에 암호 코드가 거쳐야 하는 독립적인 검토는 받지 않았습니다. 지금은 배우고 실험하는 데 쓰고, 중요한 일에는 감사받은 라이브러리를 쓰세요.

## 해시와 MAC

알고리듬은 `rok::core::crypto`의 항목(`HashAlgorithm`, `Hmac`, `Aead`, `X25519`, `Ed25519` 등)이고, 각각 자기 하위 모듈(`sha2`, `sha3`, `hmac`, `aead`, `x25519`, `ed25519`, `ecdsa`, `rsa`, `mlkem`)에도 있습니다. 바이트를 받는 모든 함수는 `List<u8>`, `Slice<u8>`, `str`(그 UTF-8 바이트), `Secret`을 받으므로 바이트 문자열과 텍스트를 바로 넘길 수 있습니다. 결과는 `List<u8>`이고, 키 재료만 `Secret`으로 돌아옵니다(아래 참고).

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

`Hmac::verify`는 상수 시간으로 비교합니다. 두 비밀을 직접 비교할 때는 `==` 대신 `crypto::constant_time_eq`를 쓰세요. `==`는 처음 다른 곳에서 멈추므로 얼마나 맞았는지가 새어 나갑니다. `Hkdf`는 공유 비밀에서 키를 유도합니다. `sha3`에는 SHA3-256 / 512와 SHAKE가 있습니다.

## 암호화와 서명

`Aead`는 한 번에 암호화와 인증을 합니다(AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305). `seal`은 16바이트 태그가 붙은 암호문을 돌려줍니다. `open`은 암호문, 태그, 연관 데이터 가운데 무엇이든 바뀌었으면 `none`을 돌려줍니다. 키와 nonce는 `random`에서 얻습니다. 알고리듬은 난수를 인자로 받을 뿐 스스로 가져오지 않습니다.

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

`X25519`와 `MlKem768`은 공유 키를 합의합니다. 앞의 것은 고전적인 방식이고 뒤의 것은 양자 내성 방식입니다. `Ecdsa`(P-256 / P-384)와 `RsaPublicKey` / `RsaPrivateKey`는 웹 인증서에 쓰는 서명을 만들고 검증합니다.

## 키 재료: `Secret`

`Secret`은 바이트를 '비밀 저장소'에 둡니다. 비밀 저장소는 수집기가 옮기거나 복사하지 않고, 해제하기 전에 0으로 덮어쓰는 버퍼입니다. `Hkdf::extract` / `expand`, `X25519::shared_secret`, ML-KEM 키 함수가 이 버퍼를 돌려줍니다. `random::secret(n)`은 난수 바이트를 곧바로 이 버퍼에 뽑아 넣습니다. 출력하면 내용이 아니라 `Secret(n bytes)`로 보입니다. `==`가 없으므로 `constant_time_eq`로 비교합니다. `wipe()`는 그 자리에서 0으로 지우고, `using` 바인딩은 블록 끝에서 지웁니다. `Aead`도 확장한 키를 비밀 저장소에 두고, `Aead::wipe`가 그 키를 지웁니다.

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

비밀의 바이트를 보통 객체(`as_bytes().to_list()`, `str`, 포맷한 문자열)로 복사하면 수집기가 지우지 않는 사본이 생깁니다. 비밀 저장소는 RAM에 고정되지도 않으므로 여전히 스왑이나 코어 덤프에 들어갈 수 있습니다.

## TLS: `net::tls`

`tls::Client::new(stream, server_name)`은 열린 `TcpStream` 위에서 TLS 1.3 핸드셰이크를 실행합니다. 서버의 인증서 체인을 OS 신뢰 저장소와 서버 이름으로 검사한 뒤 연결을 돌려줍니다. 이 연결에는 소켓과 같은 `read` / `write_all` / `read_to_end` 메서드가 있습니다. 기본 설정에서 핸드셰이크는 X25519MLKEM768 양자 내성 하이브리드를 제안합니다. TLS 1.2 이하는 거부합니다.

```rok norun
use rok::std::net::{tcp_connect, tls};

let conn = try tls::Client::new(try tcp_connect("example.com:443"), "example.com");
try conn.write_all(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n");
let resp = try conn.read_to_end();
try conn.close_notify();
println((try str::from_utf8(resp)).lines().next() ?? "");   // HTTP/1.1 200 OK
```

실제 HTTP에는 이 일을 대신해 주는 [`http` stone](18-features-and-stones.html#rok-프로젝트의-stone)을 쓰세요. 리다이렉트, 풀링, HTTP/2까지 처리합니다.

서버에는 인증서 체인과 그 개인 키가 필요하고, 보통 PEM 파일에서 읽습니다. 아래 프로그램은 루프백 인터페이스에서 양쪽을 모두 실행합니다. 한쪽은 자체 서명 인증서를 쓰는 서버 task이고, 다른 쪽은 `ClientConfig.root_certs`로 정확히 그 인증서만 신뢰하는 클라이언트입니다. 모든 호출에는 task용 `_async` 형태가 있습니다([12장](12-concurrency.html)).

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
    let sni = s.server_name() ?? "-";
    s.close();
    f"server read {n} bytes, SNI {sni}"
});

let ccfg = try! tls::ClientConfig::try_new();
ccfg.root_certs = chain;               // trust only our certificate
let tcp = try! await TcpStream::connect_async(addr);
let c = try! await tls::Client::with_config_async(tcp, "localhost", ccfg);
try! await c.write_all_async(b"ping");
let reply = try! await c.read_to_end_async();
c.close();
println(try! str::from_utf8(reply));
println(await server);
```

```output
pong
server read 4 bytes, SNI localhost
```

`tls::system_roots()`는 OS 신뢰 저장소를 읽습니다(`SSL_CERT_FILE`로 덮어쓸 수 있음). `ClientConfig.alpn_protocols`는 `h2` 같은 프로토콜을 협상하고, `negotiated_alpn()` / `peer_certificates()`는 합의된 결과를 알려 줍니다. 클라이언트 인증서, 세션 재개, 폐기 검사는 아직 없습니다.

## 요약

- `rok::core::crypto`에는 SHA-2 / SHA-3, HMAC, HKDF, AEAD, X25519, ML-KEM, Ed25519, ECDSA, RSA가 있습니다. 바이트 인자는 `List<u8>`, `Slice<u8>`, `str` 타입을 받습니다.
- 난수는 오직 `rok::std::crypto::random`(`bytes(n)`, `fill(buf)`, `secret(n)`)에서 얻어 인자로 넘깁니다.
- 키는 `Secret`으로 돌아오고, `wipe()`나 `using`으로, 또는 수집될 때 지워집니다.
- 비밀은 `==`가 아니라 `constant_time_eq`나 `Hmac::verify`로 비교합니다.
- `tls::Client` / `tls::Server`는 `TcpStream`을 TLS 1.3으로 감싼 뒤 소켓처럼 읽고 씁니다.
- 구현은 새것이고 감사를 받지 않았습니다.

## 더 읽을거리

- 표준 라이브러리 사양: [2.21 `crypto`](../spec/refs-standard-libraries/200-core/221-crypto.html), [3.10 `crypto::random`](../spec/refs-standard-libraries/300-std/310-crypto.html), [3.2.5 TLS 1.3](../spec/refs-standard-libraries/300-std/302-net.html#325-tls-13-nettls)
- API: [rok::core::crypto](../../std/core/crypto.html), [rok::std::net::tls](../../std/std/net/tls.html)
- [예제](../examples/index.html)
- 다음 장: [20. 어떻게 하나요](20-cookbook.html)
