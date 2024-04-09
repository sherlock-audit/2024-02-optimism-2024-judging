Noisy Pewter Woodpecker

medium

# Game implementation can be set before bond, allowing unbonded games to be created

## Summary

The `DisputeGameFactory` allows the owner to create a new type of game by setting the implementation contract (`gameImpls`) and the bond price (`initBonds`). If the implementation is not set, the game cannot be created. However, if the bond is not set, the game can be created at no cost. Since these two values are set independently, this allows for backrunning attacks to create unbonded games.

## Vulnerability Detail

When a new game is created, the factory owner can implement it by calling the following two functions:
```solidity
function setImplementation(GameType _gameType, IDisputeGame _impl) external onlyOwner {
    gameImpls[_gameType] = _impl;
    emit ImplementationSet(address(_impl), _gameType);
}

function setInitBond(GameType _gameType, uint256 _initBond) external onlyOwner {
    initBonds[_gameType] = _initBond;
    emit InitBondUpdated(_gameType, _initBond);
}
```
As soon as `setImplementation()` is called, the `create()` function can then be called permissionlessly to create a game using the corresponding implementation.

Since these two functions must be called independently, in the moment between the two executions a game can have an implementation set and a bond of `0`. All validation that a sufficient bond has been paid is done in the factory (not in the game itself, which trusts that `msg.value` is sufficient), so this situation allows for the game to be created without a bond.

Without the rewards of a bond, there is no incentive for anyone to defend against the claim, which could lead to invalid yet undisputed games.

## Impact

As new games are deployed, malicious actors can time transactions to create games with no bond, which removes the incentive for anyone to challenge the game and could lead to incorrect claims being validated.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L188-L198

## Tool used

Manual Review

## Recommendation

The `setImplementation()` function should also include setting the bond value to ensure we never have an implementation set without a corresponding bond:

```diff
- function setImplementation(GameType _gameType, IDisputeGame _impl) external onlyOwner {
+ function setImplementation(GameType _gameType, IDisputeGame _impl, uint256 _initBond) external onlyOwner {
      gameImpls[_gameType] = _impl;
      emit ImplementationSet(address(_impl), _gameType);

+     initBonds[_gameType] = _initBond;
+     emit InitBondUpdated(_gameType, _initBond);
  }
```