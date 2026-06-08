# H-04 | `mintToken()`, `mintWithBudget()`, and `forge()` Will Fail Due to Wrong Modifier in `EntropyGenerator.initializeAlphaIndices()`

**Author:** hailthelord
**Proof Link :** https://github.com/code-423n4/2024-07-traitforge-findings/issues/213
**Severity:** High  
**Category:** AccessControl  
**Type:** Wrong Access Control Modifier / DoS  
**Protocol:** TraitForge  
**Submitted via:** Code4rena  

## Summary

The `initializeAlphaIndices()` function in the `EntropyGenerator` contract is protected by the `onlyOwner` modifier instead of `onlyAllowedCaller`. Since this function is called internally by `TraitForgeNft._incrementGeneration()` (which is used by `mintToken()`, `mintWithBudget()`, and `forge()`), these core functions will revert when the generation limit is reached, causing a Denial of Service.

## Vulnerability Details

### Vulnerable Code

In `EntropyGenerator.sol`:

```solidity
function initializeAlphaIndices() public whenNotPaused onlyOwner {   // ← Wrong modifier
    // ...
}
```

This function is called from `TraitForgeNft._incrementGeneration()`:

```solidity
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;

    entropyGenerator.initializeAlphaIndices();   // ← Will revert because onlyOwner is used

    emit GenerationIncremented(currentGeneration);
}
```

### Root Cause

The `EntropyGenerator` contract has an `onlyAllowedCaller` modifier specifically designed to allow the `TraitForgeNft` contract to call certain privileged functions. However, `initializeAlphaIndices()` incorrectly uses `onlyOwner` instead.

As a result, when `TraitForgeNft` tries to call `initializeAlphaIndices()` after a generation is full, the transaction reverts because `TraitForgeNft` is not the owner of `EntropyGenerator`.

## Impact

This vulnerability causes a **Denial of Service** on three critical functions:

- `mintToken()`
- `mintWithBudget()`
- `forge()`

Once the current generation reaches `maxTokensPerGen`, any further calls to these functions will fail because `_incrementGeneration()` cannot successfully call `initializeAlphaIndices()`.

This breaks the core minting and forging mechanics of the entire TraitForge protocol.

## Proof of Concept

The vulnerability can be triggered as follows:

1. Deploy `TraitForgeNft` and `EntropyGenerator`, and set the allowed caller correctly.
2. Mint tokens until `generationMintCounts[currentGeneration] >= maxTokensPerGen`.
3. Attempt to call `mintToken()`, `mintWithBudget()`, or `forge()`.
4. The call to `_incrementGeneration()` → `entropyGenerator.initializeAlphaIndices()` will revert with an `Ownable: caller is not the owner` error (or equivalent).

No additional setup is needed beyond reaching the generation limit — the bug is triggered automatically on the next mint/forge after the limit.

## Recommendation

Change the modifier from `onlyOwner` to `onlyAllowedCaller`:

```diff
- function initializeAlphaIndices() public whenNotPaused onlyOwner {
+ function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
```

This ensures that the `TraitForgeNft` contract (which should be set as the allowed caller) can successfully initialize the alpha indices when incrementing generations.

---
