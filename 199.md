Tart Ultraviolet Kitten

medium

# Threat of unlimited parallelized games undermine core properties of the dispute game

## Summary

Threat of highly parallelized games undermine core properties of the dispute game

## Vulnerability Detail

The fault dispute game allows any number of players to make challenges on any other claim, until the MAX_DEPTH of the game tree is reached. A siginificant risk which has not been disclosed at any point, but is a valid way for malicious claims to resolve as valid, is that the honest team *runs out of bonds*. In fact, the game can be reduced to stake wars between the Honest and Dishonest parties. It is easy to see that a dishonest team with more funds can always invalidate a specific claim, by way of sending enough challenges on it / it's parents until the opposing side is no longer able to challenge back.

The key is that until MAX_DEPTH, where the actual Fault Proof VM is consulted, all claims are taken at face value. But for this to be a sufficient safety mechanism, it must be certain that the honest team will be able to escalate up to MAX_DEPTH for each opposing claim. However, many claims can permissionlessly be sent in parallel for the same depth, meaning to safeguard all intermediate claims, the honest team must have enough funds to cover all of them in parallel (not that they cannot reuse funds from one challenge for another, as they are locked until the game is over and a safety delay amount passes).

In fact, it can be shown that a minority stake team can win any particular claim. This is because each depth is more expensive than the previous one, so for the honest team to have the final say, they have to cover the cost of a more expensive claim.

This game theory issue undermines the entire point of fraud proofs, whose invariant is that an honest team can ALWAYS defend an honest claim.

## Impact

A party with sufficient funds can beat any particular claim, up to the root claim which unlocks access to the entire L2 locked funds. They can take over any particular bond posted in a game tree.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Allow additional mechanisms to make use of the FPVM without reaching max depth, or limit the amount of concurrent challenges in a safe way.


