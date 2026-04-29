# Contributing to geoclear-io/trust-root

> This repository is **automated infrastructure**, not a human-authored project. Most pushes here come from a dedicated GitHub App tied to the GeoClear KMS rotation pipeline; humans should generally not push directly.

## What pushes are accepted

| Source | Accepted? | Notes |
|---|---|---|
| Trust-root rotation Lambda (cosign-signed) | ✅ Yes — automated | Pushes new `jwks.json` on every HSM key rotation; tags releases `vYYYY-MM-DD-rotation-N`. Append-only (never modifies or deletes existing entries). |
| Human PRs to `README.md` / `CONTRIBUTING.md` / `MERKLE.md` | ✅ Yes — via PR | Documentation improvements welcome. Branch protection requires PR review for non-automated paths. |
| Human PRs to `jwks.json` / `revoked-kids.json` / `history/*` | ❌ No | These are owned by the rotation pipeline. Direct human edits would break the cryptographic chain of custody. If a key needs to be revoked manually (security incident), the rotation Lambda is the path; runbook: `geoclear/docs/runbooks/RUNBOOK-TRUST-ROOT-GITHUB-MIRROR.md`. |

## Branch protection

The `main` branch is protected:

- Force-push: **disabled**.
- Required PR reviews: **1 approval** for non-automated paths.
- Required signed commits: **enforced** (once cosign identity for the rotation Lambda is provisioned — interim period: not enforced until then).
- Push restrictions: only the rotation Lambda's GitHub App identity may push directly to `main`; humans use PRs.

## Append-only contract

Once published, an entry in `jwks.json` (current) or `history/jwks-YYYY-MM-DD.json` (past) is **never removed**. Receipts issued under that key must remain verifiable indefinitely. This is a hard governance rule, not a soft convention. Removing or modifying a historical entry would invalidate every receipt that depended on it — a permanent, unrecoverable harm to customers' audit trails.

If a key is compromised, the correct response is:

1. Add the kid to `revoked-kids.json` with a reason and timestamp.
2. Issue a new key (next rotation publishes a new entry to `jwks.json`).
3. Receipts signed under the revoked kid remain mathematically verifiable but are flagged operationally as untrusted by any compliant verifier.

This separation — verifiability vs. trust — preserves the mathematical truth of historical receipts while letting operators communicate compromise status to verifiers.

## Reporting security issues

If you believe you've found a security issue with the GeoClear trust-root infrastructure (e.g. you can prove the published `jwks.json` here doesn't match what's actually used by the GeoClear API, or you've found a bypass in the verification protocol), please file a security report via the standard GeoClear vulnerability disclosure channel: see [https://geoclear.io/.well-known/security.txt](https://geoclear.io/.well-known/security.txt).

Do **not** open public GitHub issues for security concerns about this repo.

## Reporting non-security issues

GitHub Issues are disabled on this repository because it's automated infrastructure. For questions about the verification protocol, the SDK, or how to consume this trust root, see the GeoClear documentation at [https://geoclear.io](https://geoclear.io) or the `geoclear-io/notary` SDK repo (forthcoming).
