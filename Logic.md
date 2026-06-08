# Incorrect Delegation Struct Assignment Locks Delegators Out of Rewards and Key Functions

## Metadata

- **Number:** #540
- **Severity:** High
- **Status:** Duplicate
- **Created by:** HailTheLord
- **Created at:** June 30, 2025 at 1:14 AM
- **Last updated:** August 22, 2025 at 1:14 AM

## Summary
The `delegateStake` function incorrectly assigns values to the **Delegation** struct, swapping the `validatorAddress` and `msg.sender` (**delegator**). This prevents **delegators** from **claiming rewards** or **unstaking**, **misdirects rewards to validators**, and **risks incorrect slashing**. This results in a denial of service (**DoS)** and **potential financial loss for delegators**.

## Finding Description
The bug is in this line of `delegateStake`:

```solidity
delegations[validatorAddress] = Delegation(blsPubkeyHash, msg.sender, validatorAddress, validatorVersion, nonce + 1);
```
The `Delegation` struct expects:

```solidity
struct Delegation {
    bytes32 blsPubkeyHash;
    address validatorAddress; // Should be validatorAddress
    address delegator;        // Should be msg.sender
    uint8 validatorVersion;
    uint64 nonce;
}
```
This flips the validatorAddress and delegator fields, misidentifying who staked.

## Impact Explanation

A critical flaw in the staking system breaks core functionality—`delegators` cannot **claim rewards** or **unstake** due to faulty **recipient verification logic**.

**Root Cause :**

In both `claimStakeRewards` and `unstake`, the contract checks if `msg.sender` matches the `validator` or the `delegation.delegator` (via **_getRecipient**). 
However, `delegation.delegator` is incorrectly set to the **validatorAddress**, not the actual delegator.

**Theoretical Implications:**

> Delegators Locked Out: 

Only the **validator** is recognized as the recipient, blocking `delegators` from accessing rewards or their staked funds.

> Reward Misallocation: 

All rewards are misdirected to validators, violating staking economics and user expectations.

> DoS on Delegators: 

Delegators effectively **lose control over their funds**, creating a **denial-of-service** on core staking actions.

**Severity Justification:**

Affects all delegations and staking flows.

Leads to direct financial loss and operational lockout for delegators.

Contradicts the intended staking model and violates user trust.

This is a **high severity** issue due to its systemic scope, financial impact, and violation of protocol guarantees.


## Likelihood Explanation

1. 100% Trigger Rate: Every call to delegateStake writes the wrong Delegation struct. No edge case needed—it’s broken by default.  
 

2. Result: Every delegator is affected, every time.

## Proof of Concept

Add this to **ConsensusRegistryTest.t.sol** below `setUp()` 

requires `import {RewardInfo} from "src/interfaces/IStakeManager.sol";`


```solidity
function test_lords_delegateStake_and_claimStakeRewards() public {
    // Setup
    uint256 validatorPrivateKey = 100;
    address validator100 = vm.addr(validatorPrivateKey);
    address delegator = vm.addr(17);
    bytes memory validator_blsPubKey = _createRandomBlsPubkey(100);

    // Validator starts with no NFT
    assertEq(consensusRegistry.balanceOf(validator100), 0);

    // Owner mints NFT for validator
    vm.prank(crOwner);
    consensusRegistry.mint(validator100);
    assertEq(consensusRegistry.balanceOf(validator100), 1);

    // Validator signs offchain data for delegator
    bytes32 digest = consensusRegistry.delegationDigest(validator_blsPubKey, validator100, delegator);
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(validatorPrivateKey, digest);
    bytes memory validatorSig = abi.encodePacked(r, s, v);

    // Delegator gets funds and stakes
    vm.deal(delegator, stakeAmount_);
    vm.prank(delegator);
    consensusRegistry.delegateStake{value: stakeAmount_}(validator_blsPubKey, validator100, validatorSig);
    assertEq(consensusRegistry.getBalance(validator100), stakeAmount_);

    // System distributes rewards
    RewardInfo[] memory rewards = new RewardInfo[](1);
    rewards[0] = RewardInfo({validatorAddress: validator100, consensusHeaderCount: 10});
    vm.prank(SYSTEM_ADDRESS);
    consensusRegistry.applyIncentives(rewards);
    assertGt(consensusRegistry.getBalance(validator100), stakeAmount_); // Rewards added

    // Delegator tries to claim rewards—FAILS
    vm.expectRevert();
    vm.prank(delegator);
    consensusRegistry.claimStakeRewards(validator100);

    // Validator claims rewards—SUCCEEDS
    vm.prank(validator100);
    consensusRegistry.claimStakeRewards(validator100);
    assertEq(consensusRegistry.getBalance(validator100), stakeAmount_); // Rewards taken
}
```

## Recommendation
Correct the Delegation struct assignment in the delegateStake function:
```diff
- delegations[validatorAddress] = Delegation(blsPubkeyHash, msg.sender, validatorAddress, validatorVersion, nonce + 1);
+ delegations[validatorAddress] = Delegation(blsPubkeyHash, validatorAddress, msg.sender, validatorVersion, nonce + 1);
```
