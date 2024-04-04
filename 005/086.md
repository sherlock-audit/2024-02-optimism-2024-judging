Noisy Pewter Woodpecker

medium

# Multiple dispute games can be created with identical block number and root

## Summary

It is intended that only one dispute game can be created per block number and root. This is enforced in the factory by checking that a UUID hasn't been previously used. However, if `extraData` is passed as less than 32 bytes, we can obtain the same `l2BlockNumber()` result while getting a different UUID, skirting this check.

## Vulnerability Detail

In the dispute game factory's `create()` function, we use the `rootClaim` (bytes32) and the L2 block number (expressed in `extraData` as `bytes`) in order to construct a game contract.

```solidity
proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(_rootClaim, parentHash, _extraData)));
```
It is intended that we cannot create two dispute games with the same block number and root. We accomplish this by calculating the UUID for the game (the hash of the game type, the root claim, and the block number), and then confirming that a game with this combination hasn't already been created by the factory.

```solidity
// Compute the unique identifier for the dispute game.
Hash uuid = getGameUUID(_gameType, _rootClaim, _extraData);

// If a dispute game with the same UUID already exists, revert.
if (GameId.unwrap(_disputeGames[uuid]) != bytes32(0)) revert GameAlreadyExists(uuid);
```
The L2 block number is passed as bytes. It's immutably set on the proxy for the game, and each call to the implementation passes along the bytes as calldata. When we access the L2 block number, we return the next 32 bytes of the calldata, starting at the relevant offset. This would theoretically allow us to pass arbitrarily longer bytestrings by appending more bytes, getting the same L2 block number, but returning a different UUID.

To prevent this, the `initialize()` function performs the following check, capping the maximum calldatasize at 102 bytes:
```solidity
assembly {
    if gt(calldatasize(), 0x66) {
        // Store the selector for `ExtraDataTooLong()` & revert
        mstore(0x00, 0xc407e025)
        revert(0x1C, 0x04)
    }
}
```
However, there is no check whether `calldatasize` is less than 102 bytes. In the event that the hex representation of the L2 block number ends with a zero, we can pass fewer than 32 bytes and still get the same L2 block number, because reading from calldata locations beyond the passed calldata will always return zero. This results in the same L2 block number but a different UUID.

## Proof of Concept

The following proof of concept can be added to `DisputeGameFactory.t.sol` to demonstrate the vulnerability:
```solidity
function testDuplicateGames() public {
    // start with a block number that ends with at least 1 zero
    uint currentBlock = 19550208; // 0x12a5000
    bytes memory blockNumBytes = abi.encode(currentBlock);
    IDisputeGame game1 = disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(0)), blockNumBytes);
    assertEq(game1.l2BlockNumber(), currentBlock);

    // create a shorter bytestring with the same number, but the final zeros removed
    bytes memory blockNumSnipped = hex"00000000000000000000000000000000000000000000000000000000012a50";
    IDisputeGame game2 = disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(0)), blockNumSnipped);

    // it is allowed because UUIDs are different, but gametype, claim, and l2 block number are the same
    assertEq(GameType.unwrap(game1.gameType()), GameType.unwrap(game2.gameType()));
    assertEq(Claim.unwrap(game1.rootClaim()), Claim.unwrap(game2.rootClaim()));
    assertEq(game1.l2BlockNumber(), game2.l2BlockNumber());
    assert(address(game1) != address(game2));
}
```

## Impact

Multiple dispute games can be created with identical block numbers and roots.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L105-L113

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L125-L135

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L546-L552

## Tool used

Manual Review

## Recommendation

The `initialize()` function should check that the calldata is exactly the right size, rather than just enforcing a cap.
```diff
assembly {
-   if gt(calldatasize(), 0x66) {
+   if not(eq(calldatasize(), 0x66)) {
        // Store the selector for `ExtraDataTooLong()` & revert
        mstore(0x00, 0xc407e025)
        revert(0x1C, 0x04)
    }
}
```
