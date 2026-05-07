# GeoClear Trust Root

> **Public verification keys for every signed receipt issued by the GeoClear API.**
> This repository is the canonical, append-only, vendor-independent publication of GeoClear's response-signing public keys. Anyone can verify any GeoClear receipt using only the contents of this repository — no GeoClear infrastructure required.

This repository exists because the verification of a signed receipt should never depend on the issuer's continued operation. If GeoClear's website, API, or business entity disappears, every receipt ever issued under the keys published here remains independently verifiable by anyone with a copy of the receipt and read access to GitHub (or to the corresponding IPFS publication, which mirrors this repository's contents).

The contents of this repository are mirrored to IPFS via multiple pinning services. See the published discovery manifest at [`https://geoclear.io/.well-known/location-trust.json`](https://geoclear.io/.well-known/location-trust.json) for current publication targets across all channels.

## What's in this repo

| File | Purpose |
|---|---|
| [`jwks.json`](jwks.json) | The current JSON Web Key Set — public keys used to verify GeoClear receipts. RFC 7517 format. |
| [`revoked-kids.json`](revoked-kids.json) | List of key IDs (`kid` values) that have been revoked. Receipts signed by a revoked kid MUST NOT be trusted. Empty at time-zero. |
| [`history/`](history/) (future) | Append-only directory of historical JWKS publications, one per rotation, named `jwks-YYYY-MM-DD.json`. The current `jwks.json` always reflects the latest publication. |
| [`MERKLE.md`](MERKLE.md) | Description of the Merkle anchoring scheme — each new JWKS publication references the previous publication's tree hash, preventing silent rewriting of history. |
| [`LICENSE`](LICENSE) | MIT — these public keys + verification protocol are free for anyone to consume, fork, mirror, or republish. |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | This repository is automated. Humans should generally not push to it; rotation events propagate signed commits via a dedicated automation identity. |

## How to verify a GeoClear receipt offline

You need three things:
1. **The receipt itself** — a JWS Compact serialization (typically arrives in the `X-GeoClear-Receipt` HTTP response header on every GeoClear API response).
2. **The receipt's `kid`** — published in the `X-GeoClear-Receipt-Kid` header alongside the receipt, and also embedded in the JWS protected header.
3. **The matching public key** — found in this repository's `jwks.json` (or in `history/jwks-YYYY-MM-DD.json` for receipts signed by a previously-rotated key).

Verification is RFC 7515 / 7518 compliant ECDSA P-384 with SHA-384 (algorithm: `ES384`). Any standards-compliant JOSE library can verify a GeoClear receipt; an example using the browser-native WebCrypto API is below.

### 10-line WebCrypto verifier (browser, Node ≥18, MCP servers)

```javascript
// Paste in browser console or run in Node ≥18 / Deno / MCP server.
// Replaces the GeoClear API in the verification path entirely — keys come from this repo, math runs locally.

async function verifyGeoClearReceipt(jwsCompact, jwksUrl = 'https://raw.githubusercontent.com/geoclear-io/trust-root/main/jwks.json') {
  const [headerB64, payloadB64, sigB64] = jwsCompact.split('.');
  const header = JSON.parse(atob(headerB64.replace(/-/g, '+').replace(/_/g, '/')));
  const jwks = await fetch(jwksUrl).then(r => r.json());
  const jwk = jwks.keys.find(k => k.kid === header.kid);
  if (!jwk) throw new Error(`Unknown kid: ${header.kid}`);
  const key = await crypto.subtle.importKey('jwk', jwk, { name: 'ECDSA', namedCurve: 'P-384' }, false, ['verify']);
  const sig = Uint8Array.from(atob(sigB64.replace(/-/g, '+').replace(/_/g, '/')), c => c.charCodeAt(0));
  const data = new TextEncoder().encode(`${headerB64}.${payloadB64}`);
  return crypto.subtle.verify({ name: 'ECDSA', hash: 'SHA-384' }, key, sig, data);
}

// Usage:
const receipt = 'eyJhbGciOiJFUzM4NCIs...'; // Paste your X-GeoClear-Receipt header value
const valid = await verifyGeoClearReceipt(receipt);
console.log(valid ? '✓ verified' : '✗ INVALID');
```

The verification path:
1. Fetches `jwks.json` from this repo (or any mirror you trust — GitHub raw URL, IPFS gateway, your own copy on disk).
2. Finds the JWK whose `kid` matches the receipt's `kid`.
3. Imports it as a WebCrypto ECDSA P-384 verification key.
4. Runs `crypto.subtle.verify` with `SHA-384` over the JWS signing-input.

No network call to `geoclear.io`. No JavaScript loaded from `geoclear.io`. The math is entirely local.

### Verifying historical receipts

After the first key rotation, this repository's root will contain only the *current* JWKS. Receipts signed under a previous key will have a `kid` that no longer matches. To verify them, look in `history/`:

```javascript
const histJwksUrl = `https://raw.githubusercontent.com/geoclear-io/trust-root/main/history/jwks-2026-04-23.json`;
await verifyGeoClearReceipt(receipt, histJwksUrl);
```

The history is append-only. Old keys are never deleted; receipts issued under them remain verifiable indefinitely.

### Revocation

Before trusting a verified receipt, also check that the receipt's `kid` is NOT in [`revoked-kids.json`](revoked-kids.json). A receipt signed by a revoked kid is mathematically valid but should not be trusted operationally — typically because the corresponding signing key has been retired or compromised.

```javascript
const revoked = await fetch('https://raw.githubusercontent.com/geoclear-io/trust-root/main/revoked-kids.json').then(r => r.json());
if (revoked.revoked.find(r => r.kid === header.kid)) {
  throw new Error(`Kid ${header.kid} is revoked: ${revoked.revoked.find(r => r.kid === header.kid).reason}`);
}
```

## What this repository is NOT

- **NOT** a place to file bugs or feature requests for the GeoClear API. Use [https://geoclear.io](https://geoclear.io) for product support.
- **NOT** a source of receipts themselves. Receipts arrive in API responses; this repo only contains keys to verify them.
- **NOT** GeoClear's product code. This repo is the public-key publication only.
- **NOT** a write target for humans. Pushes are automated; see [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE). Public keys are inherently public; the verification protocol described here is free for anyone to implement.

## Related

- **GeoClear API:** [https://geoclear.io](https://geoclear.io)
- **Discovery manifest** (lists all publication channels): [https://geoclear.io/.well-known/location-trust.json](https://geoclear.io/.well-known/location-trust.json)
- **Verifier npm package:** [`@geoclear/verify-receipt`](https://www.npmjs.com/package/@geoclear/verify-receipt) (a thin wrapper around the WebCrypto verifier above)
- **Compliance / SOC 2 mapping:** the publication of public keys to a vendor-independent location is a control documented in GeoClear's SOC 2 program

---

*This repository is the public-key publication for the GeoClear trust root. For commercial inquiries, see [https://geoclear.io](https://geoclear.io).*
