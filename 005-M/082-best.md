Noisy Pewter Woodpecker

medium

# OptimismPortal cannot be upgraded to OptimismPortal2

## Summary

When upgrading the Portal, it is required to call `initialize()` in order to set the `gameDisputeFactory` value. However, calls to `initialize()` will fail because the Portal has already been initialized, so the storage value is set.

## Vulnerability Detail

When `OptimismPortal2` is deployed, the `initialize()` function must be called in order to set `disputeGameFactory`. Note that there is nowhere else in the code where this can be set, so it is required that `initialize()` is callable.

```solidity
function initialize(
    DisputeGameFactory _disputeGameFactory,
    SystemConfig _systemConfig,
    SuperchainConfig _superchainConfig
)
    public
    initializer
{
    disputeGameFactory = _disputeGameFactory;
    systemConfig = _systemConfig;
    superchainConfig = _superchainConfig;
    if (l2Sender == address(0)) {
        l2Sender = Constants.DEFAULT_L2_SENDER;
    }
    __ResourceMetering_init();
}
```
As we can see, this function contains the `initializer` modifier. This function comes from the OpenZeppelin `Initializable` contract, which sets a storage slot upon executing the function the ensures it cannot be called again.

If we inspect the storage layout for the two contracts, we can see that they use the same storage slot (0).

`forge inspect OptimismPortal storage --pretty`
| Name                 | Type                                                       | Slot | Offset | Bytes | Contract                                 |
|----------------------|------------------------------------------------------------|------|--------|-------|------------------------------------------|
| _initialized         | uint8                                                      | 0    | 0      | 1     | src/L1/OptimismPortal.sol:OptimismPortal |
| _initializing        | bool                                                       | 0    | 1      | 1     | src/L1/OptimismPortal.sol:OptimismPortal |
...

`forge inspect OptimismPortal2 storage --pretty`
| Name                       | Type                                                                            | Slot | Offset | Bytes | Contract                                   |
|----------------------------|---------------------------------------------------------------------------------|------|--------|-------|--------------------------------------------|
| _initialized               | uint8                                                                           | 0    | 0      | 1     | src/L1/OptimismPortal2.sol:OptimismPortal2 |
| _initializing              | bool                                                                            | 0    | 1      | 1     | src/L1/OptimismPortal2.sol:OptimismPortal2 |

Further, if we check the on chain value for this slot, we can see that `_initialized = 1`.

`cast storage 0x5fb30336a8d0841cf15d452afa297cb6d10877d7 0`
```solidity
0x0000000000000000000000000000000000000000000000000000000000000001
```

## Impact

Upgrading the Portal to `OptimismPortal2` will fail because the `initialize()` function cannot be called again. This function cannot be skipped, or else the `disputeGameFactory` will not be able to be set, and therefore the contract will not function.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L147-L162

## Tool used

Manual Review

## Recommendation

Use the `reinitializer` modifier to allow for the `initialize()` function to be called again.