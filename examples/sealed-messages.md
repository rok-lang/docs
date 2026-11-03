# Sealed messages

Encrypt a message so only its recipient can read it, and sign it so the recipient knows who sent it, with nothing but `rok::core::crypto` and the OS random source.

> [!WARNING]
> Rok's cryptography is new and unaudited. This program shows how the pieces fit; do not protect real secrets with it yet.

## Code

```rok
// Send a signed, encrypted message: X25519 key agreement, HKDF, ChaCha20-Poly1305 and Ed25519.
use rok::core::crypto::{Aead, AeadAlgorithm, HashAlgorithm, Secret};
use rok::core::crypto::{ed25519, hkdf, x25519};
use rok::std::random;

struct Sealed {
    sender_key: x25519::PublicKey;   // the sender's one-time X25519 public key
    nonce: List<u8>;
    body: List<u8>;                  // ciphertext and tag
    signature: List<u8>;             // Ed25519 over sender_key, nonce and body
}

/// The AEAD key both sides derive from an X25519 shared secret.
fn derive(shared: Secret) -> Secret {
    let prk = hkdf::extract(HashAlgorithm::Sha256, b"sealed-box v1", shared);
    try! hkdf::expand(HashAlgorithm::Sha256, prk, "aead key", 32)
}

fn signed_part(s: Sealed) -> List<u8> {
    s.sender_key.to_bytes().to_list() + s.nonce + s.body
}

fn seal(to: x25519::PublicKey, signer: ed25519::SigningKey, text: str) -> Sealed {
    let eph = try! x25519::PrivateKey::from_bytes(random::secret(32));
    let shared = eph.diffie_hellman(to) ?? panic("bad peer key");
    let nonce = random::bytes(12);
    let body = try! (try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared))).seal(nonce, "", text);
    let s = Sealed { sender_key: eph.public_key(), nonce: nonce, body: body, signature: [] };
    s.signature = signer.sign(signed_part(s));
    s
}

fn open(s: Sealed, me: x25519::PrivateKey, from: ed25519::VerifyingKey) -> str | none {
    if !from.verify(signed_part(s), s.signature) { return none; }
    let shared = me.diffie_hellman(s.sender_key)?;
    let aead = try! Aead::new(AeadAlgorithm::ChaCha20Poly1305, derive(shared));
    let plain = (try! aead.open(s.nonce, "", s.body))?;
    try? str::from_utf8(plain)
}

// Bob's encryption key and Alice's signing key.
let bob = try! x25519::PrivateKey::from_bytes(random::secret(32));
let alice = try! ed25519::SigningKey::from_seed(random::secret(32));
let alice_public = alice.verifying_key();

let msg = seal(bob.public_key(), alice, "the key is under the mat");
println(f"sealed: {msg.body.len()} bytes, signature {msg.signature.len()} bytes");
println(f"bob reads: {open(msg, bob, alice_public) ?? "(rejected)"}");

let eve = try! x25519::PrivateKey::from_bytes(random::secret(32));
println(f"eve reads: {open(msg, eve, alice_public) ?? "(rejected)"}");

msg.body[0] ^= 1;
println(f"tampered: {open(msg, bob, alice_public) ?? "(rejected)"}");
```

```output
sealed: 40 bytes, signature 64 bytes
bob reads: the key is under the mat
eve reads: (rejected)
tampered: (rejected)
```

## Walkthrough

- Each message gets a fresh X25519 key pair. Its secret half and Bob's public key give the same shared secret that Bob computes from his secret and the one-time public key sent along Keys are typed values: `x25519::PrivateKey` keeps its bytes in secret storage, and its `PublicKey` half is a plain value that goes in the message ([`x25519`](../std/core/crypto/x25519.html), [2.21.3 key exchange](../spec/refs-standard-libraries/200-core/221-crypto.html#2213-key-exchange)).
- A raw shared secret is not used as a key directly: `hkdf::extract` / `expand` turn it into a 32-byte AEAD key, with a label (`b"sealed-box v1"`) that ties the key to this protocol ([2.21.1](../spec/refs-standard-libraries/200-core/221-crypto.html#2211-hashes-hmac-hkdf)).
- ChaCha20-Poly1305 encrypts and authenticates. The 24-byte text becomes 40 bytes, the extra 16 being the tag. `open` returns `none` for a wrong key or a changed byte, so Eve and the tampered copy are both rejected before any plaintext appears ([`Aead`](../std/core/crypto/aead.html), [2.21.2](../spec/refs-standard-libraries/200-core/221-crypto.html#2212-aead)).
- Alice signs the one-time key, nonce and ciphertext together with Ed25519, so nobody can swap any of them. `open` checks the signature first, with Alice's `VerifyingKey` ([`ed25519`](../std/core/crypto/ed25519.html)).
- Private keys are made from `random::secret`, which puts the bytes straight into secret storage, and the shared secret and derived key are `Secret`s too, so no key byte lands in an ordinary `List`. Nonces are public and come from `random::bytes`. The algorithms never fetch randomness themselves ([Key material](../learn/19-crypto-and-tls.html#key-material-secret)). Optional results are unwrapped with postfix `?` and `??` ([Sum types](../learn/05-sum-types.html)).
- More in [Cryptography and TLS](../learn/19-crypto-and-tls.html).
