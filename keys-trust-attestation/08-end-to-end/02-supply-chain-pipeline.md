---
title: "Full supply chain pipeline"
lede: "End-to-end walkthrough of a modern signed software pipeline — from developer commit through hardened build, SBOM and SLSA generation, Sigstore signing, Rekor logging, and consumer verification at deploy time."
description: "End-to-end walkthrough of a modern signed software pipeline — from developer commit through hardened build, SBOM and SLSA generation, Sigstore signing, Rekor logging, and consumer verification at deploy time."
---

*Builds on: §7.2 SLSA, §7.3 in-toto, §7.5 Sigstore keyless.*

## The mental model

The pieces of modern supply chain assurance — SBOM, SLSA, in-toto, Sigstore, Rekor — were covered in section 7. Each is a focused tool. This page assembles them into a single end-to-end pipeline: a developer commits source, a build produces an artifact, attestations accumulate, and a downstream consumer verifies everything before deploying.

This is what a SLSA L3-or-better pipeline actually looks like in 2026.

## The high-level shape

```mermaid
flowchart TB
    subgraph dev["DEVELOPMENT"]
        D1[Developer signs commit]
        D2[PR review with required signers]
        D3[Merge to main]
    end

    subgraph build["BUILD (HARDENED PLATFORM)"]
        B1[Hermetic, ephemeral build env]
        B2[Generate artifact]
        B3[Generate SBOM]
        B4[Generate SLSA provenance]
    end

    subgraph attest["ATTESTATION"]
        A1[Wrap SBOM and provenance in DSSE]
        A2[Sign via Sigstore keyless]
        A3[Log in Rekor transparency log]
    end

    subgraph dist["DISTRIBUTION"]
        DI1[Push artifact and attestations to registry]
        DI2[Customer pulls everything]
    end

    subgraph verify["VERIFICATION"]
        V1[Verify Sigstore signatures]
        V2[Verify Rekor log inclusion]
        V3[Check SLSA policy]
        V4[Scan SBOM for CVEs]
        V5[Deploy or reject]
    end

    dev --> build --> attest --> dist --> verify

    refs1["See pages: 2.1 issuance, 7.5 sigstore"] -.- attest
    refs2["See pages: 7.1 SBOM, 7.2 SLSA, 7.3 in-toto"] -.- build
    refs3["See pages: 7.4 transparency logs"] -.- attest
```

## The end-to-end sequence

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant Repo as Source Repo
    participant CI as Hardened Build Runner
    participant Fulcio
    participant Rekor
    participant Reg as OCI Registry
    participant Cust as Customer Verifier
    participant Policy as Customer Policy Engine
    participant Prod as Production Cluster

    rect rgb(230, 240, 250)
    Note over Dev,Repo: PHASE 1 - SOURCE
    Dev->>Dev: Sign commit using personal key or Sigstore
    Dev->>Repo: git push
    Dev->>Repo: Open PR
    Repo->>Repo: Required reviewers approve, signatures verified
    Repo->>Repo: Merge to main
    Repo->>CI: Trigger build on tagged release
    end

    rect rgb(245, 240, 220)
    Note over CI: PHASE 2 - BUILD
    CI->>CI: Spin up ephemeral isolated environment
    CI->>Repo: Fetch source by commit hash
    CI->>CI: Run hermetic build with declared inputs only
    CI->>CI: Produce artifact, e.g. container image, binary
    CI->>CI: Compute SHA-256 of artifact
    CI->>CI: Generate SBOM via Syft, scanning artifact
    CI->>CI: Generate SLSA provenance - source ref, builder, inputs, commands
    end

    rect rgb(245, 230, 240)
    Note over CI,Rekor: PHASE 3 - ATTESTATION AND LOGGING
    CI->>CI: Wrap SBOM in in-toto Statement, DSSE envelope
    CI->>CI: Wrap SLSA provenance in in-toto Statement, DSSE envelope
    CI->>Fulcio: Request ephemeral cert using OIDC workload identity
    Fulcio->>CI: Short-lived signing cert, 10 min
    CI->>CI: Sign DSSE envelopes using ephemeral key
    CI->>Rekor: Log each attestation
    Rekor->>Rekor: Append entries, return inclusion proofs
    Rekor-->>CI: Log entry indices, signed root
    CI->>CI: Zeroize ephemeral private key
    end

    rect rgb(220, 240, 240)
    Note over CI,Reg: PHASE 4 - DISTRIBUTION
    CI->>Reg: Push artifact, signed SBOM attestation, signed SLSA attestation
    Reg->>Reg: Store everything indexed by artifact digest
    end

    Note over Dev,Prod: Some time later, customer deploys

    rect rgb(250, 240, 220)
    Note over Cust,Prod: PHASE 5 - VERIFICATION AND DEPLOYMENT
    Cust->>Reg: Pull artifact plus all attestations
    Reg-->>Cust: Artifact, SBOM attestation, SLSA attestation
    Cust->>Cust: Verify each DSSE signature using Fulcio cert
    Cust->>Cust: Verify Fulcio cert chains to Sigstore root in trust store
    Cust->>Rekor: Verify log inclusion proofs
    Rekor-->>Cust: Confirmed in log at time T
    Cust->>Cust: Verify signature time T was within Fulcio cert validity
    Cust->>Policy: Evaluate SLSA provenance against policy
    Policy->>Policy: Check - source commit matches expected, builder is trusted, no off-policy steps
    Policy-->>Cust: SLSA policy pass
    Cust->>Policy: Evaluate SBOM against vuln database
    Policy->>Policy: No critical CVEs in declared components
    Policy-->>Cust: SBOM policy pass
    Cust->>Prod: Deploy artifact
    end
```

## What's being proven at the end

By the time the artifact reaches production, the customer has cryptographically verified:

- The artifact came from a specific source commit (SLSA provenance)
- The build ran on a hardened, isolated platform with no off-policy steps (SLSA provenance)
- The build was signed by an identity the customer trusts (Sigstore cert from Fulcio)
- The signature was logged publicly in Rekor (audit trail exists)
- The artifact's declared contents have no known critical vulnerabilities (SBOM scan)
- None of this evidence has been tampered with since publication (signatures + log inclusion proofs)

None of this requires trusting any intermediary (registry, network, CI vendor) — only the upstream signing identities and the public log.

## How this defends against real attacks

| Attack | What it tries to do | What blocks it |
| --- | --- | --- |
| Tampered binary in registry | Substitute malicious artifact under same name/tag | SHA-256 hash in provenance won't match; verification fails |
| SolarWinds-style build pipeline compromise | Inject malicious code during build | SLSA L3+ isolation + hermetic builds; provenance shows what actually ran |
| Stolen long-lived signing key | Sign arbitrary artifacts | No long-lived keys exist; identity is the only authority, monitored via Rekor |
| Dependency confusion attack | Slip malicious dependency into build | SBOM shows actual dependency, scan flags unexpected source |
| Backdated signature | Sign artifact, claim it was signed earlier | Rekor's signed-entry timestamp, cross-checked by log monitors (and optionally an independent RFC 3161 timestamp), establishes when signing happened — its trust depends on Rekor's auditability, not the bare timestamp |
| Unauthorized signature by valid identity | Compromise an OIDC account, sign things | Identity owner monitors Rekor, detects unexpected entries |

## What's still hard

<div class="callout warn"><div class="callout-label">Where the model is incomplete</div><p>Source-side compromise (developer's account / dev environment) still passes through this pipeline if the malicious code gets signed-committed. Branch protection, two-person review, and source-level signing help, but the chain is only as strong as its weakest link. The 2024 xz-utils / liblzma backdoor (CVE-2024-3094) — malicious code introduced upstream by a long-trusted maintainer — shows how source-side compromise slips past even strong build provenance.</p></div>

<div class="takeaway"><div class="label">Takeaway</div><p>Modern supply chain assurance is the composition of focused tools: SBOM for contents, SLSA for build integrity, in-toto for signed claims, Sigstore for keyless identity-based signing, Rekor for transparency. Together they give end-to-end cryptographic evidence from source to production, verifiable by any consumer without trusting any intermediary.</p></div>
