# Proposal: Post-Quantum Cryptography Support for AgentCard Signing

**Author**: Abdel Fane (OpenA2A, @abdelsfane)
**Date**: 2026-02-17
**Status**: Draft
**Target**: A2A Protocol v1.x

## Problem Statement

The A2A protocol's AgentCard signing mechanism (Section 8.4) currently supports classical JWS algorithms such as ES256 and RS256. These algorithms are vulnerable to attacks by cryptographically relevant quantum computers (CRQCs). While large-scale CRQCs do not yet exist, the threat is significant: a quantum computer running Shor's algorithm could derive private signing keys from public keys, enabling an attacker to forge new AgentCard signatures that appear authentic. Any entity whose public key is exposed today — which includes every AgentCard publisher — would be vulnerable to impersonation once CRQCs exist.

NIST finalized post-quantum cryptography standards in August 2024 (FIPS 203, 204, 205) and released draft transition guidance in November 2024 (NIST IR 8547, Initial Public Draft), recommending that organizations begin migration planning. The A2A protocol should provide a clear path for implementations to adopt quantum-resistant signing without breaking backward compatibility.

### Specific Gaps

1. **No PQC algorithm identifiers**: The `alg` field in the JWS protected header has no defined values for post-quantum algorithms. Implementations that want to use PQC have no interoperable way to identify the algorithm.

2. **No hybrid signature support**: NIST IR 8547 discusses hybrid approaches as a strategy for the PQC transition period. The current `AgentCardSignature` structure supports multiple signatures in the `signatures` array but provides no guidance on hybrid pairing or verification semantics.

3. **No JWKS extension for PQC keys**: The `jku` field references a JWKS endpoint, but JWK does not yet have registered key types for ML-DSA. Implementations need guidance on representing PQC keys.

4. **Key size impact undocumented**: ML-DSA public keys and signatures are significantly larger than their classical counterparts. The specification should document expected sizes so implementations can plan for bandwidth and storage.

## Proposed Changes

### 1. PQC Algorithm Identifiers

Add the following algorithm identifiers for use in the JWS `alg` protected header parameter:

| `alg` Value | Algorithm | NIST Standard | Security Level | Signature Size |
|-------------|-----------|---------------|----------------|----------------|
| `ML-DSA-44` | ML-DSA (Dilithium2) | FIPS 204 | 2 | ~2,420 bytes |
| `ML-DSA-65` | ML-DSA (Dilithium3) | FIPS 204 | 3 | ~3,309 bytes |
| `ML-DSA-87` | ML-DSA (Dilithium5) | FIPS 204 | 5 | ~4,627 bytes |

These identifiers follow the naming convention established by NIST FIPS 204 and align with the IETF draft for ML-DSA in JOSE (draft-ietf-cose-dilithium).

### 2. Hybrid Signature Scheme

Define a convention for hybrid signatures that pair a classical algorithm with a PQC algorithm. A hybrid signature uses two entries in the `signatures` array, linked by a shared `hybridGroup` identifier in the unprotected header.

**`hybridGroup` format:** The value MUST be a non-empty string that uniquely identifies a hybrid pair within the AgentCard. Implementations SHOULD use a UUID v4 (e.g., `"f47ac10b-58cc-4372-a567-0e02b2c3d479"`). The `hybridRole` field MUST be one of `"classical"` or `"pqc"`. Each `hybridGroup` MUST contain exactly two signatures: one classical and one PQC.

**Hybrid signature pair:**

```json
{
  "signatures": [
    {
      "protected": "<base64url: {\"alg\":\"ES256\",\"typ\":\"JOSE\",\"kid\":\"key-classical-1\"}>",
      "signature": "<base64url-encoded ES256 signature>",
      "header": {
        "hybridGroup": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "hybridRole": "classical"
      }
    },
    {
      "protected": "<base64url: {\"alg\":\"ML-DSA-65\",\"typ\":\"JOSE\",\"kid\":\"key-pqc-1\"}>",
      "signature": "<base64url-encoded ML-DSA-65 signature>",
      "header": {
        "hybridGroup": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
        "hybridRole": "pqc"
      }
    }
  ]
}
```

**Hybrid verification rule**: When a client encounters signatures with matching `hybridGroup` values, it MUST verify ALL signatures in the group. The hybrid group is valid only if every signature in the group verifies successfully.

**Backward compatibility**: Clients that do not understand `hybridGroup` treat each signature independently and will verify the classical signature normally. This means hybrid AgentCards degrade gracefully to classical-only verification on older clients.

**Signature ordering**: The order of entries in the `signatures` array is not significant. Implementations MUST NOT assume a fixed ordering (e.g., classical first). Hybrid pairs are identified by matching `hybridGroup` values, not by position.

### 3. PQC Key Representation in JWKS

The IETF is developing JWK key types for ML-DSA (see draft-ietf-cose-dilithium). Until the registration is finalized, implementations SHOULD use the following provisional representation:

```json
{
  "kty": "ML-DSA",
  "alg": "ML-DSA-65",
  "pub": "<base64url-encoded public key>",
  "kid": "key-pqc-1",
  "use": "sig"
}
```

Where:
- `kty`: `"ML-DSA"` — provisional key type, aligned with the direction of current IETF drafts. This value MAY change when the IETF finalizes JWK registration for ML-DSA.
- `alg`: The ML-DSA parameter set (`ML-DSA-44`, `ML-DSA-65`, or `ML-DSA-87`)
- `pub`: The raw public key bytes, base64url-encoded

**Migration note:** When IETF finalizes the JWK registration for ML-DSA, implementations MUST migrate to the registered key type and field names. Implementations SHOULD design their key parsing to tolerate changes in field names.

### 4. Specification Text Changes

#### Section 8.4.2 — Signature Format

Add to the "JWS Protected Header Parameters" section:

> **Post-Quantum Algorithm Support:**
>
> The `alg` parameter MAY specify a post-quantum algorithm. Implementations SHOULD support the following PQC algorithms for quantum-resistant signing:
>
> - `ML-DSA-44`: FIPS 204, NIST Security Level 2 (minimum for PQC)
> - `ML-DSA-65`: FIPS 204, NIST Security Level 3 (recommended)
> - `ML-DSA-87`: FIPS 204, NIST Security Level 5 (high security)
>
> For hybrid signing, implementations SHOULD generate two `AgentCardSignature` entries: one using a classical algorithm (e.g., `ES256`) and one using a PQC algorithm (e.g., `ML-DSA-65`). The entries SHOULD be linked using a `hybridGroup` identifier in the unprotected `header` field.

#### Section 8.4.3 — Signature Verification

Add:

> **Hybrid Signature Verification:**
>
> When the unprotected `header` contains a `hybridGroup` field, the client MUST verify all signatures sharing the same `hybridGroup` value. The hybrid group is considered valid only if every signature in the group passes verification.
>
> Clients that do not support hybrid verification MAY verify signatures individually. In this case, at least one classical signature MUST verify for the AgentCard to be considered authentic.

#### AgentCard Proto — Documentation Update

Add documentation comment to the `signatures` field in `AgentCard`:

```protobuf
// JSON Web Signatures computed for this AgentCard.
// Supports classical algorithms (ES256, RS256) and post-quantum
// algorithms (ML-DSA-44, ML-DSA-65, ML-DSA-87) per NIST FIPS 204.
// Hybrid signatures pair classical and PQC entries using a
// "hybridGroup" identifier in the unprotected header.
repeated AgentCardSignature signatures = 17;
```

### 5. Security Considerations

Add to Section 8.4.3 — Security Considerations:

> **Post-Quantum Considerations:**
>
> - Implementations SHOULD generate hybrid signatures (classical + PQC) during the transition period to maintain security under both classical and quantum threat models.
> - ML-DSA implementations MUST use a cryptographically secure random number generator for key generation.
> - PQC public keys and signatures are significantly larger than classical equivalents. Implementations should account for increased payload sizes:
>   - ML-DSA-65 public key: ~1,952 bytes (vs. 33 bytes for ES256)
>   - ML-DSA-65 signature: ~3,309 bytes (vs. 64 bytes for ES256)
> - Organizations subject to compliance requirements (e.g., CNSA 2.0 Suite) may require PQC adoption by specific dates. The A2A protocol's algorithm agility enables compliance without protocol changes.
> - The `hybridGroup` and `hybridRole` values reside in the JWS unprotected header and are NOT integrity-protected by the signature. An active attacker could strip these fields to prevent hybrid-aware clients from recognizing the pairing. Deployments that require hybrid verification SHOULD enforce this via policy (e.g., requiring both classical and PQC signatures) rather than relying solely on the presence of unprotected header metadata.

## Backward Compatibility

All proposed changes are additive:

- **New `alg` values**: Clients that do not recognize PQC algorithm identifiers will skip those signatures and verify classical signatures in the `signatures` array, as per existing behavior (Section 8.4.3: "Clients SHOULD verify at least one signature").
- **`hybridGroup` in unprotected header**: The `header` field is already defined as optional in `AgentCardSignature`. Clients that do not understand `hybridGroup` ignore it and verify signatures independently.
- **PQC JWKS entries**: Clients that do not recognize the `ML-DSA` key type skip those entries when fetching keys from the `jku` endpoint.

No existing behavior is changed. No existing fields are modified. No migration is required for current implementations.

## Implementation Notes

### Library Support

PQC implementations are available in major languages. The following table lists libraries known to support ML-DSA as of February 2026. Implementers should verify library maturity and audit status before production use.

| Language | Library | ML-DSA Support | Notes |
|----------|---------|----------------|-------|
| C | [liboqs](https://github.com/open-quantum-safe/liboqs) (Open Quantum Safe) | ML-DSA-44/65/87 | Reference implementation; bindings available for other languages |
| Go | [cloudflare/circl](https://github.com/cloudflare/circl) | ML-DSA-44/65/87 | Production use at Cloudflare |
| Rust | [pqcrypto](https://github.com/rustpq/pqcrypto) | ML-DSA-44/65/87 | Wraps reference C implementation |
| Python | [oqs-python](https://github.com/open-quantum-safe/liboqs-python) (Open Quantum Safe) | ML-DSA-44/65/87 | Python bindings for liboqs |
| JavaScript | [liboqs](https://github.com/open-quantum-safe/liboqs) (via WebAssembly) | ML-DSA-65 | Compile liboqs to Wasm with Emscripten; ecosystem still maturing |
| Java | [Bouncy Castle](https://www.bouncycastle.org/) 1.77+ | ML-DSA-44/65/87 | Widely used in enterprise Java; finalized FIPS 204 support from 1.79 |

### Performance Characteristics

Key and signature sizes are defined by NIST FIPS 204. Timing values are approximate and vary by hardware and implementation; the figures below are representative of optimized implementations on modern x86-64 hardware (source: Open Quantum Safe benchmarks).

| Operation | ES256 | ML-DSA-65 | Impact |
|-----------|-------|-----------|--------|
| Public key size | 33 bytes | 1,952 bytes | +1,919 bytes in JWKS |
| Signature size | 64 bytes | 3,309 bytes | +3,245 bytes per signature |
| Key generation | sub-ms | sub-ms | Negligible (one-time) |
| Signing | sub-ms | low single-digit ms | Negligible (per card update) |
| Verification | sub-ms | sub-ms | Negligible (per card fetch) |

AgentCard signing occurs infrequently (card creation/update), so computational performance impact is minimal. The primary impact is increased payload size for the `signatures` array and JWKS endpoint.

## References

- [NIST FIPS 204: Module-Lattice-Based Digital Signature Standard (ML-DSA)](https://csrc.nist.gov/pubs/fips/204/final)
- [NIST IR 8547: Transition to Post-Quantum Cryptography Standards (November 2024)](https://csrc.nist.gov/pubs/ir/8547/ipd)
- [RFC 7515: JSON Web Signature (JWS)](https://tools.ietf.org/html/rfc7515)
- [RFC 8785: JSON Canonicalization Scheme (JCS)](https://tools.ietf.org/html/rfc8785)
- [IETF draft-ietf-cose-dilithium: ML-DSA for JOSE and COSE](https://datatracker.ietf.org/doc/draft-ietf-cose-dilithium/)
- [CNSA 2.0 Suite: Commercial National Security Algorithm Suite 2.0](https://media.defense.gov/2022/Sep/07/2003071834/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS_.PDF)
- [A2A Protocol Specification v1.0 RC, Section 8.4](https://a2a-protocol.org/specification)

