Calm Cyan Falcon

medium

# bond can not be withdrawn after timelock

## Summary
The `resolveClaim()` function determines the winner of a `subgame`. In `resolveClaim()`, `_distributeBond()` is called to pay out the winner:
```javascript
 /// @inheeitdoc IFaultDisputeGame
    function resolveClaim(uint256 _claimIndex) external payable {
//..Ommitted code

        // Assume parent is honest until proven otherwise
        address countered = address(0);
        Position leftmostCounter = Position.wrap(type(uint128).max);
        for (uint256 i = 0; i < challengeIndicesLen; ++i) {
            uint256 challengeIndex = challengeIndices[i];

            // INVARIANT: Cannot resolve a subgame containing an unresolved claim
            if (subgames[challengeIndex].length != 0) revert OutOfOrderResolution();

            ClaimData storage claim = claimData[challengeIndex];

            // If the child subgame is uncountered and further left than the current left-most counter,
            // update the parent subgame's `countered` address and the current `leftmostCounter`.
            // The left-most correct counter is preferred in bond payouts in order to discourage attackers
            // from countering invalid subgame roots via an invalid defense position. As such positions
            // cannot be correctly countered.
            // Note that correctly positioned defense, but invalid claimes can still be successfully countered.
            if (claim.counteredBy == address(0) && leftmostCounter.raw() > claim.position.raw()) {
                countered = claim.claimant;
                leftmostCounter = claim.position;
            }
        }

        // If the parent was not successfully countered, pay out the parent's bond to the claimant.
        // If the parent was successfully countered, pay out the parent's bond to the challenger.
->       _distributeBond(countered == address(0) ? parent.claimant : countered, parent);

        // Once a subgame is resolved, we percolate the result up the DAG so subsequent calls to
        // resolveClaim will not need to traverse this subgame.
        parent.counteredBy = countered;

        // Resolved subgames have no entries
        delete subgames[_claimIndex];

        // Indicate the game is ready to be resolved globally.
        if (_claimIndex == 0) {
            subgameAtRootResolved = true;
        }
    }
```
When `_distributeBond()` gets called, it increments the `credit` of the user with the `bond`
```javascript
    /// @notice Pays out the bond of a claim to a given recipient.
    /// @param _recipient The recipient of the bond.
    /// @param _bonded The claim to pay out the bond of.
    function _distributeBond(address _recipient, ClaimData storage _bonded) internal {
        // Set all bits in the bond value to indicate that the bond has been paid out.
        uint256 bond = _bonded.bond;
        if (bond == CLAIMED_BOND_FLAG) revert ClaimAlreadyResolved();
        _bonded.bond = CLAIMED_BOND_FLAG;

        // Increase the recipient's credit.
->     credit[_recipient] += bond;

        // Unlock the bond.
        WETH.unlock(_recipient, bond);
    }
```
Afterwards, the `unlock()` function gets called.
```javascript

    /// @inheritdoc IDelayedWETH
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
The `unlock()` function sets the `wd.timestamp` to `block.timestamp`. This is to ensure that later, when the function `withdraw` gets called, the timestamp can be checked with the `DELAY_SECONDS` to ensure that the timelock has passed.
```javascript

    /// @inheritdoc IDelayedWETH
    function withdraw(address _guy, uint256 _wad) public {
        require(!config.paused(), "DelayedWETH: contract is paused");
        WithdrawalRequest storage wd = withdrawals[msg.sender][_guy];
        require(wd.amount >= _wad, "DelayedWETH: insufficient unlocked withdrawal");
        require(wd.timestamp > 0, "DelayedWETH: withdrawal not unlocked");
 ->     require(wd.timestamp + DELAY_SECONDS <= block.timestamp, "DelayedWETH: withdrawal delay not met");
        wd.amount -= _wad;
        super.withdraw(_wad);
    }
```


## Vulnerability Detail
However, a problem occurs when a person wins multiple `subgames`, resulting in a temporary DoS.

## Impact
Proof of Concept:
- Bob wins a `subgame1` on the `1st of January`, resulting in `_distributeBond()` being called.
- `DELAY_SECONDS = 7 days`.
- After `wd.timestamp + DELAY_SECONDS`, Bob will be able to withdraw this `bond` earned from `subgame1`; this will be on `January 8th`.
- Bob can claim after `wd.timestamp + DELAY_SECONDS` -> `January 8th`.
- Bob wins `subgame2` on `January 7th`, resulting in `_distributeBond` being called again.
- This time, the previously set `wd.timestamp` from `subgame1`  gets overwritten with a new `block.timestamp`
- This means that the bond that Bob won from `subgame1`, which was supposed to unlock on `January 8th`, will now be unlocked on `January 14th`.
- With every additional win, `wd.timestamp` keeps getting overwritten.

This means that Bob is unable to withdraw his rightfully earned bonds at the agreed upon `block.timestamp + DELAY_SECONDS`.


## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L56-L81

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L405-L477
## Tool used

Manual Review

## Recommendation
Handle distribution of bonds on a per subgame basis.
