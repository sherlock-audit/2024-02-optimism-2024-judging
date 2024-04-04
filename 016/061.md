Old Shadow Wombat

medium

# Double deduction from the user's Eth balance in a single withdrawal operation

## Summary
The `withdraw()` function deducts a specified amount (`wad`) from the `msg.sender`'s balance and then attempts to transfer this amount to `msg.sender` using the `transfer()` function.
When the `withdraw()` function internally calls `transfer()`, which then calls `transferFrom()`, if `src == msg.sender`, the `transferFrom()` redundantly deducts the amount from `msg.sender`'s balance again, even though the `withdraw()` function already did so.

## Vulnerability Detail
##### Exploit scenario:
- A user has deposited `ETH` into the smart contract, and  has `5 ETH` in their `balanceOf` mapping within the contract.
- The user decides to withdraw `1 ETH` using the `withdraw()` function.
- Within the `withdraw()` function, the contract deducts `1 ETH` from the user's balance in `balanceOf[msg.sender]` immediately. Now, the contract's record shows the user has `4 ETH` left.
- The contract then attempts to send `1 ETH` to the user by invoking `payable(msg.sender).transfer(wad)`.
- The `transfer()` function calls `transferFrom()`, which again checks the balance (already reduced to `4 ETH`).
- Since  `src == msg.sender`  within `transferFrom()`, the function deducts another `1 ETH` from the user's balance. 
- Now, the user's balance reflects `3 ETH` instead of `4 ETH`.

## Impact
- Double Deduction: 
Users lose twice the amount of tokens they intended to withdraw from their balances due to the redundant deduction.

## Code Snippet
- https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L46-L51
- https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L66-L85

## Tool used
Manual Review

## Recommendation
Remove the balance deduction from the `withdraw()` function, leaving it responsible only for transferring `ETH` to the user. This means that the `withdraw()` function will only trigger the transfer of `ETH` to the user without reducing their balance within the contract. The `transferFrom()` function will handle the deduction of the user's balance.

```solidity
    function withdraw(uint256 wad) public virtual {
        require(balanceOf[msg.sender] >= wad);
        payable(msg.sender).transfer(wad);
        emit Withdrawal(msg.sender, wad);
    }
```