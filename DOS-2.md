# M-12 | Exposed `_removeCredIdPerAddress` & `_addCredIdPerAddress` Allows Anyone to Grief Current and Future Cred Holders

**Author:** hailthelord

**Protocol:** Phi

**Proof Link :** https://github.com/code-423n4/2024-08-phi-findings/issues/51

**Severity:** Medium  

**Category:** DoS  

**Type:** Exposed Internal Functions / Griefing  

**Protocol:** Phi Protocol  

**Submitted via:** Code4rena  

## Summary

The functions `_addCredIdPerAddress` and `_removeCredIdPerAddress` in `Cred.sol` are declared `public` instead of `internal`. This allows any external caller to arbitrarily manipulate a user's `_credIdsPerAddress` array. An attacker can spam fake `credId`s, bloat arrays to cause gas griefing, corrupt index mappings, and force legitimate `sellShareCred` / `buyShareCred` transactions to revert.

## Vulnerability Details

Two critical helper functions that manage per-user cred ownership arrays were mistakenly made `public`:

```solidity
function _addCredIdPerAddress(uint256 credId_, address sender_) public { ... }

function _removeCredIdPerAddress(uint256 credId_, address sender_) public { ... }
```

These functions directly modify:
- `_credIdsPerAddress[sender_]`
- `_credIdsPerAddressCredIdIndex[sender_][credId_]`
- `_credIdsPerAddressArrLength[sender_]`

Because they are `public`, anyone can call them with arbitrary `credId_` and `sender_` values.

### Attack Vectors

1. **Array Bloat / Gas Griefing**
   - Attacker repeatedly calls `_addCredIdPerAddress` with non-existent `credId` values for a target user.
   - The target user's `_credIdsPerAddress` array grows extremely large.
   - Future operations that iterate over this array (or copy it) can hit block gas limits, especially on cheap L2s like Base or Avalanche.

2. **Index Corruption & Forced Reverts**
   - Attacker can call `_removeCredIdPerAddress` followed by `_addCredIdPerAddress` in specific orders to desync the index mapping (`_credIdsPerAddressCredIdIndex`).
   - When a legitimate user later tries to sell shares, the check fails:
     ```solidity
     if (indexToRemove >= _credIdsPerAddress[sender_].length) revert IndexOutofBounds();
     if (credId_ != credIdToRemove) revert WrongCredId();
     ```
   - This can be done repeatedly to grief active shareholders.

3. **Disruption for New Holders**
   - New users who buy shares can have their `_credIdsPerAddress` array polluted before or immediately after their first purchase, causing subsequent operations to fail.

## Impact

- Legitimate users can be **prevented from selling** their cred shares.
- New shareholders can be **griefed immediately** upon acquiring shares.
- Protocol can suffer **gas griefing** on low-fee chains (Base, Avalanche, etc.).
- Overall degradation of user experience and trust in the cred/share system.
- No direct theft, but persistent denial-of-service against core user actions.

## Proof of Concept

An attacker can execute the following actions permissionlessly:

```solidity
// Spam fake credIds for a victim address
for (uint i = 0; i < 1000; i++) {
    cred._addCredIdPerAddress(i + 100000, victim);
}

// Or corrupt indexes to break legitimate removes
cred._removeCredIdPerAddress(realCredId, victim);
cred._addCredIdPerAddress(realCredId, victim); // re-add with wrong index state
```

Any subsequent call by the victim to `sellShareCred` (which internally uses these helpers) can be made to revert with `IndexOutofBounds` or `WrongCredId`.

## Recommendation

Change both functions from `public` to `internal` so they can only be called from within the `Cred` contract.

```diff
- function _addCredIdPerAddress(uint256 credId_, address sender_) public {
+ function _addCredIdPerAddress(uint256 credId_, address sender_) internal {

- function _removeCredIdPerAddress(uint256 credId_, address sender_) public {
+ function _removeCredIdPerAddress(uint256 credId_, address sender_) internal {
```

No other changes are required. These functions were clearly intended to be internal helpers.
