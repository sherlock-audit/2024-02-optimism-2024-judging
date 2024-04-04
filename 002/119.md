Bitter Ivory Baboon

medium

# User might not be able to finalize their transaction

## Summary

There might be a case when a user needs to prove their transaction again but cannot do so, and as a result, cannot finalize it.

## Vulnerability Detail

A user can prove their transaction just after a dispute game is created, while it is still in the `IN_PROGRESS` status.

There might be cases when the user needs to prove their transaction again (for example, if the game is blacklisted or if the `respectedGameType` is changed).

Additionally, there might be different game types, and the `respectedGameType` can be changed and rolled back afterwards.

An example provided by a protocol team member:

> Let's say the fault proof system failed and we needed to fall back to the `PermissionedDisputeGame`, which follows the same security model as the OP Stack today with permissioned actors who are allowed to propose and challenge outputs.

> In this case, we'd change the respected game type to `2` from `1`. We have the ability to upgrade the implementation of game type `1` through the `DisputeGameFactory`'s `setImplementation` function as well. Once the bug in the permissionless game (`FaultDisputeGame.sol`) was resolved, we'd be able to change the respected game type in the `OptimismPortal` back to `1`. Because of the `respectedGameTypeUpdatedAt` timestamp check, it is intended that games that were created prior to rolling back would not be considered valid in the context of being able to finalize withdrawals that were proven against them.

Now let's examine the following scenario:

- A user proves a withdrawal transaction but does not finalize it yet.
- The `respectedGameType` is changed (e.g due to a bug in the `FaultDisputeGame` implementation).
- The `respectedGameType` is reverted back to its previous value (e.g. once the bug is resolved).
- During these updates, the game with which the user proved their transaction is resolved with the `DEFENDER_WINS` status.
- The user is now unable to finalize the transaction because the `respectedGameTypeUpdatedAt` timestamp was updated, and the creation time of the game with which the user proved their transaction is earlier.
- The user needs to re-prove their transaction to finalize it.
- However, the user cannot do this because none of the conditions for this are met.

```solidity
require(
    provenWithdrawal.timestamp == 0 || oldGame.status() == GameStatus.CHALLENGER_WINS
    || disputeGameBlacklist[oldGame] || oldGame.gameType().raw() != respectedGameType.raw(),
    "OptimismPortal: withdrawal hash has already been proven, and the old dispute game is not invalid"
);
```

- `provenWithdrawal.timestamp > 0` - the user has already proved the transaction before.
- `oldGame.status() == DEFENDER_WINS` - the game was resolved in favor of the defender.
- The dispute game was not blacklisted, and the `respectedGameType` is the same.

## Impact

This issue causes valid withdrawals to temporarily become non-finalizable.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L287-L291

## Tool used

Manual Review

## Recommendation

```diff
         require(
             provenWithdrawal.timestamp == 0 || oldGame.status() == GameStatus.CHALLENGER_WINS
-                || disputeGameBlacklist[oldGame] || oldGame.gameType().raw() != respectedGameType.raw(),
+                || disputeGameBlacklist[oldGame] || oldGame.gameType().raw() != respectedGameType.raw()
+                || oldGame.createdAt().raw() < respectedGameTypeUpdatedAt,
             "OptimismPortal: withdrawal hash has already been proven, and the old dispute game is not invalid"
         );
```
