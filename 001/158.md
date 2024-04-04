Prehistoric Rouge Osprey

medium

# Users can avoid withdrawal delay time after first request.

## Summary
The withdrawal process is done in two steps, the `unlock` and the `withdraw`. A user can put a request to unlock a large amount of wad, even greater than his `WETH` balance. Once the delay time his over, he's able to continually withdraw without any putting in any unlock request.

## Vulnerability Detail

The `unlock` function allows users to put in a request for withdrawal. After putting in the request, they're required to wait for a delay period before their withdrawal can be processed.
```solidity
    function unlock(address _guy, uint256 _wad) external {
...
        wd.amount += _wad; //@note _wad is not validated
    }
```
Looking at the unlock function, it increases the `wd.amount` i.e the amount to be unlocked by the user specified `wad` parameter. Noting that there's no input validation even against the user's WETH balance, a user can specify an arbitrarily large amount, for instance `type(uint256).max` and this will be set as his `wd.amount`.

All that is left is for the user to wait for the required delay time and to start withdrawing. Since the `withdraw` function deducts the amount to withdraw from the wd.amount requested, the user can theoretically continue withdrawing directly without any more delays until `type(uint256).max` is completely zeroed out without making any more unlock requests.

```solidity
    function withdraw(address _guy, uint256 _wad) public {
        require(!config.paused(), "DelayedWETH: contract is paused");
        WithdrawalRequest storage wd = withdrawals[msg.sender][_guy];
        require(wd.amount >= _wad, "DelayedWETH: insufficient unlocked withdrawal");
        require(wd.timestamp > 0, "DelayedWETH: withdrawal not unlocked");
        require(wd.timestamp + DELAY_SECONDS <= block.timestamp, "DelayedWETH: withdrawal delay not met");
        wd.amount -= _wad; //@note wad to withdraw is deducted from wad requested in unlock.
        super.withdraw(_wad);
    }
```
## Impact

Bypassing the withdrawal delay mechanism.

## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L65

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L80

## Tool used

Manual Review

## Recommendation
THere are two ways this can be fixed.

1. Validating the user's `wad` input in the `unlock` function against the `guy`'s balance and not allowing requests more that the balance, hence once the user's balance is exhausted, he has to put in a new request.
2. Zeroing out the wd.amount in the `withdraw` function, so the user will be forced to put in a new request for everytime he wants to withdraw.