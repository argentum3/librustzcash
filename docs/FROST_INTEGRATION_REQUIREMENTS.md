# FROST Integration Requirements for ycash-stark-librustzcash

**Target Repository**: `https://github.com/argentum3/librustzcash`
**Date**: 2025-11-11
**Purpose**: Enable FROST threshold signature support for Zcash/Ycash Sapling transactions

## Executive Summary

This document outlines the changes needed to support FROST (Flexible Round-Optimized Schnorr Threshold) signatures in the `librustzcash` library. FROST enables distributed threshold signing where multiple parties can cooperatively sign transactions without any single party having access to the complete spending key.

## Background

### The Problem

The current `zcash_primitives` architecture uses a Builder pattern that expects to have full access to spending keys when creating authorized transactions. The authorization flow is:

```
TransactionData<Unauthorized> → [Builder with SpendingKey] → TransactionData<Authorized>
```

However, in FROST threshold signing:
- No single party has the complete spending key
- Signatures are generated cooperatively through a multi-party protocol
- The transaction template must be created first, then signed externally

This requires a different flow:

```
TransactionData<Unauthorized> → [External FROST Protocol] → Signatures → TransactionData<Authorized>
```

### Current Limitations

The `zcash_primitives` crate does **not** expose public constructors for:
1. `SpendDescription<Authorized>` - Cannot attach external spend authorization signatures
2. `OutputDescription<GrothProofBytes>` - Limited construction options
3. `Bundle<Authorized>` - Cannot create with external binding signatures

These types come from the `sapling-crypto` crate and use internal constructors that are not publicly accessible.

## Required Changes

### Overview

We need to add **public constructor functions** that allow creating fully authorized Sapling transaction components with externally-generated FROST signatures, while maintaining the security guarantees of the existing types.

### File Modifications

#### 1. `zcash_primitives/src/transaction/components/sapling.rs`

**Location**: Add after line ~497 (after `write_v5_bundle` function)

**Changes**: Add three public constructor functions

##### 1.1 Create Spend Description with FROST Signature

```rust
// ============================================================================
// FROST Threshold Signature Support
// ============================================================================
//
// The following functions expose constructors for creating authorized Sapling
// transaction components with external threshold signatures (FROST).
//
// IMPORTANT: These bypass normal authorization checks. Callers MUST ensure:
// - Signatures are valid for their respective sighashes
// - ZK proofs are valid
// - Randomized keys match the authorizations
//
// These are intended for threshold signature schemes like FROST where
// signatures are generated externally by multiple parties.

/// Create an authorized Sapling spend description with external signature
///
/// This allows constructing a `SpendDescription<Authorized>` with a pre-computed
/// FROST threshold signature, bypassing the normal Builder pattern.
///
/// # Parameters
/// - `cv`: Value commitment
/// - `anchor`: Merkle tree root (anchor)
/// - `nullifier`: Note nullifier
/// - `rk`: Randomized verification key
/// - `zkproof`: Groth16 ZK-SNARK proof
/// - `spend_auth_sig`: Spend authorization signature (from FROST)
///
/// # Safety
/// The caller MUST ensure the signature is valid for this spend's sighash.
pub fn create_spend_description_with_frost_sig(
    cv: ValueCommitment,
    anchor: jubjub::Base,
    nullifier: Nullifier,
    rk: redjubjub::VerificationKey<SpendAuth>,
    zkproof: GrothProofBytes,
    spend_auth_sig: redjubjub::Signature<SpendAuth>,
) -> SpendDescription<Authorized> {
    SpendDescription::from_parts(cv, anchor, nullifier, rk, zkproof, spend_auth_sig)
}
```

##### 1.2 Create Output Description

```rust
/// Create an authorized Sapling output description
///
/// This allows constructing an `OutputDescription<GrothProofBytes>` with pre-computed
/// note encryption ciphertexts, useful for threshold signature workflows.
///
/// # Parameters
/// - `cv`: Value commitment
/// - `cmu`: Note commitment
/// - `ephemeral_key`: Ephemeral public key for note encryption
/// - `enc_ciphertext`: Encrypted note plaintext
/// - `out_ciphertext`: Encrypted outgoing plaintext
/// - `zkproof`: Groth16 ZK-SNARK proof
///
/// # Safety
/// The caller MUST ensure the proof and ciphertexts are valid.
pub fn create_output_description(
    cv: ValueCommitment,
    cmu: ExtractedNoteCommitment,
    ephemeral_key: EphemeralKeyBytes,
    enc_ciphertext: [u8; ENC_CIPHERTEXT_SIZE],
    out_ciphertext: [u8; OUT_CIPHERTEXT_SIZE],
    zkproof: GrothProofBytes,
) -> OutputDescription<GrothProofBytes> {
    OutputDescription::from_parts(cv, cmu, ephemeral_key, enc_ciphertext, out_ciphertext, zkproof)
}
```

##### 1.3 Create Bundle with FROST Signatures

```rust
/// Create an authorized Sapling bundle with external binding signature
///
/// This allows constructing a complete Sapling `Bundle<Authorized>` with a
/// pre-computed binding signature, as required for FROST threshold signatures.
///
/// # Parameters
/// - `shielded_spends`: Vector of authorized spend descriptions
/// - `shielded_outputs`: Vector of output descriptions
/// - `value_balance`: Net value balance (positive = value leaving shielded pool)
/// - `binding_sig`: Binding signature over value balance (from FROST)
///
/// # Safety
/// The caller MUST ensure:
/// - The binding signature is valid for the value balance
/// - All spend signatures are valid
/// - All proofs are valid
///
/// # Returns
/// `None` if spends and outputs are both empty, otherwise `Some(bundle)`.
pub fn create_bundle_with_frost_signatures(
    shielded_spends: Vec<SpendDescription<Authorized>>,
    shielded_outputs: Vec<OutputDescription<GrothProofBytes>>,
    value_balance: ZatBalance,
    binding_sig: redjubjub::Signature<redjubjub::binding_sig::Binding>,
) -> Option<Bundle<Authorized, ZatBalance>> {
    if shielded_spends.is_empty() && shielded_outputs.is_empty() {
        None
    } else {
        Some(Bundle::from_parts(
            shielded_spends,
            shielded_outputs,
            value_balance,
            Authorized { binding_sig },
        ))
    }
}
```

#### 2. Dependency: `sapling-crypto` Crate Modifications

**CRITICAL**: The `from_parts` methods called above are **not currently public** in the `sapling-crypto` crate.

You will need to either:

**Option A (Recommended)**: Submit a PR to `sapling-crypto` to add these methods:

```rust
// In sapling-crypto's bundle.rs

impl<P: BundleProof> SpendDescription<P, Authorized> {
    /// Create a spend description from its components
    ///
    /// # Safety
    /// Caller must ensure all components are valid and the signature is correct.
    pub fn from_parts(
        cv: ValueCommitment,
        anchor: jubjub::Base,
        nullifier: Nullifier,
        rk: redjubjub::VerificationKey<SpendAuth>,
        zkproof: P,
        spend_auth_sig: redjubjub::Signature<SpendAuth>,
    ) -> Self {
        SpendDescription {
            cv,
            anchor,
            nullifier,
            rk,
            zkproof,
            spend_auth_sig,
        }
    }
}

impl<P: BundleProof> OutputDescription<P> {
    /// Create an output description from its components
    ///
    /// # Safety
    /// Caller must ensure all components are valid.
    pub fn from_parts(
        cv: ValueCommitment,
        cmu: ExtractedNoteCommitment,
        ephemeral_key: EphemeralKeyBytes,
        enc_ciphertext: [u8; ENC_CIPHERTEXT_SIZE],
        out_ciphertext: [u8; OUT_CIPHERTEXT_SIZE],
        zkproof: P,
    ) -> Self {
        OutputDescription {
            cv,
            cmu,
            ephemeral_key,
            enc_ciphertext,
            out_ciphertext,
            zkproof,
        }
    }
}

impl<P: BundleProof, V> Bundle<P, Authorized, V> {
    /// Create a bundle from its components with authorization
    ///
    /// # Safety
    /// Caller must ensure the binding signature is valid.
    pub fn from_parts(
        shielded_spends: Vec<SpendDescription<P, Authorized>>,
        shielded_outputs: Vec<OutputDescription<P>>,
        value_balance: V,
        authorization: Authorized,
    ) -> Self {
        Bundle {
            shielded_spends,
            shielded_outputs,
            value_balance,
            authorization,
        }
    }
}
```

**Option B (Temporary)**: Vendor `sapling-crypto` in your fork and add the methods yourself until upstream accepts them.

## Implementation Checklist

- [ ] **Step 1**: Decide on dependency strategy (upstream PR vs vendoring sapling-crypto)
- [ ] **Step 2**: Add `from_parts` methods to `sapling-crypto` crate
- [ ] **Step 3**: Add FROST constructor functions to `zcash_primitives/src/transaction/components/sapling.rs`
- [ ] **Step 4**: Add unit tests for the new constructors
- [ ] **Step 5**: Update `CHANGELOG.md` documenting the new FROST support
- [ ] **Step 6**: Tag release (suggest minor version bump: e.g., 0.26.1 → 0.27.0)

## Usage Example

Once implemented, FROST-based transaction authorization would work like this:

```rust
use zcash_primitives::transaction::components::sapling::{
    create_spend_description_with_frost_sig,
    create_output_description,
    create_bundle_with_frost_signatures,
};

// 1. Create spend descriptions with FROST signatures
let authorized_spends: Vec<SpendDescription<Authorized>> = frost_signatures
    .iter()
    .zip(proof_data.iter())
    .map(|(sig, proof)| {
        create_spend_description_with_frost_sig(
            proof.cv,
            proof.anchor,
            proof.nullifier,
            proof.rk,
            proof.zkproof,
            sig.clone(),
        )
    })
    .collect();

// 2. Create output descriptions
let outputs: Vec<OutputDescription<GrothProofBytes>> = output_proofs
    .iter()
    .map(|proof| {
        create_output_description(
            proof.cv,
            proof.cmu,
            proof.ephemeral_key,
            proof.enc_ciphertext,
            proof.out_ciphertext,
            proof.zkproof,
        )
    })
    .collect();

// 3. Create bundle with binding signature
let bundle = create_bundle_with_frost_signatures(
    authorized_spends,
    outputs,
    value_balance,
    binding_sig,
)?;
```

## Security Considerations

### Why These Constructors Are Safe

1. **Type Safety**: The authorization state is still enforced through the type system (`Authorized` vs `Unauthorized`)
2. **No Validation Bypass**: The Zcash network will still validate all signatures and proofs when the transaction is broadcast
3. **Explicit Safety Contracts**: Documentation clearly states caller responsibilities
4. **Network Consensus**: Invalid transactions will be rejected by consensus rules

### Security Checklist for Callers

When using these constructors, **you MUST ensure**:

- ✅ Spend authorization signatures are valid RedJubjub signatures over the correct sighash
- ✅ Binding signature is valid over the value balance commitment
- ✅ All Groth16 ZK-SNARK proofs are valid
- ✅ Nullifiers are correctly derived and not previously used
- ✅ Value commitments match the actual note values
- ✅ Randomized verification keys match the signatures

**Failure to ensure these properties will result in invalid transactions that will be rejected by the network.**

## Benefits

1. **Enables FROST**: Full support for threshold signing protocols
2. **Maintains Security**: No reduction in transaction security guarantees
3. **Clean API**: Simple, well-documented constructor functions
4. **Upstream Compatible**: Changes follow Zcash conventions and can be upstreamed
5. **Type Safe**: Leverages Rust's type system to prevent misuse

## Migration Path

These additions are **purely additive** and require no changes to existing code:

- ✅ **No breaking changes** to existing APIs
- ✅ **Opt-in functionality** - existing Builder pattern still works
- ✅ **Backward compatible** - all existing code continues to work
- ✅ **Can be upstreamed** - useful for the broader Zcash ecosystem

## Testing Requirements

### Unit Tests

Add tests in `zcash_primitives/src/transaction/components/sapling.rs`:

```rust
#[cfg(test)]
mod frost_tests {
    use super::*;

    #[test]
    fn test_create_spend_with_frost_sig() {
        // Test that constructor creates valid SpendDescription
    }

    #[test]
    fn test_create_output_description() {
        // Test that constructor creates valid OutputDescription
    }

    #[test]
    fn test_create_bundle_with_frost_sigs() {
        // Test that constructor creates valid Bundle
    }

    #[test]
    fn test_empty_bundle_returns_none() {
        // Verify empty spends and outputs returns None
    }
}
```

### Integration Tests

Ideally, add an integration test that:
1. Creates a transaction using FROST constructors
2. Serializes it
3. Verifies it matches the format from a traditional Builder-created transaction
4. (Optional) Broadcasts to testnet and verifies acceptance

## Timeline Estimate

- **sapling-crypto PR**: 1-2 days (assuming quick upstream review)
- **zcash_primitives changes**: 2-4 hours
- **Testing**: 4-8 hours
- **Documentation**: 2-4 hours
- **Total**: ~1 week with upstream coordination

## Questions?

If you have questions about this request or need clarification, please reach out to the ycash-stark-bridge project:

- **GitHub**: https://github.com/your-org/ycash-stark-bridge
- **Context**: This is being used to build a trustless Ycash-Starknet bridge using FROST threshold signatures

## References

- **FROST Paper**: https://eprint.iacr.org/2020/852
- **Zcash Sapling Spec**: https://zips.z.cash/protocol/protocol.pdf#saplingbalance
- **sapling-crypto Crate**: https://github.com/zcash/sapling-crypto
- **zcash_primitives Crate**: https://github.com/zcash/librustzcash/tree/main/zcash_primitives

---

**Thank you for considering these changes!** This will enable secure, decentralized threshold custody solutions for Zcash and Ycash.
