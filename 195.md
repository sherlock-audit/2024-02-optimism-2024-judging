Tart Ultraviolet Kitten

medium

# An attacker can cause temporary freeze of funds to a participant in the FaultDisputeGame

## Summary

## Vulnerability Detail

A fault dispute game is built from the factory, which initializes the first claim in the array below:

```solidity
claimData.push(
    ClaimData({
        parentIndex: type(uint32).max,
        counteredBy: address(0),
        claimant: tx.origin,
        bond: uint128(msg.value),
        claim: rootClaim(),
        position: ROOT_POSITION,
        clock: LibClock.wrap(Duration.wrap(0), Timestamp.wrap(uint64(block.timestamp)))
    })
);
```

The sender passes a `msg.value` which equals the required bond amount, and the registered `claimant` is `tx.origin`.  At the end of the game, if the claim is honest, the funds will be returned to the `claimant`.

Suppose the following pre-conditions hold:
- A participant in the game performs an interactions that yields a malicious actor code execution (Simply paying them ETH, or trading an NFT which will reach the receiver's callback, etc)


Now, an attacker can propose an honest root to the DisputeGamseFactory. The registered `claimant` will be the victim. After a week, their claim will be resolvable, as the game has ended. Upon resolution, the bond will be credited to the victim. 
`_distributeBond(countered == address(0) ? parent.claimant : countered, parent);`
The issue is that this line represents an unauthorized temporary freeze of funds. The function requests unlocking of the WETH:
```solidity
WETH.unlock(_recipient, bond);
```

It appends to a current withdrawal request, and updates the block timestamp:
```solidity
function unlock(address _guy, uint256 _wad) external {
    // Note that the unlock function can be called by any address, but the actual unlocking capability still only
    // gives the msg.sender the ability to withdraw from the account. As long as the unlock and withdraw functions
    // are called with the proper recipient addresses, this will be safe. Could be made safer by having external
    // accounts execute withdrawals themselves but that would have added extra complexity and made DelayedWETH a
    // leaky abstraction, so we chose this instead.
    WithdrawalRequest storage wd = withdrawals[msg.sender][_guy];
    wd.timestamp = block.timestamp;
    wd.amount += _wad;
}
```

Withdrawal will now be impossible until the DELAY_SECONDS holding time completes:
`require(wd.timestamp + DELAY_SECONDS <= block.timestamp, "DelayedWETH: withdrawal delay not met");`

So, a participant can lose access to any fully unlocked / moments before unlocked funds, for the next full `DELAY_SECONDS`. This could possibly be chained to keep relocking access to the funds. Temporary freeze of funds is highly dangerous because it contradicts  the assumption of the owner that they will be able to mobilize those funds.

For example, if they have taken a loan, and knew the funds would be unlocked in time to repay it (they responsibility worked out the safe loan period), due to this attack they will be liquidated and penalized. This is just one of the many options in which temporary freeze of funds results in monetary losses for the victim.

## Impact

An attacker can cause temporary freeze of funds to a participant in the FaultDisputeGame.

## Code Snippet

```solidity
claimData.push(
    ClaimData({
        parentIndex: type(uint32).max,
        counteredBy: address(0),
        claimant: tx.origin,
        bond: uint128(msg.value),
        claim: rootClaim(),
        position: ROOT_POSITION,
        clock: LibClock.wrap(Duration.wrap(0), Timestamp.wrap(uint64(block.timestamp)))
    })
);
```


## Tool used

Manual Review

## Recommendation

Consider refactoring the `unlocking` mechanism - new withdrawal requests should not reset the previous countdowns or issues like the one described could manifest.
