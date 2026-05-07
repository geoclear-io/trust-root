# Merkle Anchoring of the Trust Root

> Append-only history, cryptographically tamper-evident.

## Why

Without Merkle anchoring, a sufficiently-privileged attacker (someone who compromises a publication identity, for example) could rewrite the historical contents of `jwks.json` to replace a genuinely-issued key with a malicious one. Receipts signed by the original key would then verify against the substituted key only if the attacker also possessed the substituted private key — which they wouldn't, because the original private key is hardware-managed and non-extractable. So the math doesn't break.

But: the *appearance* that history was preserved unchanged is itself a property worth defending. Anchoring each new publication's hash to the previous publication's hash creates a chain that an outside observer can verify without trusting any single point in time. If anyone tampers with a historical entry, the chain breaks at that point and is detectable by any party with a snapshot from after the tampering.

## How (planned, not yet active at time-zero)

Each new JWKS publication (and each `revoked-kids.json` update, and each tagged release) includes a `previous_root_hash` field referencing the SHA-256 of the previous publication's tree (computed over the canonical-JSON of the publication's contents).

```json
{
  "keys": [...],
  "publication_metadata": {
    "publication_id": "2026-04-29-rotation-0",
    "previous_root_hash": null,
    "this_root_hash": "sha256:6f7a478aa028344bd851af495194d58347737eea126287c4bfcbdb6a69f63dca",
    "issued_at": "2026-04-29T00:00:00Z"
  }
}
```

For the time-zero seed (no prior publication), `previous_root_hash` is `null`. Every subsequent publication references its predecessor.

A verifier walking the chain can confirm:
- Each publication's `this_root_hash` equals the canonical-JSON-hash of its `keys` field.
- Each publication's `previous_root_hash` equals the previous publication's `this_root_hash`.
- The chain is unbroken from time-zero to the present.

A partial third-party snapshot from any point in time (e.g. an auditor's archive of `jwks.json` taken in 2027) anchors any subsequent claim about contemporaneous history: even if this repository is compromised after the snapshot, the auditor's snapshot proves what was published before the compromise.

## Status

The Merkle anchoring scheme is **described here** and **not yet actively enforced** at time-zero (this seed publication). Activation lands with the automated rotation pipeline — the first rotation will populate `previous_root_hash` from this seed's `this_root_hash`, and every subsequent rotation continues the chain.

Until automated rotation is live, the seed publication's `this_root_hash` is computed as `sha256:6f7a478aa028344bd851af495194d58347737eea126287c4bfcbdb6a69f63dca` over the byte-identical contents of `jwks.json` as published here on 2026-04-29.

## Out of scope (for now)

- **Certificate Transparency-style log integration.** A future enhancement could submit each publication to a public append-only log (e.g. Sigstore's Rekor, or a dedicated CT log for trust-root publications). This adds a fourth-party witness to the chain. Tracked as a roadmap item.
- **Sigstore commit signing.** Each commit in this repository will be signed with a separate cosign identity (distinct from the response-signing key) once the automated rotation pipeline is provisioned. This protects against publication-side tampering at the commit level, in addition to the file-content Merkle chain described here.
