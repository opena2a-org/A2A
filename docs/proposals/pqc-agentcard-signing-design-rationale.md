# Design Rationale: Post-Quantum Cryptography Support for AgentCard Signing

**Companion to**: [PQC AgentCard Signing Proposal](./pqc-agentcard-signing.md)
**Date**: 2026-02-17
**Author**: Abdel Fane (@abdelsfane)

This document explains the design decisions behind the PQC AgentCard signing proposal. It follows the format of an Architecture Decision Record (ADR) to provide structured reasoning.

## Context

The A2A protocol supports digital signing of AgentCards using JWS (RFC 7515) with classical algorithms (ES256, RS256). Quantum computers capable of breaking these algorithms are projected within 10-15 years. An attacker with a sufficiently powerful quantum computer could derive private signing keys from public keys (via Shor's algorithm), enabling forgery of new AgentCard signatures that impersonate legitimate agents.

NIST published FIPS 204 (ML-DSA) in August 2024 and released draft IR 8547 (PQC transition guidance, Initial Public Draft) in November 2024. Government agencies and regulated industries are beginning to mandate PQC adoption timelines (e.g., CNSA 2.0 requires PQC by 2030 for national security systems).

The A2A protocol needs a way to support PQC algorithms without breaking existing implementations.

## Decision Drivers

* NIST FIPS 204 provides standardized, peer-reviewed PQC signature algorithms
* NIST IR 8547 discusses hybrid approaches as a transition strategy
* The existing `signatures` array in AgentCard already supports multiple signatures
* Backward compatibility with current implementations is essential
* The IETF is developing ML-DSA integration for JOSE (draft-ietf-cose-dilithium)

## Considered Options

* **Option 1**: Add PQC algorithm identifiers and hybrid signature convention using existing structures
* **Option 2**: Create a new `pqcSignatures` field separate from classical signatures
* **Option 3**: Wait for IETF to finalize ML-DSA JOSE registration before acting

## Decision Outcome

**Chosen option: Option 1** — Add PQC algorithm identifiers and hybrid signature convention using existing structures.

This option requires no protocol buffer schema changes. It uses the existing `signatures` array and `header` (unprotected header) field to carry PQC signatures and hybrid group metadata.

### Consequences

**Positive:**

* No breaking changes to the protobuf schema or JSON format
* Existing clients continue to work — they verify classical signatures and ignore PQC entries
* Aligns with NIST IR 8547 hybrid transition guidance
* Provides a PQC migration path before quantum threats materialize

**Negative:**

* ML-DSA signatures are larger (3,309 bytes for ML-DSA-65 vs 64 bytes for ES256), increasing AgentCard payload size
* PQC library ecosystem is still maturing — not all languages have production-ready implementations
* The provisional `ML-DSA` key type for JWKS may need to change when IETF finalizes JWK registration

**Neutral:**

* The `hybridGroup` convention in the unprotected header is a soft convention, not enforced by the schema

## Analysis of Options

### Option 1: PQC via Existing Structures

Use the existing `signatures` array and unprotected `header` for PQC entries and hybrid grouping.

**Pros:**

* Zero schema changes required
* Fully backward compatible
* Leverages existing JWS infrastructure

**Cons:**

* Hybrid grouping is a convention, not schema-enforced
* Clients must understand the convention to do hybrid verification

### Option 2: Separate `pqcSignatures` Field

Add a new `pqcSignatures` repeated field to the AgentCard message.

**Pros:**

* Clear separation between classical and PQC signatures
* Schema enforces the distinction

**Cons:**

* Requires protobuf schema change (new field)
* Duplicates the signature structure
* Clients that don't understand the field ignore it entirely (no graceful hybrid behavior)

### Option 3: Wait for IETF

Defer PQC support until IETF finalizes ML-DSA integration for JOSE/JWS.

**Pros:**

* Aligns perfectly with future IETF standards
* No risk of choosing wrong key type identifiers

**Cons:**

* IETF timeline is uncertain (drafts have been in progress since 2023)
* Leaves A2A without a PQC migration path during the transition period
* Defers a problem that becomes harder to address after ecosystem adoption

## Implementation

1. Document PQC algorithm identifiers (`ML-DSA-44`, `ML-DSA-65`, `ML-DSA-87`) in the specification
2. Document hybrid signature convention using `hybridGroup` and `hybridRole` in unprotected headers
3. Document PQC key representation in JWKS using provisional `ML-DSA` key type
4. Add security considerations for PQC key/signature sizes and implementation guidance
5. Provide reference implementation demonstrating hybrid signing and verification

## References

* [NIST FIPS 204: Module-Lattice-Based Digital Signature Standard](https://csrc.nist.gov/pubs/fips/204/final)
* [NIST IR 8547: Transition to Post-Quantum Cryptography Standards](https://csrc.nist.gov/pubs/ir/8547/ipd)
* [RFC 7515: JSON Web Signature](https://tools.ietf.org/html/rfc7515)
* [IETF draft-ietf-cose-dilithium: ML-DSA for JOSE and COSE](https://datatracker.ietf.org/doc/draft-ietf-cose-dilithium/)
* [A2A Specification Section 8.4: Agent Card Signing](https://a2a-protocol.org/specification)

