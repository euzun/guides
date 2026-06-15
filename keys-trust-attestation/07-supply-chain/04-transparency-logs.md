---
title: "Transparency logs — public, append-only auditability"
lede: "How Merkle-tree-backed append-only logs make signing operations publicly verifiable. The same construction underpins Certificate Transparency, Sigstore Rekor, and key transparency systems."
description: "How Merkle-tree-backed append-only logs make signing operations publicly verifiable. The same construction underpins Certificate Transparency, Sigstore Rekor, and key transparency systems."
---

*Builds on: §1.1 Signing & verification.*

## The mental model

A transparency log is an append-only, publicly verifiable record of events. Once something is in the log, it cannot be removed, modified, or hidden — and anyone can verify both these properties cryptographically.

The technique was invented for **Certificate Transparency** in 2013, after a series of malicious CA failures (DigiNotar, etc.). It now underpins Sigstore (Rekor), npm provenance, key transparency for end-to-end encrypted messengers, and emerging AI model provenance systems.

## The core construction: Merkle trees

```mermaid
flowchart TB
    Root[Merkle Root<br>signed by log operator]
    H12["H(H1, H2)"]
    H34["H(H3, H4)"]
    L1[Entry 1<br>hash H1]
    L2[Entry 2<br>hash H2]
    L3[Entry 3<br>hash H3]
    L4[Entry 4<br>hash H4]

    Root --> H12
    Root --> H34
    H12 --> L1
    H12 --> L2
    H34 --> L3
    H34 --> L4
```

Every entry is a leaf in a Merkle tree. Internal nodes are hashes of their children. The root is the hash that represents the entire log state.

The log operator periodically signs the current root with a key the public knows. The signed root is the cryptographic commitment to the log's contents at that moment.

## What the log enables

| Property | How it's proven |
| --- | --- |
| Inclusion — "entry X is in the log" | Inclusion proof: a list of sibling hashes from the leaf to the root |
| Consistency — "the new log root is an extension of the old root" | Consistency proof: hashes showing the old tree is a subtree of the new tree |
| Append-only — "no entries were modified or deleted" | Consistency proofs over time prove the log only grew |
| Discoverability — "I can find all entries about X" | Log clients query and download; auditors monitor continuously |

## Inclusion and consistency proofs in detail

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Log
    participant Auditor

    rect rgb(230, 240, 250)
    Note over Client,Log: PHASE 1 - Submit entry
    Client->>Log: Submit entry
    Log->>Log: Append entry as leaf, recompute Merkle tree
    Log->>Log: Compute new signed log root
    Log-->>Client: Inclusion proof, signed log root at this point
    end

    rect rgb(245, 240, 220)
    Note over Auditor,Log: PHASE 2 - Continuous auditing
    Auditor->>Log: Fetch latest signed root
    Log-->>Auditor: New signed root
    Auditor->>Log: Request consistency proof from previous root to new root
    Log-->>Auditor: Consistency proof
    Auditor->>Auditor: Verify proof - new root genuinely extends old root
    Auditor->>Auditor: Confirm log is append-only between these two snapshots
    end

    rect rgb(245, 230, 240)
    Note over Client: PHASE 3 - Verifier checks an entry later
    Client->>Log: Fetch entry plus inclusion proof
    Log-->>Client: Entry, proof
    Client->>Client: Recompute root from leaf plus sibling hashes
    Client->>Client: Compare to a signed root that auditors have checked for consistency
    Note over Client: Inclusion alone only proves "matches some signed root" - trusting that root requires the gossip/consistency checking from Phase 2
    Client->>Client: Entry confirmed present on the audited, append-only history
    end
```

## Why this matters cryptographically

The cryptographic guarantee is: **the log operator cannot lie about what's in the log without being caught by independent observers.** If a log tries to:

- Remove an entry — the consistency proof fails for any continuous observer
- Modify an entry — its leaf hash changes, which changes every node on the path to the root, so the recomputed root no longer matches the previously signed root; inclusion and consistency proofs against that earlier root fail
- Hide an entry from some observers but not others (split view attack) — observers comparing notes detect the divergence

The defense is operator-independent monitoring. Anyone can run an auditor that polls the log and verifies consistency proofs. If a fork is detected, it's a major event — the operator is publicly caught misbehaving.

## Real-world transparency logs

| Log | What it records | Who runs auditors |
| --- | --- | --- |
| Certificate Transparency (CT) | Every TLS certificate ever issued by a public CA | Browser vendors, security researchers, domain owners |
| Sigstore Rekor | Signing entries and attestations (keyless and key-based) | Sigstore project, downstream verifiers, identity owners |
| npm provenance log | SLSA provenance for npm package releases | npm, GitHub, security tooling |
| Key Transparency (signal, WhatsApp) | Public key bindings for end-to-end encryption | App developers, third-party auditors |
| Binary transparency (early proposals) | Software binary releases | Software distros, package managers |

## The operator-independence property

<div class="callout info"><div class="callout-label">Why transparency logs don't replace PKI</div><p>A transparency log doesn't replace the underlying signing; it adds public observability. CT logs don't issue certificates — CAs still do that. Sigstore Rekor doesn't sign artifacts — Fulcio-issued certs do. The log is the audit trail, not the trust authority. The combination — signing + transparency — is what gives the cryptographic + operational guarantee.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>Transparency logs add public auditability on top of signing. Cryptographic operator independence — the log cannot lie without being caught — is the property that makes them powerful. They turn 'trust the operator' into 'trust the operator, but verify continuously.'</p></div>
