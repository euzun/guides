---
title: "HSM architecture and the hardware trust boundary"
lede: "What makes a Hardware Security Module more than a server with keys on disk — tamper-resistant hardware, internal TRNG, certified design, and the operations-on-handles contract."
description: "What makes a Hardware Security Module more than a server with keys on disk — tamper-resistant hardware, internal TRNG, certified design, and the operations-on-handles contract."
---

*Builds on: §1.3 PKCS#11.*

## The mental model

An HSM (Hardware Security Module) is a physical device whose only job is to hold cryptographic keys and perform operations with them — never exposing the keys themselves to the outside world. It's a black box: you send it data and a key handle, it returns a result. The trust boundary lives at the HSM's edge.

What makes an HSM more than just a server with a key on disk:

- **Tamper-resistant hardware** — sensors detect physical attack and zeroize keys
- **Crypto coprocessors** — dedicated silicon for AES, RSA, ECC operations
- **Internal TRNG** — hardware entropy source for key generation
- **Strict API contract** — typically PKCS#11; operations on handles only
- **Certified to FIPS standards** — now **FIPS 140-3** (which aligns with ISO/IEC 19790); FIPS 140-2 is legacy and moved to the historical list, security levels 1–4

## FIPS levels in one paragraph each

- **Level 1** — basic security requirements, mostly software-based. Smartphones at this level if certified.
- **Level 2** — role-based authentication, tamper-evident seals. Most cloud HSMs.
- **Level 3** — identity-based authentication, tamper-resistant (zeroize on attempted intrusion). Where most enterprise HSMs operate.
- **Level 4** — environmental failure protection (voltage, temperature), full envelope of detection. Defense and high-assurance use.

## The trust contract

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Driver as PKCS#11 Driver
    participant HSM

    rect rgb(245, 240, 220)
    Note over App,HSM: Setup (one time)
    App->>Driver: C_OpenSession, C_Login
    Driver->>HSM: Authenticate
    HSM-->>Driver: session handle
    end

    rect rgb(230, 240, 250)
    Note over App,HSM: Key generation
    App->>Driver: GenerateKeyPair non-extractable
    Driver->>HSM: Generate inside hardware
    HSM->>HSM: TRNG generates random key
    HSM->>HSM: Store with non-extractable flag
    HSM-->>Driver: handle to key
    Driver-->>App: handle
    Note over HSM: Private key value never crosses HSM boundary
    end

    rect rgb(245, 230, 240)
    Note over App,HSM: Routine signing
    App->>Driver: Sign data with key handle
    Driver->>HSM: C_Sign data, handle
    HSM->>HSM: Hash, sign with referenced key
    HSM-->>Driver: signature
    Driver-->>App: signature
    end

    Note over App,HSM: At no point does the app possess the key
```

## HSM types in the wild

| Type | Form factor | Use case |
| --- | --- | --- |
| Network HSM | 1U or 2U server rack appliance | Enterprise PKI, code signing |
| PCIe HSM card | PCIe card in a server | High-throughput crypto in a single host |
| USB HSM | Small USB device | Developer / individual use, dev environments |
| Cloud HSM | Managed service | AWS CloudHSM, Azure Dedicated HSM, GCP Cloud HSM |
| HSM-as-a-Service | Multi-tenant managed | AWS KMS, Azure Key Vault, GCP Cloud KMS — abstracted |

## HSM clustering and high availability

Single HSMs are single points of failure. Production deployments use clusters:

- Multiple HSMs in a trust group share the same keys via **cloning**
- Cloning happens hardware-to-hardware over encrypted channels; plaintext keys never exposed
- Operations can fail over from one HSM to another with the same key
- Different vendors handle this differently — Thales Luna has its own cluster model, AWS CloudHSM clusters are AWS-managed, etc.

<div class="callout info"><div class="callout-label">HSM cloning vs key escrow</div><p>Cloning is hardware-to-hardware key transfer within an authorized trust group. It maintains the property that keys never exist outside hardware. Key escrow — keeping a plaintext copy somewhere — is a different concept and is generally avoided because it creates exactly the vulnerability HSMs are designed to prevent.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>An HSM is the hardware boundary inside which key material exists. The whole stack above — PKCS#11 APIs, signing services, PKI hierarchies — is built around the contract that applications hold handles, the HSM holds values.</p></div>
