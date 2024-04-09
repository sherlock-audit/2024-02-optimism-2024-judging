Refined Juniper Copperhead

medium

# Users can call `attack` and `claim` on the same claim

## Summary
See bellow...
## Vulnerability Detail
The vulnerability in the provided contract lies in the fact that users can call both the `attack` and `defend` functions on the same claim, potentially bypassing intended game rules and leading to unexpected outcomes. Let's break down the vulnerability in detail:
- The contract implements a game mechanism where participants can either attack or defend against a claim. These moves affect the outcome of the dispute resolution process.
- The contract does not impose any restrictions to prevent a user from calling both `attack` and `defend` functions on the same claim. As a result, a user can potentially perform both actions on the same claim, which is not intended according to the game rules.
- By allowing a user to perform both actions (attack and defend) on a single claim, the contract opens up possibilities for gaming the system. For instance, a malicious user could initially make an attack move to potentially gain an advantage, and subsequently make a defend move to counter their own attack or disrupt the game flow.
- Consider a scenario where User A initiates an attack on a claim by calling the `attack` function. Before the dispute is resolved, User A, or another user, calls the `defend` function on the same claim. This results in conflicting moves on the claim, leading to confusion and potentially impacting the outcome of the dispute unfairly.
```solidity
// Example vulnerability scenario
// User A initiates an attack
faultDisputeGame.attack(parentIndex, claim);

// Before resolution, User B calls defend on the same claim
faultDisputeGame.defend(parentIndex, claim);
```
## Impact
Allowing users to make conflicting moves on the same claim compromises the integrity of the dispute resolution game. It undermines the fairness and reliability of the process, as the actions of participants may no longer accurately reflect their true intentions or positions in the dispute.
## Code Snippet
[FaultDisputeGame.sol#L321-L323](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L321-L323)
[FaultDisputeGame.sol#L326-L328](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L326-L328)
## Tool used

Manual Review

## Recommendation
Introduce a mapping `userHasMadeMove` to track whether a user has made a move associated with a specific claim. The key of the mapping is the claim index, and the value is another mapping where the key is the user's address and the value is a boolean indicating whether the user has made a move.