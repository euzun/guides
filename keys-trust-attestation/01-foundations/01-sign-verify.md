---
title: "Digital signing and verification"
lede: "The atomic primitive every other concept in this guide depends on. Public-key signatures prove authenticity and integrity — and, with PKI and trusted timestamps, non-repudiation."
description: "The atomic primitive every other concept in this guide depends on. Public-key signatures prove authenticity and integrity — and, with PKI and trusted timestamps, non-repudiation."
---

*This is the starting primitive — nothing else is required first.*

## The mental model

Digital signing is the atomic primitive that everything else in this guide rests on. A keypair has two halves: a **private key** kept secret by the signer, and a **public key** shared with everyone. The math is asymmetric — what one half does, only the other half can undo.

To sign: hash the message, run a **private-key signing operation** over that hash, and attach the result. To verify: hash the message yourself, then run the **public-key verify operation** over the message hash and the signature — it returns valid or invalid.

What this gives you:

- **Authenticity** — only the private key holder could have produced this signature
- **Integrity** — if the message changed, the hashes won't match
- **Non-repudiation** — *given* a trusted key-to-identity binding (PKI, §2) and evidence the key wasn't compromised at signing time (usually a trusted timestamp or transparency log, §7.4), the signer can't later deny signing. A bare signature from an anonymous key proves possession, not identity.

<div class="callout textbook"><div class="callout-label">Textbook vs. reality</div><p><strong>You may have learned:</strong> "signing = encrypt the hash with the private key; verifying = decrypt the signature with the public key."</p><p><strong>What actually happens:</strong> only legacy <em>textbook RSA</em> looks like that. ECDSA, EdDSA, and RSA-PSS run a verification <em>equation</em> that outputs valid/invalid — nothing is "decrypted" and no hash is "recovered." The intuition that survives is: the private key produces a signature that only the matching public key can check.</p></div>

## The sequence

```mermaid
sequenceDiagram
    autonumber
    participant S as Signer with private key
    participant V as Verifier with public key

    Note over S: SIGNING
    S->>S: h = SHA256 of message
    S->>S: signature = Sign h with private key
    S->>V: message plus signature

    Note over V: VERIFICATION
    V->>V: h2 = SHA256 of message
    V->>V: ok = Verify signature h2 with public key
    Note over V: ok true means message is authentic and untampered
```

## Walkthrough

**1–2.** The signer hashes the message to a fixed-length digest. Hashing first lets you sign any size of data with a small cryptographic operation, and it's where collision resistance becomes important — if two different messages produce the same hash, the signature is ambiguous about which one was signed.

**3.** The signer runs the signing operation over the hash with their private key, producing the signature. (As the box above notes, "encrypt with the private key" is only a loose analogy for legacy RSA — modern schemes perform algorithm-specific operations, but the intuition "private key turns hash into signature" holds.)

**4.** The message and signature travel together. The signature alone tells you nothing; the signature plus the message lets a verifier check the binding.

**5–7.** The verifier independently hashes the message, then runs the verification algorithm with the public key, the message hash, and the signature. It outputs valid or invalid. (Only textbook RSA literally "recovers" a hash to compare; ECDSA/EdDSA/RSA-PSS just check an equation.) A valid result means the message is authentic and unmodified.

## Modern algorithms

You'll see these in the wild:

- **RSA-PSS** — legacy but ubiquitous. Large keys, large signatures, well understood.
- **ECDSA** — smaller signatures, faster operations, more failure modes (random nonce reuse breaks it entirely).
- **EdDSA / Ed25519** — newer, deterministic signatures, no random nonce needed, simpler implementation.
- **ML-DSA / Dilithium** — NIST-standardized post-quantum signature scheme, much larger keys and signatures but resistant to quantum attacks.

<div class="callout warn"><div class="callout-label">ECDSA nonce reuse</div><p>If you ever sign two different messages with the same ECDSA nonce, the math leaks your private key — the failure that broke Sony's PlayStation 3 firmware signing (see §1.4 TRNG for that and other RNG disasters). Always use a fresh, cryptographically random nonce per signature, or deterministic ECDSA (RFC 6979), which derives the nonce from the message hash and private key.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>Signing proves three things at once — that the signer holds the private key, that the message is the one signed, and that the message hasn't changed. The whole edifice of cryptographic trust is recursive application of this primitive.</p></div>
