---
title: "Firmware lifecycle — sign, boot, attest"
lede: "End-to-end walkthrough from signing a firmware binary in a secure facility, through the customer GPU's boot verification, to remote attestation at runtime. Three acts, three trust chains, one continuous story."
description: "End-to-end walkthrough from signing a firmware binary in a secure facility, through the customer GPU's boot verification, to remote attestation at runtime. Three acts, three trust chains, one continuous story."
---

*Builds on: §3.4 Secure boot, §3.6 EK/AIK, §5.1 Remote attestation.*

## The mental model

Pieces of the firmware story have been covered in earlier sections — secure boot, measured boot, PCRs, attestation, key ceremonies. This page connects them end to end: from the moment a firmware binary is signed in a secure facility, through the boot verification on a customer's GPU, to the attestation a customer relies on at runtime.

The story has three acts, each backed by a different trust chain.

## The high-level shape

```mermaid
flowchart TB
    subgraph act1["ACT 1 - SIGNING (in NVIDIA secure facility)"]
        S1[Build firmware binary]
        S2[Submit to Signing Service]
        S3[HSM signs binary with leaf key]
        S4[Bundle binary + signature + cert chain]
    end

    subgraph act2["ACT 2 - BOOT VERIFICATION (on customer GPU)"]
        B1[Boot ROM reads fuses]
        B2[Read firmware bundle from flash]
        B3[Hash, extend PCR, verify chain]
        B4[Execute or HALT]
    end

    subgraph act3["ACT 3 - REMOTE ATTESTATION (at runtime)"]
        A1[Customer sends nonce]
        A2[GPU signs PCRs + nonce with AIK]
        A3[Customer verifies hardware + firmware]
    end

    act1 --> act2 --> act3

    sign["See pages: 4.2 generation, 3.1 HSM, 2.1 issuance"] -.- act1
    boot["See pages: 3.3 fuses, 3.4 secure boot, 3.5 measured boot"] -.- act2
    att["See pages: 3.6 EK/AIK, 5.1 remote attestation"] -.- act3
```

## Detailed sequence: signing, boot, attestation as one chain

```mermaid
sequenceDiagram
    autonumber
    participant Eng as NVIDIA Firmware Engineer
    participant SS as Signing Service
    participant HSM as Production Signing HSM
    participant Pub as Public Reference Values
    participant Flash as GPU Flash Storage
    participant ROM as GPU Boot ROM
    participant Fuses as OTP Fuses
    participant PCR as PCR Registers
    participant FW as Firmware running
    participant AIK as AIK in shielded silicon
    participant Cust as Customer Verifier

    rect rgb(230, 240, 250)
    Note over Eng,Pub: ACT 1 - SIGNING IN NVIDIA SECURE FACILITY
    Eng->>SS: Submit firmware binary plus metadata
    SS->>SS: Authenticate engineer, check policy
    SS->>SS: Multi-party approval if release class
    SS->>HSM: Sign hash of binary using current leaf private key
    HSM-->>SS: Signature
    SS->>SS: Bundle binary plus signature plus leaf cert plus intermediate cert plus root cert
    SS->>Pub: Publish reference PCR value computed from binary hash
    Pub->>Pub: Reference value signed by NVIDIA Firmware Root
    SS-->>Eng: Signed firmware bundle ready for distribution
    end

    Note over Eng,Flash: Bundle distributed via update channels, eventually written to GPU flash

    rect rgb(245, 240, 220)
    Note over Flash,FW: ACT 2 - BOOT VERIFICATION ON GPU
    Note over ROM: Power-on. ROM begins executing from silicon.
    ROM->>Fuses: Read firmware root pubkey hash and anti-rollback minimum
    Fuses-->>ROM: hash and min version, immutable
    ROM->>Flash: Read firmware bundle
    Flash-->>ROM: binary, signature, cert chain, root cert
    ROM->>ROM: hash_fw = SHA256 of binary
    ROM->>PCR: Extend a boot-stage PCR with hash_fw
    ROM->>ROM: Hash root cert pubkey, compare to fuses. HALT if mismatch.
    Note over ROM: Root cert came from untrusted flash - the fuse-hash match is what makes it trustworthy
    ROM->>ROM: Verify intermediate signature using root pubkey
    ROM->>ROM: Verify leaf signature using intermediate pubkey
    ROM->>ROM: Verify binary signature using leaf pubkey
    ROM->>ROM: Check version against anti-rollback minimum from fuses, else HALT
    ROM->>FW: Jump to firmware entry point if all checks passed
    Note over FW: Firmware now running. PCRs hold ordered measurements.
    end

    rect rgb(245, 230, 240)
    Note over Cust,AIK: ACT 3 - REMOTE ATTESTATION AT RUNTIME
    Cust->>Cust: Generate fresh random nonce
    Cust->>FW: Attestation request with nonce
    FW->>PCR: Read current PCR values
    PCR-->>FW: PCRs as set during Act 2 boot
    FW->>FW: Build quote - PCRs plus nonce
    FW->>AIK: Sign quote with AIK private key in shielded silicon
    AIK-->>FW: Signature
    FW->>Cust: Send quote plus signature plus AIK cert plus EK cert

    Cust->>Cust: Verify EK cert chain to NVIDIA Manufacturing CA in trust store
    Note over Cust: Confirms genuine NVIDIA hardware
    Cust->>Cust: Verify AIK bound to EK via credential activation
    Cust->>Cust: Verify quote signature using AIK pubkey
    Cust->>Cust: Confirm echoed nonce matches sent nonce
    Cust->>Pub: Look up expected PCR measurement for the claimed firmware build
    Pub-->>Cust: Signed reference PCR measurement
    Cust->>Cust: Compare reported PCR to the expected measurement
    Note over Cust: Confirms expected firmware version is running
    end

    Note over Eng,Cust: Customer trusts NVIDIA Mfg CA plus Firmware CA plus cryptographic math. Nobody else.
```

## The trust chains in this story

| Chain | Anchor | What it proves | Used in |
| --- | --- | --- | --- |
| Firmware signing chain | Root pubkey hash in OTP fuses | Binary came from NVIDIA's signing service | Act 2 (boot) |
| Manufacturing identity chain | NVIDIA Manufacturing CA in customer's trust store | Hardware is a genuine NVIDIA chip | Act 3 (attestation) |
| Reference value chain | NVIDIA Firmware Root in customer's trust store | Expected PCR for this firmware version | Act 3 (attestation) |

Three separate trust chains, three separate roots, all rooted in NVIDIA but designed for different blast radii. Compromise of one doesn't compromise the others.

## What can go wrong, where

| Stage | Failure mode | Mitigation |
| --- | --- | --- |
| Signing | Insider produces unauthorized signing | M-of-N approval, audit logs, customer-side publish-time verification |
| Signing | HSM compromised, leaf key extracted | Short-lived leaf certs, intermediate revocation, fast rotation |
| Signing | Root key compromised | Catastrophic, recovery via backup root in fuses if designed in |
| Boot | Boot ROM exploit (e.g., checkm8 class) | Frozen at fabrication, cannot patch, mitigated by code minimalism |
| Boot | Downgrade to vulnerable firmware version | Anti-rollback minimum version fused at manufacturing |
| Attestation | EK extracted from chip | Side-channel resistant silicon, physical attack cost |
| Attestation | Trust store poisoning at customer | Hardware-rooted trust stores, signed updates, TOFU pinning |

<div class="callout info"><div class="callout-label">Why three acts and not one</div><p>Signing happens once per release. Boot happens every power-cycle on every device. Attestation happens on demand from customers. Different cadences, different security boundaries, different trust questions. The architecture separates them so the operational tempo of one doesn't force compromises on the others.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>Firmware lifecycle is three acts with three trust chains. The signing act produces a signed binary anchored in the firmware root. The boot act verifies offline using a hardware-anchored hash in fuses. The attestation act binds hardware identity and software state into one signed statement. Each act stands alone; together they give end-to-end trust.</p></div>
