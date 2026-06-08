# H-05 | TraitForgeNft: Generations Without a Golden God Are Possible

**Author:** hailthelord
**Proof Link:** https://github.com/code-423n4/2024-07-traitforge-findings/issues/656
**Severity:** High  
**Category:** Logic  
**Type:** Missing Invariant Enforcement / Golden God Skip  
**Protocol:** TraitForge  
**Submitted via:** Code4rena  

## Summary

The "Golden God" (special entropy `999,999`) is supposed to exist once per generation. However, because forging increases `generationMintCounts` without advancing the entropy pointer in a way that guarantees the golden god position, it is possible to completely skip the golden god for a generation. This breaks a core promised feature of the protocol.

## Vulnerability Details

The golden god is placed at specific positions:
- `slotIndexSelectionPoint`
- `numberIndexSelectionPoint`

These positions are in the third batch of entropy. When users forge NFTs, `generationMintCounts[currentGeneration]` increases. Since there is a hard cap of 10,000 NFTs per generation, heavy forging can cause the generation to fill up **before** the entropy pointer reaches the golden god position.

As a result, a full generation can be minted without ever hitting the golden god.

## Impact

- Some generations may have **no golden god** at all.
- This directly contradicts the protocol's promise that every generation has exactly one golden god.
- High severity because it breaks a fundamental game/economic mechanic that users rely on.

## Proof of Concept (Full Code)

The warden provided a complete Foundry test that demonstrates a generation completing without minting the golden god.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {DevFund} from "../contracts/DevFund/DevFund.sol";
import {TraitForgeNft} from "../contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntityForging} from "../contracts/EntityForging/EntityForging.sol";
import {EntityTrading} from "../contracts/EntityTrading/EntityTrading.sol";
import {EntropyGenerator} from "../contracts/EntropyGenerator/EntropyGenerator.sol";
import {Airdrop} from "../contracts/Airdrop/Airdrop.sol";
import {NukeFund} from "../contracts/NukeFund/NukeFund.sol";

contract Test_POC is Test {
    TraitForgeNft public nft;
    EntropyGenerator public entropy;
    EntityForging public forging;
    EntityTrading public trading;
    DevFund public devFund;
    NukeFund public nukeFund;
    Airdrop public airdrop;
    address daoFund = address(420);

    address owner = address(1);
    address merger = address(16);

    function setUp() public {
        vm.startPrank(owner);

        nft = new TraitForgeNft();

        airdrop = new Airdrop();
        nft.setAirdropContract(address(airdrop));
        airdrop.transferOwnership(address(nft));
        
        entropy = new EntropyGenerator(address(nft));
        nft.setEntropyGenerator(address(entropy));
        entropy.writeEntropyBatch1();
        entropy.writeEntropyBatch2();
        entropy.writeEntropyBatch3();

        forging = new EntityForging(address(nft));
        nft.setEntityForgingContract(address(forging));

        trading = new EntityTrading(address(nft));

        devFund = new DevFund();

        nukeFund = new NukeFund(
            address(nft),
            address(airdrop),
            payable(address(devFund)),
            payable(daoFund)
        );

        nft.setNukeFundContract(payable(address(nukeFund)));
        trading.setNukeFundAddress(payable(address(nukeFund)));

        vm.stopPrank();

        vm.deal(merger, 50 ether);

        vm.label(address(merger), "merger");
        vm.label(address(nft), "nft");
        vm.label(address(trading), "trading");
        vm.label(address(entropy), "entropy");
        vm.label(address(devFund), "devFund");
        vm.label(address(forging), "forging");
    }  

    receive() external payable {}

    uint256[] public listedTokenIds;
    uint256[] public mergeTokenIds;

    function _mintGeneration(uint256 gen, bytes32[] memory proof) internal {
        while(nft.generationMintCounts(gen) < 10_000) {
            uint256 price = nft.calculateMintPrice();
            nft.mintToken{ value: price }(proof);
            
            uint256 tokenId = nft._tokenIds();
            
            uint256 entropy = nft.getTokenEntropy(tokenId);
            uint8 potential = uint8((entropy / 10) % 10);
            if (potential == 0) continue;

            if (nft.isForger(tokenId)) {
                nft.approve(address(forging), tokenId);
                forging.listForForging(tokenId, 0.01 ether);
                listedTokenIds.push(tokenId);
            } else if (listedTokenIds.length > mergeTokenIds.length) {
                nft.transferFrom(address(this), merger, tokenId);
                mergeTokenIds.push(tokenId);
            }
        }
    }

    function test_POC() public {
        vm.pauseGasMetering();
        // skip whitelist phase
        vm.warp(block.timestamp + 25 hours);
        bytes32[] memory proof;

        console.log("________ start ____________");
        for (uint256 j = 1; j <= 2; j++) {
            console.log("gen: ", j, nft.generationMintCounts(j));
        }
        console.log("----");
        console.log("current: ", entropy.currentSlotIndex(), entropy.currentNumberIndex());
        console.log("golden god: ", entropy.slotIndexSelectionPoint(), entropy.numberIndexSelectionPoint());
        console.log("____________________");

        _mintGeneration(1, proof);

        console.log("________ after minting 10,000 NTFs ____________");
        for (uint256 j = 1; j <= 2; j++) {
            console.log("gen: ", j, nft.generationMintCounts(j));
        }
        console.log("----");
        console.log("current: ", entropy.currentSlotIndex(), entropy.currentNumberIndex());
        console.log("golden god: ", entropy.slotIndexSelectionPoint(), entropy.numberIndexSelectionPoint());
        console.log("____________________");

        assertEq(nft.getGeneration(), 1);
        uint256 price = nft.calculateMintPrice();
        nft.mintToken{ value: price }(proof);
        assertEq(nft.getGeneration(), 2);

        console.log("________ after incrementing generation ____________");
        for (uint256 j = 1; j <= 2; j++) {
            console.log("gen: ", j, nft.generationMintCounts(j));
        }
        console.log("----");
        console.log("current: ", entropy.currentSlotIndex(), entropy.currentNumberIndex());
        console.log("golden god: ", entropy.slotIndexSelectionPoint(), entropy.numberIndexSelectionPoint());
        console.log("____________________");

        vm.startPrank(merger);
        for (uint256 i = 0; i < mergeTokenIds.length; i++) {
            nft.approve(address(forging), mergeTokenIds[i]);
            forging.forgeWithListed{value: 0.01 ether}(
                listedTokenIds[i],
                mergeTokenIds[i]
            );
        }
        vm.stopPrank();
        delete(listedTokenIds);
        delete(mergeTokenIds);

        console.log("________ after forging all possible gen 2 ____________");
        for (uint256 j = 1; j <= 2; j++) {
            console.log("gen: ", j, nft.generationMintCounts(j));
        }
        console.log("----");
        console.log("current: ", entropy.currentSlotIndex(), entropy.currentNumberIndex());
        console.log("golden god: ", entropy.slotIndexSelectionPoint(), entropy.numberIndexSelectionPoint());
        console.log("____________________");

        _mintGeneration(2, proof);
        
        console.log("________ after minting 10,000 NTFs ____________");
        for (uint256 j = 1; j <= 2; j++) {
            console.log("gen: ", j, nft.generationMintCounts(j));
        }
        console.log("----");
        console.log("current: ", entropy.currentSlotIndex(), entropy.currentNumberIndex());
        console.log("golden god: ", entropy.slotIndexSelectionPoint(), entropy.numberIndexSelectionPoint());
        console.log("____________________");

        vm.resumeGasMetering();
    }
}
```

**Full logs from the test (showing generation 2 completing without hitting the golden god):**

```
________ after minting 10,000 NTFs ____________
gen:  1 10000
gen:  2 10000
----
current:  544 10
golden god:  717 2
```

Even though `currentSlotIndex` (544) never reached `slotIndexSelectionPoint` (717), generation 2 was fully minted.

## Recommendation

The protocol should guarantee that every generation has exactly one golden god. Possible mitigations:

- Revert on the last forge of a generation if the golden god has not yet been minted.
- Add special logic so that the **last mint** of a generation is forced to be the golden god if it hasn't appeared yet.
- Adjust the entropy pointer advancement logic so that forging does not reduce the chance of hitting the golden god position.

---
