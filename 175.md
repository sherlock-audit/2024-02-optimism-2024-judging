Interesting Fossilized Pony

medium

# New dispute game cannot be created for the same root claim with a different L1 parent hash

## Summary

The `DisputeGameFactory.create` function computes the UUID of a new dispute game based on the `gameType`, `rootClaim`, and `extraData` parameters. However, it does not include the L1 parent hash in the UUID computation. As a result, if a new dispute game needs to be created for the same `rootClaim` but with a different L1 parent hash (e.g., due to an issue with the original game), it is not possible because the UUID will collide with the existing game.


## Vulnerability Detail

In the `DisputeGameFactory.create` function, the UUID of a new dispute game is computed [using the `getGameUUID` function](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L110):

```solidity
Hash uuid = getGameUUID(_gameType, _rootClaim, _extraData);
```

However, [the L1 parent hash](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L103)  is not included in the UUID computation. Despite not being part of UUID, [it is passed as an argument to the `clone` function](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L106C80-L106C90) when creating the dispute game, and is then used by the dispute game logic.

The [L1 parent hash is used within the dispute game contract](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L340-L342) to set a pre-image local value that can be used during the on-chain VM step. 

If the original dispute game encounters an issue due to an unsuitable L1 parent hash (e.g., the L1 parent block is too early, or doesn't match correctly the anchored `startingOutputRoot`), and a new game needs to be created for the same `rootClaim` with a later L1 parent hash, it is not possible because the [UUID will collide with the existing game and revert the creation](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L113) of a new game.


## Impact

If a dispute game encounters an issue due to an unsuitable L1 parent hash and cannot be correctly resolved, it is not possible to create a new game for the same `rootClaim` with a different L1 parent hash. This is because the UUID computation does not include the L1 parent hash, resulting in a collision with the previously created game.

If this happens considerable delays will be caused by the mitigation actions, and will cause a significant DoS impact:
1. A new `AnchorStateRegistry` will need to be deployed, or the previous one upgraded to populate its anchored roots which [can only be set once during initialization ](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L46-L51)so will be 0 for the new game type.
2. A new implementation of the dispute game, with a new [game type value, and new anchor registry (which are immutable)](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L46-L52) will need to be deployed.
3. The factory will need to be updated [by the owner (SLA of 72 hours) to include the new implementation](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L189).
4. The respected game type for the portal would [need to be updated](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L448-L449) by guardian (SLA of 24 hours).
7. New dispute games will need to be created by proposers for the withdrawals backlog caused by the delays.
8. Only after all these steps the proving can be restarted.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L103-L113

## Tool used

Manual Review

## Recommendation

To allow creating a new dispute game for the same `rootClaim` with a different L1 parent hash, include the L1 parent hash in the UUID computation.
