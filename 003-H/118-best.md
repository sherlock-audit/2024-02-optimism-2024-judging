Square Honey Sloth

high

# Incorrect logic around resolving claims and distributing bonds.

## Summary
When a claim is resolved, defense move is considered as counter, which results in incorrect distribution of bonds.

## Vulnerability Detail
[FaultDisputeGame.sol L456-459](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L456-L459)
```Solidity
if (claim.counteredBy == address(0) && leftmostCounter.raw() > claim.position.raw()) {
    countered = claim.claimant;
    leftmostCounter = claim.position;
}
```

When resolving a claim happens, it iterates through sub-games and chooses the leftmost child which is not countered by, and set it as countered, distribute the bond of parent to the child claimer.
This is because when the child claimer is not counted by anyone, it means it is assumed as correct.
However, the problem with the code above is, it iterates through sub-games with defense move and possibly can set it as countered one.

Here's an example scenario of how it becomes an issue:
1. The root claimer creates a game with correct root claim.
2. Alice(assumed malicious) comes in, attack on the root claim.
3. The root claimer again attacks Alice because he knows Alice's claim is not correct.
4. Another observer Bob comes in, defend the the claim by root claimer on step 3.
5. Since the root claim is absolutely correct, there happens no more attack/defense happening to the game.(Also Alice might notice she was wrong and does not attack anymore)

Here's a diagram to visually show the scenario above:
![Incorrect claim resolution](https://i.ibb.co/hRZc79h/Untitled-Diagram-drawio.png)

As shown in the diagram, after time passes and claims are resolved, the bond is distributed in incorrect way.
1. Bob has no-one who countered on his claim, he retrieves his bond back.
2. Bob becomes a party who countered the root claimer and he has no countered party, thus Root Claimer's bond goes to Bob while Bob is set as countered.
3. It comes to Alice, her countered party becomes zero, means her bond is returned to herself.
4. Finally it reaches to the root claim, Alice becomes his counter, and the bond of root claimer goes to Alice.

Here's a test case written in Foundry:
```Solidity
function test_auditInvalidPayout() public {
    uint256 bal = 1000 ether;
    address alice = address(0xa11ce);
    address bob = address(0xb0b);

    vm.deal(address(this), bal);
    vm.deal(alice, bal);
    vm.deal(bob, bal);

    // Alice attack the root claim with incorrect claim
    uint256 firstBond = _getRequiredBond(0);
    vm.prank(alice);
    gameProxy.attack{ value: firstBond }(0, _dummyClaim());
    vm.stopPrank();

    // The root claimer attacks alice because she is not correct
    uint256 secondBond = _getRequiredBond(1);
    gameProxy.attack{ value: secondBond }(1, _dummyClaim());

    // Bob defends the root claimer since he believes the root claim is correct
    uint256 thirdBond = _getRequiredBond(2);
    vm.prank(bob);
    gameProxy.defend{ value: thirdBond }(2, _dummyClaim());
    vm.stopPrank();

    // There comes no more attacks or defends coming, all assumed the root claim is correct.
    // Also this time Alice notices she was incorrect and doesn't attack the root claim anymore.
    // Thus time passes and the game is resolved.
    vm.warp(block.timestamp + 3 days + 12 hours + 1 seconds);

    // Resolve all claims
    for (uint256 i = gameProxy.claimDataLen(); i > 0; i--) {
        (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
        assertTrue(success);
    }
    gameProxy.resolve();

    // Wait for the withdrawal delay.
    vm.warp(block.timestamp + delayedWeth.delay() + 1 seconds);

    // Root claimer has no credit to claim at the end, thus revert.
    vm.expectRevert();
    gameProxy.claimCredit(address(this));

    // Alice should have her bond paid to the root claimer, but she claims her bond
    gameProxy.claimCredit(alice);
    assertEq(alice.balance, bal);

    // Bob should keep his original balance, but he claims the bond of root claimer
    gameProxy.claimCredit(bob);
    assertEq(bob.balance, bal + secondBond, "incorrect bob balance");
}
```
The test succeeds:
```bash
[PASS] test_auditInvalidPayout() (gas: 760726)
Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 107.83ms

Ran 1 test suite in 107.83ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact
The bonds are distributed incorrectly.

## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L456-L459

## Tool used
Manual Review, Foundry

## Recommendation
When it iterates through its sub-games, it should check if the child is an attack.
