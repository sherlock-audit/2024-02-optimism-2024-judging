Noisy Pewter Woodpecker

high

# Games created before OptimismPortal2 was deployed will be considered valid

## Summary

To finalize a withdrawal, it is required that the game was created after the last time the `respectedGameType` was updated. This is because the off-chain watchers are not guaranteed to be monitoring the system until this update has been made. However, when `respectedGameType` is initially set, we don't update `respectedGameTypeUpdatedAt`, leaving it set at zero and allowing games created before the upgrade to slip through.

## Vulnerability Detail

When finalizing a withdrawal on `OptimismPortal2`, there is an explicit check that the game must be created after the respected game type was updated on the portal (and has therefore been monitored by off chain watchers). This is to prevent users from creating games before the rest of the system is set up and watching, and then finalizing withdrawals against them.

```solidity
// The game must have been created after `respectedGameTypeUpdatedAt`. This is to prevent users from creating
// invalid disputes against a deployed game type while the off-chain challenge agents are not watching.
require(
    createdAt >= respectedGameTypeUpdatedAt,
    "OptimismPortal: dispute game created before respected game type was updated"
);
```
This check works in all cases, except the first time the game type is set. When the `respectedGameType` is first set in the constructor (should be moved to `initialize()`, which I've reported in a separate issue), there is no update to `respectedGameTypeUpdatedAt`. Therefore, it will be set to `0` until the first time the game is changed.

This creates a risk that, if the factory is deployed before the Portal is upgraded and watchers are set up, games can be created that are not watched by the off-chain challenger.

Such a game could slip through unnoticed, and will pass both the checks that (a) the game was created before it was proven, (b) the game was created after the last update to the game type, and (c) the game was resolved more than `DISPUTE_GAME_FINALITY_DELAY_SECONDS` ago.

## Impact

Unwatched games that should be caught because the `createdAt` time is before the respected game type is upgraded upon deployment, would be allowed to finalize and withdraw funds.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L126-L162

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L502-L507

## Tool used

Manual Review

## Recommendation

Update `respectedGameTypeUpdatedAt` upon deployment when the `respectedGameType` is set.
