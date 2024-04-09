Quick Iron Salamander

medium

# `call()` should be used instead of `transfer()` on an address payable

## Summary
The [withdraw()](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L49C9-L49C43) function of contract [WETH98](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L26) is used to withdraw native asset funds, this is done via the `transfer()` function on address payable, which is discouraged. This affects also the [DelayedWETH](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L22) contract as the same function is reused in its own [withdraw()](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L81) functions.

The same issue is observed in [recover(uint256)](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L85) function of [DelayedWETH](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L22) contract.

## Vulnerability Detail
The `transfer()` and `send()` functions provide a static 2300 gas, recommended for reentrancy protection. However, gas cost changes from hard forks, like EIP 1884's SLOAD adjustment, can break contracts based on fixed gas estimates. Also the [withdraw()](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L49C9-L49C43) and [recover(uint256)](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L85) functions adhere to the checks-effects-interactions(CEI) pattern, making them impervious to reentrancy attacks.

## Impact

2300 is the set up gas for `transfer()` function, so its usage is discouraged because it will inevitably make the transaction fail when:

1. The claimer smart contract does not implement a payable function.
2. The claimer smart contract does implement a payable fallback which uses more than 2300 gas unit.
3. The claimer smart contract implements a payable fallback function that needs less than 2300 gas units but is called through proxy, raising the call's gas usage above 2300.
4. Additionally, using higher than 2300 gas might be mandatory for some multisig wallets.

Medium, as it arises when the recipient cannot accept the funds due to any of the aforementioned reasons, leading to transaction failures.

## Code Snippet
[withdraw()](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L49C9-L49C43) function in contract [WETH98](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L26):
```Solidity
    function withdraw(uint256 wad) public virtual {
        require(balanceOf[msg.sender] >= wad);
        balanceOf[msg.sender] -= wad;
        payable(msg.sender).transfer(wad);
        emit Withdrawal(msg.sender, wad);
    }
```
[recover(uint256)](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L85) function in [DelayedWETH](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L22):
```Solidity
    function recover(uint256 _wad) external {
        require(msg.sender == owner(), "DelayedWETH: not owner");
        uint256 amount = _wad < address(this).balance ? _wad : address(this).balance;
        payable(msg.sender).transfer(amount);
    }
```
## Tool used

Manual Review

## Recommendation
Use `call()` instead of `transfer()`. The CEI pattern is already respected, so reentrancy is not an issue.

More info on; https://swcregistry.io/docs/SWC-134
