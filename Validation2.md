# M-15 | Forger Entities Can Forge More Times Than Intended

**Author:** hailthelord
**Proof Link:** https://github.com/code-423n4/2024-07-traitforge-findings/issues/211 
**Severity:** Medium  
**Category:** Validation  
**Type:** Incorrect Validation Timing / Off-by-One  
**Protocol:** TraitForge  
**Submitted via:** Code4rena  

## Summary

The forging limit for "Forger" entities is enforced incorrectly. The `forgingCounts[forgerTokenId]` is incremented **after** the limit check in `forgeWithListed`, while the check itself happens in `listForForging`. This allows any forger entity to forge **one extra time** than its `forgePotential` permits (e.g., an entity with `forgePotential = 1` can forge twice instead of once).

## Vulnerability Details

According to the protocol documentation, an entity’s `forgePotential` (extracted from the 5th digit of its entropy) determines how many times it can forge per year:

- `forgePotential = 0` → infertile (cannot forge)
- `forgePotential = 1` → can forge **1 time**
- `forgePotential = n` → can forge **n times**

The `forgingCounts` mapping is used to track how many times an entity has forged.

### Problematic Code Flow

**In `listForForging` (check happens here):**

```solidity
require(
    forgePotential > 0 && forgingCounts[tokenId] <= forgePotential,
    'Entity has reached its forging limit'
);
```

**In `forgeWithListed` (increment happens here — too late):**

```solidity
forgingCounts[forgerTokenId]++;           // ← Increment after check
// ... forging logic ...
```

### Why It’s Broken

- The check `forgingCounts[tokenId] <= forgePotential` is performed when **listing**.
- The counter is only incremented when **forging** actually happens.
- This creates a window where a forger entity can list and forge one extra time.

**Example (forgePotential = 1):**

| Step                    | forgingCounts[123] | Check (`<= 1`) | Result     |
|-------------------------|--------------------|----------------|------------|
| 1st Listing             | 0                  | 0 <= 1         | Pass       |
| 1st Forge               | 1                  | -              | Success    |
| 2nd Listing             | 1                  | 1 <= 1         | Pass       |
| 2nd Forge               | 2                  | -              | Success    |
| 3rd Listing (should fail) | 2                | 2 <= 1         | **Fail**   |

→ Entity with `forgePotential = 1` successfully forged **2 times** instead of 1.

The same pattern allows an entity with `forgePotential = 9` to forge **10 times**.

Note: The `mergerTokenId` side is implemented correctly (increment happens **before** the check).

## Impact

- Forger entities can exceed their intended forging limit by 1.
- This breaks the core economic design of the forging system.
- Entities effectively get **+1 extra forge** for free (2–10 instead of 1–9).
- Medium severity because it gives unfair advantage and distorts the intended scarcity mechanics.

## Proof of Concept

Assume a Forger Entity:
- `tokenId = 123`
- `forgePotential = 1`
- Initial `forgingCounts[123] = 0`

**Execution:**

1. **First Listing** → Check: `0 <= 1` → Pass → Forge succeeds → `forgingCounts[123] = 1`
2. **Second Listing** → Check: `1 <= 1` → Pass → Forge succeeds → `forgingCounts[123] = 2`
3. **Third Listing** → Check: `2 <= 1` → Reverts

This proves a `forgePotential = 1` entity can forge **twice**.

## Recommendation

Change the condition in `listForForging` from `<=` to `<`:

```diff
require(
-   forgePotential > 0 && forgingCounts[tokenId] <= forgePotential,
+   forgePotential > 0 && forgingCounts[tokenId] < forgePotential,
    'Entity has reached its forging limit'
);
```

This ensures the counter is checked strictly before allowing an additional forge, matching the intended behavior (`forgePotential = n` allows exactly `n` forges).

---
