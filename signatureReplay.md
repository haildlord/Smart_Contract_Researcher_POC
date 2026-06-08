# H-03 | Signature Replay in `createArt` Allows Impersonation of Artist and Theft of Royalties

**Author:** Hailthelord
**Protocol: ** Phi
**Proof Link: ** https://github.com/code-423n4/2024-08-phi-findings/issues/70
**Severity:** High  
**Category:** Signature  
**Type:** Signature Replay / Missing Domain Separation  
**Protocol:** Phi Protocol  
**Submitted via:** Code4rena  

## Summary

The `createArt()` function in `PhiFactory` accepts a signature that only covers `signCreateData` (art ID, expiry, cred data). It does **not** bind the signature to the `CreateConfig` struct (especially the `receiver` — royalty recipient — and `artCreator`). Combined with the fact that `createERC1155Internal` succeeds even when the contract already exists, this allows anyone to front-run a legitimate `createArt` transaction, reuse the signature with a malicious config, and steal all future royalties.

## Vulnerability Details

### Root Causes

1. **Signature does not include `CreateConfig`**  
   The signed message only contains `expiresIn + artIdURL + credData`. Critical economic parameters such as:
   - `receiver` (royalties recipient)
   - `artCreator`
   - `royaltyBPS` related settings
   are completely unsigned and can be supplied arbitrarily by an attacker.

2. **No binding to specific submitter**  
   There is no check that `msg.sender` matches the intended creator.

3. **Idempotent creation allows double success**  
   `createERC1155Internal` does not revert if the PhiNFT1155 contract for that `artId` already exists. Both the attacker’s front-run transaction and the original transaction succeed.

### Attack Flow

1. Legitimate user prepares a valid signature + `CreateConfig` and broadcasts `createArt(...)`.
2. Attacker sees the transaction in the mempool and front-runs it with the **same signature** but their own `CreateConfig` where `receiver = attacker`.
3. Attacker’s transaction succeeds → `ArtData.receiver` is set to the attacker.
4. Original user’s transaction also succeeds (because creation is idempotent) → events are emitted normally.
5. All future royalty payments and rewards routed through `PhiRewards` go to the attacker instead of the legitimate artist/recipient.
6. Additionally, the attacker (now recorded as `artCreator`) can call privileged functions like `updateRoyalties()` and `updateArtSettings()`.

## Impact

- **Direct theft of royalties** — Attacker receives all future royalty payments intended for the artist.
- **Privilege escalation** — Attacker gains the ability to call `PhiNFT1155::updateRoyalties()` and `PhiFactory::updateArtSettings()`, allowing them to change royalty rates, max supply, mint fees, etc.
- **Loss of funds for legitimate artists and creators**.
- High severity because the attack is cheap to execute (just front-running gas) and has permanent economic consequences.

## Proof of Concept

The warden provided a Foundry test (`testKuprum_ImpersonateArtist`) that demonstrates the full attack:

- Owner prepares a legitimate signature + config.
- `user1` front-runs using the same signature but with `receiver = user1`.
- After both transactions, `phiFactory.artData(artId).receiver == user1`.

The test confirms that the attacker becomes the royalties recipient while the original transaction still succeeds without any visible error to the legitimate user.

## Recommendation

Implement the following mitigations:

1. **Include `CreateConfig` in the signed message** (or at minimum the critical fields: `receiver`, `artCreator`, `maxSupply`, etc.).

2. **Bind the signature to a specific submitter** (include `msg.sender` or a dedicated `submitter` field in the signed data).

3. **Make `createERC1155Internal` revert if the contract already exists** for that `artId`, so only the first valid transaction succeeds.

Example direction for the fix in `createArt()`:

```solidity
function createArt(
    bytes memory signCreateData,
    bytes memory signature,
    CreateConfig memory config
) external payable {
    // 1. Recover signer from signature
    // 2. Verify that signCreateData now also commits to keccak256(abi.encode(config))
    // 3. Optionally verify msg.sender == intended submitter
    // 4. Proceed only if checks pass
}
```

Additionally, consider using EIP-712 typed data hashing with proper domain separation for better security and readability.

---

**Status:** Open  
**Tags:** `#signature` `#replay` `#front-running` `#royalties` `#high-severity`
