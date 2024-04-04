Sunny Dijon Mouse

medium

# Frontrunning of Approval in WETH98 may cause double spending against User

medium

## Frontrunning of Approval in WETH98 may cause double spending against User


## Summary

Frontrunning of approval to cause multiple transfer of user balance by malicious spender may occur due to how approval is set in the `WETH98.approve` and also `DELAYEDWETH.approve` function.

## Vulnerability Detail

The `WETH98` contract is expected to be a simple contract which implements a basic ERC20 interface to represent WETH. This is used in the Dispute Games and also inherited directly by the `DELAYEDWETH` contract. The contract implements a simple approve method which directly sets approval amount as specified by a user parameter when the function is called.

```javascript
src: optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol

 function approve(address guy, uint256 wad) external returns (bool) {
        allowance[msg.sender][guy] = wad;
        emit Approval(msg.sender, guy, wad);
        return true;
    }

```

Due to how transactions are executed and mined on ethereum mainnet, by the use of a mempool, it is possible to see other users transactions before they get executed. If a user had previously approved a contract or third party to spend from their balance, and then wants to set a new approval or reset the previous approval, then this transaction can be frontrun by the spender to cause a double spend, effectively stealing tokens from the approver.

- This scenario is well known and is described here

1. https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit#heading=h.b32yfk54vyg9

## Impact

Users of the `WETH98`, `DELAYEDWETH` and any other contract which integrates `WETH98` would be susceptible to doublespending frontrunning attack by a malicious spender.

## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/weth/WETH98.sol#L59-L61

## Proof of Concept

Consider the following scenario

1. User Alice has previously approved Bob to spend 100 tokens on her balance.
2. Alice now wants to change Bob's allowance either to 0 or to a new amount.
3. Bob notices this and immediately frontruns this, calling `WETH98.transferFrom` from Alice's account before the approval transaction gets executed. This increases Bob's balance and decreases Alice's balance. Note that Bob may have already set up a bot to spy transactions from Alice and execute this attack when `approve` is called to change his allowance.
4. After this transfer, approval is set to the new number, if the new allowance is greater than 0, this allows Bob to spend from Alice's balance a second time.
5. Finally, Bob can withdraw his balance from the contract at any time.

- Generally, this is a well-documented vulnerability and included below is link to a similar case;

1. https://solodit.xyz/issues/m-01-lockeerc20-is-vulnerable-to-frontrun-attack-code4rena-streaming-protocol-streaming-protocol-contest-git

## Tool used

Manual Review

## Recommendation

1. A good mitigation is to implement the recommended standard which is accept the previous approval value as a parameter, then confirm if this value has been altered before setting allowance to new value

```diff

function approve(address guy, uint256 _previousWad, uint256 wad) external returns (bool) {
+        require(allowance[msg.sender][guy] == _previousWad, "Allowance value altered");
        allowance[msg.sender][guy] = wad;
        emit Approval(msg.sender, guy, wad);
        return true;
    }

```
