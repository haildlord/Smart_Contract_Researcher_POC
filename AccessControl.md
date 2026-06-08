# H-01 | Anyone Can Add Themselves as a Validator, Enabling Malicious Influence

**Author:** hail_the_lord
**Protocol:** virtuals protocol
**Proof Link:** https://code4rena.com/audits/2025-04-virtuals-protocol/submissions?uid=sg8wuLa4oxV
**Severity:** High  
**Category:** Access Control / Authorization  
**Type:** Missing Access Control (CWE-862)  
**Discovered:** 333 days ago  
**Chain:** Base  

## Summary

The `addValidator` function is public and lacks any access control, allowing any user to add themselves (or any address) as a validator for any `virtualId`. This enables malicious actors to instantly gain validator privileges, which can be used to disrupt governance at AgentDAO, manipulate reward distributions, or approve harmful contributions.

## Description

The `addValidator` function (and its internal helpers) contains no authentication, authorization, staking requirement, or approval mechanism. Anyone with a wallet on Base can call it directly and become a validator with immediate voting power and reward eligibility.

### Vulnerable Code

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return; // already a validator
    }
    _addValidator(virtualId, validator);
    _initValidatorScore(virtualId, validator);
}

function _addValidator(uint256 virtualId, address validator) internal {
    _validatorsMap[virtualId][validator] = true;
    _validators[virtualId].push(validator);
    emit NewValidator(virtualId, validator);
}

function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}
```

**Root Cause:** The function is declared `public` with zero modifiers or checks. The protocol implicitly trusts that only legitimate parties will call it.

## Impact

- **Governance Takeover** — Validators vote on contribution proposals at AgentDAO. A malicious validator can approve backdoored upgrades, malicious `ServiceNft` mints, or block legitimate proposals.
- **Reward Dilution & Theft** — Validators receive rewards via `AgentReward.distributeRewards`. An attacker can flood the validator set, redirecting or diluting rewards away from honest participants.
- **Financial Loss via Bonding Curve** — Maliciously approved contributions can drain staked VIRTUAL tokens. Because the protocol uses a bonding curve (`y = kx`), even a modest liquidity drain can trigger cascading price impact and multimillion-dollar losses for users.
- **Systemic Risk** — A single malicious validator approving a fake core service update can compromise the entire VIRTUAL ecosystem.

## Likelihood

**High** — The attack is trivial to execute:

- **Public & Permissionless** — No whitelist, stake, identity check, or governance vote is required.
- **Instant Privilege Escalation** — Validator status and scoring are granted immediately.
- **High Economic Incentive** — Control over proposals + reward claims makes this extremely attractive to attackers.
- **No Timelock or Vetting** — Privileges take effect in the same transaction.

## Proof of Concept

**Setup**  
`virtualId = 42` is an active AgentDAO managing contributions.

**Attack Steps**
1. Attacker calls `addValidator(42, attackerAddress)`.
2. Attacker is instantly registered as a validator for `virtualId 42`.
3. Attacker can now:
   - Vote on AgentDAO proposals (including `ServiceNft` mints and core upgrades)
   - Claim rewards from `AgentReward`

**Result**  
Attacker proposes + approves a fraudulent `ServiceNft` mint that updates the core service ID to a malicious contract, enabling theft of user funds.

## Recommendation

**Immediate Fix** — Add proper access control:

```diff
- function addValidator(uint256 virtualId, address validator) public {
+ function addValidator(uint256 virtualId, address validator) public onlyAdmin {
      if (isValidator(virtualId, validator)) {
          return;
      }
      _addValidator(virtualId, validator);
      _initValidatorScore(virtualId, validator);
  }
```
