Old Shadow Wombat

medium

# Lack of access control in both the `unlock()` and `withdraw()` functions exposes the contract to potential exploitation.

## Summary
Overall, the lack of access control in both the `unlock()` and `withdraw()` functions exposes the contract to potential exploitation. 

## Vulnerability Detail
1. `unlock()` Function:
The unlock function allows any user to unlock funds for withdrawal by specifying the recipient address (_guy) and the amount (_wad) they want to unlock.
There is no access control in place for this function, meaning any user can call it and unlock funds for any recipient address.

2. `withdraw()` Function:
The withdraw function  intended to be called by any user to withdraw funds after they've been unlocked and the withdrawal delay has passed.
However, there is no access control in place for this function, meaning any user can call it and attempt to withdraw funds.

##### Exploit scenario:
- Malicious `User A` initiates the `unlock()` function with one of their other Ethereum addresses (controlled by themselves) as the `recipient` and a significant amount of funds to be unlocked.
- The contract accepts the unlocking request without verifying the recipient's identity, allowing `User A` to unlock the funds to their other address.
- Once the delay period expires, `User A` calls the `withdraw()` function, specifying their other address as the recipient.
- The contract processes the withdrawal request without authentication, transferring the funds to the specified address controlled by `User A`.

## Impact
Malicious `User A` successfully withdraws funds from the contract to their own address, exploiting the lack of access control.
The contract's balance is depleted, potentially impacting other users' ability to withdraw funds as intended.

## Code Snippet
- https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L57-L66
- https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/DelayedWETH.sol#L74-L82

## Tool used
Manual Review

## Recommendation
The contract must implement robust `access control` mechanisms to verify the legitimacy of withdrawal requests.
Access controls could include requiring authentication from the `recipient` address or implementing role-based access controls to restrict certain functions to authorized users.
