Tart Ultraviolet Kitten

high

# Theft of initial bonds from proposers who are using smart wallets

## Summary

Proposal of output roots through the DisputeGameFactory from Smart Wallets is vulnerable to frontrunning attacks which will steal the initial bond of the proposer.

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

The  sender passes a `msg.value` which equals the required bond amount, and the registered `claimant` is `tx.origin`.  At the end of the game, if the claim is honest, the funds will be returned to the `claimant`.

Smart Wallets are extremely popular ways of holding funds and are used by all types of entities for additional security properties and/or flexibility. A typical smart wallet will receive some `execute()` call with parameters, verify it's authenticity via signature / multiple signatures, and perform the requested external call. That is how the highly popular Gnosis Safe operates among many others. Smart Wallets are agnostic to whoever actually  called the `execute()` function, as long as the data is authenticated.

These properties as well as the use of `tx.origin` in the FaultDisputeGame make it easy to steal the bonds of honest proposals:
- Scan the mempool for calls to [Gnosis](https://github.com/safe-global/safe-smart-account/blob/1cd7568769128717c1a6862d22fe34873d7c79c8/contracts/Safe.sol#L104) `execTransaction()`  or any other variants.
- Copy the TX content and call it from the attacker's EOA.
- The Smart Wallet will accept the call and send the `msg.value` to the DisputeGameFactory.
- The `claimant` will now be the attacker.
- Upon resolution of the root claim, the attacker will receive the initial bond.

## Impact

Theft of funds from an honest victim who did not interact with the system in any wrong way.

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

The Factory needs to pass down the real msg.sender to the FaultDisputeGame. 
