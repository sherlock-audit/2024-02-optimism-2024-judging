Noisy Pewter Woodpecker

high

# Guardian role cannot be changed once it is set, risking ability to block malicious games

## Summary

The `GUARDIAN` role is the only one who is able to blacklist games or change the game type. This address is set immutably on the `SuperchainConfig` contract. If the address associated with this role cannot be used for any reason, there will not be sufficient time to get a contract upgrade through the governance process to reset `GUARDIAN`, and the malicious games will be allowed to withdraw.

## Vulnerability Detail

The core protection against exploited games on the Portal is performed by either (a) blacklisting individual games or (b) changing the game type so all games on the old type are considered invalid. Both actions are performed exclusively by the `GUARDIAN` role.

This role is accessed by querying the `SuperchainConfig` contract:
```solidity
function guardian() public view returns (address) {
    return superchainConfig.guardian();
}
```

If we look at this contract, we can see that the only place `_setGuardian()` is called is from within the `initialize()` function. This function is only called once, when the contract is deployed, and thereore the `GUARDIAN` role cannot be changed.

In order to update this role, we would need to do a full upgrade of the `SuperchainConfig` contract. This would require to be passed through governance, which could take longer than the 3.5 days allocated for the `GUARDIAN` to act on a malicious game.

Based on the trust assumptions of the system, the `GUARDIAN` role is `RESTRICTED` (not fully trusted) and should not be able to remove ETH from the Portal. However, because of the length of time required to update the role, they would effectively be able to remove ETH from the Portal by refusing to blacklist an exploited game, since they would not be able to be replaced sufficiently quickly enough to stop the action.

## Impact

A malicious guardian or accidental loss of access to the guardian role could result in an extended period with the inability to block malicious games from withdrawing funds from the Portal.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L184-L186

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/SuperchainConfig.sol#L90-L93

## Tool used

Manual Review

## Recommendation

Update `SuperchainConfig.sol` to allow the `proxyAdmin` to update the guardian role on shorter notice.