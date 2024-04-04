Noisy Pewter Woodpecker

medium

# Blacklisted games still update Anchor State Registry, bricking game type

## Summary

When an invalid game is resolved, the `AnchorStateRegistry` is immediately updated to reflect the new state. This update does not have any delay for the protocol team to fix things if it's incorrect, and there is no way to override the updated state once it's discovered.

## Vulnerability Detail

When a game is resolved, three things happen, all of which should account for the possibility of a fraudulent game:

✅ The status is set based on who wins. If a game incorrectly resolves to `DEFENDER_WINS`, the guardian has time to blacklist the game before any withdrawals can be finalized against it.

✅ Bonds are paid out. If a game incorrectly resolves, the `DelayedWETH` owner has time to take the funds before any bonds can be withdrawn.

❌ The `AnchorStateRegistry` is updated. If a game is resolved incorrectly, the `AnchorStateRegistry` is immediately updated to reflect the new state. This is an irreversible operation that permanently updates the last trusted state to the `rootClaim` of the game.

In the event that the game type is still being used (ie if the exploited game was blacklisted instead of the whole game type being abandoned), this leads to an `AnchorStateRegistry` that requires all games of this type to build off the incorrect state.

## Impact

In the best case, this will lead to valid claims not being able to be proven, and the game type needing to be changed to manually reset the system.

In the worst case, it could open up the door to an attacker skirting the off-chain monitoring by performing a valid state transition from an invalid starting point.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L386-L402

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L59-L87

## Tool used

Manual Review

## Recommendation

`tryUpdateAnchorState()` should act similarly to the other impacts of a resolved game, implementing some time delay before trusting the result.

For example, the `AnchorStateRegistry` could store an array of state updates along with the timestamp they were submitted. Once a period of time has passed, anyone can permissionlessly integrate these updates into the `anchors` array.
