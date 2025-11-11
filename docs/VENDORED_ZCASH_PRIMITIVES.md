# Vendored zcash_primitives for FROST Integration

**Date**: 2025-11-10
**Purpose**: Enable FROST transaction authorization by exposing necessary constructors

## Setup

We've vendored `zcash_primitives` v0.26.1 and `zcash_proofs` v0.26.1 to enable modifications that support FROST threshold signatures.

### Location

```
/Users/boy/projects/ycash-stark-bridge/vendored/librustzcash/
├── zcash_primitives/
└── zcash_proofs/
```

### Patch Configuration

In the workspace root `Cargo.toml`:

```toml
[patch.crates-io]
zcash_primitives = { path = "vendored/librustzcash/zcash_primitives" }
zcash_proofs = { path = "vendored/librustzcash/zcash_proofs" }
```

This transparently replaces all `zcash_primitives = "0.26"` dependencies with our vendored version.

## Why Vendoring Was Necessary

From [ycash-client/src/frost/transaction.rs:589-613](../ycash-client/src/frost/transaction.rs#L589-L613):

> "The zcash_primitives authorization flow:
>
> TransactionData<Unauthorized> has:
>   - SpendDescription<Unauthorized> (no spend_auth_sig)
>   - OutputDescription<Unauthorized> (same as Authorized)
>
> TransactionData<Authorized> has:
>   - SpendDescription<Authorized> (with spend_auth_sig: Signature)
>   - OutputDescription<Authorized>
>   - binding_sig: redjubjub::Signature
>
> However, zcash_primitives doesn't expose constructors for these types directly.
> The Builder pattern expects to create them internally with spending keys.
>
> For FROST integration, we need to either:
> A) Use unsafe transmute (not recommended)
> B) Fork zcash_primitives to expose constructors (maintenance burden) ← **WE'RE DOING THIS**
> C) Build transactions using a lower-level approach (complex)
> D) Wait for upstream zcash_primitives to support threshold signing"

## What Needs to Be Modified

### 1. SpendDescription<Authorized> Constructor

**File**: `vendored/librustzcash/zcash_primitives/src/transaction/components/sapling.rs`

We need to expose a public constructor that allows creating `SpendDescription<Authorized>` with:
- cv (value commitment)
- anchor (merkle root)
- nullifier
- rk (randomized verification key)
- zkproof (spend proof)
- **spend_auth_sig** (RedJubjub signature from FROST)

Currently, this can only be created internally by the Builder with a spending key.

### 2. OutputDescription Constructor (if needed)

**File**: Same as above

May need similar exposure for `OutputDescription<Authorized>`, though outputs don't require spend authorization signatures.

### 3. SaplingBundle Constructor

**File**: `vendored/librustzcash/zcash_primitives/src/transaction/components/sapling/bundle.rs`

Need ability to create `Bundle<Authorized>` with:
- Vector of `SpendDescription<Authorized>`
- Vector of `OutputDescription<Authorized>`
- value_balance
- **binding_sig** (binding signature over value balance)

## Implementation Approach

### Step 1: Examine Current Structure

Look at how the types are currently defined and what fields are private/public.

### Step 2: Add Public Constructors

Add `pub fn new_authorized(...)` methods that bypass the Builder pattern.

Example (conceptual):

```rust
impl SpendDescription<Authorized> {
    /// Create an authorized spend description with external signatures
    ///
    /// # Safety
    /// This bypasses normal authorization checks. The caller MUST ensure:
    /// - The spend_auth_sig is valid for this spend's sighash
    /// - The zkproof is valid
    /// - The rk matches the authorization
    pub fn new_with_external_sig(
        cv: ValueCommitment,
        anchor: bls12_381::Scalar,
        nullifier: Nullifier,
        rk: redjubjub::VerificationKey<SpendAuth>,
        zkproof: GrothProofBytes,
        spend_auth_sig: redjubjub::Signature<SpendAuth>,
    ) -> Self {
        // Implementation...
    }
}
```

### Step 3: Update Our Code

Modify `ycash-client/src/frost/transaction.rs::authorize_transaction_with_frost()` to use these new constructors.

## Reverting the Vendor

To remove the vendor and go back to crates.io versions:

1. Remove the `[patch.crates-io]` lines from workspace `Cargo.toml`
2. Delete `vendored/librustzcash/`
3. Run `cargo clean && cargo build`

This is why we documented this clearly - it's temporary until upstream supports threshold signing.

## Benefits of This Approach

1. **Clean**: Uses Cargo's patch system, easy to remove
2. **Isolated**: Changes are only in vendored code
3. **Testable**: Can verify our changes work correctly
4. **Temporary**: Clear path to remove once upstream supports FROST
5. **Compatible**: Produces identical transaction format to ywallet

## Next Steps

1. ✅ Vendor zcash_primitives v0.26.1
2. ✅ Set up Cargo patch configuration
3. ⏳ Examine Sapling module structure
4. ⏳ Add public constructors for authorized descriptions
5. ⏳ Update `authorize_transaction_with_frost()` to use new constructors
6. ⏳ Test transaction serialization matches ywallet format
7. ⏳ Broadcast test transaction to verify network acceptance

---

**Status**: Vendor setup complete, ready for modifications
**Blocking**: None - can proceed with examining the vendored code
**Estimated Time**: 2-4 hours to implement constructors and integrate
