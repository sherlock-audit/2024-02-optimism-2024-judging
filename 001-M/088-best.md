Noisy Pewter Woodpecker

medium

# `respectedGameType` will not be set according to the config

## Summary

`OptimismPortal2` uses the `respectedGameType` value to determine which games are currently valid. This value is set in the constructor (rather than the `initialize()` function), so it will be set in the storage of the implementation, but not on the proxy. As a result, `respectedGameType` will always be deployed as `0`, regardless of the chain's deployment configuration.

## Vulnerability Detail

When `OptimismPortal2.sol` is deployed, the constructor sets the initial immutable values and calls `initialize()` with zero values to ensure it can't be reinitialized.

```solidity
constructor(
    uint256 _proofMaturityDelaySeconds,
    uint256 _disputeGameFinalityDelaySeconds,
    GameType _initialRespectedGameType
) {
    PROOF_MATURITY_DELAY_SECONDS = _proofMaturityDelaySeconds;
    DISPUTE_GAME_FINALITY_DELAY_SECONDS = _disputeGameFinalityDelaySeconds;
    respectedGameType = _initialRespectedGameType;

    initialize({
        _disputeGameFactory: DisputeGameFactory(address(0)),
        _systemConfig: SystemConfig(address(0)),
        _superchainConfig: SuperchainConfig(address(0))
    });
}
```

However, as we can see, the `respectedGameType` is also set in the `constructor`. Looking at the storage slot, we can see that it is not an immutable value:
```solidity
/// @notice The game type that the OptimismPortal consults for output proposals.
GameType public respectedGameType;
```

Therefore, it will be saved in storage of the implementation, which does not impact the proxy. When `initialize()` is called on the proxy, there is no value set for `respectedGameType`, so it will default to `0`, regardless of the chain's config.

## Impact

All new chains deployed (including those where `respectedGameType != 0`) will have the game type set to `0` on deployment. In the event that game type `0` is exploitable, this would make new chains vulnerable to being exploited.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L127-L141

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L147-L162

## Tool used

Manual Review

## Recommendation

Move `respectedGameType = _initialRespectedGameType` from the constructor to the `initialize()` function to ensure that the value is set on the proxy, rather than the implementation.
