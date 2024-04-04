Tame Orchid Snail

high

# `FaultDisputeGame`: The bond will be locked if the leaf is not resolved before the entire `resolve`

## Summary

The function `FaultDisputeGame:resolveClaim` let nodes to be resolved even when the children leaf nodes are not yet resolved as long as the children do not have subnodes. As the result the bond for the leaf nodes will be locked, if the `resolveClaim` is not called on the leaf node before the whole game is resolved by `FaultDisputeGame::resolve`.

## Vulnerability Detail

Below is a test case to demonstrate this issue. The test is based on the `FaultDisputeGame_Test`.

The demonstrated scenario is following.
1. There are three attacks, therefore four nodes exist including the rootClaim: `0(rootClaim) - 1 - 2 - 3`
2. Time passes long enough for resolve/resolveClaim
3. case 1: the leaf node 3 can be resolved later then the other nodes, as the node 3 does not have any children.
4. case 2: however, if the `resolve` for the whole game is called before the leaf node 3 is resolved, the node 3 cannot be resolved anymore, as the game is not anymore in progress.

Note that
1. as the depth goes deeper the required bond is growing rapidly 
2. depending on how the time was spent by each team, there may be a situation where the last move was made only a few seconds before the entire game can be resolved. In that case, it may be possible a malicious actor can resolve up the tree and the entire game in one transaction, therefore the victim (the creator of the leaf node) might not have enough time to react.
3. if the bond should be locked, only manual recovery in DelayedWeth can rescue the bond.


```solidity
// As a function in `FaultDisputeGame_Test`
    function test_unresolved_leaf() public {
        gameProxy.attack{ value: _getRequiredBond(0) }(0, _dummyClaim());
        gameProxy.attack{ value: _getRequiredBond(1) }(1, _dummyClaim());
        gameProxy.attack{ value: _getRequiredBond(2) }(2, _dummyClaim());

        // random time long enough to resolveClaims
        vm.warp(block.timestamp + 7 days);

        // SNAPSHOT TO COMBACK
        uint256 snapshot = vm.snapshot();

        gameProxy.resolveClaim(2);
        gameProxy.resolveClaim(1);
        gameProxy.resolveClaim(0);
        uint creditBefore = gameProxy.credit(address(this));
        gameProxy.resolveClaim(3);
        // There is credit left to be claimed
        assertEq(_getRequiredBond(2), gameProxy.credit(address(this)) - creditBefore);
        assertGt(_getRequiredBond(2), 0);
        assertEq(uint8(gameProxy.resolve()), uint8(GameStatus.CHALLENGER_WINS));


        // GO BACK TO THE SNAPSHOT
        vm.revertTo(snapshot);

        gameProxy.resolveClaim(2);
        gameProxy.resolveClaim(1);
        gameProxy.resolveClaim(0);
        assertEq(uint8(gameProxy.resolve()), uint8(GameStatus.CHALLENGER_WINS));
        
        // if the resolveClaim is called in time, it cannot be claimed anymore
        vm.expectRevert(abi.encodeWithSignature("GameNotInProgress()"));
        gameProxy.resolveClaim(3);
    }
```

The cause of the bug is that the `FaultDisputeGame:resolveClaim` does not explicitly check the leaf nodes are resolved as long as they do not have children themsolves.
In the above example, the node 3 does not have any children, therefore `resolveClaim` can be called on the node 2 (the parent of node 3) without resolving the node 3 itself.

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L428-L438


## Impact

The bonds for leaf nodes will be locked unless the owner of `DelayedWETH` uses `recover` or `hold` function on the game.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L428-L438

## Tool used

Manual Review

## Recommendation

Due to the complexity of the FaultDisputeGame resolution functionality, it is unclear what is most clean way to mitigate this.
One possible way to mitigate might be adding a check whether the leaf nodes are resolved, instead of just checking the length of their subgames.

