Tart Ultraviolet Kitten

high

# Loss of bond amounts on re-org attacks

## Summary

The `move()` function lacks proper identification of the target of the move, leading to successful re-org attacks which can take the honest participant's funds.

## Vulnerability Detail

Participants in the game can call `attack()`, `defend()` or `move()`, each accepting a `parentIndex` which corresponds to the claim being challenged, and a `_claim` commitment. 

When participants claim, they have a particular claim in mind which they wish to challenge, and then pass on that claim's index. However, between the moment they sent the TX and the moment that TX is executed, a block reorg can take place. When it occurs, the challenge corresponding to that ID may change to another challenge, which may be valid or invalid in a different way. Regardless, the participant's commitment to that `move()` will be wrong, and they stand to lose their bond amount.

Chain reorgs are very prevalent in Ethereum mainnet, where the contract is deployed. You can check [this](https://etherscan.io/blocks_forked) index of reorged blocks on etherscan. It is **incorrect** to assume the attacker will wait until it achieved finality, because there's no warnings or documentation available for them to identify this as a threat. Therefore, it remains a very valid concern with reasonable hypotheticals.

Note that in high depths, the bond amount is very large, leading to a large loss of funds.

Possible flow:
- Attacker submits invalid claim hash
- Honest defenders rush to prove the claim wrong (note that only first defender gets the bond, so they will rush to submit the TX. They would not be concerned about waiting against reorgs without warning)
- A block re-org occurs
- The attacker replaces the invalid claim with a valid claim hash
- Defender's TXs are applied on top of the valid claim.
- Attacker can scoop up all the defenders' bonds

## Impact

Loss of bond value for honest participants of the dispute game.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Every move needs to include the key parameters which it wishes to attack/defend - the claim hash and the Position in the game tree.
