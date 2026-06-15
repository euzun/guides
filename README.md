# Guides

Short, beginner-friendly field guides to security, cryptography, and systems
topics — a clear, practical reference for anyone learning the material or
teaching it.

📖 **Read the rendered version (recommended):** https://euzun.github.io/guides

This repo is the **source**. It's plain Markdown with [Mermaid](https://mermaid.js.org/)
diagrams, so every page is also readable directly on GitHub — but the polished,
styled, dark-mode version with zoomable diagrams lives on the site above.

If a guide helps you, a ⭐ is appreciated — it tells me which topics to expand.

---

## Guides

### Keys, Trust & Attestation
How systems establish and prove trust with cryptography — signatures, PKI,
hardware roots of trust, key ceremonies, attestation, and supply-chain
integrity. → [`keys-trust-attestation/`](keys-trust-attestation/)

1. **Foundations** — signing/verification, key hierarchy, PKCS#11, TRNG, TLS, mTLS
2. **PKI and Trust** — certificate issuance, chain verification, key rotation, trust domains
3. **Hardware Security** — HSMs, smartcards, OTP fuses, secure/measured boot, EK/AIK
4. **Key Ceremonies** — generation, Shamir backup, recovery, share refresh
5. **Attestation** — remote attestation, the universal pattern, FIDO2/WebAuthn
6. **Token Security** — PKCE, DPoP (OAuth/OIDC)
7. **Supply Chain** — SBOM, SLSA, in-toto, transparency logs, Sigstore keyless, AI model provenance
8. **End-to-End** — firmware lifecycle, full supply-chain pipeline, signing-service design

### Coming later
OAuth / OIDC & token-based flows · secure multi-party computation (2PC) ·
fully homomorphic encryption (FHE).

---

## Contributing / feedback

Spotted an error or have a suggestion? Open an
[issue](https://github.com/euzun/guides/issues) or a pull request — every page
on the site links back to its source file here.

## Structure

```
keys-trust-attestation/      # one guide (folder name == slug)
  guide.json                 # guide metadata (title, slug, sections, order, note)
  NN-section/                # e.g. 01-foundations
    NN-topic.md              # one page; section + order derived from the numbering
```

Frontmatter per page: `title`, `lede`, `description`. Navigation order comes from
the folder/file numbering, so there's nothing to hand-maintain.

## License

Content is licensed under [CC BY 4.0](LICENSE) — free to share and adapt for any
purpose, with attribution to Erkam Uzun.
