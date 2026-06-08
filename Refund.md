# M-13 | Refunds Sent to Incorrect Addresses in Certain Claim Paths

**Author:** hailthelord
**Proof Link :** https://github.com/code-423n4/2024-08-phi-findings/issues/11
**Severity:** Medium  
**Category:** Refund  
**Type:** Incorrect msg.sender Handling / Caller Assumption  
**Protocol:** Phi Protocol  
**Submitted via:** Code4rena  

## Summary

The `_processClaim` function in `PhiFactory` refunds excess ETH using `_msgSender()`. However, when claims are made through `batchClaim`, `claim`, or via the `Claimable` contract inside `PhiNFT1155`, `msg.sender` becomes the contract itself instead of the actual user. As a result, refunds are sent to `PhiFactory` or `PhiNFT1155` instead of the user, and in some cases the funds become permanently stuck.

## Vulnerability Details

### Root Cause

The refund logic in `_processClaim` relies on `msg.sender`:

```solidity
if ((etherValue_ - mintFee) > 0) {
    _msgSender().safeTransferETH(etherValue_ - mintFee);  // ← Wrong address in some call paths
}
```

This works correctly only when the user directly calls `signatureClaim` or `merkleClaim` on `PhiFactory`.

### Problematic Call Chains

**Case 1: `batchClaim` → `claim` → `signatureClaim`/`merkleClaim`**

```solidity
function batchClaim(...) external payable {
    for (...) {
        this.claim{ value: ethValue_[i] }(encodeDatas_[i]);  // ← msg.sender becomes PhiFactory
    }
}
```

Inside `claim`, it further delegates to `signatureClaim` / `merkleClaim`, so `_msgSender()` resolves to `PhiFactory` instead of the original user.

**Case 2: Claims through `Claimable` in `PhiNFT1155`**

When users call `claimFromFactory` (or similar) on the `PhiNFT1155` contract, the `Claimable` logic forwards the call to `PhiFactory`. Here `msg.sender` becomes the `PhiNFT1155` contract itself.

Result: Refunds are sent to `PhiNFT1155`, and because there is no mechanism for the owner to withdraw stuck ETH from it, the funds are effectively locked.

## Impact

- Users who overpay during claims (via `batchClaim` or through the 1155 contract) lose their excess ETH.
- Funds sent to `PhiNFT1155` become **permanently stuck** (no withdrawal path for owner or users).
- Breaks the intended refund logic that the protocol specifically asked auditors to review.
- Affects both `signatureClaim` and `merkleClaim` paths.

## Proof of Concept

The warden provided a test (`test_claim_1155_refund`) showing that when a user sends `mintFee + 0.1 ETH` through the `PhiNFT1155` contract’s claim mechanism, the 0.1 ETH refund ends up in the `PhiNFT1155` contract balance instead of being returned to the user.

## Recommendation

Pass the original user address explicitly through the call chain instead of relying on `msg.sender` for refunds.

Recommended approach:

1. Modify `batchClaim`, `claim`, `signatureClaim`, and `merkleClaim` to accept (or propagate) the original `minter` / `caller` address.
2. Change `_processClaim` (and related functions) to accept the **actual recipient address** for refunds as a parameter.
3. Update all internal call sites to forward the correct user address.

Example direction:

```diff
- function _processClaim(..., address minter_) private {
+ function _processClaim(..., address minter_, address refundRecipient_) private {
      ...
-     _msgSender().safeTransferETH(etherValue_ - mintFee);
+     refundRecipient_.safeTransferETH(etherValue_ - mintFee);
      ...
}
```

This ensures refunds always go to the real user regardless of how the claim was initiated.

---

**Status:** Open  
**Tags:** `#refund` `#msg-sender` `#batch-claim` `#medium-severity`
