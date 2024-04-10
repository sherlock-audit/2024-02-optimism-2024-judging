Tart Ultraviolet Kitten

medium

# Various defenses can be bypassed to make a created game unresolvable

## Summary

Various defenses can be bypassed to make a created game unresolvable

## Vulnerability Detail

A game is created by passing parameters to the Factory, which builds and initializes the proxy:
```solidity
proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(_rootClaim, parentHash, _extraData)));
proxy_.initialize{ value: msg.value }();
```

The architecture uses Clone with Immutable args, so the `_extraData` is stored at offset 0x40 bytes from the start of the args.
```solidity
function extraData() public pure returns (bytes memory extraData_) {
    // The extra data starts at the second word within the cwia calldata and
    // is 32 bytes long.
    extraData_ = _getArgDynBytes(0x40, 0x20);
}
```

In the Fault Dispute Game, it represents the L2 block number:
```solidity
function l2BlockNumber() public pure returns (uint256 l2BlockNumber_) {
    l2BlockNumber_ = _getArgUint256(0x40);
}
```

There is a safety guarantee in the factory that we cannot create two games for the same parameters:

```solidity
Hash uuid = getGameUUID(_gameType, _rootClaim, _extraData);
// If a dispute game with the same UUID already exists, revert.
if (GameId.unwrap(_disputeGames[uuid]) != bytes32(0)) revert GameAlreadyExists(uuid);
```

In fact, there is a way to create two games with the exact same parameters, bypassing the check above. The impact will be described later.

`extraData` is encoded as an arbitrary length bytes:
`bytes calldata _extraData`
It is enpacked via encodePacked() as above.

The `initialize()` function of the game makes sure the calldata does not exist 0x66 bytes:
```solidity
// Revert if the calldata size is too large, which signals that the `extraData` contains more than expected.
// This is to prevent adding extra bytes to the `extraData` that result in a different game UUID in the factory,
// but are not used by the game, which would allow for multiple dispute games for the same output proposal to
// be created.
// Expected length: 0x66 (0x04 selector + 0x20 root claim + 0x20 l1 head + 0x20 extraData + 0x02 CWIA bytes)
assembly {
    if gt(calldatasize(), 0x66) {
        // Store the selector for `ExtraDataTooLong()` & revert
        mstore(0x00, 0xc407e025)
        revert(0x1C, 0x04)
    }
}
```

This prevents passing an `extraData` which is longer than the expected 0x20 bytes, making for another game with the same parameters. However, the check should also verify the `calldatasize()` is not *less than 0x66*. A sophisticated attacker can pass `extraData` of length 0x1F, instead of 0x20.  When the argument is read using the `extraData()` or `l2BlockNumber()`, the CWIA implementation will read *past* the 0x1F extra data bytes and into the 2 byte CWIA size bytes. The assembly-level code has no notion of lengths, it just sees them all as bytes.
```solidity
function _getArgUint256(uint256 argOffset) internal pure returns (uint256 arg) {
    uint256 offset = _getImmutableArgsOffset();
    assembly {
        arg := calldataload(add(offset, argOffset))
    }
}
```


To understand the impact of creating two games which look the exact same for all purposes, let's look at `resolve()`:
```solidity
// Try to update the anchor state, this should not revert.
ANCHOR_STATE_REGISTRY.tryUpdateAnchorState();
```

At the end of the function, it tries updating the registry. It should never fail. The registry has the check below:
```solidity
IFaultDisputeGame game = IFaultDisputeGame(msg.sender);
(GameType gameType, Claim rootClaim, bytes memory extraData) = game.gameData();
// Grab the verified address of the game based on the game data.
// slither-disable-next-line unused-return
(IDisputeGame factoryRegisteredGame,) =
    DISPUTE_GAME_FACTORY.games({ _gameType: gameType, _rootClaim: rootClaim, _extraData: extraData });
// Must be a valid game.
require(
    address(factoryRegisteredGame) == address(game),
    "AnchorStateRegistry: fault dispute game not registered with factory"
);
```

The check is meant to make sure the caller is the true game registered by the factory. However, since there are **two** registered games, and the shortened data one will not be fetched through the `games()` call (it will return the full extraData game in both cases). This will always trip the check in the Anchor registry, which makes `resolve()` revert.
Therefore, we've created a valid game which cannot be resolved, breaking the invariant that each game should end with either DEFENDER_WINS or CHALLENGER_WINS.

Another attack avenue would be to create two challenges with the same attributes together. It is likely that defenders will trust the assumption that only one game can played for the same parameters, so they would not defend all such games. It is possible to create such a game whenever the last CWIA byte lines up with the proposed L2 block number, meaning 1/256 of the timeslots can be attackable this way.

Note that the scope excludes incorrect game resolutions (DEFENDER WINS instead of ATTACKER WINS or vice versa), but does not exclude broken game resolution.

## Impact

A game can be engineered to not be resolvable.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Verify that the number of bytes in calldata is exactly 0x66.
