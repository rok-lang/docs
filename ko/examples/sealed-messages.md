# 봉인된 메시지

`rok::core::crypto`와 OS 난수 소스만으로, 수신자만 읽을 수 있도록 메시지를 암호화하고 누가 보냈는지 수신자가 알 수 있도록 서명합니다.

> [!WARNING]
> Rok의 암호 기능은 새로 만들어졌고 감사를 받지 않았습니다. 이 프로그램은 부품이 어떻게 맞물리는지 보여 줄 뿐이니, 아직 실제 비밀을 보호하는 데 쓰지 마세요.

## 코드

```rok
// Send a signed, encrypted message: X25519 key agreement, HKDF, ChaCha20-Poly1305 and Ed25519.
use rok::core::crypto::{Aead, AeadAlgorithm, Ed25519, Hkdf, HashAlgorithm, Secret, X25519};
use rok::std::crypto::random;

struct Sealed {
    sender_key: List<u8>;     // the sender's one-time X25519 public key
    nonce: List<u8>;
    body: List<u8>;           // ciphertext and tag
    signature: List<u8>;      // Ed25519 over sender_key, nonce and body
}

/// The AEAD key both sides derive from an X25519 shared secret.
fn derive(shared: Secret) -> Secret {
    let prk = Hkdf::extract(HashAlgorithm::Sha256, b"sealed-box v1", shared);
    try! Hkdf::expand(HashAlgorithm::Sha256, prk, "aead key", 32)
}

fn signed_part(s: Sealed) -> List<u8> {
    let all: List<u8> = [];
    for part in [s.sender_key, s.nonce, s.body] { all.extend(part); }
    all
}

fn seal(to_public: List<u8>, signing_seed: Secret, text: str) -> Sealed {
    let eph = random::secret(32);
    let shared = try! X25519::shared_secret(eph, to_public) ?? panic("bad peer key");
    let nonce = random::bytes(12);
    let body = try! (try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared))).seal(nonce, "", text);
    let s = Sealed { sender_key: try! X25519::public_key(eph), nonce: nonce, body: body, signature: [] };
    s.signature = try! Ed25519::sign(signing_seed, signed_part(s));
    s
}

fn open(s: Sealed, my_secret: Secret, signer: List<u8>) -> str | none {
    if !Ed25519::verify(signer, signed_part(s), s.signature) { return none; }
    let shared = (try! X25519::shared_secret(my_secret, s.sender_key))?;
    let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared));
    let plain = (try! aead.open(s.nonce, "", s.body))?;
    try? str::from_utf8(plain)
}

// Bob's encryption key pair and Alice's signing key pair.
let bob_secret = random::secret(32);
let bob_public = try! X25519::public_key(bob_secret);
let alice_seed = random::secret(32);
let alice_public = try! Ed25519::public_key(alice_seed);

let msg = seal(bob_public, alice_seed, "the key is under the mat");
println(f"sealed: {msg.body.len()} bytes, signature {msg.signature.len()} bytes");
println(f"bob reads: {open(msg, bob_secret, alice_public) ?? "(rejected)"}");

let eve_secret = random::secret(32);
println(f"eve reads: {open(msg, eve_secret, alice_public) ?? "(rejected)"}");

msg.body[0] = msg.body[0] ^ 1u8;
println(f"tampered: {open(msg, bob_secret, alice_public) ?? "(rejected)"}");
```

```output
sealed: 40 bytes, signature 64 bytes
bob reads: the key is under the mat
eve reads: (rejected)
tampered: (rejected)
```

## 설명

- 메시지마다 새 X25519 키 쌍을 만듭니다. 그 비밀 키와 Bob의 공개 키로 얻은 공유 비밀은, Bob이 자기 비밀 키와 함께 전달된 일회용 공개 키로 계산한 공유 비밀과 같습니다([`X25519`](../../std/core/crypto/x25519.html), [2.21.3 키 교환](../spec/refs-standard-libraries/200-core/221-crypto.html#2213-키-교환)).
- 원시 공유 비밀을 키로 바로 쓰지 않습니다. `Hkdf::extract` / `expand`가 이 비밀을 32바이트 AEAD 키로 바꾸며, 라벨(`b"sealed-box v1"`)이 그 키를 이 프로토콜에 묶습니다([2.21.1](../spec/refs-standard-libraries/200-core/221-crypto.html#2211-해시-hmac-hkdf)).
- ChaCha20-Poly1305는 암호화와 인증을 함께 합니다. 24바이트 텍스트는 40바이트가 되며, 늘어난 16바이트가 태그입니다. `open`은 키가 틀리거나 바이트가 하나라도 바뀌면 `none`을 돌려주므로, Eve와 변조된 사본은 평문이 나오기 전에 모두 거부됩니다([`Aead`](../../std/core/crypto/aead.html), [2.21.2](../spec/refs-standard-libraries/200-core/221-crypto.html#2212-aead)).
- Alice는 일회용 키, nonce, 암호문을 한꺼번에 Ed25519로 서명하므로 누구도 그중 하나를 바꿔치기할 수 없습니다. `open`은 서명부터 확인합니다([`Ed25519`](../../std/core/crypto/ed25519.html)).
- 비밀 키와 시드는 `random::secret`에서 얻으며, 이 함수는 값을 곧바로 비밀 저장소에 넣습니다. 공유 비밀과 유도한 키도 `Secret`이므로, 키 바이트가 일반 `List`에 담기는 일이 없습니다. nonce는 공개 값이며 `random::bytes`에서 얻습니다. 알고리즘이 직접 난수를 가져오는 일은 없습니다([키 재료](../learn/19-crypto-and-tls.html#키-재료-secret)). 있을 수도 없을 수도 있는 결과는 후위 `?`와 `??`로 꺼냅니다([합 타입](../learn/05-sum-types.html)).
- 자세한 내용은 [암호와 TLS](../learn/19-crypto-and-tls.html)에 있습니다.
