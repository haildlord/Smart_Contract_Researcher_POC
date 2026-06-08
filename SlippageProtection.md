# M-10 | Missing Slippage Protection in `LendingPool.deposit()`

**Author:** hailthelord 

**Protocol:** RAAC

**Proof Link:** https://codehawks.cyfrin.io/c/2025-02-raac/s/5905

**Severity:** Medium  

**Category:** Slippage  

**Type:** Missing Slippage Protection / Front-Running  

**Protocol:** RAAC Protocol  

**Submitted via:** Cyfrin Audit (2025-02-raac)  

## Summary

The `deposit()` function in `LendingPool` lacks slippage protection. A malicious actor can front-run a user's deposit by borrowing assets, which increases the `liquidityIndex` and causes the user to receive fewer `rToken` than expected. This also leads to reduced rewards (`raacToken`) in the StabilityPool.

## Description

When a user calls `LendingPool.deposit(amount)`, the amount of `rToken` they receive is calculated based on the current `liquidityIndex`:

```solidity
rTokenAmount = amount * RAY / liquidityIndex
```

Because `liquidityIndex` is updated based on utilization and borrowing activity, it can be manipulated within the same block by an attacker who borrows a large amount of the underlying asset (crvUSD) before the deposit transaction is mined.

### Attack Scenario

1. **Initial State**: `liquidityIndex = 1e27`
2. User intends to deposit `100` crvUSD and expects:
   ```solidity
   rTokenAmount = (100 * 1e27) / 1e27 = 100 rToken
   ```
3. Attacker front-runs by borrowing a significant amount of crvUSD. This increases utilization → increases `currentLiquidityRate` → increases `liquidityIndex` (e.g., to `1.1e27`).
4. User's deposit is executed with the new higher index:
   ```solidity
   rTokenAmount = (100 * 1e27) / 1.1e27 ≈ 90.91 rToken
   ```
5. User receives **~9% fewer rToken** than expected, resulting in permanently lower rewards in the StabilityPool.

The user has no way to specify a minimum acceptable `rToken` amount, making the deposit vulnerable to this form of slippage / sandwich attack.

## Impact

- Users receive **fewer rToken** than anticipated for their deposit.
- Reduced `rToken` balance leads to **lower deToken and raacToken rewards** in the StabilityPool.
- Degrades user experience and trust in the protocol.
- Can be especially damaging during periods of high volatility or when large deposits are being made.

## Proof of Concept

**Affected Code:**  
`LendingPool.sol` lines 225–236 (deposit function)

**Attack Flow:**
- No minimum output amount is checked in `deposit()`.
- `liquidityIndex` is updated in `ReserveLibrary.updateReserveState()` before minting `rToken`.
- An attacker can increase `liquidityIndex` in the same block via a borrow transaction.
- The depositor has no protection against receiving significantly less `rToken` than expected at the time they submitted the transaction.

This is a classic **slippage / MEV** vulnerability commonly seen in AMMs and lending protocols that lack `minAmountOut` style checks.

## Recommendation

Add a slippage protection parameter (`minRTokenAmount` or `expected`) to the `deposit()` function and revert if the actual minted amount is below the user's expectation.

```diff
- function deposit(uint256 amount) 
+ function deposit(uint256 amount, uint256 minRTokenAmount) 
      external 
      nonReentrant 
      whenNotPaused 
      onlyValidAmount(amount) 
  {
      ReserveLibrary.updateReserveState(reserve, rateData);
      uint256 mintedAmount = ReserveLibrary.deposit(reserve, rateData, amount, msg.sender);

+     if (mintedAmount < minRTokenAmount) revert SlippageExceeded();

      _rebalanceLiquidity();
      emit Deposit(msg.sender, amount, mintedAmount);
  }
```

**Additional Suggestions:**
- Consider also exposing a `previewDeposit(uint256 amount)` view function so users can query the expected `rToken` amount before submitting the transaction.
- Document the new parameter clearly in the frontend and SDK.
- Apply similar slippage protection to `withdraw()` and other functions that depend on dynamic indices (e.g. `borrow`, `repay`).

