Curly Turquoise Ram

medium

# `OptimismPortal2` upgrade will delay `OptimismPortal` proven withdrawals.

## Summary

The new `OptimismPortal2` implementation will be deployed on-top of the old `OptimismPortal`.
The different implementations use different variables to store the proven withdrawal. 

Therefore withdrawals that:
1. Are already proven but not finalized
2. Almost finished the 7 day proof window

Will need to reprove their withdrawals and wait twice as intended to execute their withdrawal

## Vulnerability Detail

In `OptimismPortal2` proven withdrawals are stored in:
```solidity
    /// @notice A mapping of withdrawal hashes to proof submitters to `ProvenWithdrawal` data.
    mapping(bytes32 => mapping(address => ProvenWithdrawal)) public provenWithdrawals;
```

In `OptimismPortal` proven withdrawals are stored in:
```solidity
    /// @notice A mapping of withdrawal hashes to `ProvenWithdrawal` data.
    mapping(bytes32 => ProvenWithdrawal) public provenWithdrawals;
```

Since these do not match, any proven withdrawals that were stored in the old `provenWithdrawals` need to be re-proved and stored in the new `provenWithdrawals`

## Impact

Delay of an additional 7 days in withdrawal execution.
Users expect an SLA of 7 days to get their funds from L2->L1. Any additional delay can loss of funds for the user if they need the funds in a timely manner to repay loans, etc..

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L82

## Tool used

Manual Review

## Recommendation

Consider leaving a legacy finalization function that can be used in the first week of the upgrade to finalize withdrawals from the old `provenWithdrawals`.