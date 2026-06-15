---
title: "Certificate chain verification"
lede: "How trust propagates from a leaf certificate back to a root in your trust store. Recursive signature checking with pre-configured anchors."
description: "How trust propagates from a leaf certificate back to a root in your trust store. Recursive signature checking with pre-configured anchors."
---

*Builds on: §2.1 Certificate issuance.*

## The mental model

When you receive a leaf cert, it doesn't stand alone. It comes with a **chain** of certificates linking it back to a trusted root. Verification walks the chain upward — each cert verified by its parent — until you reach a root your trust store recognizes.

The trust store is the only thing the verifier decides. Everything else is signature math against that anchor.

## The hierarchy

```mermaid
flowchart TB
    Root[Root CA<br>self-signed<br>in trust store]
    Int1[Intermediate CA]
    Int2[Intermediate CA]
    Leaf1[Leaf cert<br>server, device, code signer]
    Leaf2[Leaf cert]
    Leaf3[Leaf cert]

    Root -->|signs| Int1
    Root -->|signs| Int2
    Int1 -->|signs| Leaf1
    Int1 -->|signs| Leaf2
    Int2 -->|signs| Leaf3
```

- **Root CA** — top of the chain. Self-signed. The only thing that makes you trust a root is that you've added it to your trust store.
- **Intermediate CA** — signed by the root or another intermediate. Used for day-to-day issuance so the root can stay offline.
- **Leaf** — end-entity cert, signed by an intermediate. The actual server, device, or code signer.

## The verification flow

```mermaid
sequenceDiagram
    autonumber
    participant V as Verifier
    participant TS as Trust Store

    Note over V: Bundle received - artifact, signature, leaf cert, intermediate cert
    V->>V: Verify artifact signature using leaf public key
    V->>V: Verify leaf cert signature using intermediate public key
    V->>TS: Does the received root match a root I already trust
    TS-->>V: Yes - its fingerprint matches a trust-store anchor (no signature checked here)
    V->>V: Verify intermediate cert signature using the trusted root public key
    V->>V: Check validity periods on all three certs
    V->>V: Check revocation status via CRL, OCSP, or bundled list
    V->>V: Check cert usage extensions allow this operation
    Note over V: All checks pass - artifact is trusted
```

## Walkthrough

**1.** Verify the artifact's signature using the leaf cert's embedded public key. This is the actual thing you care about — the leaf signed something, and you want to verify it.

**2–3.** But why trust the leaf? Because it's signed by the intermediate. And why trust the intermediate? Because it's signed by the root. **Each step uses the same signature-verification operation** — just applied recursively.

**4–5.** The root is the anchor — but you don't *cryptographically verify* it at all. It's self-signed, so its own signature proves nothing. Instead you check that the root you received **matches (by fingerprint) a root already in your trust store**, and then use that trusted copy's public key to verify the intermediate. The trust store is configured out-of-band — pre-installed in OSes, browsers, distributions, or firmware fuses. (Whether the anchor is a full certificate or just a pinned hash/fuse value, the principle is the same: trust comes from the pre-installed copy, not from any signature on the root.)

**6.** Time bounds matter. Every cert has not-before / not-after dates. Even a mathematically valid signature on an expired cert is rejected.

**7.** Revocation: was this cert revoked before its scheduled expiry? Mechanisms vary (CRL, OCSP, bundled lists for offline verifiers).

**8.** Usage extensions: a code-signing cert can't be used to authenticate a TLS server, and vice versa. The cert states what it's allowed to do.

<div class="callout warn"><div class="callout-label">Where the trust really lives</div><p>Cryptography doesn't bootstrap trust — it extends trust you already have. The root's authority comes from being in your trust store, which someone decided to put there. If an attacker can write to your trust store, no signature math above it can save you.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>Chain verification is recursive signature checking that terminates at a pre-configured trust anchor. The math is mechanical; the trust is operational — it lives in whichever entity decides what goes into the trust store.</p></div>
