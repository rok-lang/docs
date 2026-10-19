# Sealed messages

Encrypt a message so only its recipient can read it, and sign it so the recipient knows who sent it, with nothing but `rok::core::crypto` and the OS random source.

> [!WARNING]
> rok's cryptography is new and unaudited. This program shows how the pieces fit; do not protect real secrets with it yet.

## Code

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

## Walkthrough

- Each message gets a fresh X25519 key pair. Its secret half and Bob's public key give the same shared secret that Bob computes from his secret and the one-time public key sent along ([`X25519`](../std/core/crypto/x25519.html), [2.21.3 key exchange](../spec/refs-standard-libraries/200-core/221-crypto.html#2213-key-exchange)).
- A raw shared secret is not used as a key directly: `Hkdf::extract` / `expand` turn it into a 32-byte AEAD key, with a label (`b"sealed-box v1"`) that ties the key to this protocol ([2.21.1](../spec/refs-standard-libraries/200-core/221-crypto.html#2211-hashes-hmac-hkdf)).
- ChaCha20-Poly1305 encrypts and authenticates. The 24-byte text becomes 40 bytes, the extra 16 being the tag. `open` returns `none` for a wrong key or a changed byte, so Eve and the tampered copy are both rejected before any plaintext appears ([`Aead`](../std/core/crypto/aead.html), [2.21.2](../spec/refs-standard-libraries/200-core/221-crypto.html#2212-aead)).
- Alice signs the one-time key, nonce and ciphertext together with Ed25519, so nobody can swap any of them. `open` checks the signature first ([`Ed25519`](../std/core/crypto/ed25519.html)).
- Secret keys and seeds come from `random::secret`, which puts them straight into secret storage, and the shared secret and derived key are `Secret`s too, so no key byte lands in an ordinary `List`. Nonces are public and come from `random::bytes`. The algorithms never fetch randomness themselves ([Key material](../learn/19-crypto-and-tls.html#key-material-secret)). Optional results are unwrapped with postfix `?` and `??` ([Sum types](../learn/05-sum-types.html)).
- More in [Cryptography and TLS](../learn/19-crypto-and-tls.html).
